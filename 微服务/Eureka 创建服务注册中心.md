# Eureka 创建服务注册中心

## 创建注册中心基本流程

* 创建Demo顶层Pom和子项目eureka-server
* 添加Eureka依赖
* 设置启动类
* Start走起

### POM依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>spring-cloud-learning-imooc</artifactId>
        <groupId>online.loopcode</groupId>
        <version>1.0-SNAPSHOT</version>
        <relativePath>../../pom.xml</relativePath>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>eureka-server</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>
</project>
```

### 2. 创建启动类

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
    public static void main(String[] args) {
        new SpringApplicationBuilder(EurekaApplication.class).web(WebApplicationType.SERVLET)
                .run(args);
    }
}
```

### 3. yml配置

```yml
spring:
  application:
    name: eureka-server
server:
  port: 20000
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false     # 是否把自己注册到eureka
    fetch-registry: false           # 是否拉取注册中心的注册列表
```

## 出现的问题

1. 在启动Eureka注册中心时报了一个错误，是由于引用了错误的pom依赖

* 错误依赖：缺少'starter'

  ```xml
  <dependencies>
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-netflix-eureka-server</artifactId>
      </dependency>
  </dependencies>
  ```

* 正确依赖：

  ```xml
  <dependencies>
      <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
      </dependency>
  </dependencies>
  ```

* 报错信息

  ![image-20210601212905678](http://typicture.loopcode.online/image/image-20210601212905678.png)

2. 关于yml使用 serviceUrl.defultZone 的问题：yml的自动提示没有 serviceUrl.defultZone 

因为service-url的值是一个map，defaultZone相当于这个map中的一个key，yml自动提示是出不来map这种结构的

# 搭建Eureka服务提供者-EurekaClient

基本流程和上面类似，主要有两点的区别：

1. 启动类Application的注解不同。注册中心需要的注解是`@EnableEurekaServer`，而服务提供者需要打上`@EnableDiscoveryClient`注解。
2. yml配置不同：作为服务的提供者，需要向eureka注册中心提供服务，所以register-with-eureka应该是true，并且需要提供eureka的url。
3. 依赖不同

### 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class EurekaClientApplication {
    public static void main(String[] args) {
        new SpringApplicationBuilder(EurekaClientApplication.class).web(WebApplicationType.SERVLET).run(args);
    }
}
```

### yml配置

```yml
server:
  port: 8001
eureka:
  client:
    fetch-registry: false
    #    service-url: http://localhost:20000/eureka/   无法使用
    serviceUrl:
      defaultZone: http://localhost:20000/eureka/
spring:
  application:
    name: eureka-client
```

### Pom依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>spring-cloud-learning-imooc</artifactId>
        <groupId>online.loopcode</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>eureka-client</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
</project>
```

## 服务启动后如下

![image-20210601220315697](http://typicture.loopcode.online/image/image-20210601220315697.png)

发现Eureka-Client已经注册到注册中心了，status是UP。

