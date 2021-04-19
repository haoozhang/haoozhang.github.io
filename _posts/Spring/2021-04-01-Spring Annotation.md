---
layout:     post
title:      5. Spring Annotation
subtitle:   
date:       2021-04-01
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring
---

在 Spring4 之后，想要使用注解，必须要引入 aop 的 jar 包\
在配置文件中，还要引入 context 约束

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

</beans>
```

## Bean

我们之前都是使用 bean 的标签进行bean注入，但是实际开发中，我们一般都会使用注解！

1、配置扫描哪些包下的注解
```xml
<!--指定注解扫描包-->
<context:component-scan base-package="com.kuang.pojo"/>
```
2、在指定包下编写类，增加注解
```java
@Component("user")
// 相当于配置文件中 <bean id="user" class="当前注解的类"/>
public class User {
   public String name = "XiaoMing";
}
```
3、测试
```java
@Test
public void test(){
   ApplicationContext applicationContext =
       new ClassPathXmlApplicationContext("beans.xml");
   User user = (User) applicationContext.getBean("user");
   System.out.println(user.name);
}
```

## 属性注入

使用注解注入属性

1、可以不用提供set方法，直接在直接名上添加@value
```java
@Component("user")
// 相当于配置文件中 <bean id="user" class="当前注解的类"/>
public class User {
   @Value("秦疆")
   // 相当于配置文件中 <property name="name" value="秦疆"/>
   public String name;
}
```
2、如果提供了set方法，在set方法上添加@value
```java
@Component("user")
public class User {

   public String name;

   @Value("秦疆")
   public void setName(String name) {
       this.name = name;
  }
}
```

## 衍生注解

@Component三个衍生注解

为了更好的进行分层，Spring可以使用其它三个注解，功能一样，目前使用哪一个功能都一样。

@Controller：web层

@Service：service层

@Repository：dao层

写上这些注解，就相当于将这个类交给Spring管理装配了！

## 自动装配

@Autowired 在上一篇已经讲过

## 作用域

@Scope：Singleton 和 Prototype 也已讲过。

```java
@Controller("user")
@Scope("prototype")
public class User {
   @Value("秦疆")
   public String name;
}
```

## XML 与注解

XML与注解比较：

+ XML可以适用任何场景 ，结构清晰，维护方便

+ 注解不是自己提供的类使用不了，开发简单方便

xml与注解整合开发 ：推荐最佳实践

+ xml管理Bean

+ 注解完成属性注入

+ 使用过程中， 可以不用扫描，扫描是为了类上的注解

## 基于 Java Config 进行配置

JavaConfig 原来是 Spring 的一个子项目，它通过 Java 类的方式提供 Bean 的定义信息，在 Spring4 的版本， JavaConfig 已正式成为 Spring4 的核心功能 。

测试：

1、编写一个实体类，Dog
```java
public class User {
    public String name = "zhao";
}
```
2、新建一个config包，编写一个Config配置类

```java
@Configuration // 表示这是一个配置类
public class Config {
    @Bean // 通过方法注册 bean，返回值为bean类型，方法名为bean 的 id
    public User user() {
        return new User();
    }
}
```

3、测试

```java
@Test
public void test() {
    ApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
    User user = context.getBean("user", User.class);
    System.out.println(user.name);
}
```

4、成功输出结果！

### 导入其他配置如何做呢？

1、我们再编写一个配置类！
```java
@Configuration  //代表这是一个配置类
public class MyConfig2 {
}
```
2、在之前的配置类中我们来选择导入这个配置类

```java
@Configuration // 表示这是一个配置类
@Import(MyConfig2.class)  //导入合并其他配置类，类似于配置文件中的 inculde 标签
public class Config {
    @Bean // 通过方法注册 bean，返回值为bean类型，方法名为bean 的 id
    public User user() {
        return new User();
    }
}
```

## 总结

本篇我们学习了 Spring 的注解，可以通过注解来注册 bean 和注入属性。另外，我们学习了通过 Java Config 完全不用配置文件来注册 bean。

参考自：
1. [狂神说Spring05：使用注解开发](https://mp.weixin.qq.com/s/dCeQwaQ-A97FiUxs7INlHw)
