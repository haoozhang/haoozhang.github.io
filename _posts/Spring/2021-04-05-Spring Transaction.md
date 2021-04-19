---
layout:     post
title:      9. Spring Transaction
subtitle:   
date:       2021-04-05
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring
---


## 回顾事务

ACID原则：
+ 原子性：
+ 一致性：
+ 隔离性：
+ 持久性：

## 测试

将上篇整合 Mybatis 的代码拷贝到一个新项目中。

在 UserMapper 接口新增两个方法，删除和增加用户；

```java
//添加一个用户
int addUser(User user);

//根据id删除用户
int deleteUser(int id);
```

mapper文件，我们故意把 deletes 写错，测试！
```xml
<insert id="addUser" parameterType="com.zhao.pojo.User">
insert into user (id,name,pwd) values (#{id},#{name},#{pwd})
</insert>

<delete id="deleteUser" parameterType="int">
deletes from user where id = #{id}
</delete>
```

编写接口的实现类，在实现类中，我们去操作一波
```java
public class UserMapperImpl extends SqlSessionDaoSupport implements UserMapper {

   //增加一些操作
   public List<User> selectUser() {
       User user = new User(4,"小明","123456");
       UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
       mapper.addUser(user);
       mapper.deleteUser(4);
       return mapper.selectUser();
  }

   //新增
   public int addUser(User user) {
       UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
       return mapper.addUser(user);
  }
   //删除
   public int deleteUser(int id) {
       UserMapper mapper = getSqlSession().getMapper(UserMapper.class);
       return mapper.deleteUser(id);
  }

}
```

测试
```java
@Test
public void test2(){
   ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
   UserMapper mapper = (UserMapper) context.getBean("userMapper");
   List<User> user = mapper.selectUser();
   System.out.println(user);
}
```

报错信息显示为 SQL 语句不正确，但仍然插入成功了。这不符合事务的特性。Spring给我们提供了事务管理，我们只需要配置即可；

## Spring 事务管理

Spring在不同的事务管理API之上定义了一个抽象层，使得开发人员不必了解底层的事务管理API就可以使用Spring的事务管理机制。Spring支持编程式事务管理和声明式的事务管理。我们主要关注声明式事务。

声明式事务管理

+ 一般情况下比编程式事务好用。
+ 将事务管理代码从业务方法中分离出来，以声明的方式来实现事务管理。
+ 将事务管理作为横切关注点，通过aop方法模块化。Spring中通过Spring AOP框架支持声明式事务管理。

使用Spring管理事务，注意头文件的约束导入 : tx

xmlns:tx="http://www.springframework.org/schema/tx"

http://www.springframework.org/schema/tx
http://www.springframework.org/schema/tx/spring-tx.xsd">

### 事务管理器

无论使用Spring的哪种事务管理策略（编程式或者声明式）事务管理器都是必须的。

就是 Spring的核心事务管理抽象，管理封装了一组独立于技术的方法。

JDBC事务
```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
       <property name="dataSource" ref="dataSource" />
</bean>
```
配置好事务管理器后我们需要去配置事务的通知
```xml
<!--配置事务通知-->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
   <tx:attributes>
       <!--配置哪些方法使用什么样的事务,配置事务的传播特性-->
       <tx:method name="add" propagation="REQUIRED"/>
       <tx:method name="delete" propagation="REQUIRED"/>
       <tx:method name="update" propagation="REQUIRED"/>
       <tx:method name="search*" propagation="REQUIRED"/>
       <tx:method name="get" read-only="true"/>
       <tx:method name="*" propagation="REQUIRED"/>
   </tx:attributes>
</tx:advice>
```
### Spring事务传播特性：

事务传播行为就是多个事务方法相互调用时，事务如何在这些方法间传播。spring支持7种事务传播行为：

+ propagation_requierd：如果当前没有事务，就新建一个事务，如果已存在一个事务中，加入到这个事务中，这是最常见的选择。

+ propagation_supports：支持当前事务，如果没有当前事务，就以非事务方法执行。

+ propagation_mandatory：使用当前事务，如果没有当前事务，就抛出异常。

+ propagation_required_new：新建事务，如果当前存在事务，把当前事务挂起。

+ propagation_not_supported：以非事务方式执行操作，如果当前存在事务，就把当前事务挂起。

+ propagation_never：以非事务方式执行操作，如果当前事务存在则抛出异常。

+ propagation_nested：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与propagation_required类似的操作

Spring 默认的事务传播行为是 PROPAGATION_REQUIRED，它适合于绝大多数的情况。

### 配置AOP

导入aop的头文件！

```xml
<!--配置aop织入事务-->
<aop:config>
   <aop:pointcut id="txPointcut" expression="execution(* com.zhao.mapper.*.*(..))"/>
   <aop:advisor advice-ref="txAdvice" pointcut-ref="txPointcut"/>
</aop:config>
```

### 测试

删掉刚才插入的数据，再次测试！
```java
@Test
public void test2(){
   ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
   UserMapper mapper = (UserMapper) context.getBean("userMapper");
   List<User> user = mapper.selectUser();
   System.out.println(user);
}
```

## 总结

本篇我们学习了 Spring 的声明式事务。

参考自：
1. [狂神说Spring09：声明式事务](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247484148&idx=1&sn=9d3edabf2443cd3a552e62e51b1f4097&scene=19#wechat_redirect)
