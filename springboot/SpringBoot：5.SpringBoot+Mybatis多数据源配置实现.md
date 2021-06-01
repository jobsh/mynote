# SpringBoot：5.SpringBoot+Mybatis多数据源配置实现

在做架构中，有的时候会遇到下面两种情况：

- 读写分离，主库负责写入，从库负责读取。
- 因为数据量较大，需要在主库中存放平台主要表结构，将会大量产生的数据按日期分表放到从库中。（比如我们公司做车载GPS的，GPS数据量就比较大，所以就把GPS信息以及相关的报警信息按日期分表放入到从库中）

对于这两种情况，就需要在项目中加入多数据源，以便操作不同的数据库。而在实际开发中，一般会根据实际情况选择数据源的管理方式：

- 在项目中集成多数据源，实现数据源的切换。
- 通过数据库中间件，例如mycat、cobar等，通过一定的规则来让指定的语句到指定的数据库中执行。

考虑到公司项目每天产生的GPS数据量并不是很大，一般一天在1000万条数据，只要在从库中按日期分表，每天生产一张表用于存放GPS信息，所以选择了在项目中集成多数据源的方式。

Spring Boot+Mybatis多数据源实现的方式

### 1.引入依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.9.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.w3cjava</groupId>
	<artifactId>05.Spring-Boot-Mul-Mybatis</artifactId>
	<version>0.1</version>
	<name>05.Spring-Boot-Mul-Mybatis</name>
	<description>Mul-Mybatis project for Spring Boot</description>

	<properties>
		<java.version>1.8</java.version>
		<maven-jar-plugin.version>3.0.0</maven-jar-plugin.version>
	</properties>

    <dependencies>
        <!-- springboot核心包-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>    
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
        <!-- springboot-aop包,AOP切面注解,Aspectd等相关注解 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>		    
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.1.1</version>
		</dependency>
        <!-- jdbcTemple  -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>		
        <!-- mysql数据库连接包 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.2</version>
        </dependency>
		<!-- 开发测试环境修改文件实时生效包,生产默认不使用 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-configuration-processor</artifactId>
			<optional>true</optional>
		</dependency>
    </dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```

### 2.动态数据源配置

#### 2.1 @DataBaseSource

用于在Service指定主从库

```java
package com.w3cjava.common.annotation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface DataBaseSource {
    String value() default "master";
}
```

#### 2.2 DataSourceContextHolder

数据源获取与设置容器

```java
package com.w3cjava.common.datasource;

public class DataSourceContextHolder {
	/**
     * 默认数据源
     */
    public static final String DEFAULT_DS = "master";
 
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();
 
    // 设置数据源名
    public static void setDB(String dbType) {
        System.out.println("切换到{"+dbType+"}数据源");
        contextHolder.set(dbType);
    }
 
    // 获取数据源名
    public static String getDb() {
        return (contextHolder.get());
    }
 
    // 清除数据源名
    public static void clearDB() {
        contextHolder.remove();
    }
}
```

#### 2.3 DynamicDataSource

动态源获取

```java
package com.w3cjava.common.datasource;


import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;

public class DynamicDataSource extends AbstractRoutingDataSource {
	@Override
	protected Object determineCurrentLookupKey() {
		System.out.println("数据源为" + DataSourceContextHolder.getDb());
		return DataSourceContextHolder.getDb();
	}
}
```

#### 2.4 动态数据源配置DataSourceConfig

通过AOP在不同数据源之间动态切换

```java
package com.w3cjava.common.config;

import java.util.HashMap;
import java.util.Map;

import javax.sql.DataSource;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;
import org.springframework.transaction.PlatformTransactionManager;

import com.w3cjava.common.datasource.DynamicDataSource;

@Configuration
public class DataSourceConfig {
	public Logger logger = LoggerFactory.getLogger(this.getClass());
	// 数据源master
	@Bean(name = "master")
	@ConfigurationProperties(prefix = "spring.datasource.master") // application.properteis中对应属性的前缀
	public DataSource masterDataSource() {
		return DataSourceBuilder.create().build();
	}

	// 数据源slave
	@Bean(name = "slave")
	@ConfigurationProperties(prefix = "spring.datasource.slave") // application.properteis中对应属性的前缀
	public DataSource slaveDataSource() {
		return DataSourceBuilder.create().build();
	}

	/**
	 * 动态数据源: 通过AOP在不同数据源之间动态切换
	 * 
	 * @return
	 */
	@Primary
	@Bean(name = "dynamicDataSource")
	public DataSource dynamicDataSource() {
		DynamicDataSource dynamicDataSource = new DynamicDataSource();
		// 默认数据源
		dynamicDataSource.setDefaultTargetDataSource(masterDataSource());
		// 配置多数据源
		Map<Object, Object> dsMap = new HashMap<Object, Object>();
		dsMap.put("master", masterDataSource());
		dsMap.put("slave", slaveDataSource());

		dynamicDataSource.setTargetDataSources(dsMap);
		return dynamicDataSource;
	}

	/**
	 * 配置@Transactional注解事物
	 * 
	 * @return
	 */
	@Bean
	public PlatformTransactionManager transactionManager() {
		return new DataSourceTransactionManager(dynamicDataSource());
	}
	
}
```

#### 2.5 DataSourceExchange切面

配置数据源改变时的切面

```java
package com.w3cjava.common.datasource;

import java.lang.reflect.Method;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.AfterReturning;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import com.w3cjava.common.annotation.DataBaseSource;

@Aspect
@Component
public class DataSourceExchange{
	public Logger logger = LoggerFactory.getLogger(this.getClass());
    @Pointcut(value="execution(* com.w3cjava.modules.*.dao.*.*(..))")
    public void dbPointCut() {

    }
	/**
	 * 方法结束后
	 */
	@AfterReturning(value="execution(* com.w3cjava.modules.*.dao.*.*(..)) ")
	public void afterReturning(JoinPoint point){
		logger.info("当前1数据源："+DataSourceContextHolder.getDb());
		DataSourceContextHolder.clearDB();
		logger.info("数据源已移除！");
		logger.info("当前2数据源："+DataSourceContextHolder.getDb());
	}
	
	/**
	 * 拦截目标方法，获取由@DataSource指定的数据源标识，设置到线程存储中以便切换数据源
	 */
	
	@SuppressWarnings("rawtypes")
	@Before(value="execution(* com.w3cjava.modules.*.dao.*.*(..))")
	public void before(JoinPoint point){
        //获得当前访问的class
        Class<?> className = point.getTarget().getClass();
        //获得访问的方法名
        String methodName = point.getSignature().getName();
        //得到方法的参数的类型
        Class[] argClass = ((MethodSignature)point.getSignature()).getParameterTypes();
        
		try {
			Method method = className.getMethod(methodName, argClass);
			DataBaseSource dataSource = AnnotationUtils.findAnnotation(method, DataBaseSource.class);
			if(dataSource!=null) {
				DataSourceContextHolder.setDB(dataSource.value());
			}else {
				DataSourceContextHolder.setDB(DataSourceContextHolder.DEFAULT_DS);
			}
			logger.info("数据源切换至："+DataSourceContextHolder.getDb());
		} catch (Exception e) {
			e.printStackTrace();
		}
		
	}
}
```

### 3. 测试业务模型

#### 3.1 User实体

```java
package com.w3cjava.modules.user.entity;

public class User{
	private String id;
	private String name;
	private Integer age;
	public String getId() {
		return id;
	}
	public void setId(String id) {
		this.id = id;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public Integer getAge() {
		return age;
	}
	public void setAge(Integer age) {
		this.age = age;
	}
	@Override
	public String toString() {
		return "User [id=" + id + ", name=" + name + ", age=" + age + "]";
	}
	
	
}

```

#### 3.2 UserDao层及xml

```java
package com.w3cjava.modules.user.dao;

import java.util.List;

import org.apache.ibatis.annotations.Mapper;

import com.w3cjava.common.annotation.DataBaseSource;
import com.w3cjava.modules.user.entity.User;

@Mapper
public interface UserDao{
	//使用xml配置形式查询
	@DataBaseSource("master")
	public int insertMaster(User entity);
	@DataBaseSource("slave")
	public int insertSlave(User entity);
	
	
	
	@DataBaseSource("slave")
    public List<User> getSlaveAllUser();
	@DataBaseSource("master")
    public List<User> getMasterAllUser();
}
```

xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.w3cjava.modules.user.dao.UserDao">
	<sql id="testColumns">
		a.id AS "id",
		a.name AS "name",
		a.age AS "age"
	</sql>
	
	<sql id="testJoins">
	</sql>
	<!-- 查询所有user -->
     <select id="getSlaveAllUser" resultType="com.w3cjava.modules.user.entity.User">
            select 
				<include refid="testColumns"/>
			 from user a
     </select>
     <select id="getMasterAllUser" resultType="com.w3cjava.modules.user.entity.User">
            select 
            	<include refid="testColumns"/>  
            from user a
     </select>       
	<insert id="insertMaster">
		INSERT INTO user(
			id,
			name,
			age
		) VALUES (
			#{id},
			#{name},
			#{age}
		)
	</insert>
	<insert id="insertSlave">
		INSERT INTO user(
			id,
			name,
			age
		) VALUES (
			#{id},
			#{name},
			#{age}
		)
	</insert>	
</mapper>
```

#### 3.3 UserService层

```java
package com.w3cjava.modules.user.service;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.w3cjava.modules.user.dao.UserDao;
import com.w3cjava.modules.user.entity.User;
@Service
public class UserService{
	@Autowired
    private UserDao userDao;
    //使用数据源master查询
	//@Transactional(readOnly=true)
    public List<User> getAllUserMaster(){
        return userDao.getMasterAllUser();
    }
    //使用数据源slave查询
	//@Transactional(readOnly=true)
    public List<User> getAllUserSlave(){
        return userDao.getSlaveAllUser();
    }
 
    
    
	@Transactional(readOnly=true)
    public String getAllUserTest(){
		List<User> list1 = userDao.getMasterAllUser();
		List<User> list2 = userDao.getSlaveAllUser();
        return "master:"+list1+"</br>slave:"+list2;
    }	
	
    //使用数据源master插入数据
    //@Transactional(readOnly=false)
    public int saveMaster(User user){
    	int m = userDao.insertMaster(user);
        return m;
    }
    
    //使用数据源slave插入数据
    //@Transactional(readOnly=false)
    public int saveSlave(User user){
    	int m = userDao.insertSlave(user);
        return m;
    }   
}
```

#### 3.4 UserController层

```java
package com.w3cjava.modules.user.controller;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import com.w3cjava.common.utils.IdGen;
import com.w3cjava.modules.user.entity.User;
import com.w3cjava.modules.user.service.UserService;

@RestController
@RequestMapping("/user")
public class UserController {
	@Autowired
    private UserService userService;
	//不使用事务注解@Transactional
    @RequestMapping(value = "/getDb1AllUser")
    public String getDbAllUser() {
        List<User> list1 = userService.getAllUserMaster();
        for (int i = 0; i < list1.size(); i++) {
        	System.out.println(list1.get(i).getId()+"-"+list1.get(i).getName()+"-"+list1.get(i).getAge());
		}
        List<User> list2 = userService.getAllUserSlave();
        for (int i = 0; i < list2.size(); i++) {
        	System.out.println(list2.get(i).getId()+"-"+list2.get(i).getName()+"-"+list2.get(i).getAge());
		}       
        return "master:"+list1+"</br>slave:"+list2;
    }

    
    
    //使用事务注解@Transactional
    @RequestMapping(value = "/getDbAllUserTest")
    public String getDbAllUserTest() {
        String list = userService.getAllUserTest();
        return list;
    }
    //主库master user信息
    @RequestMapping(value = "/getAllUserMaster")
    public String getAllUserMaster() {
        List<User> list = userService.getAllUserMaster();
        return "master:"+list;
    } 
    
    //从库slave user信息
    @RequestMapping(value = "/getAllUserSlave")
    public String getAllUserSlave() {
        List<User> list = userService.getAllUserSlave();
        return "slave:"+list;
    }
 
    @SuppressWarnings("unused")
	@RequestMapping(value = "/saveMaster")
    public String saveMaster() {
        User user = new User();
        user.setId(IdGen.uuid());
        user.setName("MasterTom");
        user.setAge(20);
        Integer rows = userService.saveMaster(user);//返回的是结果行数
        return "{id:"+user.getId()+"}";
    }
    
    
    
    @SuppressWarnings("unused")
	@RequestMapping(value = "/saveSlave")
    public String saveSlave() {
        User user = new User();
        user.setId(IdGen.uuid());
        user.setName("SlaveTom");
        user.setAge(20);
        Integer rows = userService.saveSlave(user);//返回的是结果行数
        return "{id:"+user.getId()+"}";
    }
}

```

user表id采用32位字符串形式，所以需要一个生成ID的工具类IdGen

```java
package com.w3cjava.common.utils;
import java.util.UUID;
/**
 * 
 * @author	w3cjava
 * @date	2018年8月29日
 * @desc	封装各种生成唯一性ID算法的工具类.
 */
public class IdGen{
	/**
	 * 封装JDK自带的UUID, 通过Random数字生成, 中间无-分割.
	 */
	public static String uuid() {
		return UUID.randomUUID().toString().replaceAll("-", "");
	}

}

```

### 4.启动类

```java
package com.w3cjava;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration;
/**
 * 
 * @class  SpringBootMulMybatisApplication
 * @version SpringBoot 2.1.9
 * @author cos
 * @desc   整合Mybatis实现多数据源配置
 *
 */
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class SpringBootMulMybatisApplication {
	public static void main(String[] args) {
		SpringApplication.run(SpringBootMulMybatisApplication.class, args);
		
	}

}
```

配置文件application.properties

```properties
server.port=10001
#springboot\u591A\u6570\u636E\u6E90\u914D\u7F6E
#\u6570\u636E\u6E901
spring.datasource.master.jdbc-url=jdbc:mysql://127.0.0.1:3306/test1?useUnicode=true&characterEncoding=utf-8&useSSL=false
spring.datasource.master.username=root
spring.datasource.master.password=123456
spring.datasource.master.driver-Class-Name=com.mysql.jdbc.Driver
spring.datasource.master.max-idle=10
spring.datasource.master.max-wait=10000
spring.datasource.master.min-idle=5
spring.datasource.master.initial-size=5
#\u6570\u636E\u6E902
spring.datasource.slave.jdbc-url=jdbc:mysql://127.0.0.1:3306/test2?useUnicode=true&characterEncoding=utf-8&useSSL=false
spring.datasource.slave.username=root
spring.datasource.slave.password=123456
spring.datasource.slave.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.slave.max-idle=10
spring.datasource.slave.max-wait=10000
spring.datasource.slave.min-idle=5
spring.datasource.slave.initial-size=5
#mybatis
mybatis.mapper-locations=classpath*:mapper/*.xml
```

数据库SQL，库test1和test2均使用如下sql创建表

```mysql
SET FOREIGN_KEY_CHECKS=0;

-- ----------------------------
-- Table structure for `user`
-- ----------------------------
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user` (
  `id` varchar(32) NOT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

### 5. 启动程序测试

通过SpringBootMulMybatisApplication run运行启动程序。

分别访问如下请求查看效果

<http://localhost:10001/user/saveMaster> 主库插入

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/4/9/1715ddf8c2652257?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

http://localhost:10001/user/saveSlave

 

从库插入

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/4/9/1715ddf9a771b5b6?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

http://localhost:10001/user/getAllUserMaster

 

获取主库数据

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/4/9/1715ddf8c2e861b2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

http://localhost:10001/user//getAllUserSlave

 

获取从库数据

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/4/9/1715ddf8c2d32b4a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

http://localhost:10001/user/getDb1AllUser

 

获取主从库数据，无事务

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/4/9/1715ddf8ddb62e6d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

http://localhost:10001/user/getDbAllUserTest

 

获取主从库数据，有事务，失败

![在这里插入图片描述](https://user-gold-cdn.xitu.io/2020/4/9/1715ddf8c34bdd0e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

备注：在获取主从库数据时，如果增加了事务@Transactional时导致获取的数据错误，经测试一般先查master库，slave库一般就查不到，先查slave库，master库查不到，通过切面切换主从库出现的异常还不知道如何解决。有人知道如何解决的可以提供下。