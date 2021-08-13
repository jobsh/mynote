## Spring源码系列之容器启动流程

### Spring启动流程

------

1. Demo创建

2. 启动

3. 入口

4. 基础概念

5. AnnotationConfigApplicationContext的构造方法

   5.1 this()调用

   5.2 register(annotatedClasses)

   5.3 执行refresh()方法6. refresh()方法

​    6.1 invokeBeanFactoryPostProcessors()

​	6.2 registerBeanPostProcessors()

​	6.3 initMessageSource(

​	6.4 initApplicationEventMulticaster()

​	6.5 onRefresh()

​	6.6 registerListeners()

​	6.7 finishBeanFactoryInitialization()

​	6.8 finishRefresh()

​	6.9 resetCommonCaches()

7. 总结
8. 计划
9. 推荐性能监控工具

------

#### 1. Demo创建

- `Demo`代码十分简单，整个工程结构如下:

![img](https://mmbiz.qpic.cn/mmbiz_png/K5cqia0uV8Gx303GNv1VCfPicIvr1xHbLElKZUAfj8fx3b4j74k0NdDucuPQz65jSHWO1Pxnz4SmZ7icaHPNAEWDQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- `pom`依赖

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.1.8.RELEASE</version>
</dependency>
```

- service包下的两个类`OrderService`、`UserService`只加了`@Service`注解，dao包下的两个类`OrderDao`、`UserDao`只加了`@Repository`注解。`MainApplication`类中只写`main()`方法。代码如下：

```
 1public static void main(String[] args) {
 2    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
 3
 4    UserService userService = applicationContext.getBean(UserService.class);
 5    System.out.println(userService);
 6
 7    OrderService orderService = applicationContext.getBean(OrderService.class);
 8    System.out.println(orderService);
 9
10    UserDao userDao = applicationContext.getBean(UserDao.class);
11    System.out.println(userDao);
12
13    OrderDao orderDao = applicationContext.getBean(OrderDao.class);
14    System.out.println(orderDao);
15
16    applicationContext.close();
17}
```

- `AppConfig`类是一个配置类，类上加了两个注解，加`@Configuration`表明`AppConfig`是一个配置类，加`@ComponentScan`是为了告诉`Spring`要扫描哪些包，代码如下:

```
1@Configuration
2@ComponentScan("com.tiantang.study")
3public class AppConfig {
4}
```

#### 2. 启动

- 运行`MainApplication`中的`main()`方法，这样一个`Spring`容器就运行起来了。控制台分别打印出了`UserService`、`OrderService`、`UserDao`、`OrderDao`的`hash`码。
- 那么问题来了，相比以往`xml`配置的方式，现在就这么几行简单的代码，一个`Spring`容器就能运行起来，我们就能从容器中获取到`Bean`，`Spring`内部是如何做到的呢？下面就来逐步分析`Spring`启动的源码。

#### 3. 入口

- 程序的入口为`main()`方法，从代码中可以发现，核心代码只有一行，`new AnnotationConfigApplicationContext(AppConfig.class)`，通过这一行代码，就将`Spring`容器给创建完成，然后我们就能通过`getBean()`从容器中获取到对象的了。因此，分析`Spring`源码，就从`AnnotationConfigApplicationContext`的有参构造函数开始。
- `AnnotationConfigApplicationContext`与`ClassPathXmlApplicationContext`作用一样，前者对应的是采用`JavaConfig`技术的应用，后者对应的是`XML`配置的应用

#### 4. 基础概念

- 在进行`Spring`源码阅读之前，需要先理解几个概念。
- \1. `Spring`会将所有交由`Spring`管理的类，扫描其`class`文件，将其解析成`BeanDefinition`，在`BeanDefinition`中会描述类的信息，例如:这个类是否是单例的，`Bean`的类型，是否是懒加载，依赖哪些类，自动装配的模型。`Spring`创建对象时，就是根据`BeanDefinition`中的信息来创建`Bean`。
- \2. `Spring`容器在本文可以简单理解为`DefaultListableBeanFactory`,它是`BeanFactory`的实现类，这个类有几个非常重要的属性：`beanDefinitionMap`是一个`map`，用来存放`bean`所对应的`BeanDefinition`；`beanDefinitionNames`是一个`List`集合，用来存放所有`bean`的`name`；`singletonObjects`是一个`Map`，用来存放所有创建好的单例`Bean`。
- \3. `Spring`中有很多后置处理器，但最终可以分为两种，一种是`BeanFactoryPostProcessor`，一种是`BeanPostProcessor`。前者的用途是用来干预`BeanFactory`的创建过程，后者是用来干预`Bean`的创建过程。后置处理器的作用十分重要，`bean`的创建以及`AOP`的实现全部依赖后置处理器。

#### 5. AnnotationConfigApplicationContext的构造方法

- `AnnotationConfigApplicationContext`的构造函数的参数，是一个可变数组，可以传多个配置类，在本次`Demo`中，只传了`AppConfig`一个类。
- 在构造函数中，会先调用`this()`，在`this()`中通过调用父类构造器初始化了`BeanFactory`，以及向容器中注册了7个后置处理器。然后调用`register()`，将构造方法的参数放入到`BeanDefinitionMap`中。最后执行`refresh()`方法，这是整个`Spring`容器启动的核心，本文也将重点分析`refresh()`方法的流程和作用。

```
1public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
2    // 会初始化一个BeanFactory,为默认的DefaultListableBeanFactory
3    // 会初始化一个beanDefinition的读取器，同时向容器中注册了7个spring的后置处理器(包括BeanPostProcessor和BeanFactoryPostProcessor)
4    // 会初始化一个扫描器，后面似乎并没有用到这个扫描器，在refresh()中使用的是重新new的一个扫描器。
5    this();
6    // 将配置类注册进BeanDefinitionMap中
7    register(annotatedClasses);
8    refresh();
9}
```

##### 5.1 this()调用

- `this()`会调用`AnnotationConfigApplicationContext`无参构造方法，而在`Java`的继承中，会先调用父类的构造方法。所以会先调用`AnnotationConfigApplicationContext`的父类`GeniricApplicationContext`的构造方法，在父类中初始化`beanFactory`，即直接`new`了一个`DefaultListableBeanFactory`。

```
1public GenericApplicationContext() {
2    this.beanFactory = new DefaultListableBeanFactory();
3}
```

- 在`this()`中通过`new AnnotatedBeanDefinitionReader(this)`实例化了一个`Bean`读取器，并向`BeanDefinitionMap`中添加了`7`个元素。通过`new ClassPathBeanDefinitionScanner(this)`实例化了一个扫描器(该扫描器在后面并没有用到)。

```
1public AnnotationConfigApplicationContext() {
2    // 此处会先调用父类的构造器，即先执行 super(),初始化DefaultListableBeanFactory
3    // 初始化了bean的读取器，并向spring中注册了7个spring自带的类，这里的注册指的是将这7个类对应的BeanDefinition放入到到BeanDefinitionMap中
4    this.reader = new AnnotatedBeanDefinitionReader(this);
5    // 初始化扫描器
6    this.scanner = new ClassPathBeanDefinitionScanner(this);
7}
```

- 执行`this.reader = new AnnotatesBeanDefinitionReader(this)`时，最后会调用到`AnnotationConfigUtils.registerAnnotationConfigProcessors(BeanDefinitionRegistry registry,Object source)`方法，这个方法向`BeanDefinitionMap`中添加了`7`个类，这`7`个类的`BeanDefinition`(关于`BeanDefinition`的介绍可以参考前面的解释)均为`RootBeanDefinition`，这几个类分别为`ConfigurationClassPostProcessor`、`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、`RequiredAnnotationBeanPostProcessor`、`PersistenceBeanPostProcessor`、`EventListenerMethodProcessor`、`DefaultEventListenerFactory`。
- 这7个类中，`ConfigurationClassPostProcessor`、`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`这三个类非常重要，这里先在下面代码中简单介绍了一下作用，后面会单独写文章分析它们的作用。本文的侧重点是先介绍完`Spring`启动的流程。

```
 1public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
 2            BeanDefinitionRegistry registry, @Nullable Object source) {
 3    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
 4    // 省略部分代码 ...
 5    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
 6    // 注册ConfigurationClassPostProcessor,这个类超级重要，它完成了对加了Configuration注解类的解析，@ComponentScan、@Import的解析。
 7    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
 8        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
 9        def.setSource(source);
10        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
11    }
12    // 注册AutowiredAnnotationBeanPostProcessor,这个bean的后置处理器用来处理@Autowired的注入
13    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
14        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
15        def.setSource(source);
16        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
17    }
18    // 注册RequiredAnnotationBeanPostProcessor
19    if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
20        RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
21        def.setSource(source);
22        beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
23    }
24    // 注册CommonAnnotationBeanPostProcessor，用来处理如@Resource，@PostConstruct等符合JSR-250规范的注解
25    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
26    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
27        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
28        def.setSource(source);
29        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
30    }
31    // 注册PersistenceAnnotationBeanPostProcessor，用来支持JPA
32    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
33    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
34        RootBeanDefinition def = new RootBeanDefinition();
35        try {
36            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
37                    AnnotationConfigUtils.class.getClassLoader()));
38        }
39        catch (ClassNotFoundException ex) {
40            throw new IllegalStateException(
41                    "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
42        }
43        def.setSource(source);
44        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
45    }
46
47    // 注册EventListenerMethodProcessor，用来处理方法上加了@EventListener注解的方法
48    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
49        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
50        def.setSource(source);
51        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
52    }
53
54    // 注册DefaultEventListenerFactory，暂时不知道干啥用的，从类名来看，是一个事件监听器的工厂
55    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
56        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
57        def.setSource(source);
58        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
59    }
60    return beanDefs;
61}
```

- 调用`this.scanner = new ClassPathBeanDefinitionScanner(this)`来初始化一个扫描器，这个扫描器在后面扫描包的时候，并没有用到，猜测是`Spring`为了满足其他的场景而初始化的，例如: 开发人员手动通过`register(configClass)`时，扫描包时使用的。

##### 5.2 register(annotatedClasses)

> 将传入的配置类`annotatedClasses`解析成`BeanDefinition`(实际类型为`AnnotatedGenericBeanDefinition`)，然后放入到`BeanDefinitionMap`中，这样后面在`ConfigurationClassPostProcessor`中能解析`annotatedClasses`，例如`demo`中的`AppConfig`类，只有解析了`AppConfig`类，才能知道`Spring`要扫描哪些包(因为在`AppConfig`类中添加了`@ComponentScan`注解)，只有知道要扫描哪些包了，才能扫描出需要交给`Spring`管理的`bean`有哪些，这样才能利用`Spring`来创建`bean`。

##### 5.3 执行refresh()方法

> `refresh()`方法是整个`Spring`容器的核心，在这个方法中进行了`bean`的实例化、初始化、自动装配、`AOP`等功能。下面先看看`refresh()`方法的代码，代码中加了部分个人的理解，简单介绍了每一行代码作用，后面会针对几个重要的方法做出详细分析

```java
 public void refresh() throws BeansException, IllegalStateException {
     synchronized (this.startupShutdownMonitor) {
         // Prepare this context for refreshing.
         // 初始化属性配置文件、检验必须属性以及监听器
         prepareRefresh();
         // Tell the subclass to refresh the internal bean factory.
         // 给beanFactory设置序列化id
         ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
         // 向beanFactory中注册了两个BeanPostProcessor,以及三个和环境相关的bean
        // 这两个后置处理器为ApplicationContextAwareProcessor和ApplicationListenerDetector
        // 前一个后置处理是为实现了ApplicationContextAware接口的类，回调setApplicationContext()方法，
        // 后一个处理器时用来检测ApplicationListener类的，当某个Bean实现了ApplicationListener接口的bean被创建好后，会被加入到监听器列表中
        prepareBeanFactory(beanFactory);
        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 空方法，由子类实现
            postProcessBeanFactory(beanFactory);
            // 执行所有的BeanFactoryPostProcessor，包括自定义的，以及spring内置的。默认情况下，容器中只有一个BeanFactoryPostProcessor,即：Spring内置的，ConfigurationClassPostProcessor(这个类很重要)
            // 会先执行实现了BeanDefinitionRegistryPostProcessor接口的类，然后执行BeanFactoryPostProcessor的类
            // ConfigurationClassPostProcessor类的postProcessorBeanFactory()方法进行了@Configuration类的解析，@ComponentScan的扫描，以及@Import注解的处理
            // 经过这一步以后,会将所有交由spring管理的bean所对应的BeanDefinition放入到beanFactory的beanDefinitionMap中
            // 同时ConfigurationClassPostProcessor类的postProcessorBeanFactory()方法执行完后，向容器中添加了一个后置处理器————ImportAwareBeanPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);
            // 注册所有的BeanPostProcessor，因为在方法里面调用了getBean()方法，所以在这一步，实际上已经将所有的BeanPostProcessor实例化了
            // 为什么要在这一步就将BeanPostProcessor实例化呢？因为后面要实例化bean，而BeanPostProcessor是用来干预bean的创建过程的，所以必须在bean实例化之前就实例化所有的BeanPostProcessor(包括开发人员自己定义的)
            // 最后再重新注册了ApplicationListenerDetector，这样做的目的是为了将ApplicationListenerDetector放入到后置处理器的最末端
            registerBeanPostProcessors(beanFactory);
            // Initialize message source for this context.
           // 初始化MessageSource，用来做消息国际化。在一般项目中不会用到消息国际化
            initMessageSource();
            // Initialize event multicaster for this context.
            // 初始化事件广播器，如果容器中存在了名字为applicationEventMulticaster的广播器，则使用该广播器
            // 如果没有，则初始化一个SimpleApplicationEventMulticaster
            // 事件广播器的用途是，发布事件，并且为所发布的事件找到对应的事件监听器。
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 执行其他的初始化操作，例如和SpringMVC整合时，需要初始化一些其他的bean，但是对于纯spring工程来说，onFresh方法是一个空方法
            onRefresh();

            // Check for listener beans and register them.
            // 这一步会将自定义的listener的bean名称放入到事件广播器中
            // 同时还会将早期的ApplicationEvent发布(对于单独的spring工程来说，在此时不会有任何ApplicationEvent发布，但是和springMVC整合时，springMVC会执行onRefresh()方法，在这里会发布事件)
            registerListeners();
            // 实例化剩余的非懒加载的单例bean(注意：剩余、非懒加载、单例)
            // 为什么说是剩余呢？如果开发人员自定义了BeanPostProcessor，而BeanPostProcessor在前面已经实例化了，所以在这里不会再实例化，因此这里使用剩余一词
            finishBeanFactoryInitialization(beanFactory);
            // 结束refresh，主要干了一件事，就是发布一个事件ContextRefreshEvent，通知大家spring容器refresh结束了。
            finishRefresh();
        }
        catch (BeansException ex) {
            // 出异常后销毁bean
            destroyBeans();
            // Reset 'active' flag.
            cancelRefresh(ex);
            // Propagate exception to caller.
            throw ex;
        }
        finally {
           // 在bean的实例化过程中，会缓存很多信息，例如bean的注解信息，但是当单例bean实例化完成后，这些缓存信息已经不会再使用了，所以可以释放这些内存资源了
            resetCommonCaches();
        }
    }
}
```

#### 6. refresh()方法

- 在`refresh()`方法中，比较重要的方法为`invokeBeanFactoryPostProcessors(beanFactory)` 和 `finishBeanFactoryInitialization(beanFactory)`。其他的方法相对而言比较简单，下面主要分析这两个方法，其他方法的作用，可以参考上面源码中的注释。

##### 6.1 invokeBeanFactoryPostProcessors()

- 该方法的作用是执行所有的`BeanFactoryPostProcessor`，由于`Spring`会内置一个`BeanFactoryPostProcessor`，即`ConfigurationClassPostProcessor`(如果开发人员不自定义，默认情况下只有这一个`BeanFactoryPostProcessor`)，这个后置处理器在处理时，会解析出所有交由`Spring`容器管理的`Bean`，将它们解析成`BeanDefinition`，然后放入到`BeanFactory`的`BeanDefinitionMap`中。

- 该方法最终会调用到`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`方法，主要作用是执行所有`BeanFactoryPostProcessor`的`postProcessorBeanFactory()`方法。`BeanFactoryPostProcessor`又分为两种情况，一种是直接实现`BeanFactoryPostProcessor`接口的类，另一种情况是实现了`BeanDefinitionRegistryPostProcessor`接口(`BeanDefinitionRegistryPostProcessor`继承了`BeanFactoryPostProcessor`接口)。

  

![img](https://mmbiz.qpic.cn/mmbiz_png/K5cqia0uV8Gx303GNv1VCfPicIvr1xHbLEzfREPNXibX2PABMLicqBwpGmgicLQOC2Qhdt7DpgWlf0ChIVfTIQIfsXg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



- 在执行过程中先执行所有的`BeanDefinitionRegistryPostProcessor`的`postProcessorBeanDefinitionRegistry()`方法，然后再执行`BeanFacotryPostProcessor`的`postProcessorBeanFactory()`方法。

```
1public interface BeanFactoryPostProcessor {
2    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
3}
1public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
2    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
3}
```

- 默认情况下，`Spring`有一个内置的`BeanFactoryPostProcessor`，即：`ConfigurationClassPostProcessor`类，该类实现了`BeanDefinitionRegistryPostProcessor`类，所以会执行`ConfigurationClassPostProcessor.postProcessorBeanDefinitionRegistry`,`ConfigurationClassPostProcessor`的`UML`图如上(删减了部分不重要的继承关系)

##### 6.2 registerBeanPostProcessors()

- 该方法的作用是找到所有的`BeanPostProcessor`，然后将这些`BeanPostProcessor`实例化(会调用`getBean()`方法，`getBean()`方法的主要逻辑是，如果`bean`存在于`BeanFactory`中，则返回`bean`；如果不存在，则会去创建。在后面会仔细分析`getBean()`的执行逻辑)。将这些`PostProcessor`实例化后，最后放入到`BeanFactory`的`beanPostProcessors`属性中。
- 问题：如何找到所有的`BeanPostProcessor`? 包括`Spring`内置的和开发人员自定义的。
- 由于在`refresh()`方法中，会先执行完`invokeBeanFactoryPostProcessor()`方法，这样所有自定义的`BeanPostProcessor`类均已经被扫描出并解析成`BeanDefinition`(扫描和解析又是谁做的呢？`ConfigurationClassPostProcessor`做的)，存入至`BeanFactory`的`BeanDefinitionMap`，所以这儿能通过方法如下一行代码找出所有的`BeanPostProcessor`，然后通过`getBean()`全部实例化，最后再将实例化后的对象加入到`BeanFactory`的`beanPostProcessors`属性中，该属性是一个`List`集合。

```
1String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
```

- 最后再重新注册了`ApplicationListenerDetector`，这样做的目的是为了将`ApplicationListenerDetector`放入到后置处理器的最末端
- `registerBeanPostProcessor()` 最终调用的是`PostProcessorRegistrationDelegate.registerBeanPostProcessors()`，下面是`PostProcessorRegistrationDelegate.registerBeanPostProcessors()`方法的代码

```
 1public static void registerBeanPostProcessors(
 2        ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
 3
 4    // 从BeanDefinitionMap中找出所有的BeanPostProcessor
 5    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
 6    // ... 省略部分代码 ...
 7
 8    // 分别找出实现了PriorityOrdered、Ordered接口以及普通的BeanPostProcessor
 9    for (String ppName : postProcessorNames) {
10        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
11            // 此处调用了getBean()方法，因此在此处就会实例化出BeanPostProcessor
12            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
13            priorityOrderedPostProcessors.add(pp);
14            if (pp instanceof MergedBeanDefinitionPostProcessor) {
15                internalPostProcessors.add(pp);
16            }
17        }
18        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
19            orderedPostProcessorNames.add(ppName);
20        }
21        else {
22            nonOrderedPostProcessorNames.add(ppName);
23        }
24    }
25
26    // First, register the BeanPostProcessors that implement PriorityOrdered.
27    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
28    // 将实现了PriorityOrdered接口的BeanPostProcessor添加到BeanFactory的beanPostProcessors集合中
29    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
30
31    // 下面这部分代码与上面的代码逻辑一致，是将实现了Ordered接口以及普通的BeanPostProcessor实例化以及添加到beanPostProcessors结合中，逻辑与处理PriorityOrdered的后置处理器一样
32    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
33    for (String ppName : orderedPostProcessorNames) {
34        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
35        orderedPostProcessors.add(pp);
36        if (pp instanceof MergedBeanDefinitionPostProcessor) {
37            internalPostProcessors.add(pp);
38        }
39    }
40    sortPostProcessors(orderedPostProcessors, beanFactory);
41    registerBeanPostProcessors(beanFactory, orderedPostProcessors);
42
43    // Now, register all regular BeanPostProcessors.
44    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
45    for (String ppName : nonOrderedPostProcessorNames) {
46        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
47        nonOrderedPostProcessors.add(pp);
48        if (pp instanceof MergedBeanDefinitionPostProcessor) {
49            internalPostProcessors.add(pp);
50        }
51    }
52    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
53
54    // Finally, re-register all internal BeanPostProcessors.
55    sortPostProcessors(internalPostProcessors, beanFactory);
56    registerBeanPostProcessors(beanFactory, internalPostProcessors);
57
58    // Re-register post-processor for detecting inner beans as ApplicationListeners,
59    // moving it to the end of the processor chain (for picking up proxies etc).
60    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
61
62    // 最后将ApplicationListenerDetector这个后置处理器一样重新放入到beanPostProcessor中，这样做的目的是为了将其放入到后置处理器的最末端
63    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
64}
```

- 从上面的源码中可以发现，`BeanPostProcessor`存在优先级，实现了`PriorityOrdered`接口的优先级最高，其次是`Ordered`接口，最后是普通的`BeanPostProcessor`。优先级最高的，会最先放入到`beanPostProcessors`这个集合的最前面，这样在执行时，会最先执行优先级最高的后置处理器(因为`List`集合是有序的)。
- 这样在实际应用中，如果我们碰到需要优先让某个`BeanPostProcessor`执行，则可以让其实现`PriorityOrdered`接口或者`Ordered`接口。

##### 6.3 initMessageSource()

- 用来支持消息国际化，现在一般项目中不会用到国际化相关的知识。

##### 6.4 initApplicationEventMulticaster()

> 该方法初始化了一个事件广播器，如果容器中存在了`beanName`为`applicationEventMulticaster`的广播器，则使用该广播器；如果没有，则初始化一个`SimpleApplicationEventMulticaster`。该事件广播器是用来做应用事件分发的，这个类会持有所有的事件监听器(`ApplicationListener`)，当有`ApplicationEvent`事件发布时，该事件监听器能根据事件类型，检索到对该事件感兴趣的`ApplicationListener`。

- `initApplicationEventMulticaster()`方法的源码如下(省略了部分日志信息):

```
 1protected void initApplicationEventMulticaster() {
 2    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
 3    // 判断spring容器中是否已经存在beanName = applicationEventMulticaster的事件广播器
 4    // 例如：如果开发人员自己注册了一个
 5    // 如果存在，则使用已经存在的；否则使用spring默认的:SimpleApplicationEventMulticaster
 6    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
 7        this.applicationEventMulticaster =
 8                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
 9    }
10    else {
11        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
12        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
13    }
14}
```

##### 6.5 onRefresh()

> 执行其他的初始化操作，例如和`SpringMVC`整合时，需要初始化一些其他的`bean`，但是对于纯`Spring`工程来说，`onRefresh()`方法是一个空方法。

##### 6.6 registerListeners()

> 这一步会将自定义的`listener`的`bean`名称放入到事件广播器中,同时还会将早期的`ApplicationEvent`发布(对于单独的`Spring`工程来说，在此时不会有任何`ApplicationEvent`发布，但是和`SpringMVC`整合时，`SpringMVC`会执行`onRefresh()`方法，在这里会发布事件)。方法源码如下:

```
 1protected void registerListeners() {
 2    // Register statically specified listeners first.
 3    for (ApplicationListener<?> listener : getApplicationListeners()) {
 4  getApplicationEventMulticaster().addApplicationListener(listener);
 5    }
 6
 7    // 从BeanFactory中找到所有的ApplicationListener，但是不会进行初始化，因为需要在后面bean实例化的过程中，让所有的BeanPostProcessor去改造它们
 8    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
 9    for (String listenerBeanName : listenerBeanNames) {
10        // 将事件监听器的beanName放入到事件广播器中
11        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
12    }
13
14    // 发布早期的事件(纯的spring工程，在此时一个事件都没有)
15    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
16    this.earlyApplicationEvents = null;
17    if (earlyEventsToProcess != null) {
18        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
19            getApplicationEventMulticaster().multicastEvent(earlyEvent);
20        }
21    }
22}
```

##### 6.7 finishBeanFactoryInitialization()

> 该方法十分重要，它完成了所有非懒加载的单例`Bean`的实例化和初始化，属性的填充以及解决了循环依赖等问题。由于微信平台对文章字数有限制，因此关于`Bean`的创建过程移到了另外一篇文章中,点击后面链接查看。([通过源码看Bean的创建过程](https://mp.weixin.qq.com/s?__biz=MzI4Mjg2NjUzNw==&mid=2247483679&idx=1&sn=03a08c6844397a6d2610d89577eb8558&scene=21#wechat_redirect))
>
> 在`《通过源码看Bean的创建过程》`这边文章中通过源码分析了所有后置处理器的作用，`Bean`的生命周期，以及后置处理的应用场景。

##### 6.8 finishRefresh()

- 执行到这一步，`Spring`容器的启动基本结束了，此时`Bean`已经被实例化完成，且完成了自动装配。执行`finishRefresh()`方法，是为了在容器`refresh()`结束时，做一些其他的操作，例如：发布`ContextRefreshedEvent`事件，这样当我们想在容器`refresh`完成后执行一些特殊的逻辑，就可以通过监听`ContextRefreshedEvent`事件来实现。`Spring`内置了四个和应用上下文(`ApplicationContextEvent`)有关的事件：`ContextRefreshedEvent`、`ContextStartedEvent`、`ContextStopedEvent`、`ContextClosedEvent`。

```
1protected void finishRefresh() {
2    clearResourceCaches();
3    initLifecycleProcessor();
4    getLifecycleProcessor().onRefresh();
5    // 发布ContextRefreshedEvent
6    publishEvent(new ContextRefreshedEvent(this));
7    LiveBeansView.registerApplicationContext(this˛);
8}
```

##### 6.9 resetCommonCaches()

> 最后在`refresh()`方法的`finally`语句块中，执行了`resetCommonCaches()`方法。因为在前面创建`bean`时，对单例`bean`的元数据信息进行了缓存，而单例`bean`在容器启动后，不会再进行创建了，因此这些缓存的信息已经没有任何用处了，在这里进行清空，释放部分内存。

```
1protected void resetCommonCaches() {
2    ReflectionUtils.clearCache();
3    AnnotationUtils.clearCache();
4    ResolvableType.clearCache();
5    CachedIntrospectionResults.clearClassLoader(getClassLoader());
6}
```

#### 7. 总结

- 本文介绍了`Spring`的启动流程，通过`AnnotationConfigApplicationContext`的有参构造方法入手，重点分析了`this()`方法和`refresh()`方法。在`this()`中初始化了一个`BeanFactory`，即`DefaultListableBeanFactory`；然后向容器中添加了7个内置的`bean`，其中就包括`ConfigurationClassPostProcessor`。
- 在`refresh()`方法中，又重点分析了`invokeBeanFactoryPostProcessor()`方法和`finishBeanFactoryInitialization()`方法。
- 在`invokeBeanFactoryPostProcessor()`方法中，通过`ConfigurationClassPostProcessor`类扫描出了所有交给`Spring`管理的类，并将`class`文件解析成对应的`BeanDefinition`。
- 在`finishBeanFactoryInitialization()`方法中，完成了非懒加载的单例`Bean`的实例化和初始化操作，主要流程为`getBean()` ——>`doGetBean()`——>`createBean()`——>`doCreateBean()`。在`bean`的创建过程中，一共出现了`8`次`BeanPostProcessor`的执行，在这些后置处理器的执行过程中，完成了`AOP`的实现、`bean`的自动装配、属性赋值等操作。
- 最后通过一张流程图，总结了`Spring`中单例`Bean`的生命周期。

#### 8. 计划

> 本文主要介绍了`Spring`的启动流程，但对于一些地方的具体实现细节没有展开分析，因此后续Spring源码分析的计划如下:

- `ConfigurationClassPostProcessor`类如何扫描包，解析配置类。

- `@Import`注解作用与`@Enable`系列注解的实现原理

- `JDK`动态代理与`CGLIB`代理

- `FactoryBean`的用途和源码分析

- `AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`如何实现自动装配,`Spring`如何解决循环依赖

- `AOP`的实现原理

- `SpringBoot`源码分析

#### 10. 推荐性能监控工具

- 最后推荐一款开源的性能监控工具——`Pepper-Metrics`

> 地址：https://github.com/zrbcool/pepper-metrics
> GitHub
> `Pepper-Metrics`是坐我对面的两位同事一起开发的开源组件，主要功能是通过比较轻量的方式与常用开源组件（`jedis/mybatis/motan/dubbo/servlet`）集成，收集并计算`metrics`，并支持输出到日志及转换成多种时序数据库兼容数据格式，配套的`grafana dashboard`友好的进行展示。项目当中原理文档齐全，且全部基于`SPI`设计的可扩展式架构，方便的开发新插件。另有一个基于`docker-compose`的独立`demo`项目可以快速启动一套`demo`示例查看效果`https://github.com/zrbcool/pepper-metrics-demo`。如果大家觉得有用的话，麻烦给个`star`，也欢迎大家参与开发，谢谢：）Spring源码系列之容器启动流程
>
> 刘进坤 [菜鸟飞呀飞](javascript:void(0);) *2019-09-12*
>
> 
>
> > 点击上方`”菜鸟飞呀飞“`，即可关注微信公众号。
>
> ### Spring启动流程
>
> ------
>
> \1. Demo创建2. 启动3. 入口4. 基础概念5. AnnotationConfigApplicationContext的构造方法5.1 this()调用5.2 register(annotatedClasses)5.3 执行refresh()方法6. refresh()方法6.1 invokeBeanFactoryPostProcessors()6.2 registerBeanPostProcessors()6.3 initMessageSource()6.4 initApplicationEventMulticaster()6.5 onRefresh()6.6 registerListeners()6.7 finishBeanFactoryInitialization()6.8 finishRefresh()6.9 resetCommonCaches()7. 总结8. 计划10. 推荐性能监控工具
>
> ------
>
> #### 1. Demo创建
>
> - `Demo`代码十分简单，整个工程结构如下:
>
> ![img](https://mmbiz.qpic.cn/mmbiz_png/K5cqia0uV8Gx303GNv1VCfPicIvr1xHbLElKZUAfj8fx3b4j74k0NdDucuPQz65jSHWO1Pxnz4SmZ7icaHPNAEWDQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
>
> - `pom`依赖
>
> ```
> 1<dependency>
> 2    <groupId>org.springframework</groupId>
> 3    <artifactId>spring-context</artifactId>
> 4    <version>5.1.8.RELEASE</version>
> 5</dependency>
> ```
>
> - service包下的两个类`OrderService`、`UserService`只加了`@Service`注解，dao包下的两个类`OrderDao`、`UserDao`只加了`@Repository`注解。`MainApplication`类中只写`main()`方法。代码如下：
>
> ```
>  1public static void main(String[] args) {
>  2    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
>  3
>  4    UserService userService = applicationContext.getBean(UserService.class);
>  5    System.out.println(userService);
>  6
>  7    OrderService orderService = applicationContext.getBean(OrderService.class);
>  8    System.out.println(orderService);
>  9
> 10    UserDao userDao = applicationContext.getBean(UserDao.class);
> 11    System.out.println(userDao);
> 12
> 13    OrderDao orderDao = applicationContext.getBean(OrderDao.class);
> 14    System.out.println(orderDao);
> 15
> 16    applicationContext.close();
> 17}
> ```
>
> - `AppConfig`类是一个配置类，类上加了两个注解，加`@Configuration`表明`AppConfig`是一个配置类，加`@ComponentScan`是为了告诉`Spring`要扫描哪些包，代码如下:
>
> ```
> 1@Configuration
> 2@ComponentScan("com.tiantang.study")
> 3public class AppConfig {
> 4}
> ```
>
> #### 2. 启动
>
> - 运行`MainApplication`中的`main()`方法，这样一个`Spring`容器就运行起来了。控制台分别打印出了`UserService`、`OrderService`、`UserDao`、`OrderDao`的`hash`码。
> - 那么问题来了，相比以往`xml`配置的方式，现在就这么几行简单的代码，一个`Spring`容器就能运行起来，我们就能从容器中获取到`Bean`，`Spring`内部是如何做到的呢？下面就来逐步分析`Spring`启动的源码。
>
> #### 3. 入口
>
> - 程序的入口为`main()`方法，从代码中可以发现，核心代码只有一行，`new AnnotationConfigApplicationContext(AppConfig.class)`，通过这一行代码，就将`Spring`容器给创建完成，然后我们就能通过`getBean()`从容器中获取到对象的了。因此，分析`Spring`源码，就从`AnnotationConfigApplicationContext`的有参构造函数开始。
> - `AnnotationConfigApplicationContext`与`ClassPathXmlApplicationContext`作用一样，前者对应的是采用`JavaConfig`技术的应用，后者对应的是`XML`配置的应用
>
> #### 4. 基础概念
>
> - 在进行`Spring`源码阅读之前，需要先理解几个概念。
> - \1. `Spring`会将所有交由`Spring`管理的类，扫描其`class`文件，将其解析成`BeanDefinition`，在`BeanDefinition`中会描述类的信息，例如:这个类是否是单例的，`Bean`的类型，是否是懒加载，依赖哪些类，自动装配的模型。`Spring`创建对象时，就是根据`BeanDefinition`中的信息来创建`Bean`。
> - \2. `Spring`容器在本文可以简单理解为`DefaultListableBeanFactory`,它是`BeanFactory`的实现类，这个类有几个非常重要的属性：`beanDefinitionMap`是一个`map`，用来存放`bean`所对应的`BeanDefinition`；`beanDefinitionNames`是一个`List`集合，用来存放所有`bean`的`name`；`singletonObjects`是一个`Map`，用来存放所有创建好的单例`Bean`。
> - \3. `Spring`中有很多后置处理器，但最终可以分为两种，一种是`BeanFactoryPostProcessor`，一种是`BeanPostProcessor`。前者的用途是用来干预`BeanFactory`的创建过程，后者是用来干预`Bean`的创建过程。后置处理器的作用十分重要，`bean`的创建以及`AOP`的实现全部依赖后置处理器。
>
> #### 5. AnnotationConfigApplicationContext的构造方法
>
> - `AnnotationConfigApplicationContext`的构造函数的参数，是一个可变数组，可以传多个配置类，在本次`Demo`中，只传了`AppConfig`一个类。
> - 在构造函数中，会先调用`this()`，在`this()`中通过调用父类构造器初始化了`BeanFactory`，以及向容器中注册了7个后置处理器。然后调用`register()`，将构造方法的参数放入到`BeanDefinitionMap`中。最后执行`refresh()`方法，这是整个`Spring`容器启动的核心，本文也将重点分析`refresh()`方法的流程和作用。
>
> ```
> 1public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
> 2    // 会初始化一个BeanFactory,为默认的DefaultListableBeanFactory
> 3    // 会初始化一个beanDefinition的读取器，同时向容器中注册了7个spring的后置处理器(包括BeanPostProcessor和BeanFactoryPostProcessor)
> 4    // 会初始化一个扫描器，后面似乎并没有用到这个扫描器，在refresh()中使用的是重新new的一个扫描器。
> 5    this();
> 6    // 将配置类注册进BeanDefinitionMap中
> 7    register(annotatedClasses);
> 8    refresh();
> 9}
> ```
>
> ##### 5.1 this()调用
>
> - `this()`会调用`AnnotationConfigApplicationContext`无参构造方法，而在`Java`的继承中，会先调用父类的构造方法。所以会先调用`AnnotationConfigApplicationContext`的父类`GeniricApplicationContext`的构造方法，在父类中初始化`beanFactory`，即直接`new`了一个`DefaultListableBeanFactory`。
>
> ```
> 1public GenericApplicationContext() {
> 2    this.beanFactory = new DefaultListableBeanFactory();
> 3}
> ```
>
> - 在`this()`中通过`new AnnotatedBeanDefinitionReader(this)`实例化了一个`Bean`读取器，并向`BeanDefinitionMap`中添加了`7`个元素。通过`new ClassPathBeanDefinitionScanner(this)`实例化了一个扫描器(该扫描器在后面并没有用到)。
>
> ```
> 1public AnnotationConfigApplicationContext() {
> 2    // 此处会先调用父类的构造器，即先执行 super(),初始化DefaultListableBeanFactory
> 3    // 初始化了bean的读取器，并向spring中注册了7个spring自带的类，这里的注册指的是将这7个类对应的BeanDefinition放入到到BeanDefinitionMap中
> 4    this.reader = new AnnotatedBeanDefinitionReader(this);
> 5    // 初始化扫描器
> 6    this.scanner = new ClassPathBeanDefinitionScanner(this);
> 7}
> ```
>
> - 执行`this.reader = new AnnotatesBeanDefinitionReader(this)`时，最后会调用到`AnnotationConfigUtils.registerAnnotationConfigProcessors(BeanDefinitionRegistry registry,Object source)`方法，这个方法向`BeanDefinitionMap`中添加了`7`个类，这`7`个类的`BeanDefinition`(关于`BeanDefinition`的介绍可以参考前面的解释)均为`RootBeanDefinition`，这几个类分别为`ConfigurationClassPostProcessor`、`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、`RequiredAnnotationBeanPostProcessor`、`PersistenceBeanPostProcessor`、`EventListenerMethodProcessor`、`DefaultEventListenerFactory`。
> - 这7个类中，`ConfigurationClassPostProcessor`、`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`这三个类非常重要，这里先在下面代码中简单介绍了一下作用，后面会单独写文章分析它们的作用。本文的侧重点是先介绍完`Spring`启动的流程。
>
> ```
>  1public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
>  2            BeanDefinitionRegistry registry, @Nullable Object source) {
>  3    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
>  4    // 省略部分代码 ...
>  5    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
>  6    // 注册ConfigurationClassPostProcessor,这个类超级重要，它完成了对加了Configuration注解类的解析，@ComponentScan、@Import的解析。
>  7    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
>  8        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
>  9        def.setSource(source);
> 10        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
> 11    }
> 12    // 注册AutowiredAnnotationBeanPostProcessor,这个bean的后置处理器用来处理@Autowired的注入
> 13    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
> 14        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
> 15        def.setSource(source);
> 16        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
> 17    }
> 18    // 注册RequiredAnnotationBeanPostProcessor
> 19    if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
> 20        RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
> 21        def.setSource(source);
> 22        beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
> 23    }
> 24    // 注册CommonAnnotationBeanPostProcessor，用来处理如@Resource，@PostConstruct等符合JSR-250规范的注解
> 25    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
> 26    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
> 27        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
> 28        def.setSource(source);
> 29        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
> 30    }
> 31    // 注册PersistenceAnnotationBeanPostProcessor，用来支持JPA
> 32    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
> 33    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
> 34        RootBeanDefinition def = new RootBeanDefinition();
> 35        try {
> 36            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
> 37                    AnnotationConfigUtils.class.getClassLoader()));
> 38        }
> 39        catch (ClassNotFoundException ex) {
> 40            throw new IllegalStateException(
> 41                    "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
> 42        }
> 43        def.setSource(source);
> 44        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
> 45    }
> 46
> 47    // 注册EventListenerMethodProcessor，用来处理方法上加了@EventListener注解的方法
> 48    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
> 49        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
> 50        def.setSource(source);
> 51        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
> 52    }
> 53
> 54    // 注册DefaultEventListenerFactory，暂时不知道干啥用的，从类名来看，是一个事件监听器的工厂
> 55    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
> 56        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
> 57        def.setSource(source);
> 58        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
> 59    }
> 60    return beanDefs;
> 61}
> ```
>
> - 调用`this.scanner = new ClassPathBeanDefinitionScanner(this)`来初始化一个扫描器，这个扫描器在后面扫描包的时候，并没有用到，猜测是`Spring`为了满足其他的场景而初始化的，例如: 开发人员手动通过`register(configClass)`时，扫描包时使用的。
>
> ##### 5.2 register(annotatedClasses)
>
> > 将传入的配置类`annotatedClasses`解析成`BeanDefinition`(实际类型为`AnnotatedGenericBeanDefinition`)，然后放入到`BeanDefinitionMap`中，这样后面在`ConfigurationClassPostProcessor`中能解析`annotatedClasses`，例如`demo`中的`AppConfig`类，只有解析了`AppConfig`类，才能知道`Spring`要扫描哪些包(因为在`AppConfig`类中添加了`@ComponentScan`注解)，只有知道要扫描哪些包了，才能扫描出需要交给`Spring`管理的`bean`有哪些，这样才能利用`Spring`来创建`bean`。
>
> ##### 5.3 执行refresh()方法
>
> > `refresh()`方法是整个`Spring`容器的核心，在这个方法中进行了`bean`的实例化、初始化、自动装配、`AOP`等功能。下面先看看`refresh()`方法的代码，代码中加了部分个人的理解，简单介绍了每一行代码作用，后面会针对几个重要的方法做出详细分析
>
> ```
>  1public void refresh() throws BeansException, IllegalStateException {
>  2    synchronized (this.startupShutdownMonitor) {
>  3        // Prepare this context for refreshing.
>  4        // 初始化属性配置文件、检验必须属性以及监听器
>  5        prepareRefresh();
>  6        // Tell the subclass to refresh the internal bean factory.
>  7        // 给beanFactory设置序列化id
>  8        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
>  9        // 向beanFactory中注册了两个BeanPostProcessor,以及三个和环境相关的bean
> 10        // 这两个后置处理器为ApplicationContextAwareProcessor和ApplicationListenerDetector
> 11        // 前一个后置处理是为实现了ApplicationContextAware接口的类，回调setApplicationContext()方法，
> 12        // 后一个处理器时用来检测ApplicationListener类的，当某个Bean实现了ApplicationListener接口的bean被创建好后，会被加入到监听器列表中
> 13        prepareBeanFactory(beanFactory);
> 14        try {
> 15            // Allows post-processing of the bean factory in context subclasses.
> 16            // 空方法，由子类实现
> 17            postProcessBeanFactory(beanFactory);
> 18            // 执行所有的BeanFactoryPostProcessor，包括自定义的，以及spring内置的。默认情况下，容器中只有一个BeanFactoryPostProcessor,即：Spring内置的，ConfigurationClassPostProcessor(这个类很重要)
> 19            // 会先执行实现了BeanDefinitionRegistryPostProcessor接口的类，然后执行BeanFactoryPostProcessor的类
> 20            // ConfigurationClassPostProcessor类的postProcessorBeanFactory()方法进行了@Configuration类的解析，@ComponentScan的扫描，以及@Import注解的处理
> 21            // 经过这一步以后,会将所有交由spring管理的bean所对应的BeanDefinition放入到beanFactory的beanDefinitionMap中
> 22            // 同时ConfigurationClassPostProcessor类的postProcessorBeanFactory()方法执行完后，向容器中添加了一个后置处理器————ImportAwareBeanPostProcessor
> 23            invokeBeanFactoryPostProcessors(beanFactory);
> 24            // 注册所有的BeanPostProcessor，因为在方法里面调用了getBean()方法，所以在这一步，实际上已经将所有的BeanPostProcessor实例化了
> 25            // 为什么要在这一步就将BeanPostProcessor实例化呢？因为后面要实例化bean，而BeanPostProcessor是用来干预bean的创建过程的，所以必须在bean实例化之前就实例化所有的BeanPostProcessor(包括开发人员自己定义的)
> 26            // 最后再重新注册了ApplicationListenerDetector，这样做的目的是为了将ApplicationListenerDetector放入到后置处理器的最末端
> 27            registerBeanPostProcessors(beanFactory);
> 28            // Initialize message source for this context.
> 29           // 初始化MessageSource，用来做消息国际化。在一般项目中不会用到消息国际化
> 30            initMessageSource();
> 31            // Initialize event multicaster for this context.
> 32            // 初始化事件广播器，如果容器中存在了名字为applicationEventMulticaster的广播器，则使用该广播器
> 33            // 如果没有，则初始化一个SimpleApplicationEventMulticaster
> 34            // 事件广播器的用途是，发布事件，并且为所发布的时间找到对应的事件监听器。
> 35            initApplicationEventMulticaster();
> 36
> 37            // Initialize other special beans in specific context subclasses.
> 38            // 执行其他的初始化操作，例如和SpringMVC整合时，需要初始化一些其他的bean，但是对于纯spring工程来说，onFresh方法是一个空方法
> 39            onRefresh();
> 40
> 41            // Check for listener beans and register them.
> 42            // 这一步会将自定义的listener的bean名称放入到事件广播器中
> 43            // 同时还会将早期的ApplicationEvent发布(对于单独的spring工程来说，在此时不会有任何ApplicationEvent发布，但是和springMVC整合时，springMVC会执行onRefresh()方法，在这里会发布事件)
> 44            registerListeners();
> 45            // 实例化剩余的非懒加载的单例bean(注意：剩余、非懒加载、单例)
> 46            // 为什么说是剩余呢？如果开发人员自定义了BeanPosrProcessor，而BeanPostProcessor在前面已经实例化了，所以在这里不会再实例化，因此这里使用剩余一词
> 47            finishBeanFactoryInitialization(beanFactory);
> 48            // 结束refresh，主要干了一件事，就是发布一个事件ContextRefreshEvent，通知大家spring容器refresh结束了。
> 49            finishRefresh();
> 50        }
> 51        catch (BeansException ex) {
> 52            // 出异常后销毁bean
> 53            destroyBeans();
> 54            // Reset 'active' flag.
> 55            cancelRefresh(ex);
> 56            // Propagate exception to caller.
> 57            throw ex;
> 58        }
> 59        finally {
> 60           // 在bean的实例化过程中，会缓存很多信息，例如bean的注解信息，但是当单例bean实例化完成后，这些缓存信息已经不会再使用了，所以可以释放这些内存资源了
> 61            resetCommonCaches();
> 62        }
> 63    }
> 64}
> ```
>
> #### 6. refresh()方法
>
> - 在`refresh()`方法中，比较重要的方法为`invokeBeanFactoryPostProcessors(beanFactory)` 和 `finishBeanFactoryInitialization(beanFactory)`。其他的方法相对而言比较简单，下面主要分析这两个方法，其他方法的作用，可以参考上面源码中的注释。
>
> ##### 6.1 invokeBeanFactoryPostProcessors()
>
> - 该方法的作用是执行所有的`BeanFactoryPostProcessor`，由于`Spring`会内置一个`BeanFactoryPostProcessor`，即`ConfigurationClassPostProcessor`(如果开发人员不自定义，默认情况下只有这一个`BeanFactoryPostProcessor`)，这个后置处理器在处理时，会解析出所有交由`Spring`容器管理的`Bean`，将它们解析成`BeanDefinition`，然后放入到`BeanFactory`的`BeanDefinitionMap`中。
>
> - 该方法最终会调用到`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`方法，主要作用是执行所有`BeanFactoryPostProcessor`的`postProcessorBeanFactory()`方法。`BeanFactoryPostProcessor`又分为两种情况，一种是直接实现`BeanFactoryPostProcessor`接口的类，另一种情况是实现了`BeanDefinitionRegistryPostProcessor`接口(`BeanDefinitionRegistryPostProcessor`继承了`BeanFactoryPostProcessor`接口)。
>
>   
>
> ![img](https://mmbiz.qpic.cn/mmbiz_png/K5cqia0uV8Gx303GNv1VCfPicIvr1xHbLEzfREPNXibX2PABMLicqBwpGmgicLQOC2Qhdt7DpgWlf0ChIVfTIQIfsXg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
>
> 
>
> - 在执行过程中先执行所有的`BeanDefinitionRegistryPostProcessor`的`postProcessorBeanDefinitionRegistry()`方法，然后再执行`BeanFacotryPostProcessor`的`postProcessorBeanFactory()`方法。
>
> ```
> 1public interface BeanFactoryPostProcessor {
> 2    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
> 3}
> 1public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
> 2    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
> 3}
> ```
>
> - 默认情况下，`Spring`有一个内置的`BeanFactoryPostProcessor`，即：`ConfigurationClassPostProcessor`类，该类实现了`BeanDefinitionRegistryPostProcessor`类，所以会执行`ConfigurationClassPostProcessor.postProcessorBeanDefinitionRegistry`,`ConfigurationClassPostProcessor`的`UML`图如上(删减了部分不重要的继承关系)
>
> ##### 6.2 registerBeanPostProcessors()
>
> - 该方法的作用是找到所有的`BeanPostProcessor`，然后将这些`BeanPostProcessor`实例化(会调用`getBean()`方法，`getBean()`方法的主要逻辑是，如果`bean`存在于`BeanFactory`中，则返回`bean`；如果不存在，则会去创建。在后面会仔细分析`getBean()`的执行逻辑)。将这些`PostProcessor`实例化后，最后放入到`BeanFactory`的`beanPostProcessors`属性中。
> - 问题：如何找到所有的`BeanPostProcessor`? 包括`Spring`内置的和开发人员自定义的。
> - 由于在`refresh()`方法中，会先执行完`invokeBeanFactoryPostProcessor()`方法，这样所有自定义的`BeanPostProcessor`类均已经被扫描出并解析成`BeanDefinition`(扫描和解析又是谁做的呢？`ConfigurationClassPostProcessor`做的)，存入至`BeanFactory`的`BeanDefinitionMap`，所以这儿能通过方法如下一行代码找出所有的`BeanPostProcessor`，然后通过`getBean()`全部实例化，最后再将实例化后的对象加入到`BeanFactory`的`beanPostProcessors`属性中，该属性是一个`List`集合。
>
> ```
> 1String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
> ```
>
> - 最后再重新注册了`ApplicationListenerDetector`，这样做的目的是为了将`ApplicationListenerDetector`放入到后置处理器的最末端
> - `registerBeanPostProcessor()` 最终调用的是`PostProcessorRegistrationDelegate.registerBeanPostProcessors()`，下面是`PostProcessorRegistrationDelegate.registerBeanPostProcessors()`方法的代码
>
> ```
>  1public static void registerBeanPostProcessors(
>  2        ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
>  3
>  4    // 从BeanDefinitionMap中找出所有的BeanPostProcessor
>  5    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
>  6    // ... 省略部分代码 ...
>  7
>  8    // 分别找出实现了PriorityOrdered、Ordered接口以及普通的BeanPostProcessor
>  9    for (String ppName : postProcessorNames) {
> 10        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
> 11            // 此处调用了getBean()方法，因此在此处就会实例化出BeanPostProcessor
> 12            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
> 13            priorityOrderedPostProcessors.add(pp);
> 14            if (pp instanceof MergedBeanDefinitionPostProcessor) {
> 15                internalPostProcessors.add(pp);
> 16            }
> 17        }
> 18        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
> 19            orderedPostProcessorNames.add(ppName);
> 20        }
> 21        else {
> 22            nonOrderedPostProcessorNames.add(ppName);
> 23        }
> 24    }
> 25
> 26    // First, register the BeanPostProcessors that implement PriorityOrdered.
> 27    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
> 28    // 将实现了PriorityOrdered接口的BeanPostProcessor添加到BeanFactory的beanPostProcessors集合中
> 29    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);
> 30
> 31    // 下面这部分代码与上面的代码逻辑一致，是将实现了Ordered接口以及普通的BeanPostProcessor实例化以及添加到beanPostProcessors结合中，逻辑与处理PriorityOrdered的后置处理器一样
> 32    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
> 33    for (String ppName : orderedPostProcessorNames) {
> 34        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
> 35        orderedPostProcessors.add(pp);
> 36        if (pp instanceof MergedBeanDefinitionPostProcessor) {
> 37            internalPostProcessors.add(pp);
> 38        }
> 39    }
> 40    sortPostProcessors(orderedPostProcessors, beanFactory);
> 41    registerBeanPostProcessors(beanFactory, orderedPostProcessors);
> 42
> 43    // Now, register all regular BeanPostProcessors.
> 44    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
> 45    for (String ppName : nonOrderedPostProcessorNames) {
> 46        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
> 47        nonOrderedPostProcessors.add(pp);
> 48        if (pp instanceof MergedBeanDefinitionPostProcessor) {
> 49            internalPostProcessors.add(pp);
> 50        }
> 51    }
> 52    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
> 53
> 54    // Finally, re-register all internal BeanPostProcessors.
> 55    sortPostProcessors(internalPostProcessors, beanFactory);
> 56    registerBeanPostProcessors(beanFactory, internalPostProcessors);
> 57
> 58    // Re-register post-processor for detecting inner beans as ApplicationListeners,
> 59    // moving it to the end of the processor chain (for picking up proxies etc).
> 60    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
> 61
> 62    // 最后将ApplicationListenerDetector这个后置处理器一样重新放入到beanPostProcessor中，这样做的目的是为了将其放入到后置处理器的最末端
> 63    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
> 64}
> ```
>
> - 从上面的源码中可以发现，`BeanPostProcessor`存在优先级，实现了`PriorityOrdered`接口的优先级最高，其次是`Ordered`接口，最后是普通的`BeanPostProcessor`。优先级最高的，会最先放入到`beanPostProcessors`这个集合的最前面，这样在执行时，会最先执行优先级最高的后置处理器(因为`List`集合是有序的)。
> - 这样在实际应用中，如果我们碰到需要优先让某个`BeanPostProcessor`执行，则可以让其实现`PriorityOrdered`接口或者`Ordered`接口。
>
> ##### 6.3 initMessageSource()
>
> - 用来支持消息国际化，现在一般项目中不会用到国际化相关的知识。
>
> ##### 6.4 initApplicationEventMulticaster()
>
> > 该方法初始化了一个事件广播器，如果容器中存在了`beanName`为`applicationEventMulticaster`的广播器，则使用该广播器；如果没有，则初始化一个`SimpleApplicationEventMulticaster`。该事件广播器是用来做应用事件分发的，这个类会持有所有的事件监听器(`ApplicationListener`)，当有`ApplicationEvent`事件发布时，该事件监听器能根据事件类型，检索到对该事件感兴趣的`ApplicationListener`。
>
> - `initApplicationEventMulticaster()`方法的源码如下(省略了部分日志信息):
>
> ```
>  1protected void initApplicationEventMulticaster() {
>  2    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
>  3    // 判断spring容器中是否已经存在beanName = applicationEventMulticaster的事件广播器
>  4    // 例如：如果开发人员自己注册了一个
>  5    // 如果存在，则使用已经存在的；否则使用spring默认的:SimpleApplicationEventMulticaster
>  6    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
>  7        this.applicationEventMulticaster =
>  8                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
>  9    }
> 10    else {
> 11        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
> 12        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
> 13    }
> 14}
> ```
>
> ##### 6.5 onRefresh()
>
> > 执行其他的初始化操作，例如和`SpringMVC`整合时，需要初始化一些其他的`bean`，但是对于纯`Spring`工程来说，`onRefresh()`方法是一个空方法。
>
> ##### 6.6 registerListeners()
>
> > 这一步会将自定义的`listener`的`bean`名称放入到事件广播器中,同时还会将早期的`ApplicationEvent`发布(对于单独的`Spring`工程来说，在此时不会有任何`ApplicationEvent`发布，但是和`SpringMVC`整合时，`SpringMVC`会执行`onRefresh()`方法，在这里会发布事件)。方法源码如下:
>
> ```
>  1protected void registerListeners() {
>  2    // Register statically specified listeners first.
>  3    for (ApplicationListener<?> listener : getApplicationListeners()) {
>  4  getApplicationEventMulticaster().addApplicationListener(listener);
>  5    }
>  6
>  7    // 从BeanFactory中找到所有的ApplicationListener，但是不会进行初始化，因为需要在后面bean实例化的过程中，让所有的BeanPostProcessor去改造它们
>  8    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
>  9    for (String listenerBeanName : listenerBeanNames) {
> 10        // 将事件监听器的beanName放入到事件广播器中
> 11        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
> 12    }
> 13
> 14    // 发布早期的事件(纯的spring工程，在此时一个事件都没有)
> 15    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
> 16    this.earlyApplicationEvents = null;
> 17    if (earlyEventsToProcess != null) {
> 18        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
> 19            getApplicationEventMulticaster().multicastEvent(earlyEvent);
> 20        }
> 21    }
> 22}
> ```
>
> ##### 6.7 finishBeanFactoryInitialization()
>
> > 该方法十分重要，它完成了所有非懒加载的单例`Bean`的实例化和初始化，属性的填充以及解决了循环依赖等问题。由于微信平台对文章字数有限制，因此关于`Bean`的创建过程移到了另外一篇文章中,点击后面链接查看。([通过源码看Bean的创建过程](https://mp.weixin.qq.com/s?__biz=MzI4Mjg2NjUzNw==&mid=2247483679&idx=1&sn=03a08c6844397a6d2610d89577eb8558&scene=21#wechat_redirect))
> >
> > 在`《通过源码看Bean的创建过程》`这边文章中通过源码分析了所有后置处理器的作用，`Bean`的生命周期，以及后置处理的应用场景。
>
> ##### 6.8 finishRefresh()
>
> - 执行到这一步，`Spring`容器的启动基本结束了，此时`Bean`已经被实例化完成，且完成了自动装配。执行`finishRefresh()`方法，是为了在容器`refresh()`结束时，做一些其他的操作，例如：发布`ContextRefreshedEvent`事件，这样当我们想在容器`refresh`完成后执行一些特殊的逻辑，就可以通过监听`ContextRefreshedEvent`事件来实现。`Spring`内置了四个和应用上下文(`ApplicationContextEvent`)有关的事件：`ContextRefreshedEvent`、`ContextStartedEvent`、`ContextStopedEvent`、`ContextClosedEvent`。
>
> ```
> 1protected void finishRefresh() {
> 2    clearResourceCaches();
> 3    initLifecycleProcessor();
> 4    getLifecycleProcessor().onRefresh();
> 5    // 发布ContextRefreshedEvent
> 6    publishEvent(new ContextRefreshedEvent(this));
> 7    LiveBeansView.registerApplicationContext(this˛);
> 8}
> ```
>
> ##### 6.9 resetCommonCaches()
>
> > 最后在`refresh()`方法的`finally`语句块中，执行了`resetCommonCaches()`方法。因为在前面创建`bean`时，对单例`bean`的元数据信息进行了缓存，而单例`bean`在容器启动后，不会再进行创建了，因此这些缓存的信息已经没有任何用处了，在这里进行清空，释放部分内存。
>
> ```
> 1protected void resetCommonCaches() {
> 2    ReflectionUtils.clearCache();
> 3    AnnotationUtils.clearCache();
> 4    ResolvableType.clearCache();
> 5    CachedIntrospectionResults.clearClassLoader(getClassLoader());
> 6}
> ```
>
> #### 7. 总结
>
> - 本文介绍了`Spring`的启动流程，通过`AnnotationConfigApplicationContext`的有参构造方法入手，重点分析了`this()`方法和`refresh()`方法。在`this()`中初始化了一个`BeanFactory`，即`DefaultListableBeanFactory`；然后向容器中添加了7个内置的`bean`，其中就包括`ConfigurationClassPostProcessor`。
> - 在`refresh()`方法中，又重点分析了`invokeBeanFactoryPostProcessor()`方法和`finishBeanFactoryInitialization()`方法。
> - 在`invokeBeanFactoryPostProcessor()`方法中，通过`ConfigurationClassPostProcessor`类扫描出了所有交给`Spring`管理的类，并将`class`文件解析成对应的`BeanDefinition`。
> - 在`finishBeanFactoryInitialization()`方法中，完成了非懒加载的单例`Bean`的实例化和初始化操作，主要流程为`getBean()` ——>`doGetBean()`——>`createBean()`——>`doCreateBean()`。在`bean`的创建过程中，一共出现了`8`次`BeanPostProcessor`的执行，在这些后置处理器的执行过程中，完成了`AOP`的实现、`bean`的自动装配、属性赋值等操作。
> - 最后通过一张流程图，总结了`Spring`中单例`Bean`的生命周期。
>
> #### 8. 计划
>
> > 本文主要介绍了`Spring`的启动流程，但对于一些地方的具体实现细节没有展开分析，因此后续Spring源码分析的计划如下:
>
> - `ConfigurationClassPostProcessor`类如何扫描包，解析配置类。
>
> - `@Import`注解作用与`@Enable`系列注解的实现原理
>
> - `JDK`动态代理与`CGLIB`代理
>
> - `FactoryBean`的用途和源码分析
>
> - `AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`如何实现自动装配,`Spring`如何解决循环依赖
>
> - `AOP`的实现原理
>
> - `SpringBoot`源码分析
>
> #### 10. 推荐性能监控工具
>
> - 最后推荐一款开源的性能监控工具——`Pepper-Metrics`
>
> > 地址：https://github.com/zrbcool/pepper-metrics
> > GitHub
> > `Pepper-Metrics`是坐我对面的两位同事一起开发的开源组件，主要功能是通过比较轻量的方式与常用开源组件（`jedis/mybatis/motan/dubbo/servlet`）集成，收集并计算`metrics`，并支持输出到日志及转换成多种时序数据库兼容数据格式，配套的`grafana dashboard`友好的进行展示。项目当中原理文档齐全，且全部基于`SPI`设计的可扩展式架构，方便的开发新插件。另有一个基于`docker-compose`的独立`demo`项目可以快速启动一套`demo`示例查看效果`https://github.com/zrbcool/pepper-metrics-demo`。如果大家觉得有用的话，麻烦给个`star`，也欢迎大家参与开发，谢谢：）