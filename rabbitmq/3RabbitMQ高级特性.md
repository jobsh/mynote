# RabbitMQ高级特性

## 1. 消息如何保证100%投递成功

### 1.1 什么是生产端的可靠性投递

1. 保障消息的成功发出
2. 保障MQ节点的成功接收
3. 发送端收到MQ节点（broker）确认应答
4. 完善的消息补偿机制

### 1.2 BAT/TMD互联网大厂解决方案

1. **消息落库，对消息状态进行打标** 

   ![1588555203994](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588555203994.png)

   需要经过至少这七步，才能保证消息的正常

   图中信息说明：

   * BIZ DB：自己的业务

   * MSG DB：消息的存根（类似日志的记录），相当于对BIZ DB 的状态进行打标，如果Producer收到了broker的ACK应答，则会去修改MSG DB记录

   * 要保证STEP 1 和 STEP 2操作是原子性的，比如在BIZ DB插入一条数据时，同时在 MSG DB插入一条数据，是对BIZ DB数据的说明（一种打标）

   * BIZ Service：消息投递服务

   * STEP 3：消息投递给MQ Broker

   * STEP 4：MQ的confirm机制，返回ACK状态

   * STEP 5： 如果成功,去数据库查询该消息,并将消息状态更新为1

   * STEP 6：如果出现意外情况，消费者未接收到或者 Listener 接收确认时发生网络闪断，导致生产端的Listener就永远收不到这条消息的confirm应答了，也就是说这条消息的状态就一直为0（初始状态）了，这时候就需要用到我们的分布式定时任务来从 MSG 数据库抓取那些超时了还未被消费的消息，重新发送一遍

     > 此时我们需要设置一个规则，比如说消息在入库时候设置一个临界值timeout，5分钟之后如果还是0的状态那就需要把消息抽取出来。这里我们使用的是分布式定时任务，去定时抓取DB中距离消息创建时间超过5分钟的且状态为0的消息。

   * Producer Component：监听broker的组件

   * 分布式定时任务：可以用ES Job，不断去监听MSG DB,如果MSG DB中记录状态没有改变，则重新调用service向MQ中重新投递，直到MSG DB中记录状态改变为止，或者超过一定时间置为失败

     #### 思考:该方案在高并发的场景下是否合适

     对于第一种方案，我们需要做两次数据库的持久化操作，在高并发场景下显然数据库存在着性能瓶颈.

     其实在我们的核心链路中只需要对业务进行入库就可以了，消息就没必要先入库了，我们可以做消息的延迟投递，做二次确认，回调检查。下面然我们看方案二

2. **消息延迟投递,两次确认,回调检查(大规模海量数据方案)**

   当然这种方案不一定能保障百分百投递成功，但是基本上可以保障大概99.9%的消息是OK的，有些特别极端的情况只能是人工去做补偿了，或者使用定时任务.

   主要就是为了减少DB操作

   ##### 方案流程图   ![img](https:////upload-images.jianshu.io/upload_images/16782311-d758785759374c2a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1020/format/webp)  image  

   - Upstream Service
      上游服务,即生产端
   - Downstream service
      下游服务,即消费端
   - Callback service
      回调服务

   ##### 方案实现流程

   - step1 一定要先将业务消息入库,然后Pro再发出消息,顺序不能错!
   - step2 在发送消息之后,紧接着Pro再发送一条消息(Second Send Delay Check),即延迟消息投递检查,这里需要设置一个延迟时间,比如5分钟之后进行投递.
   - step3 Con监听指定的队列,处理收到的消息.
   - step4 处理完成之后,发送一个confirm消息,也就是回送响应,但是这不是普通的ACK,而是重新生成一条消息,投递到MQ,表示处理成功.
   - Callback service是一个单独的服务,它扮演MSG DB角色,它通过MQ监听下游服务发送的confirm消息,如果监听到confirm消息,那么就对其持久化到MSG DB.
   - step6 5分钟之后延迟消息发送到MQ,然后Callback service还是去监听延迟消息所对应的队列，收到Check消息后去检查DB中是否存在消息，如果存在，则不需要做任何处理，如果不存在或者消费失败了，那么Callback service就需要主动发起RPC通信给上游服务，告诉它延迟检查的这条消息我没有找到，你需要重新发送，生产端收到信息后就会重新查询BIZ DB然后将消息发送出去.

   ##### 设计目的

   少做一次DB的存储,在高并发场景下,最关心的不是消息百分百投递成功,而是一定要保证性能，保证能抗得住这么大的并发量。所以能节省数据库的操作就尽量节省，异步地进行补偿.

   其实在主流程里面是没有Callback service的，它属于一个补偿的服务，整个核心链路就是生产端入库业务消息，发送消息到MQ，消费端监听队列，消费消息。其他的步骤都是一个补偿机制。

   ##### 小结

   这两种方案都是可行的，需要根据实际业务来进行选择,方案二也是互联网大厂更为经典和主流的解决方案.但是若对性能要求不是那么高,方案一要更简单.

## 2. 幂等性

### 2.1 什么是幂等性

一个幂等操作的特点是其任意多次执行所产生的影响均与一次执行的影响相同。幂等函数或幂等方法是指可以使用相同参数重复执行，并能获得相同结果的函数/方法。这些函数/方法不会影响系统状态，因此不用担心重复执行会对系统造成改变。

我们可以借鉴数据库的乐观锁机制，比如我们执行一条更新库存的SQL语句

```mysql
UPDATE T_REPS SET COUNT = COUNT-1， VERSION = VERSION+1 WHERE VERSION= 1
```

### 2.2 在海量订单产生的业务高峰期,如何避免消息的重复消费问题?

消费端实现幂等性,就意味着,我们的消息永远不会多次消费,即使我们收到了多条一样的消息

### 2.3 业界主流的幂等性操作

业务唯一ID(或)指纹码机制（订单号+流水号+时间戳等）,**利用数据库主键去重**（最通用的方式）

```mysql
SELECT COUNT(1) FROM T_ORDER WHERE D = 唯一ID(或)指纹码
```

先查出来，如果不是0，那就是有，就不能insert了

把大概率事件用一个极小的性能损耗去完成，小概率事件用一个兜底的方案去搞定。

## 3. Confirm消息确认机制

### 3.1 理解 Confirm消息确认机制

* 消息的确认,是指生产者投递消息后,如果 Brokerl收到消息,则会给我们生产者一个应答（一个异步的过程）

* 生产者进行接收应答,用来确定这条消息是否正常的发送到 Broker,这种方式也是消息的可靠性投递的核心保障

![1588558224213](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588558224213.png)

### 3.2 如何实现 Confirm确认消息?

* #### 第一步:发送端代码中在 channel上开启确认模式: channel.confirmSelect()

* #### 第二步:在 channel上添加监听: add Confirm Listener,监听成功和失败的返回结果,根据具体的结果对消息进行重新发送、或记录日志等后续处理!

* #### 代码如下

  ```java
  package com.bfxy.rabbitmq.api.confirmlistener;
  
  import java.io.IOException;
  
  import com.rabbitmq.client.Channel;
  import com.rabbitmq.client.ConfirmListener;
  import com.rabbitmq.client.Connection;
  import com.rabbitmq.client.ConnectionFactory;
  
  public class Sender4ConfirmListener {
  
  	
  	public static void main(String[] args) throws Exception {
  		
  		//1 创建ConnectionFactory
  		ConnectionFactory connectionFactory = new ConnectionFactory();
  		connectionFactory.setHost("192.168.248.151");
  		connectionFactory.setPort(5672);
  		connectionFactory.setVirtualHost("/");
  		
  		//2 创建Connection
  		Connection connection = connectionFactory.newConnection();
  		//3 创建Channel
  		Channel channel = connection.createChannel();  
  		
  		//4 声明
  		String exchangeName = "test_confirmlistener_exchange";
  		String routingKey1 = "confirm.save";
  		
      	//5 发送
  		String msg = "Hello World RabbitMQ 4 Confirm Listener Message ...";
  		
  		channel.confirmSelect();
          channel.addConfirmListener(new ConfirmListener() {
  			@Override
  			public void handleNack(long deliveryTag, boolean multiple) throws IOException {
  				System.err.println("------- error ---------");
  			}
  			@Override
  			public void handleAck(long deliveryTag, boolean multiple) throws IOException {
  				System.err.println("------- ok ---------");
  			}
  		});
          
  		channel.basicPublish(exchangeName, routingKey1 , null , msg.getBytes()); 
  
   
  	}
  	
  }
  ```

## 4. return消息机制

* Return Listener用于处理一些不可路由的消息
* 我们的消息生产者,通过指定一个 Exchange 和 Routing key，把消息送达到某一个队列中去,然后我们的消费者监听队列,进行消费处理操作
* 但是在某些情况下,如果我们在发送消息的时候,当前的 exchange不存在或者指定的路由key路由不到,这个时候如果我们需要监听这种不可达的消息,就要使用 Return Listener!

### 4.1 实现

* 在基础APl中有一个关键的配置项:

  **Mandatory**：**Mandatory**设置为**true**，监听器会接收到路由不可达的消息后，才会进行后续的处理,**如果Mandatory为 false,那么 broker端会自动删除该消息!**

![1588558713514](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1588558713514.png)

## 5. 消费端限流
> * 假设一个场景,首先,我们 RabbitMQ服务器有上万条未处理的消息,我们随便打开一个消费者客户端,会出现下面情况:
>   巨量的消息瞬间全部推送过来,但是我们单个客户端无法同时处理这么多数据。此时很有可能导致服务器崩溃，严重的可能导致线上的故障。
>
> * 还有一些其他的场景，比如说单个Producer一分钟产生了几百条数据,但是单个Consumer一分钟可能只能处理60条,这个时候Pro-Con肯定是不平衡的。通常Pro是没办法做限制的。所以Con肯定需要做一些限流措施，否则如果超出最大负载，可能导致Con性能下降，服务器卡顿甚至崩溃等一系列严重后果。
>
> * RabbitMQ提供了一种qos(服务质量保证)功能,即在非自动确认消息的前提下,如果一定数目的消息(通过基于 consumer或者 channel设置Qos的值)未被确认前,不进行消费新的消息。

代码设置：

```java
void Basicqos(uint prefetchsize, ushort prefetchcount, bool global);
```

参数解释：

* prefetchSize: 单条消息的大小限制，Con通常设置为0，表示不做限制

* prefetchCount: 一次最多能处理多少条消息，会告诉 Rabbitmq不要同时给一个消费者推送多
  于N个消息,即一旦有N个消息还没有ACK,则该 consumer将 block掉,直到有消息ACK。

* global: 是否将上面设置true应用于channel级别还是取false代表Con级别，简单点说,就是上面限制是 channels级别的还是 consumers级别

>prefetchSize和global这两项,RabbitMQ没有实现,暂且不研究
>prefetchCount在 `autoAck=false` 的情况下生效,即在自动应答的情况下该值无效



## 6.消费端ACK & 重回队列机制

### 6.1 手工ACK

`void basicAck(Integer deliveryTag，boolean multiple)`
调用这个方法就会主动回送给Broker一个应答，表示这条消息我处理完了，你可以给我下一条了。参数multiple表示是否批量签收，由于我们是一次处理一条消息，所以设置为false

当我们设置`autoACK=false` 时,就可以使用手工ACK方式了,其实手工方式包括了手工ACK与NACK

* 当我们手工 ACK 时,会发送给Broker一个应答,代表消息处理成功,Broker就可回送响应给producer。

* NACK 则表示消息处理失败,如果设置了重回队列,Broker端就会将没有成功处理的消息重新发送。消费端进行消费的时候,如果由于业务异常我们可以进行日志的记录，然后进行补偿！
* 说明：
  * 如果消费失败了就回复一个NACK，后面做人工补偿，千万不要做重回队列，后面说为什么。
    * 即使回复的是NACK，offset也会偏移
  * 如果由于服务器宕机等严重问题,那我们就需要手工进行ACK保障消费端消费成功!
  * 在实际的生产中我们都不会选择 自动ACK，都会去选择手工ACK。

### 6.2 重回队列

- 重回队列是为了对没有处理成功的消息,将消息重新投递给Broker。（有可能会造成消息的不断重新投递，会造成consumer无限地消费这条处理不了的消息，这是应该避免的）
- 重回队列,会把消费失败的消息重新添加到队列的尾端,供Con继续消费
- 一般在实际应用中,都会关闭重回队列,即设置为false

## 7. TTL机制

### 7.1 什么是TTL

- TTL(Time To Live),即生存时间
- RabbitMQ支持消息的过期时间，在消息发送时可以进行指定
- RabbitMQ支持为每个队列设置消息的超时时间，从消息入队列开始计算，只要超过了队列的超时时间配置，那么消息会被自动清除

## 8. 死信队列机制

### 8.1 什么是死信队列

DLX - 死信队列(dead-letter-exchange)
 利用DLX,当消息在一个队列中变成死信 (dead message) 之后,它能被重新publish到另一个Exchange中,这个Exchange就是DLX.

### 8.2  死信队列的产生场景

- 消息被拒绝(basic.reject / basic.nack),并且requeue = false
- 消息因TTL过期
- 队列达到最大长度

### 8.3 死信的处理过程

- DLX亦为一个普通的Exchange,它能在任何队列上被指定,实际上就是设置某个队列的属性
- 当某队列中有死信时,RabbitMQ会自动地将该消息重新发布到设置的Exchange,进而被路由到另一个队列
- 可以监听这个队列中的消息做相应的处理.该特性可以弥补RabbitMQ 3.0以前支持的`immediate`参数的功能

### 8.4 死信队列的配置

- 设置死信队列的exchange和queue,然后进行绑定
  - Exchange:dlx.exchange
  - Queue: dlx.queue
  - RoutingKey:#
- 正常声明交换机、队列、绑定，只不过我们需要在队列加上一个参数即可

```jsx
arguments.put(" x-dead-letter-exchange"，"dlx.exchange");
```

这样消息在过期、requeue、 队列在达到最大长度时，消息就可以直接路由到死信队列！

