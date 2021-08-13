## Springboot 三大特性

1. 组件自动装配：WEB MVC、web flux、JDBC等组件
2. 嵌入式web容器：tomcat、Jetty、Underflow
3. 生产准备特性，一套完备地运维方案，健康检查、外部化配置

## 组件自动装配

* 如何激活自动装活：使用`@EnableAutoConfiguration`

  言外之意，在SpringBoot中是默认没有激活自动装配的。

* 配置：/META-INF/spring.factories

  该文件是一个**规约文件**。

  使用的是一种工厂机制，文件内部是key-value形式，key是接口，value是实现。实现既可以是SpringBoot内部的实现，也可以是用户自己的实现，

* 实现：XXXAutoConfiguration

## WEB应用

### 传统web应用

#### 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### servlet组件

* Servlet
  * 实现
  * URL映射
  * 注册
* Filter
* Listener