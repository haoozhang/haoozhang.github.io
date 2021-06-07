---
layout:     post
title:      12. 异步、邮件、定时任务
subtitle:   
date:       2021-06-06
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SpringBoot
---

随着互联网技术的发展，网站的规模在不断地扩大，单一应用架构已经不再能满足需求，由于架构的原因，系统中某一处如果要进行修改，整个应用就需要重新部署，代价非常大。而在分布式架构中就能很好地解决上述问题，每一个模块就拆分为一个单独的服务，当模块的访问量变大时，就可以将同一个服务部署到不同的机器中同时运行。

## RPC

当不同的模块之间出现调用关系时就会用到RPC (Remote Procedure Call)，当前最主流的两个框架一个是Dubbo，一个是SpringCloud。这里我们主要介绍Dubbo，Dubbo是Alibaba开源的分布式服务框架，它最大的特点是按照分层的方式来架构，使用这种方式可以使各层之间解耦(或者最大限度地松耦合)。从服务模型的角度看，Dubbo采用的是一种非常简单的模型，要么是提供方提供服务，要么是消费方消费服务，所以基于这一点可以抽象出服务提供方(Provider)和服务消费方(Consumer)两个角色。

![img](/img/post/SpringBoot/dubbo1.png)

## 注册中心

当需要被调用的模块在不同的多态机器上进行部署时，就会出现调用方不知道该调用哪台机器上的服务，这时候就需要注册中心来帮助我们。注册中心会保存每个模块所在服务器的位置，当调用方需要调用时，首先去注册中心查看被调用模块的位置，然后根据注册中心返回的位置信息，去对应的服务器上进行调用。ZooKeeper就是一个应用非常广泛的注册中心应用，它是一个分布式的应用程序协调服务，它为分布式应用提供一致性服务，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

## Dubbo

Apache Dubbo 是一款高性能、轻量级的开源Java RPC框架，其架构如下

![img](/img/post/SpringBoot/dubbo2.png)

+ 服务容器 (Container) 在启动时会加载和运行服务提供者 (Provider)
+ 服务提供者在启动后会将能提供的自身的服务信息注册到注册中心 (Registry)
+ 服务消费方 (Consumer) 在启动后会订阅 (Subscribe) 注册中心中相关服务的信息
+ 注册中心会将消费方所需的服务的地址列表提供 (Notify) 给消费方，如果服务信息变更，注册中心会基于长连接的方式推送给消费方。
+ 消费方当需要调用 (Invoke) 时，根据注册中心提供的地址列表基于负载均衡机制找到服务提供方的位置进行调用
+ 如果调用失败，会再去找下一个服务提供者进行调用，直到调用成功为止。
+ 监控中心 (Monitor) 会监控服务的提供放和消费方

## Spring Boot + Dubbo + Zookeeper 整合

### 构建 Zookeeper 环境

Zookeeper 是服务注册中心的一种实现，调用者需要在注册中心订阅服务提供者的信息，并根据注册中心提供的服务地址调用，因此我们首先要部署 Zookeeper 的环境，在[官网](https://zookeeper.apache.org/releases.html)下载 Zookeeper 压缩包

```bash
# 解压
tar -zxvf zookeeper-3.4.13.tar.gz
# 切换到解压目录
cd zookeeper-3.4.13/
# 新建 data 和 logs 目录
mkdir data
mkdir logs
# 复制zoo_sample.cfg 为 zoo.cfg
cd conf/
cp zoo_sample.cfg zoo.cfg
# 修改配置
vi zoo.cfg
dataDir=刚刚新建的data目录的绝对路径
logDir=刚刚新建的data目录的绝对路径
# 启动
cd ../bin
./zkServer.sh start
# 检测是否启动成功
./zkServer.sh status
```

### Dubbo admin

为了让用户更好的管理监控众多的dubbo服务，官方提供了一个可视化的监控程序dubbo-admin，不过这个监控即使不装也不影响使用

1、下载 dubbo-admin，地址 ：https://github.com/apache/dubbo-admin/tree/master

2、解压进入目录，修改 dubbo-admin 下的 application.properties，指定 zookeeper 地址

```
dubbo.registry.address=zookeeper://127.0.0.1:2181
```

3、打包项目

```
mvn clean package -Dmaven.test.skip=true
```

4、执行 dubbo-admin\target 下打包的 dubbo-admin-0.0.1-SNAPSHOT.jar，然后访问 http://localhost:7001/，登录账户和密码默认均为 root，登录成功就可以看到如下界面，这里我们看到提供的服务

![img](/img/post/SpringBoot/dubbo_admin.png)

### 服务提供者

首先创建一个空项目，然后创建 Spring Boot Web 模块：Provider，引入以下依赖

```xml
<!--  dubbo的starter引入  -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.3</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!--  引入dubbo  -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo</artifactId>
    <version>2.7.3</version>
</dependency>

<!--    对zookeeper的底层api的一些封装    -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.0.1</version>
</dependency>

<!-- 封装了一些高级特性，如：Cache事件监听、选举、分布式锁、分布式Barrier -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.8.0</version>
</dependency>

<!--  引入zookeeper  -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.13</version>
    <type>pom</type>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!--  zookeeper连接客户端  -->
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.10</version>
</dependency>
```

然后，我们需要在 application.properties 文件中配置 dubbo 的相关信息

```bash
# Dubbo provider configuration
dubbo.application.name=provider-ticket
dubbo.registry.address=zookeeper://127.0.0.1:2181
# 扫描注解包通过该设置将服务注册到zookeeper
dubbo.scan.base-packages=com.zhao.service
```

其中，我们需要将 Service 下的类 (即服务) 发布到 Zookeeper 中，通过注册中心同志调用者该服务的地址。接下来我们编写 Service

```java
public interface TicketService {
    public String ticket();
}
```

```java
import org.apache.dubbo.config.annotation.Service;
import org.springframework.stereotype.Component;

@Component  // 作用等同于 Service，只是区别于下面的注解
@Service  // 注意这个 Service 注解是 Dubbo 下的
public class TicketServiceImpl implements TicketService {
    @Override
    public String ticket() {
        return "lan ticket";
    }
}
```

编写完提供的服务后，我们就可以将 TicketService 发布到 Zookeeper 上，启动项目，在 Zookeeper 上查看是否已经发布，我们通过 zkClient 查看

```bash
# 进入 Zookeeper 的 bin 目录
# 启动 zkCli 脚本
zkCli.sh
# 查看注册的服务
[zk: localhost:2181(CONNECTED) 0] ls /dubbo
# 出现如下信息，表示服务已经注册成功
[com.zhao.service.TicketService]
```

### 服务消费者

服务消费者通过 Dubbo 进行远程调用，我们再新建一个 Spring Boot Web 模块：Consumer，需要引入的依赖与 Provider 相同，配置文件内容如下

```
dubbo.application.name=consumer-ticket
dubbo.registry.address=zookeeper://127.0.0.1:2181
```

服务消费者无需将自身服务发布出去，因为它本身就是消费者

正常步骤应该是将服务提供者的接口打包，然后使用 pom 文件导入。这里我们使用简单的方式，直接将服务的接口拿过来，放在相同的路径下

![img](/img/post/SpringBoot/ticketService.png)

并写入相同的服务提供方法

```java
public interface TicketService {
    public String ticket();
}
```

编写消费者的服务

```java
public interface UserService {
    public void getTicket();
}
```
```java
@Service
public class UserServiceImpl implements UserService {

    @Reference  // 该注解帮助我们根据注册中心提供的地址调用服务
    TicketService ticketService;

    @Override
    public void getTicket() {
        String ticket = ticketService.ticket();
        System.out.println("get " + ticket);
    }
}
```

在测试类中调用 UserService 测试

```java
@SpringBootTest
class ConsumerApplicationTests {

    @Autowired
    private UserService userService;

    @Test
    void contextLoads() {

        userService.getTicket();
    }

}
```

![img](/img/post/SpringBoot/test_Result.png)

## 总结

以上是SpringBoot+Dubbo+ZooKeeper的整合测试

参考自：
1. [SpringBoot+Dubbo+ZooKeeper整合总结](https://www.rossontheway.com/2020/05/28/SpringBoot-Dubbo-ZooKeeper%E6%95%B4%E5%90%88%E6%80%BB%E7%BB%93/)
