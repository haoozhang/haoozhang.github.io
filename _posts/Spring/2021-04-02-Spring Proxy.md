---
layout:     post
title:      6. Spring Proxy
subtitle:   
date:       2021-04-02
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring
---

为什么要学习代理模式？因为 Spring AOP 的底层机制就是动态代理，所以在学习 AOP 之前，我们先了解一下代理模式。代理模式分为两种：静态代理和动态代理，整个代理的过程如下图所示。

![img](/img/post/Spring/proxyPattern.png)

我们以一个租房的例子解释上图的过程，Client 就是租房者，Real Subject 就是房东，Proxy 就是租房中介，房东和中介都要做的事是出租房子，所以 Abstract Object 就是出租房子。最初情况下，租房者是直接与房东沟通来租房的。但随着手续越来越麻烦，房东要做的事就越来越多，比如贴广告、带看房、签合同等。为了简化房东的事务，就引入 **租房中介** 这个角色。租房中介负责帮房东出租房子、经办上面一系列繁琐的事情。而房东只需要每月收到租金即可。此时，租房者就只会和中介打交道，而看不到背后的房东，这就是代理模式。

## 静态代理

### 静态代理角色分析：
+ Abstract Object: 抽象角色，一般使用接口或者抽象类来实现
+ Real Object: 被代理的真实角色
+ Proxy: 代理角色，代理真实角色后，一般会做一些附属的操作
+ Client: 通过代理角色来进行一些操作

### 代码实现

抽象角色，Rent.java

```java
// 抽象角色：租房
public interface Rent {
   public void rent();
}
```

真实角色，Host.java

```java
// 真实角色: 房东，房东要出租房子
public class Host implements Rent{
   public void rent() {
       System.out.println("房屋出租");
  }
}
```

代理角色，Proxy.java

```java
// 代理角色：租房中介，帮房东出租房子
public class Proxy implements Rent {
 
    private Host host;
 
    public Proxy() { 
 
    }
    
    public Proxy(Host host) {
        this.host = host;
    }
 
    // 租房
    public void rent(){
        seeHouse();
        sign();
        host.rent();
        fee();
    }

    // 看房
    public void seeHouse(){
        System.out.println("带房客看房");
    }

    public void sign() {
        System.out.println("签合同");
    }

    // 收费
    public void fee(){
        System.out.println("收中介费");
    }
}

客户，Client.java

```java
// 客户，一般客户都会去找代理
public class Client {
    public static void main(String[] args) {
        // 房东要租房
        Host host = new Host();
        // 中介帮助房东
        Proxy proxy = new Proxy(host);
        // 客户去找中介！
        proxy.rent();
    }
}
```

我们就实现了上面的租房例子，那这样的代理有何优缺点呢？

优点:

+ 简化真实角色的功能，不用去关注公共的业务
+ 公共的业务由代理来完成，实现业务分工
+ 公共业务发生扩展时更加集中和方便

缺点:
+ 一个真实角色对应一个代理角色，代码量翻倍，开发效率降低

## 静态代理再理解

我们再聚一个例子，加深对静态代理的理解

### 练习步骤

1、创建一个抽象角色，比如一般的用户业务，抽象起来就是增删改查

```java
// 抽象角色：增删改查业务
public interface UserService {
   void add();
   void delete();
   void update();
   void query();
}
```

2、创建一个真实对象来完成这些增删改查操作

```java
//真实对象，完成增删改查操作
public class UserServiceImpl implements UserService {

   public void add() {
       System.out.println("增加了一个用户");
  }

   public void delete() {
       System.out.println("删除了一个用户");
  }

   public void update() {
       System.out.println("更新了一个用户");
  }

   public void query() {
       System.out.println("查询了一个用户");
  }
}
```

3、此时新的需求来了，我们要增加一个日志功能，怎么实现？
+ 思路1：在实现类上增加代码 **【麻烦】**
+ 思路2：通过代理完成，能够在不改变原来业务的情况下实现此功能

4、设置一个代理代理角色类来处理日志

```java
// 代理角色，增加日志的实现
public class UserServiceProxy implements UserService {
   private UserServiceImpl userService;

   public void setUserService(UserServiceImpl userService) {
       this.userService = userService;
  }

   public void add() {
       log("add");
       userService.add();
  }

   public void delete() {
       log("delete");
       userService.delete();
  }

   public void update() {
       log("update");
       userService.update();
  }

   public void query() {
       log("query");
       userService.query();
  }

   public void log(String msg){
       System.out.println("执行了"+msg+"方法");
  }
}
```

5、测试
```java
public class Client {
    public static void main(String[] args) {
        // 真实业务
        UserServiceImpl userService = new UserServiceImpl();
        // 代理类
        UserServiceProxy proxy = new UserServiceProxy();
        // 使用代理类实现日志功能！
        proxy.setUserService(userService);
        // 直接访问代理
        proxy.add();
    }
}
```

OK，到这里我们就学习了静态代理，它最核心的思想是：在不改变原来的代码的情况下，实现了对原有功能的增强，这是 AOP 中最核心的思想

### AOP：纵向开发、横向开发

![img](/img/post/Spring/hengzong.png)

## 动态代理

前面我们说到，静态代理有一个缺点是，一个真实对象对应一个代理类。为了解决这个问题，提出了动态代理。动态代理的角色和静态代理一样，不同的是，动态代理的代理类是动态生成的，静态代理的代理类是我们提前写好的。

动态代理分为两类，一类是基于接口动态代理，一类是基于类的动态代理
+ 基于接口的动态代理：JDK 动态代理
+ 基于类的动态代理：cglib
+ 现在用的比较多的是 javasist 来生成动态代理

这里我们使用 JDK 动态代理。JDK 动态代理需要了解两个类，InvocationHandler 和 Proxy，打开 JDK 文档可以查看这两个类：

![img](/img/post/Spring/invocationHandler.png)

![img](/img/post/Spring/invoke.png)

![img](/img/post/Spring/proxy.png)

![img](/img/post/Spring/newProxyIns.png)

创建动态代理类：

```java
Foo f = (Foo) Proxy.newProxyInstance(
                    Foo.class.getClassLoader(),                                      
                    new Class<?>[] { Foo.class },
                    handler);
```

### 代码实现

抽象角色和真实角色和之前的一样

```java
//抽象角色：租房
public interface Rent {
    public void rent();
}

//真实角色: 房东要出租房子
public class Host implements Rent{
    public void rent() {
        System.out.println("房屋出租");
    }
}
```

代理角色，ProxyInvocationHandler.java 

```java
public class ProxyInvocationHandler implements InvocationHandler {
    private Rent rent;
 
    public void setRent(Rent rent) {
        this.rent = rent;
    }
 
    // 创建动态代理类，重点是第二个参数，获取要代理的抽象角色。
    // 之前都是一个角色，现在可以代理一类角色
    public Object getProxy(){
        return Proxy.newProxyInstance(this.getClass().getClassLoader(),
                rent.getClass().getInterfaces(),this);
    }
 
    // proxy: 代理类，method: 代理类的调用处理程序的方法对象
    // 处理代理实例上的方法调用并返回结果
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        seeHouse();
        //核心：本质利用反射实现！
        Object result = method.invoke(rent, args);
        fare();
        return result;
    }
 
    //看房
    public void seeHouse(){
        System.out.println("带房客看房");
    }

    //收中介费
    public void fare(){
        System.out.println("收中介费");
    }
}
```

Client.java
```java
//租客
public class Client {
 
    public static void main(String[] args) {
        // 真实角色
        Host host = new Host();
        // 代理实例的调用处理程序
        ProxyInvocationHandler pih = new ProxyInvocationHandler();
        // 将真实角色放置进去
        pih.setRent(host); 
        // 创建动态代理类
        Rent proxy = (Rent)pih.getProxy(); 

        proxy.rent();
    }
}
```

**核心：一个动态代理一般代理某一类业务，一个动态代理可以代理多个类，代理的是接口**

## 动态代理再理解

我们来使用动态代理实现我们后面写的 UserService 例子

我们也可以编写一个通用的动态代理实现的类，所有的代理对象设置为 Object 即可！

```java
public class ProxyInvocationHandler implements InvocationHandler {
   private Object target;

   public void setTarget(Object target) {
       this.target = target;
  }

   // 生成代理类
   public Object getProxy(){
       return Proxy.newProxyInstance(this.getClass().getClassLoader(),
               target.getClass().getInterfaces(),this);
  }

   // proxy: 代理类
   // method: 代理类的调用处理程序的方法对象
   public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
       log(method.getName());
       Object result = method.invoke(target, args);
       return result;
  }

   public void log(String methodName){
       System.out.println("执行了"+methodName+"方法");
  }

}
```

测试增删改查，查看结果！

```java
public class Test {
   public static void main(String[] args) {
       // 真实对象
       UserServiceImpl userService = new UserServiceImpl();
       // 代理对象的调用处理程序
       ProxyInvocationHandler pih = new ProxyInvocationHandler();
       // 设置要代理的对象
       pih.setTarget(userService); 
       // 动态生成代理类！
       UserService proxy = (UserService)pih.getProxy(); 
       proxy.delete();
  }
}
```

### 动态代理的好处

静态代理有的它都有，静态代理没有的，它也有：

+ 简化真实角色的功能，不用去关注公共的业务
+ 公共的业务由代理来完成，实现业务分工
+ 公共业务发生扩展时更加集中和方便
+ 一个动态代理一般代理某一类业务
+ 一个动态代理可以代理多个类，代理的是接口！

## 总结

本篇我们学习了代理模式，了解了静态代理和动态代理，这是 Spring AOP 的基础，下一篇我们学习 AOP。

参考自：
1. [狂神说Spring05：使用注解开发](https://mp.weixin.qq.com/s/dCeQwaQ-A97FiUxs7INlHw)
