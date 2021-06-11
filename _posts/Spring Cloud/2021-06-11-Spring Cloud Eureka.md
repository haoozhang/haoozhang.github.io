---
layout:     post
title:      Spring Cloud Eureka
subtitle:   
date:       2021-06-11
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

本文学习服务的注册和发现中心 Eureka，并修改服务提供者和消费者的环境实现服务的注册、发现和调用

## What is Eureka 

Eureka 是 Netflix 的一个核心子模块，以基于 REST 的方式实现服务的注册与发现。
服务的注册于发现对微服务架构十分重要，有了服务注册和发现，我们就可以根据服务标识符直接访问服务，而无需修改服务调用的配置文件。

Spring Cloud 封装了 Netflix 开发的 Eureka 模块来实现服务注册和发现。
Eureka 采用 CS 架构，Eureka Server 作为服务注册的服务器，也即服务注册中心；
而系统中的其它微服务，使用 Eureka Client 连接到 Eureka Server 并维持心跳连接，这样系统运维人员就可以通过 Eureka Server 来监控系统中的微服务是否正常运行，Spring Cloud 中的一些其他模块 (例如 Zuul 组件执行路由转发和过滤) 就可以通过 Eureka Server 来发现其它微服务，并执行相关逻辑。

因此，系统中一共出现了三种角色：
+ Eureka Server：服务注册中心
+ Service Provider：注册自身服务到 Eureka 中，从而方便消费者找到并调用
+ Service Consumer：消费者从 Eureka 中获取服务注册列表，从而消费某个服务

## Eureka 环境搭建

我们先搭建 Eureka 环境，上一篇文章已经搭建了 Spring Cloud 的提供者消费者环境，所以我们继续在它上面修改。

1、新建 Maven 模块，pom.xml 中引入 Eureka Server 的依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    <version>3.0.3</version>
</dependency>
```

2、在 resources 目录下新建 application.yml 文件，编写配置文件

```yml
server:
  port: 7001

eureka:
  instance:
    a-s-g-name: localhost  # Eureka Server 名字
  client:
    register-with-eureka: false  # 是否向 Eureka Server 注册自己
    fetch-registry: false  # 表示自己为注册中心
    service-url:  # Eureka Server 监控页面
      defaultZone: http://${eureka.instance.a-s-g-name}:${server.port}/eureka/
```

首先指定了这个服务的端口为 7001；\
然后通过 a-s-g-name 字段设置了这个服务的名字；\
register-with-eureka 字段表示是否向 Eureka Server 注册自己，因为这是 Eureka Server，所以无需再注册；\
fetch-registry 为 false，表示自己为注册中心；\
我们点开 service-url 字段的源码，可以看到它默认为 http://localhost:8761/eureka/，这里我们端口设置为7001，所以自定义 defaultZone

```java
public EurekaClientConfigBean() {
    this.serviceUrl.put("defaultZone", "http://localhost:8761/eureka/");
    ....
}
```

3、编写启动类，这里引入了 @EnableEurekaServer 注解

```java
@SpringBootApplication
@EnableEurekaServer  // 开启 Eureka 服务器，接收别的服务注册进来
public class EurekaServer {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServer.class, args);
    }
}
```

Eureka Server 的环境就搭建好了，我们访问 http://localhost:7001/ 测试，可看到如下页面：

![img](/img/SpringCloud/eureka.png)

由于目前还没有注册服务，所以 Instances 为空。

## 服务提供者

接下来，我们修改服务提供者，将其注册到 Eureka Server。

1、pom.xml 文件中引入 Eureka Client 的依赖

```xml
<!-- Eureka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>3.0.3</version>
</dependency>
```

2、application.yml 配置文件中配置 Eureka 相关信息

```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

3、在主启动类上添加 @EnableEurekaClient 注解，开启 Eureka 客户端，自动注册该服务到 Eureka Server

4、启动项目，我们可以从 Eureka Server 页面上看到 注册的服务

![img](/img/SpringCloud/provider1.png)

5、可以看到注册的服务右边有一个描述信息的链接，目前点进去为 error 页面。我们可以修改描述链接

```yml
eureka:
  instance:
    instance-id: springcloud-provider-8081  # 修改默认的描述信息
```

6、并添加点击链接后的信息，引入如下依赖

```xml
<!-- 完善监控信息 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

在配置文件中添加描述信息

```yml
# actuator/info 配置
info:
  app.name: springcloud-provider-dept
  company.name: hao.blog
```

7、启动项目，查看 Eureka Server 页面，可以看到 Provider 服务描述信息变化

![img](/img/SpringCloud/provider2.png)

点击链接后，可以看到我们添加的描述信息

```
{"app":{"name":"springcloud-provider-dept"},"company":{"name":"hao.blog"}}
```

8、当我们结束提供者服务后，Eureka Server 仍然显示这个服务存在。等待一会儿，页面会出现如何的信息

![img](/img/SpringCloud/selfprot.png)

这是由于 Eureka 的自我保护机制。默认情况下，Eureka Server 在一定时间内没有收到某个服务的心跳 (默认90秒)，就会注销该服务。但当网络异常时，微服务与 Eureka 无法正常通信，Eureka 的注销行为就很危险。因为此时微服务是在正常运行，不应该被注销。所以 Eureka 引入自我保护机制处理这种问题，当 Eureka Server 短时间内丢失过多客户端时，它就进入自我保护模式，在该模式下，Eureka Server 会保护服务注册表中的信息，不再注销任何微服务。当网络故障恢复后，它自动退出自我保护模式。

这样的设计理念就是宁可保留错误的服务信息，也不能注销任何健康的服务。使用自我保护模式，Eureka 可以更加健壮稳定。

Spring Cloud 可以使用 eureka.server.enable-self-preservation=false 来禁用自我保护模式，但不一般建议禁用。
 
## 服务消费者

服务消费者的修改与提供者类似。

1、引入 Eureka Client 的依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>3.0.3</version>
</dependency>
```

2、配置 Eurake 信息

```yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

3、主启动类添加 @EnableEurekaClient 注解，使该服务自动注册到 Eureka Server

4、启动项目，查看 Eureka 页面，可看到注册的服务。之后参照上面的步骤修改描述链接和信息

5、修改 controller 的接口。之前我们是直接按照提供者的访问路径访问服务的，但有了 Eureka 之后，我们可以根据服务的标识符来访问服务，以 getDept 为例：

```java
@Autowired
private DiscoveryClient client;

@GetMapping("/dept/get/{id}")
public Dept getDept(@PathVariable("id") Long id) {
    // 根据提供者名称获取服务的访问路径
    REST_URL_PREFIX = client.getInstances("provider").get(0).getUri().toString();
    return restTemplate.getForObject(REST_URL_PREFIX+"/dept/get/"+id, Dept.class);
}
```

可以看到 REST_URL_PREFIX 是根据服务标识符 provider 获取到。我们可以添加一个接口，查看服务的具体信息

```java
@GetMapping("/svcInfo")
public String serviceInfo() {
    List<ServiceInstance> instances = client.getInstances("provider");
    return instances.toString();
}
```

6、启动消费者项目，我们访问消费者的端口 http://localhost:8080/dept/get/1，可以看到如下输出

```
{"id":1,"name":"develop","dbSource":"cloud"}
```

访问 http://localhost:8080/svcInfo 端口，可以看到注册的提供者服务的信息

```
[[EurekaServiceInstance@66850306 instance = InstanceInfo [instanceId = springcloud-provider-8081, appName = PROVIDER, hostName = 192.168.1.103, status = UP, ipAddr = 192.168.1.103, port = 8081, securePort = 443, dataCenterInfo = com.netflix.appinfo.MyDataCenterInfo@7490e2ea]]
```

## Eureka 集群

目前我们的 Eureka Server 只有一个，一旦它断了，那服务注册中心就崩溃了。为了保证可用性，我们建立一个 Eureka 集群。

1、我们编辑 Eureka 模块的配置信息，将它设置为可以并行运行

![img](/img/SpringCloud/parallelRun.png)

2、配置第一个 Server，它的端口为7001，Server 名字为 localhost7001，然后挂载 http://localhost:7002/eureka/ 页面，启动该项目

```yml
server:
  port: 7001

eureka:
  instance:
    a-s-g-name: localhost7001  # Eureka Server 名字
  client:
    register-with-eureka: false  # 是否向 Eureka Server 注册自己
    fetch-registry: false  # 表示自己为注册中心
    service-url:  # Eureka Server 监控页面
      defaultZone: http://localhost:7002/eureka/ 
```

3、对应地，第二个 Server 端口为7002，Server 名字为 localhost7002，然后挂载 http://localhost:7001/eureka/ 页面，启动该项目，此时 Server 就同时运行了，两个节点的集群就建立了，如果要扩为3个节点，只需每个节点挂载其它两个页面即可

```yml
server:
  port: 7002

eureka:
  instance:
    a-s-g-name: localhost7002  # Eureka Server 名字
  client:
    register-with-eureka: false  # 是否向 Eureka Server 注册自己
    fetch-registry: false  # 表示自己为注册中心
    service-url:  # Eureka Server 监控页面
      defaultZone: http://localhost:7001/eureka/ 
```

4、配置要注册的服务的 application.yml

```yml
# Eureka 配置
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka, http://localhost:7002/eureka
```

5、启动项目，我们就可以在 http://localhost:7001 和 http://localhost:7002 两个页面看到相同的注册服务，可以看到 7001 页面的集群信息挂载 7002，7002 页面挂载 7001

![img](/img/SpringCloud/eurekacluster.png)

## Eureka vs. Zookeeper

首先，我们了解一下 CAP 原则，即 Consistency (强一致性)、Availability (可用性)、Partition tolerance (分区容错性)。在学习关系型数据库时，了解到 MySQL、SQLServer 等数据库要遵循 ACID 原则，但诸如 Redis 的 NoSQL 一般遵循 CAP 原则。

CAP 理论指出，一个分布式系统不可能同时满足 CAP 三个需求。由于分区容错性 P 在分布式系统中必须保证，因此我们只能在 A 和 C 之间权衡：
+ Zookeeper 保证的是 CP
+ Eureka 保证的是 AP

### Zookeeper 保证 CP

Zookeeper 的一致性要求高于可用性。在 Zookeeper 中会出现一种情况，当 master 节点因为网络故障与其他节点失去联系时，剩余节点会进行 leader 选举，在选举 leader 的这段时间内整个 zk 集群都不可用，导致注册服务瘫痪。

### Eureka 保证 AP

Eureka 注意到了以上的问题，所以设计时优先保证可用性，即各个节点都是平等的。几个节点断连不影响其他节点工作，只要有一台 Eureka Server 在线，就能保证服务的可用性，只不过查询的信息可能不是最新的。此外，Eureka 还有自我保护机制，如果 15 分钟内 85% 的节点都没有正常心跳，则认为客户端与注册中心之间出现了网络故障，此时出现以下几种情况：

1. Eureka 不再从服务注册表中移除服务
2. Eureka 仍能接受新的服务注册和查询请求，但不会痛不到其他节点
3. 当网络稳定时，该节点的注册信息同步到其他节点上

因此，Eureka 可以很好处理部分节点断连的情况，避免了像 Zookeeper 造成整个注册服务的瘫痪

