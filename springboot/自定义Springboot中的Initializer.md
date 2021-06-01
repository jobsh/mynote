# 自定义Springboot中的Initializer

## 实现一

1. 实现ApplicationContextInitializer接口

   ```java
   @Order(1)
   public class FirstInitializer implements ApplicationContextInitializer {
       @Override
       public void initialize(ConfigurableApplicationContext applicationContext) {
           ConfigurableEnvironment environment = applicationContext.getEnvironment();
           Map<String, Object> map = new HashMap();
           map.put("key1", "value1");
           MapPropertySource mapPropertySource = new MapPropertySource("firstInitializer", map);
           environment.getPropertySources().addLast(mapPropertySource);
           System.out.println("run firstInitializer");
       }
   }
   ```

2. spring.factories填写接口实现

```properties
org.springframework.context.ApplicationContextInitializer=com.imooc.initializer.FirstInitializer
```

## 实现二

1. 第一步相同：都是实现ApplicationContextInitializer接口

   ```java
   @Order(2)
   public class SecondInitializer implements ApplicationContextInitializer {
       @Override
       public void initialize(ConfigurableApplicationContext applicationContext) {
           ConfigurableEnvironment environment = applicationContext.getEnvironment();
           Map<String, Object> map = new HashMap();
           map.put("key2", "value2");
           MapPropertySource mapPropertySource = new MapPropertySource("secondInitializer", map);
           environment.getPropertySources().addLast(mapPropertySource);
           System.out.println("run secondInitializer");
       }
   }
   ```

2. 在springApplication初始化时设置进去

   ```java
   @SpringBootApplication
   public class Application {
       public static void main(String[] args) {
   //        SpringApplication.run(Application.class);
           new SpringApplicationBuilder(Application.class)
                   .web(WebApplicationType.SERVLET)
               	// 设置initializer
                   .initializers(new SecondInitializer())
                   .run(args);
       }
   }
   ```

## 实现三

1. 实现ApplicationContextInitializer接口

   ```java
   @Order(3)
   public class ThirdInitializer implements ApplicationContextInitializer {
       @Override
       public void initialize(ConfigurableApplicationContext applicationContext) {
           ConfigurableEnvironment environment = applicationContext.getEnvironment();
           Map<String, Object> map = new HashMap();
           map.put("key3", "value3");
           MapPropertySource mapPropertySource = new MapPropertySource("thirdInitializer", map);
           environment.getPropertySources().addLast(mapPropertySource);
           System.out.println("run thirdInitializer");
       }
   }
   ```

2. application.yml中填写实现类

   ```yml
   context:
     initializer:
       classes: com.imooc.initializer.ThirdInitializer
   ```

**思考：**为什么ThirdInitializer明明时@Order(3)，这里确实第一个执行呢？

![1597567661700](C:\Users\HP\AppData\Roaming\Typora\typora-user-images\1597567661700.png)

## service层

```java
@Service
public class TestService implements ApplicationContextAware{

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    public String test1() {
        return applicationContext.getEnvironment().getProperty("key1");
    }

    public String test2() {
        return applicationContext.getEnvironment().getProperty("key2");
    }

    public String test3() {
        return applicationContext.getEnvironment().getProperty("key3");
    }
}
```

## controller测试

```java
package com.imooc.controller;

import com.imooc.service.TestService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @desc:
 * @author: Mr.Han
 */
@RestController
@RequestMapping("demo")
public class DemoController {

    @Autowired
    private TestService testService;

    @GetMapping("test1")
    public String test1() {
        return testService.test1();// 打印出value1 
    }

    @GetMapping("test2")
    public String test2() {
        return testService.test2();// 打印出value2       
    }

    @GetMapping("test3")
    public String test3() {
        return testService.test3();   // 打印出value3
    }

}
```

