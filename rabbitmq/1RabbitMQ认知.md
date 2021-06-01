# 一、RabbitMQ认知

## RabbitMQ四种架构模式

### 1. 主备模式

#### 1.1 介绍

> 也称为 Warren (兔子窝) 模式。实现 rabbitMQ 的高可用集群，一般在并发和数据量不高的情况下，这种模式非常的好用且简单。
>
> 也就是一个主/备方案，主节点提供读写，备用节点不提供读写。如果主节点挂了，就切换到备用节点，原来的备用节点升级为主节点提供读写服务，当原来的主节点恢复运行后，原来的主节点就变成备用节点，和 activeMQ 利用 zookeeper 做主/备一样，也可以一主多备。
>
> activeMQ是通过zk实现的主备切换，而rabbitMQ是通过 haProxy 

#### 1.2 架构原理

**架构图：**

![1588324116049](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588324116049.png)

**注：**这里的consumer, 不能只是理解为消费者，应该理解成是一个需求方，通过HaProxy默认路由到 主节点（Master），默认主节点提供服务

#### 1.3 HaProxy配置

```shell
listen rabbitmq_cluster # 主备模式的集群名字

bind 0.0.0.0:567  

mode tcp  # 配置 tcp 模式

balance roundrobin  # 负载均衡策略：roundrobin  轮询

server 你的76机器 hostname  192.168.11.76:5672 check inter 5000 rise 2 fall 2 	# 主节点

server 你的77机器 hostname  192.168.11.77:5672 backup check inter 5000 rise 2 fall 2  # 备用节点
```

 **注意：**上面的 rabbitMQ 集群节点配置 # inter 每隔 5 秒对 mq 集群做健康检查， 2 次正确证明服务可用，2 次失败证明服务器不可用，并且配置主备机制

### 2. 远程模式

#### 2.1 简介

远程模式可以实现双活的一种模式，简称 shovel 模式，所谓的 shovel 就是把消息进行不同数据中心的复制工作，可以跨地域的让两个 MQ 集群互联，远距离通信和复制。

Shovel 就是我们可以把消息进行数据中心的复制工作，我们可以跨地域的**让两个 MQ 集群互联。**

可靠性有待提高，且配置麻烦 。

#### 2.2 架构原理

**架构图：**

![1588324900158](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588324900158.png)

如图所示，有两个异地的 MQ 集群（可以是更多的集群），当用户在地区 1 这里下单了，系统发消息到 1 区的 MQ 服务器，发现 MQ 服务已超过设定的阈值，负载过高，这条消息就会被转到 地区 2 的 MQ 服务器上，由 2 区的去执行后面的业务逻辑，相当于分摊我们的服务压力。

在使用了 shovel 插件后，模型变成了近端同步确认，远端异步确认的方式，大大提高了订单确认速度，并且还能保证可靠性。

**Shovel 集群拓扑图：**

![1588325009359](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588325009359.png)

#### 2.3 配置

​	略。一般不使用

### 3. 镜像模式

#### 3.1 简介

​	非常经典的 mirror 镜像模式，保证 100% 数据不丢失。在实际工作中也是用得最多的，并且实现非常的简单，一般互联网大厂都会构建这种镜像集群模式。

​	镜像模式其实就是数据的备份，类似与esSearch中replication的概念，还有MongoDB中  复制集的概念

​	mirror 镜像队列，目的是为了保证 rabbitMQ 数据的高可靠性解决方案，主要就是实现数据的同步，一般来讲是 2 - 3 个节点实现数据同步。对于 100% 数据可靠性解决方案，一般是采用 3 个节点。

#### 3.2 架构原理

**架构图：*

![1588325473930](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588325473930.png)

先从最下面看，是三个节点的RabbitMQ服务器，这三个节点数据保存都是一致的

用 KeepAlived 做了 HA-Proxy 的高可用，然后有 3 个节点的 MQ 服务，消息发送到主节点上，主节点通过 mirror 队列把数据同步到其他的 MQ 节点，这样来实现其高可靠。

官方建议使用3个节点，节点太多需要数据同步，会降低网络吞吐量

缺陷：不能支持横向扩展，想要横向扩展可以采用多活模式

### 4. 多活 模式

#### 4.1 简介

 也是实现**异地数据复制**的主流模式，因为 shovel 模式配置比较复杂，所以一般来说，实现异地集群的都是采用这种双活 或者 多活模型来实现的。这种模式**需要依赖 rabbitMQ 的 federation 插件**，可以实现持续的，可靠的 AMQP 数据通信，多活模式在实际配置与应用非常的简单。

rabbitMQ 部署架构采用双中心模式(多中心)，那么在两套(或多套)数据中心各部署一套 rabbitMQ 集群，各中心的rabbitMQ 服务除了需要为业务提供正常的消息服务外，中心之间还需要实现部分队列消息共享。

#### 4.2 架构原理

![1588326140265](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588326140265.png)

> > > > > > > > > > > > > > > > > > > > > > > > federation 插件是一个不需要构建 cluster ，而在 brokers 之间传输消息的高性能插件，federation 插件可以在 brokers 或者 cluster 之间传输消息，连接的双方可以使用不同的 users 和 virtual hosts，双方也可以使用不同版本的 rabbitMQ 和 erlang。federation 插件使用 AMQP 协议通信，可以接受不连续的传输。federation 不是建立在集群上的，而是建立在单个节点上的，如图上黄色的 rabbit node 3 可以与绿色的 node1、node2、node3 中的任意一个利用 federation 插件进行数据同步。

![1588327473141](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588327473141.png)

如上图所示，federation exchanges 可以看成 downstream 从 upstream 主动拉取消息，但是并不是拉取所有消息，必须是在 downstream 上已经明确定义 Bingdings 关系的 exchange，也就是有实际的物理 queue 来接收消息，才会从 upstream 拉取消息到 downstream 。

它使用 AMQP 协议实现代理间通信，downstream 会将绑定关系组合在一起，绑定/解绑命令将发送到 upstream 交换机。因此，federation exchange 只接收具有订阅的消息。