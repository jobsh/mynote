# SpringBoot源码解析一：Springboot启动流程

## 前言

springboot通过`SpringApplication.run(Application.class, args);`一行代码就可以启动整个springboot服务，我们来探讨run方法背后运行的事情

![1597544216562](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1597544216562.png)

进入run方法

![1597544330077](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1597544330077.png)

最后是调用的这个run方法

![1597544363327](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1597544363327.png)

我们可以看到，这里其实做了两件事情：

1. new SpringApplication(..)

   这里是初始化SpringApplication容器

2. .run()

   调用run方法真正的启动

## SpringApplication的初始化

![1597544525906](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1597544525906.png)

![1597544549897](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1597544549897.png)

可以看到，springApplication的初始化主要流程为

1. 配置资源加载器

2. 配置`primarySources`

   `primarySources`一般就是我们在main方法中传入的启动类：`Application.class`

3. 应用环境类型检测：`WebApplicationType.deduceFromClasspath()`

   WebApplicationType是一个枚举类型，Application容器一共有三种：

   * NONE：非web环境
   * SERVLET：web环境
   * REACTIVE：springboot 2.x  引入的响应式编程模型

4. 配置应用初始化器：`setInitializer()`

   从spring.factories文件中找出key为ApplicationContextInitializer的类并实例化后设置到SpringApplication的initializers属性中。这个过程也就是找出所有的应用程序初始化器

5. 配置应用监听器：`setListeners()`

   从spring.factories文件中找出key为ApplicationListener的类并实例化后设置到SpringApplication的listeners属性中。这个过程就是找出所有的应用程序事件监听器  

6. 配置main方法所在的类，一般与primarySource类是一样的

## run方法

```java
public ConfigurableApplicationContext run(String... args) {
   // 构造一个任务执行观察器，比如应用的启动所消耗的时间，就是通过stopwatch来实现的
   StopWatch stopWatch = new StopWatch();
   // 开始执行，记录开始时间
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    // Headless模式赋值
   configureHeadlessProperty();
   SpringApplicationRunListeners listeners = getRunListeners(args);
   // 发送ApplicationStartingEvent
   listeners.starting();
   try {
      ApplicationArguments applicationArguments = new 		         DefaultApplicationArguments(
            args);
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
      configureIgnoreBeanInfo(environment);
      Banner printedBanner = printBanner(environment);
      context = createApplicationContext();
      exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
      refreshContext(context);
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```



 ![springboot启动流程](C:\Users\HP\Desktop\springboot启动流程.png)

## 框架自动化装配

1. 收集配置文件中的配置工厂类
2. 加载组件工厂
3. 注册组件内定义bean

