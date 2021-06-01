# Kafka学习

## Kafka概述

Kafka是最初由Linkedin公司开发，是一个分布式、支持分区的（partition）、多副本的（replica），基于zookeeper协调的分布式消息系统，它的最大的特性就是可以实时的处理大量数据以满足各种需求场景：比如基于hadoop的批处理系统、低延迟的实时系统、storm/Spark流式处理引擎，web/nginx日志、访问日志，消息服务等等，用scala语言编写，Linkedin于2010年贡献给了Apache基金会并成为顶级开源 项目。

## KafKa的应用场景

* 异步化、服务间的解耦、削峰填谷（MQ的普遍功能）
* 海量日志的收集
* Kafka之数据同步的应用
* Kafka之实时计算分析

## Kafka集群

 Kafka集群架构图

![image-20210405095353675](http://typicture.loopcode.online/image/image-20210405095353675.png)

- Kafka broker：缓存代理，Kafka集群中的一台或多台服务器统称broker
  1. broker没有副本机制，一旦broker宕机，该broker的消息将都不可用。
  2. broker不保存订阅者的状态，由订阅者自己保存。
  3. 无状态导致消息的删除成为难题（可能删除的消息正在被订阅），Kafka采用基于时间的SLA（服务保证），消息保存一定时间（通常7天）后会删除。
  4. 消费订阅者可以rewind back到任意位置重新进行消费，当订阅者故障时，可以选择最小的offset(id)进行重新读取消费消息

- Producer：消息生产者

- Consumer：消息消费者，一般来说Kafka都会采取pull（拉）的方式，去把消息拉取到本地的Consumer节点上，然后去做一个实际的消息处理。

- Zookeeper：对Kafka集群进行维护、管理。

## Kafka基本概念——topic&partition

![image-20210405113619492](http://typicture.loopcode.online/image/image-20210405113619492.png)

* Topic：一组消息抽象归纳为一个topic，是对消息的一个**逻辑分类**；

  Topic相当于传统消息系统MQ中的一个队列queue，producer端发送的message必须指定是发送到哪个topic，但是不需要指定topic下的哪个partition，因为kafka会把收到的message进行load balance，均匀的分布在这个topic下的不同的partition上（ hash(message) % [broker数量]  ）

- Partition	分区是kafka消息队列组织的最小单位；物理上存储上

  一个topic 可以有多个partition，每个partion是一个有序的队列，一个partition 可以有多个副本；

  partion中每条消息都会被分配一个 有序的Id(offset)，因此往partition中写入和读取消息事都是顺序的。

  分区是可以扩展并动态改变的。

- topic和partiton是一对多的关系。

## Kafka基本概念——Replica（副本）

![image-20210405114921989](http://typicture.loopcode.online/image/image-20210405114921989.png)

副本是包括Leader副本和Follower（Slave） 副本的，为了方便起见，后面Leader副本直接称为Leader，而副本特指的是Slave副本

上图中，每一个黄色的Broker代表有三个不同的kafka，这里有三个Broken表明是由三个Kafka构成的集群，我们上面谈到过，Patition（分片）才是Broker的最小存储单位，所有的消息都会落地到每一个patition中。在上图中，每一个P1、P2、P3…就代表每一个分片，而这些在同一个Borker中的不同的分片，相当于构成了partition的集群，比如Broken1中的P1、P3、P4、多个分片就构成了Patition的集群，当消息打在了Broker1上时，Broker1会通过负载均衡算法决定最终落到哪一个分片上。

如果一个Broker有多个分片，但是每个分片只存在一份，那么如果这台Broker宕机了，就会造成不可用，或者消息的丢失，这时候就有了Patition副本的概念，我们可以为每一个Patition创建一个或多个副本，而这个副本分摊在不同的Broker上，不就实现了Kafka的高可用吗。如上图中的Broker1的绿色的P1和Broker2中蓝色的P1，就存在副本的关系。同样Broker1中的P3和P4也存在着副本。由此当Broker1挂掉时， Broker2和Broker3中的副本就可以工作，从而保证数据的完整性，仍然可以对外提供服务的能力。

在上图中，我们假如绿色的Patition作leader，紫色的作为Replica。在Kafka中，leader负责和客户端的通信，包括消息的读写；而Replica只负责和Leader进行通信，实时的拉取Leader中的消息，进行数据的同步。

学习过ES和HBase的小伙伴们是不是很熟悉这种设计模式，其实在分布式存储中，很多都是采取了分片 + 副本的形式来实现当集群中的某台机器故障时，可以进行故障转移，不影响整个集群的对外服务，比如ElasticSearch中也有分片和副本的概念，我们在学习的过程中可以对比学习，相同的可以总结归纳，慢慢就可以融会贯通了。

## Kafka基本概念——In Sync Replicas（主从数据的同步）

上面我们谈到了Kafka是通过Replica副本的机制来保证其高可用的，既然引入了副本的概念，那么就一定会带来数据同步的问题，下面我们就来讨论，Patition的不同副本和Leader是如何数据同步的，以及如果同步失败发生数据不一致问题是如何解决的，如果主节点宕机了，那么副本会成为Leader吗，其中的机制是如何的，这些问题我们来一一分析。

我们现来看下图，其中上面三个黑色并列的方框，代表的是三个不同的Broker；我们可以看到每个Broker中的有一个P1，其中第一个绿色的P1作为Leader存在。其他两个P1 S1 和 P1 S2作为Slave节点存在。我们来分析他们之间是如何进行数据同步的。

![image-20210405121112353](http://typicture.loopcode.online/image/image-20210405121112353.png)

在谈ISR之前先说什么AR的概念，分区中的所有副本，包括Leader统称为AR（Assigned Repllicas）。

当副本P1 S1 、 P1 S2向Leader拉取数据时，也就是Leader和副本之间的同步，而拉取数据这个过程是需要一定的时间的，如果副本向Leader拉取数据的时间在Kafka的容忍时间之内（这个容忍时间是可以配置的），那么Leader就会把这个副本加入到上图的ISR集合中，而如果某个副本拉取数据的时间超过了主从同步的容忍时间，那么Leader就会把该副本丢到OSR集合中，后期如果该副本正常了，我们会说follower副本“追上”了Leader副本，Leader又会重新把它放到ISR集合中去，由此可以看出，ISR和OSR是个动态变化的过程，且AR = ISR + OSR，正常的话OSR应该是空的。

为什么要弄ISR 和 OSR这两个集合呢，其中之一和Kafka的选举策略有关，当Leader节点挂掉时，只有处于ISR集合中的副本才有选举成为Leader的资格。

我们再谈几个ISR的相关概念：

* HW：High Watermark,高水位线，它表示了一个特定消息的偏移量（offset），消费者只能拉取低于高水位线的消息。如下图HW是6，也就是只能获取到0~5之间的消息。
* LEO：Log End Offset，表示了当前日志文件中下一条待写入消息的offset，如下图offset为9的位置即为当前日志文件LEO
* 分区ISR集合中的每个副本都会维护自身的LEO，一般而言**ISR集合**中最小的LEO即为分区的HW，我说一般而言，就说明我们可以对这种策略进行修改，比如可以设置有一般的Follower同步完成就可以对外提供服务，对消费这而言只能消费HW之前的消息。
* ISR集合与HW和LEO直接存在着密不可分的关系

 如下图，代表一个日志文件，这个日志文件中有9条消息，第一消息的offset（LogStartOffset）为0，最后的一条消息offset为8，offset为9的消息用虚线框表示，代表下的一个待写入的消息。日志文件的HW为6.表示消费者只能拉取到offset0至5之间的消息，而offset为6之后的消息对消费者而言是不可见的。

![image-20210405192650719](http://typicture.loopcode.online/image/image-20210405192650719.png)

从上图发现我的HW在6而LEO却已经到9了这是为什么呢，

 如下图，假设某个分区的ISR集合中有三个副本，即一个leader和两个follower副本，此时分区的LEO和HW都为3。消息3和消息4从生产者发出之后会被先存入leader。

<img src="http://typicture.loopcode.online/image/image-20210405194827809.png" alt="image-20210405194827809" style="zoom:80%;" />

 在消息写入leader副本之后，follower副本会发送拉取请求来拉取消息3和消息4以进行消息同步。

<img src="http://typicture.loopcode.online/image/image-20210405194945984.png" alt="image-20210405194945984" style="zoom:80%;" />

 在同步过程中，不同的follower副本的同步效率也不尽相同。如下图，在某一时刻follower1完全跟上了leader副本而follower2只同步了消息3，如此leader副本的LEO为5，follower1的LEO为5，Follower2的LEO为4。那么当前分区的HW最小值4，此时消费者可以消费到offset为0-3之间的消息，可以看出HW存在的意义就是在一定程度上保证数据的一致性，Consumer只能消费数据已经同步的部分，HW就是所有分片的公共部分（同步完成的部分）。

HW其实是和我们的offset返回策略有关系的，这里我们默认的是所有的follower都同步成功才返回offset，返回的offset就是最新的HW，我们也可以更改这种返回策略，比如我们可以设置为半数以上的副本同步成功就返回offset，途中我们有三个replicate，也就是如果两个分片同步成功了（包括leader在内），即follower1同步成功了，那么就返回offset，这样的话，下图的HW就会是4.

![image-20210405195417730](http://typicture.loopcode.online/image/image-20210405195417730.png)

当所有的副本都成功写入了消息3和消息4，整个分区的HW和LEO为5，因此消费者就可以消费offset为4的消息了。

由此可见,kafka的复制机制不是完全的同步复制，也不是单纯的异步复制，事实上，同步复制要求所有能工作的Follower副本都复制完，这条消息才会被确认为成功提交，这种复制方式影响了性能。而在异步复制的情况下， follower副本异步地从leader副本中复制数据，数据只要被leader副本写入就被认为已经成功提交。在这种情况下，如果follower副本都没有复制完而落后于leader副本，如果突然leader副本宕机，则会造成数据丢失。Kafka使用这种ISR的方式有效的权衡了数据可靠性与性能之间的关系。

