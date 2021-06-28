---
layout:     post
title:      Spring Cloud LoadBalancer 作负载均衡
subtitle:   
date:       2021-06-15
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

本文学习 Spring Cloud 中的负载均衡工具 Spring Cloud LoadBalancer，并基于前面搭建的工程环境进一步修改实现

## What is LoadBalancer 

我们先从 Ribbon 讲起，Spring Cloud Ribbon 是基于 Netflix Ribbon 实现的一套 **客户端负载均衡工具**。\
具体地，Ribbon 是 Netflix 发布的开源项目，提供客户端的负载均衡策略。通过在配置文件中列出 Load Balancer 后面的服务，它可以基于某种规则 (轮询，随机) 去连接服务，我们也可以使用 Ribbon 自定义负载均衡策略。

LB，即负载均衡，指将用户的请求平摊到分配的多个服务上，以达到系统的高可用性，是微服务和分布式集群中常用的一种功能。\
常见的复杂均衡工具有 Nginx、Lvs 等；Dubbo 和 Spring Cloud 均提供了负载均衡。
负载均衡简单分为以下两类：
+ 集中式 LB：在消费方和提供方之间使用独立的 LB 设施 (如 Nginx)，由该设施负责把服务请求按照某种策略转发到提供方
+ 进程内 LB：将 LB 逻辑集成到消费方，消费方从服务中心获取可用服务列表，并按照策略选择某个服务

Ribbon 就属于进程内 LB，它只是一个类库，集成在消费方进程，消费方通过它来获取服务的具体地址。\
而在 Spring Cloud 2020 版本以后，Spring Cloud 团队移除了 Ribbon 组件，而用自己实现的产品 Spring Cloud LoadBalancer 替代。它的功能与 Ribbon 相同，但配置方式略有不同。\
我们之前学习 Eureka 时引入了 spring-cloud-netflix-eureka 依赖。**在 Spring Cloud 2020.0.3 版本以后，
spring-cloud-netflix-eureka 已经包含了负载均衡，所以我们无需再显式引入 Spring Cloud LoadBalancer 依赖**

![img](/img/SpringCloud/lb-depend.png)

接下来，我们要配置负载均衡，因为我们是使用 RestTemplate 来调用请求的，所以我们让 RestTemplate 实现负载均衡即可。给 RestTemplate bean 实例添加如下注解

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}
```

使用 LoadBalancer 负载均衡之后，我们不用再把远程访问的地址写死，因为它要选择其中的一个服务 (即便我们现在只有一个服务)，我们可以修改消费方的 controller 中远程访问的地址前缀

```java
private static String REST_URL_PREFIX = "http://provider";
```

配置以上内容后，我们就可以启动项目体验一下负载均衡了。依次启动 Eureka、Provider、Consumer，然后按照消费方的 controller 中的访问路径测试链接，可以看到访问成功。

## LoadBalancer 实现负载均衡

上面只是简单配置了负载均衡，但因为目前只有一个服务提供者，所以我们并没有显式地体会到它均衡的效果。接下来我们创建两个 Provider 来实现负载均衡。

1、我们首先创建底层的数据库，它的内容和之前的数据库内容一样，除了 db_source 列显示的库名不同

![img](/img/SpringCloud/cloud2.png)

2、新建一个和之前 Provider 相同的模块，然后将配置文件中端口改为 8082，访问的数据库改为 cloud02，并修改默认的描述信息\
要注意，两个服务提供者的 application name 必须相同

![img](/img/SpringCloud/provider2yml.png)

3、我们依次启动 Eureka、两个提供者、消费者，可以通过 Eureka 页面看到三个服务都已经注册

![img](/img/SpringCloud/svc3.png)

然后我们测试消费方访问服务，可以看到每次查询到的信息的 db_source 不同，这就代笔了它每次访问的服务不同，负载均衡生效！

目前我们的整体架构应该如下图所示

![img](/img/SpringCloud/ribbon_archi.png)

## LoadBalancer 自定义负载均衡策略

LoadBalancer 已经提供了一些负载均衡策略，比如轮询、随机等，默认为轮询。当然，我们也可以自定义一些负载均衡策略。

首先，我们更改它的默认负载均衡策略，根据[官网文档](https://docs.spring.io/spring-cloud-commons/docs/3.0.3/reference/html/#spring-cloud-loadbalancer)示例，创建如下类

```java
public class CustomLoadBalancerConfig {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
                                                            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(loadBalancerClientFactory
                .getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

它返回了随机的负载均衡策略。为了使其生效，我们在消费者的主启动类上添加如下注解

```java
....
@LoadBalancerClients(defaultConfiguration = CustomLoadBalancerConfig.class)
public class DeptConsumer {
  ....
}
```

启动项目，测试消费者访问请求的结果，多测试几次，可以观察到随机负载均衡的效果！

我们 Ctrl+B 查看 ReactorLoadBalancer 的源码，发现这个接口有两个实现类，分别就对应了轮询和随机两种负载均衡策略

![img](/img/SpringCloud/LBr.png)

为了实现自定义的负载均衡策略，我们 copy RoundRobinLoadBalancer 的实现，然后在对应位置改为我们的策略即可

```java
public class CustomLBRule implements ReactorServiceInstanceLoadBalancer {

    private static final Log log = LogFactory.getLog(CustomLBRule.class);

    final AtomicInteger position;

    final String serviceId;

    ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;

    public CustomLBRule(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
                                  String serviceId) {
        this(serviceInstanceListSupplierProvider, serviceId, new Random().nextInt(1000));
    }

    public CustomLBRule(ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
                                  String serviceId, int seedPosition) {
        this.serviceId = serviceId;
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
        this.position = new AtomicInteger(seedPosition);
    }

    @SuppressWarnings("rawtypes")
    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
                .getIfAvailable(NoopServiceInstanceListSupplier::new);
        return supplier.get(request).next()
                .map(serviceInstances -> processInstanceResponse(supplier, serviceInstances));
    }

    private Response<ServiceInstance> processInstanceResponse(ServiceInstanceListSupplier supplier,
                                                              List<ServiceInstance> serviceInstances) {
        Response<ServiceInstance> serviceInstanceResponse = getInstanceResponse(serviceInstances);
        if (supplier instanceof SelectedInstanceCallback && serviceInstanceResponse.hasServer()) {
            ((SelectedInstanceCallback) supplier).selectedServiceInstance(serviceInstanceResponse.getServer());
        }
        return serviceInstanceResponse;
    }

    private int duration = 0;
    private int curIndex = 0;

    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            if (log.isWarnEnabled()) {
                log.warn("No servers available for service: " + serviceId);
            }
            return new EmptyResponse();
        }
        // TODO: Custom Load Balance Rule
        if (duration < 3) {
            duration++;
        } else {
            duration = 0;
            curIndex++;
            if (curIndex >= instances.size()) {
                curIndex = 0;
            }
        }

        ServiceInstance instance = instances.get(curIndex);

        return new DefaultResponse(instance);
    }

}
```

在上面的 TODO 位置处，即为我们自定义的负载均衡策略，我们让每个服务访问3次再跳转到下个服务，这是个很简单的逻辑实现。然后，我们在 CustomLoadBalancerConfig 配置类中返回自定义的负载均衡类即可

```java
public class CustomLoadBalancerConfig {

    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment,
                                                            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new CustomLBRule(loadBalancerClientFactory
                .getLazyProvider(name, ServiceInstanceListSupplier.class),
                name);
    }
}
```

参考自：
1. [[狂神说Java]SpringCloud最新教程IDEA版](https://www.bilibili.com/video/BV1jJ411S7xr?p=12)
2. [Spring Cloud之Ribbon负载均衡（Spring Cloud 2020.0.3版）](https://www.cnblogs.com/seanRay/p/14781110.html)
3. [spring cloud 2020.0.1 LoadBalancer负载均衡算法切换](https://blog.csdn.net/qq_35799668/article/details/114534023)
4. [Switching between the load-balancing algorithms](https://docs.spring.io/spring-cloud-commons/docs/3.0.3/reference/html/#spring-cloud-loadbalancer)

