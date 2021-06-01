# 你真的懂Spring Java Config 吗？Full @Configuration vs lite @Bean mode



`Full @Configuration`和`lite @Bean mode` 是 Spring Java Config 中两个非常有意思的概念。

先来看一下官方文档关于这两者的相关内容：

> The @Bean methods in a regular Spring component are processed differently than their counterparts inside a Spring @Configuration class. The difference is that @Component classes are not enhanced with CGLIB to intercept the invocation of methods and fields. CGLIB proxying is the means by which invoking methods or fields within @Bean methods in @Configuration classes creates bean metadata references to collaborating objects. Such methods are not invoked with normal Java semantics but rather go through the container in order to provide the usual lifecycle management and proxying of Spring beans, even when referring to other beans through programmatic calls to @Bean methods. In contrast, invoking a method or field in a @Bean method within a plain @Component class has standard Java semantics, with no special CGLIB processing or other constraints applying.

> 机器翻译结果：常规Spring组件中的@Bean方法与Spring @Configuration类中的对应方法处理方式不同。不同之处在于，@Component类没有使用CGLIB增强来拦截方法和字段的调用。CGLIB代理是通过调用@Configuration类中的@Bean方法中的方法或字段来创建对协作对象的bean元数据引用的方法。这些方法不是用普通的Java语义调用的，而是通过容器来提供Spring bean通常的生命周期管理和代理，即使在通过对@Bean方法的编程调用引用其他bean时也是如此。相反，在普通@Component类中调用@Bean方法中的方法或字段具有标准的Java语义，不需要使用特殊的CGLIB处理或其他约束。

> When @Bean methods are declared within classes that are not annotated with @Configuration, they are referred to as being processed in a “lite” mode. Bean methods declared in a @Component or even in a plain old class are considered to be “lite”, with a different primary purpose of the containing class and a @Bean method being a sort of bonus there. For example, service components may expose management views to the container through an additional @Bean method on each applicable component class. In such scenarios, @Bean methods are a general-purpose factory method mechanism. Unlike full @Configuration, lite @Bean methods cannot declare inter-bean dependencies. Instead, they operate on their containing component’s internal state and, optionally, on arguments that they may declare. Such a @Bean method should therefore not invoke other @Bean methods. Each such method is literally only a factory method for a particular bean reference, without any special runtime semantics. The positive side-effect here is that no CGLIB subclassing has to be applied at runtime, so there are no limitations in terms of class design (that is, the containing class may be final and so forth). In common scenarios, @Bean methods are to be declared within @Configuration classes, ensuring that “full” mode is always used and that cross-method references therefore get redirected to the container’s lifecycle management. This prevents the same @Bean method from accidentally being invoked through a regular Java call, which helps to reduce subtle bugs that can be hard to track down when operating in “lite” mode.

> 机器翻译：当@Bean方法在没有使用@Configuration注释的类中声明时，它们被称为在“lite”模式下处理。在@Component中声明的Bean方法，甚至在普通的旧类中声明的Bean方法，都被认为是“lite”，包含类的主要目的不同，而@Bean方法在这里是一种附加功能。例如，服务组件可以通过每个适用组件类上的附加@Bean方法向容器公开管理视图。在这种情况下，@Bean方法是一种通用的工厂方法机制。与完整的@Configuration不同，lite @Bean方法不能声明bean之间的依赖关系。相反，它们对包含它们的组件的内部状态进行操作，并可选地对它们可能声明的参数进行操作。因此，这样的@Bean方法不应该调用其他@Bean方法。每个这样的方法实际上只是一个特定bean引用的工厂方法，没有任何特殊的运行时语义。这里的积极副作用是在运行时不需要应用CGLIB子类，所以在类设计方面没有限制(也就是说，包含的类可能是final类，等等)。在常见的场景中，@Bean方法将在@Configuration类中声明，以确保始终使用“full”模式，并因此将交叉方法引用重定向到容器的生命周期管理。这可以防止通过常规Java调用意外调用相同的@Bean方法，这有助于减少在“lite”模式下操作时难以跟踪的细微bug。

------

## 相关定义

- `lite @Bean mode` ：当`@Bean`方法在没有使用`@Configuration`注解的类中声明时称之为`lite @Bean mode`
- `Full @Configuration`：如果`@Bean`方法在使用`@Configuration`注解的类中声明时称之为`Full @Configuration`

## 有何区别

总结一句话，`Full @Configuration`中的`@Bean`方法会被CGLIB所代理，而 `lite @Bean mode`中的`@Bean`方法不会被CGLIB代理。

## 举个例子

有一普通Java类，现在分别使用`Full @Configuration`和 `lite @Bean mode`这两种模式注册到Spring容器中。

```java
public class User {
    public User() {
        System.out.println("user create... hashCode :" + this.hashCode());
    }
}
```

### Full @Configuration

配置类：

```java
@Configuration
public class AppConfig {

    @Bean
    public User user() {
        return new User();
    }

    @Bean
    public String name(User user) {
        System.out.println(user.hashCode());
        System.out.println(user().hashCode());
        return "123";
    }
}
```

主程序：

```java
public class FullMain {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
        User user = context.getBean(User.class);
        System.out.println(user.hashCode());
        context.close();
    }
}
```

运行结果：

```java
user create... hashCode :1709537756
1709537756
1709537756
1709537756
```

分析：`@Bean`是在使用了`@Configuration`注解的类上被声明的，属于`Full @Configuration`。`@Configuration`类中的`@Bean`地方会被CGLIB进行代理。Spring会拦截该方法的执行，在默认单例情况下，容器中只有一个Bean，所以我们多次调用`user()`方法，获取的都是同一个对象。

**注意：被CGLIB的方法是不能被声明为private和final，因为CGLIB是通过生成子类来实现代理的，private和final方法是不能被子类Override的，也就是说，Full @Configuration模式下，@Bean的方法是不能不能被声明为private和final，不然在启动时Spring会直接报错。**

### lite @Bean mode

配置类：

```java
@Component
public class LiteAppConfig {

    @Bean
    public User user() {
        System.out.println("user() 方法执行");
        return new User();
    }

    @Bean
    public String name(User user) {
        System.out.println("name(User user) 方法执行");
        System.out.println(user.hashCode());
        System.out.println("再次调用user()方法: " + user().hashCode());
        return "123";
    }
}
```

主程序：

```java
public class LiteMain {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(LiteAppConfig.class);
        User user = context.getBean(User.class);
        System.out.println(user.hashCode());
        context.close();
    }
}
```

运行结果：

```java
user() 方法执行
user create... hashCode :254413710
name(User user) 方法执行
254413710
user() 方法执行
user create... hashCode :1793329556
再次调用user()方法: 1793329556
254413710
```

分析：在`lite @Bean mode`模式下， `@Bean`方法不会被CGLIB代理，所以多次调用`user()`会生成多个user对象。