---
layout:     post
title:      7. Spring AOP
subtitle:   
date:       2021-04-03
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring
---

## What is AOP

AOP (Aspect Oriented Programming) 译为：面向切面编程，该技术通过预编译方式和运行期间动态代理来实现程序功能的统一维护。AOP 是 OOP 的延续，是软件开发中的一个热点，也是 Spring 框架中的一个重要内容，是函数式编程的一种衍生范型。利用 AOP 可以对业务逻辑的各个部分进行隔离，从而使业务逻辑各部分之间耦合度降低，提高程序的可重用性，同时提高了开发的效率。

![img](/img/post/Spring/aop.png)

## AOP 在 Spring 中的作用

允许用户自定义切面；提供声明式事务；

以下名词需要了解下：

+ 横切关注点：跨越应用多个模块的方法或功能，即与业务逻辑无关，但需要关注的部分，就是横切关注点。如日志、安全、缓存、事务等；

+ 切面 (Aspect)：横切关注点被模块化的对象，即它是一个类；

+ 通知 (Advice)：切面必须要完成的工作。即类的一个方法；

+ 目标 (Target)：被通知对象，即真实角色；

+ 代理 (Proxy)：向目标对象应用通知之后创建的对象，即代理角色。

+ 切入点 (PointCut)：切面通知执行的位置。

+ 连接点 (JointPoint)：与切入点匹配的执行点。

![img](/img/post/Spring/aopConcept.png)

Spring AOP 通过 Advice 定义横切逻辑，支持5种类型的Advice。即 AOP 在不改变原有代码的情况下，去增加新的功能。

![img](/img/post/Spring/advice.png)

## 使用 Spring 实现 AOP

首先需要导入包

```xml
<!-- https://mvnrepository.com/artifact/org.aspectj/aspectjweaver -->
<dependency>
   <groupId>org.aspectj</groupId>
   <artifactId>aspectjweaver</artifactId>
   <version>1.9.4</version>
</dependency>
```

### 通过 Spring API 实现

1、编写我们的业务接口和实现类

```java
public interface UserService {

   public void add();

   public void delete();

   public void update();

   public void search();

}
public class UserServiceImpl implements UserService{

   @Override
   public void add() {
       System.out.println("增加用户");
  }

   @Override
   public void delete() {
       System.out.println("删除用户");
  }

   @Override
   public void update() {
       System.out.println("更新用户");
  }

   @Override
   public void search() {
       System.out.println("查询用户");
  }
}
```

2、写增强类，这里我们写两个，一个前置增强，一个后置增强

```java
public class BeforeLog implements MethodBeforeAdvice {

   // method : 要执行的目标对象的方法
   // objects : 被调用的方法的参数
   // Object : 目标对象
   @Override
   public void before(Method method, Object[] objects, Object o) throws Throwable {
       System.out.println( o.getClass().getName() + "的" + method.getName() + "方法被执行了");
  }
}

public class AfterLog implements AfterReturningAdvice {
   // returnValue 返回值
   // method被调用的方法
   // args 被调用的方法的对象的参数
   // target 被调用的目标对象
   @Override
   public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
       System.out.println("执行了" + target.getClass().getName()
       +"的"+method.getName()+"方法,"
       +"返回值："+returnValue);
  }
}
```

3、最后去 Spring 的配置文件中注册，并实现 AOP 切入，注意导入约束

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:aop="http://www.springframework.org/schema/aop"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/aop
       http://www.springframework.org/schema/aop/spring-aop.xsd">

   <!-- 注册 bean -->
   <bean id="userService" class="com.kuang.service.UserServiceImpl"/>
   <bean id="beforelog" class="com.kuang.log.BeforeLog"/>
   <bean id="afterLog" class="com.kuang.log.AfterLog"/>

   <!-- aop的配置 -->
   <aop:config>
       <!-- 切入点 expression:表达式匹配要执行的方法 -->
       <aop:pointcut id="pointcut" expression="execution(* com.kuang.service.UserServiceImpl.*(..))"/>
       <!--执行环绕; advice-ref执行方法，pointcut-ref切入点-->
       <aop:advisor advice-ref="log" pointcut-ref="pointcut"/>
       <aop:advisor advice-ref="afterLog" pointcut-ref="pointcut"/>
   </aop:config>

</beans>
```

4、测试

```java
public class MyTest {
   @Test
   public void test(){
       ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
       UserService userService = (UserService) context.getBean("userService");
       userService.search();
  }
}
```
Spring AOP 是将公共业务 (日志，安全等) 和领域业务结合起来，当执行领域业务时，公共业务横切进来，实现公共业务的重复利用，开发人员关注更纯粹的领域业务。其本质还是动态代理。

### 通过自定义类实现

1、业务接口和实现类不变

2、便携式自己的切入类

```java
public class DiyPointcut {

   public void before(){
       System.out.println("---------方法执行前---------");
  }
   public void after(){
       System.out.println("---------方法执行后---------");
  }
   
}
```

3、配置文件中注册

```xml
<!--注册bean-->
<bean id="diy" class="com.kuang.config.DiyPointcut"/>

<!--aop的配置-->
<aop:config>
   <!--第二种方式：使用AOP的标签实现-->
   <aop:aspect ref="diy">
       <aop:pointcut id="diyPonitcut" expression="execution(* com.kuang.service.UserServiceImpl.*(..))"/>
       <aop:before pointcut-ref="diyPonitcut" method="before"/>
       <aop:after pointcut-ref="diyPonitcut" method="after"/>
   </aop:aspect>
</aop:config>
```

4、测试

```java
public class MyTest {
   @Test
   public void test(){
       ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
       UserService userService = (UserService) context.getBean("userService");
       userService.add();
  }
}
```

### 通过注解实现

1、编写一个注解实现的增强类

```java
@Aspect
public class AnnotationPointcut {

   @Before("execution(* com.kuang.service.UserServiceImpl.*(..))")
   public void before(){
       System.out.println("---------方法执行前---------");
  }

   @After("execution(* com.kuang.service.UserServiceImpl.*(..))")
   public void after(){
       System.out.println("---------方法执行后---------");
  }

   @Around("execution(* com.kuang.service.UserServiceImpl.*(..))")
   public void around(ProceedingJoinPoint jp) throws Throwable {
       System.out.println("环绕前");
       System.out.println("签名:"+jp.getSignature());
       //执行目标方法proceed
       Object proceed = jp.proceed();
       System.out.println("环绕后");
       System.out.println(proceed);
  }
}
```

2、配置文件中注册 bean，并增加支持注解的配置

```xml
<bean id="annotationPointcut" class="com.kuang.config.AnnotationPointcut"/>
<aop:aspectj-autoproxy/>
```

aop:aspectj-autoproxy 声明自动为 Spring 容器中配置 @aspectJ 切面的 bean 创建代理，插入切面。有一个 proxy-target-class 属性，默认为 false，表示使用jdk动态代理插入切面。当配为 true 时，表示使用CGLib动态代理技术织入增强。

```xml
<aop:aspectj-autoproxy poxy-target-class="true"/>
```

## 总结

本篇我们学习了 Spring 的 AOP 及三种实现方式，目前比较常用第二种自定义类的方式和第三种注解的方式。

参考自：
1. [狂神说Spring07：AOP就这么简单](hhttps://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247484138&idx=1&sn=9fb187c7a2f53cc465b50d18e6518fe9&scene=19#wechat_redirect)
