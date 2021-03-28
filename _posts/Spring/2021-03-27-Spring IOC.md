---
layout:     post
title:      Spring IOC
subtitle:   
date:       2021-03-28
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring
---

## IOC 基础

新建一个空白 Maven 项目，我们先用原来的方式写一段代码。

1、先写一个 UserDao 接口

```java
public interface UserDao {
    public void getUser();
}
```

2、再去写 Dao 的实现类

```java
public class UserDaoImpl implements UserDao {
    @Override
    public void getUser() {
        System.out.println("Get User Data");
    }
}
```

3、然后去写 UserService 的接口

```java
public interface UserService {
    public void getUser();
}
```

4、最后写Service的实现类

```java
public class UserServiceImpl implements UserService {

    private UserDao userDao = new UserDaoImpl();

    @Override
    public void getUser() {
        userDao.getUser();
    }
}
```

5、测试一下

```java
@Test
public void test(){
    UserService service = new UserServiceImpl();
    service.getUser();
}
```

这是我们原来的方式，一开始大家都是这么去写的。现在我们修改一下。

把 Userdao 的实现类增加一个

```java
public class UserDaoMysqlImpl implements UserDao {
   @Override
   public void getUser() {
       System.out.println("Get User Data from Mysql");
  }
}
```

如果我们要使用 Mysql，我们就要去 Service 实现类中修改对应的实现

```java
public class UserServiceImpl implements UserService {

    private UserDao userDao = new UserDaoMysqlImpl();
 
    @Override
    public void getUser() {
        userDao.getUser();
    }
}
```

假如我们再增加一个 Userdao 的实现类

```java
public class UserDaoOracleImpl implements UserDao {
   @Override
   public void getUser() {
       System.out.println("Get User Data from Oracle");
  }
}
```

如果我们要使用Oracle，又需要去 Service 实现类里面修改对应的实现，显然在公司很大的项目工程中，牵一发而动全身，这种方式太过于繁琐。每次变动我们都要修改对应位置的多处代码，设计的耦合性太高了。

那我们如何解决呢？我们可以在使用到它的位置，不去手动 new 一个对象，而是设置一个 set 方法。

```java
public class UserServiceImpl implements UserService {

    private UserDao userDao;
 
    // 利用 set 实现
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
 
    @Override
    public void getUser() {
        userDao.getUser();
    }
}
```

现在我们再去测试类中测试：

```java
@Test
public void test(){
    UserServiceImpl service = new UserServiceImpl();
    // 只需在此设置要使用的对象
    service.setUserDao(new UserDaoMySqlImpl());
    service.getUser();
}
```

这两者有啥区别呢？可能你觉得表面上没有什么区别，但是这两者根本上已经发生了变化。思考一下，前一种方式的对象完全是由程序去控制创建，而后一种方式是由用户自行创建，把主动权交给了调用者，程序不用控制创建，只是提供给用户一个接口。

这种思想从本质上解决了问题，程序员不用再去管理对象的创建了，更多的关注业务实现，大大降低了耦合性，这就是 IOC 的原型。

## IOC 本质

控制反转 IOC (Inversion of Control) 是一种设计思想，DI (依赖注入) 是实现 IOC 的一种方式。在没有 IOC 的程序中，我们使用面向对象编程，对象的创建与对象间的依赖关系完全硬编码在程序中，对象的创建由程序自己控制，控制反转后将对象的创建转移给用户。

IOC 是 Spring 框架的核心内容，我们可以使用 XML 文件配置，也可以使用注解方式配置。使用 XML 文件配置 IOC 的时候，Spring 容器初始化时读取配置文件，根据配置文件内容创建对象并保存在容器中，程序使用时从 IOC 容器中直接取出使用。

采用 XML 文件配置 Bean 的时候，Bean 的定义信息和实现分离，而采用注解的方式可以把两者合为一体，Bean 的定义信息直接以注解的形式定义在实现类中，从而达到了零配置的目的。

## Hello Spring

理解了 IOC 的基本思想之后，我们在 Spring 应用中通过 XML 文件配置 IOC。

新建 Maven 项目，并导入 Spring 的 jar 包。

```xml
<dependency>
   <groupId>org.springframework</groupId>
   <artifactId>spring-webmvc</artifactId>
   <version>5.3.5.RELEASE</version>
</dependency>
```

1、编写一个Hello实体类

```java
public class Hello {

    private String name;
 
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
 
    public void show(){
        System.out.println("Hello,"+ name );
    }
}
```

2、编写配置文件，这里我们命名为 beans.xml\

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

   <!-- bean就是java 对象 , 由 Spring 创建和管理 -->
   <bean id="hello" class="com.zhao.Hello">
       <property name="name" value="Spring"/>
   </bean>

</beans>
```

3、测试一下

```java
@Test
public void test(){
   // 解析 beans.xml 文件 , 生成管理相应的 Bean 对象
   ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
   // getBean: 参数即为spring配置文件中bean的id .
   Hello hello = (Hello) context.getBean("hello");
   hello.show();
}
```

+ 控制：谁来创建对象，以前是程序自身创建，现在是 Spring 容器创建。
+ 反转：程序本身不创建对象，只是 getBean 被动接收。

我们可以修改上面第一节中的程序，增加一个配置文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

   <bean id="MysqlImpl" class="com.zhao.dao.impl.UserDaoMySqlImpl"/>
   <bean id="OracleImpl" class="com.zhao.dao.impl.UserDaoOracleImpl"/>

   <bean id="ServiceImpl" class="com.zhao.service.impl.UserServiceImpl">
       <!-- 这里的 name 不是属性,而是 set 方法的参数,首字母小写 -->
       <!-- 引用另外一个bean,不是用 value 而是用 ref -->
       <property name="userDao" ref="OracleImpl"/>
   </bean>

</beans>
```

然后测试

```java
@Test
public void test2(){
   ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
   UserServiceImpl serviceImpl = (UserServiceImpl) context.getBean("ServiceImpl");
   serviceImpl.getUser();
}
```

到此为止，我们就不用再在程序中改动了。要实现不同的操作，只需要在 XML 配置文件中修改，所有的对象都由 Spring 创建、管理、装配。

## IOC 创建对象的方式

### 通过无参构造函数创建

1、User.java

```java
public class User {

    private String name;
 
    public User() {
        System.out.println("default constructor");
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public void show(){
        System.out.println("name="+ name );
    }
 
}
```
2、beans.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

   <bean id="user" class="com.zhao.pojo.User">
       <property name="name" value="zhao"/>
   </bean>

</beans>
```

3、测试

```java
@Test
public void test(){
   ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
   // getBean 之前，user对象已经通过无参构造函数被创建好了
   User user = (User) context.getBean("user");
}
```

### 通过有参构造函数创建

1、UserT.java
```java
public class UserT {

   private String name;

   public UserT(String name) {
       this.name = name;
  }

   public void setName(String name) {
       this.name = name;
  }

   public void show(){
       System.out.println("name="+ name );
  }

}
```

2、beans.xml 有三种方式编写

```XML
<!-- 第一种根据index参数下标设置 -->
<bean id="userT" class="com.zhao.pojo.UserT">
   <!-- index指构造方法 , 下标从0开始 -->
   <constructor-arg index="0" value="zhao"/>
</bean>

<!-- 第二种根据参数名字设置 -->
<bean id="userT" class="com.zhao.pojo.UserT">
   <!-- name指参数名 -->
   <constructor-arg name="name" value="zhao"/>
</bean>

<!-- 第三种根据参数类型设置 -->
<bean id="userT" class="com.zhao.pojo.UserT">
   <constructor-arg type="java.lang.String" value="zhao"/>
</bean>
```

3、测试

```java
@Test
public void testT(){
   ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
   UserT user = (UserT) context.getBean("userT");
   user.show();
}
```

结论：在配置文件加载的时候。其中管理的对象都已经初始化了！

## 几个配置选项

+ alias：为 bean 设置别名；
```xml
<alias name="userT" alias="userNew"/>
```
+ name：为 bean 设置多个别名；

```xml
<bean id="hello" name="hello2 h2,h3;h4"
```

+ import：在 applicationContext.xml 文件中合并多个配置文件为一个；

```xml
<import resource="{path}/beans.xml"/>
```

## 总结

本篇我们学习了 Spring IOC，通过实例详细了解了 IOC 的本质和使用方式。


参考自：
1. [狂神说Spring02：快速上手Spring](https://mp.weixin.qq.com/s/Sa39ulmHpNFJ9u48rwCG7A)
