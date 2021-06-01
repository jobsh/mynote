# RabbitMQ与Springboot整合

## 1 引入Maven依赖

## 2 配置properties/yaml配置文件

### 2.1 生产端核心配置

```properties
spring rabbitmq.publisher-confirms=true
spring rabbitmq.publisher-returns=true
spring rabbitmqtemplate.mandatory=true
```

### 2.2 消费端核心配置

```properties
spring. abbitmq listener simple, acknowledge-mode=MANUAL # MANUAL 手动确认ACK
spring. abbitmq listener simple concurrency=1 # 并行数量，并发比较大的话可以根据cpu数量，线程数进行调整
spring rabbitmqlistener. simple. max-concurrency=5 
```

### 2.3 @RabbitListener注解使用

> `@RabbitListener`是个复合型注解，包含有`@QueueBinding @Queue @Exchange`

代码中使用

```java
@RabbitListener(bindings = @Queuebinding(
				value = @Queue(value="queue-1", durable="true"),
				exchange =  Exchange(value ="exchange-i
				durable " true",
				type = "topic"
				IgmoredeclarationExceptions ="true"),
                key ="springboot.x)
@RabbitHandler
public void onmessage (Message message, Channel channel)throws Exception {
     // 表示这个方法收到什么消息，可以通过channel获取某些channel对应的信息：如手工ACK时可能需要d			deliveryTag这样的信息
}
```

**PS:**由于类配置写在代码里非常不友好,所以强烈建议大家使用配置文件配置

## 3. P端代码

### Maven依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>rabbit-consumer</artifactId>
    <version>1.0-SNAPSHOT</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-parent</artifactId>
        <version>2.1.5.RELEASE</version>
    </parent>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- springboot 整合 rabbitMQ-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

### application.yml 配置

```yml
server:
  port: 8001
  servlet:
    context-path: /
spring:
  application:
    name: spring-producer
  rabbitmq:
    addresses: 192.168.248.151:5672,192.168.248.152:5672,192.168.248.153:5672
    username: guest
    password: guest
    # 实际工作中可以按照不同的项目分virtual-host
    virtual-host: /
    connection-timeout: 10000 # 连接超时时间为10s
    # 是否启用消息确认模式
    publisher-confirms: true
    # 设置 return消息模式，注意要和mandatory一起配合使用
    publisher-returns: true
    template:
      mandatory: true

```

### 启动类 Application.java （P端、C端相同）

```java
package com.hk.rabbit.producer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

```

### 创建发送消息的component类——RabbitSender.java

```java
package com.hk.rabbit.producer.component;

import org.springframework.amqp.AmqpException;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.core.MessagePostProcessor;
import org.springframework.amqp.rabbit.connection.CorrelationData;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.UUID;

@Component
public class RabbitSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     *  final RabbitTemplate.ConfirmCallback confirmCallback = new RabbitTemplate.ConfirmCallback(){
     *
     *              * @param correlationData :作为一个唯一标识
     *              * @param b ack broker是否落盘成功
     *              * @param s 失败的一些异常信息
     *
     *
             @Override
             public void confirm(CorrelationData correlationData, boolean b, String s) {

             }
     */
    final RabbitTemplate.ConfirmCallback confirmCallback = (correlationData, b,s) -> {

    };

    /**
     * @param message   具体的消息内容
     * @param properties    额外的附加属性
     * @throws Exception
     */
    public void send(Object message, Map<String, Object> properties) throws Exception{
        // 对附件的属性进行的封装
        MessageHeaders messageHeaders = new MessageHeaders(properties);
        org.springframework.messaging.Message<?> msg = MessageBuilder.createMessage(message, messageHeaders);
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.setConfirmCallback(confirmCallback);
        rabbitTemplate.convertAndSend("exchange-1", "springboot.rabbit", msg,
                // 对消息进行后期的处理
                new MessagePostProcessor() {
                    @Override
                    public Message postProcessMessage(Message message) throws AmqpException {
                        System.out.println("---> post to do " + message);
                        return message;
                    }
                },
                correlationData);
    }
}

```

## 4. C端代码

### Maven依赖同上

### application.yml配置

```yml
server:
  port: 8002
  servlet:
    context-path: /
spring:
  application:
    name: spring-producer
  rabbitmq:
    addresses: 192.168.248.151:5672,192.168.248.152:5672,192.168.248.153:5672
    username: guest
    password: guest
    # 实际工作中可以按照不同的项目分virtual-host
    virtual-host: /
    connection-timeout: 10000 # 连接超时时间为10s
    listener:
      simple:
        acknowledge-mode: manual # 表示消费者消费成功后需要手工进行签收（ACK）默认为auto
        concurrency: 5
        prefetch: 1 # 关于消息批量消费，此处设置为1：一条一条消费
        max-concurrency: 10
rabbitmq:
  exchange:
    name: exchange-1
    type: topic
    durable: true
    key: springboot.*
    ignoreDeclarationExceptions: true
  queue:
    name: queue-1
    durable: true
```

监听消息队列 RabbitReceiver.java 用于接受消息，处理消息，手工返回ACK

```java
package consumer.component;

import com.rabbitmq.client.Channel;
import org.springframework.amqp.rabbit.annotation.*;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.Message;
import org.springframework.messaging.MessageHeaders;
import org.springframework.stereotype.Component;

@Component
public class RabbitReceiver {

    @RabbitListener(
            /*
            可以以大括号的形式绑定多个，每个 @QueueBinding是一套绑定
            bindings = {
                   @QueueBinding(value = @Queue(),exchange = @Exchange()),
                   @QueueBinding(...),
                   @QueueBinding(...)
            }
             */
            bindings =  @QueueBinding(
                            value = @Queue(value = "${rabbitmq.queue.name}", durable = "${rabbitmq.queue.durable}"),
                            exchange = @Exchange(name = "${rabbitmq.exchange.name}", durable = "${rabbitmq.exchange.durable}", type = "${rabbitmq.exchange.type}", ignoreDeclarationExceptions = "${rabbitmq.exchange.ignoreDeclarationExceptions}"),
                            key = "${rabbitmq.exchange.key}"
                         )
    )
    @RabbitHandler
    public void onMessage(Message message, Channel channel) throws Exception {
        // 1.收到消息后进行业务端消费处理
        System.out.println("消费消息：" + message.getPayload());
        // 2.处理成功后 获取deliveryTag，并进行手动ACK,因为配置文件里配置了
        // acknowledge-mode: manual 手动签收
        MessageHeaders headers = message.getHeaders();
        Long deliveryTag = (Long)headers.get(AmqpHeaders.DELIVERY_TAG);
        channel.basicAck(deliveryTag,false);
    }

}

```

## 5. 发送消息进行测试

```java
package com.hk.rabbit.producer.test;

import com.hk.rabbit.producer.component.RabbitSender;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.HashMap;
import java.util.Map;

@RunWith(SpringRunner.class)
@SpringBootTest
public class ApplicationTests {
    @Autowired
    private RabbitSender rabbitSender;

    @Test
    public void sendMessage() throws Exception{
        Map<String,Object> properties = new HashMap<>();
        properties.put("argument1","hello1");
        properties.put("argument2","hello2");
        rabbitSender.send("hello word",properties);
        Thread.sleep(10000);

    }
}

```

