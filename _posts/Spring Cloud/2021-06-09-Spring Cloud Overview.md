---
layout:     post
title:      Spring Cloud Overview
subtitle:   
date:       2021-06-09
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

## What is Spring Cloud

Spring Cloud 是一系列框架的集合。它利用 Spring Boot 的开发便利性，巧妙地简化了分布式系统基础设施的开发，如服务注册、服务发现、配置中心、消息总线、负载均衡、断路器、数据监控等，这些都可以用 Spring Boot 的开发风格做到一键启动和部署。

通俗地讲，Spring Cloud 就是用于构建微服务开发和治理的框架集合，并不是具体的一个框架

![img](/img/SpringCloud/sc_def.png)

## Spring Cloud 和 Spring Boot 的关系

+ Spring Boot 专注于快速构建和开发单个微服务
+ Spring Cloud 是关注全局的微服务协调框架，它整合 Spring Boot 开发的一个个微服务，为各个微服务提供：服务发现、配置管理、断路器、路由等服务
+ Spring Boot 可离开 Spring Cloud 独立使用和开发项目；但 Spring Cloud 依赖 Spring Boot 开发

## Spring Cloud 和 Dubbo 对比

我们先从 Nginx 说起，最初的服务化解决方案是给相同服务提供一个统一的域名，然后服务调用者向这个域发送 HTTP 请求，由 Nginx 负责请求的分发和跳转。

![img](/img/SpringCloud/nginx.png)

这种架构存在很多问题：Nginx 作为中间层，在配置文件中耦合了服务调用的逻辑，这削弱了微服务的完整性，也使得 Nginx 在一定程度上不轻量化；由于 Nginx 作为中间层，服务消费方并不知道有哪些实例在给他们提供服务。

为了解决上面的问题，我们需要一个现成的中心组件对服务进行整合，汇总服务的信息，包括服务的组件名称、地址、数量等。

服务的调用方在请求某项服务时首先通过中心组件获取提供服务的实例信息（IP、端口等），再通过默认或自定义的策略选择该服务的某一提供方直接进行访问，所以考虑引入 Dubbo。

下图为 Dubbo 的基本架构图，使用 Dubbo 构建的微服务已经可以较好地解决上面提到的问题：
+ 消费方可以直接以 RPC 通信访问服务提供方；
+ 服务信息被集中到 Registry 中，形成了服务治理的中心组件；
+ 通过 Monitor 监控系统，可以直观地展示服务调用的统计信息；
+ 服务消费者可以进行负载均衡、服务降级的选择；

![img](/img/SpringCloud/dubbo.png)

但是对于微服务架构而言，Dubbo 也有一些缺陷，比如：
+ Registry 严重依赖第三方组件（如 ZooKeeper），当这些组件出现问题时，服务调用很快就会中断。
+ Dubbo 只支持 RPC 调用。这使得服务提供方与消费方在代码上产生了强依赖，服务提供方需要不断将包含公共代码的 Jar 包打包出来供消费方使用。一旦打包出现问题，就会导致服务调用出错。

Dubbo 的定位始终是一款 RPC 框架，而 Spring Cloud 的目标是微服务架构下的一站式解决方案。作为新一代的服务框架，Spring Cloud 提出的口号是开发"面向云的应用程序"，它为微服务架构提供了更加全面的技术支持。结合我们一开始提到的微服务的诉求，参见下表，把Spring Cloud 与 Dubbo 进行一番对比。

![img](/img/SpringCloud/sc_vs_dubbo.png)

Spring Cloud 抛弃了 Dubbo 的 RPC 通信，采用的是基于 HTTP 的 REST 方式。严格来说，这两种方式各有优劣。虽然从一定程度上来说，后者牺牲了服务调用的性能，但也避免了上面提到的原生 RPC 带来的问题。而且 REST 相比 RPC 更为灵活，服务提供方和调用方不存在代码级别的强依赖，这在强调快速演化的微服务环境下显得更加合适。

## Spring Cloud 2020 版本

在 Spring Cloud Hoxton 版本之后，Spring Cloud 取消了伦敦地铁站命名的方式，推出了 2020.0.x 版本，从官方文档可知，当前的最新版本为 2020.0.3。

新版本最大的变化就是移除了 spring cloud netflix 模块，仅仅包括了eureka模块。\
如 spring cloud 2020.0 release notes 中 [Breaking changes](https://github.com/spring-cloud/spring-cloud-release/wiki/Spring-Cloud-2020.0-Release-Notes#breaking-changes) 所述，the following modules have been removed from spring-cloud-netflix:

+ spring-cloud-netflix-archaius
+ spring-cloud-netflix-concurrency-limits
+ spring-cloud-netflix-core
+ spring-cloud-netflix-dependencies
+ spring-cloud-netflix-hystrix
+ spring-cloud-netflix-hystrix-contract
+ spring-cloud-netflix-hystrix-dashboard
+ spring-cloud-netflix-hystrix-stream
+ spring-cloud-netflix-ribbon
+ spring-cloud-netflix-sidecar
+ spring-cloud-netflix-turbine
+ spring-cloud-netflix-turbine-stream
+ spring-cloud-netflix-zuul
+ spring-cloud-starter-netflix-archaius
+ spring-cloud-starter-netflix-hystrix
+ spring-cloud-starter-netflix-hystrix-dashboard
+ spring-cloud-starter-netflix-ribbon
+ spring-cloud-starter-netflix-turbine
+ spring-cloud-starter-netflix-turbine-stream
+ spring-cloud-starter-netflix-zuul
+ Support for ribbon, hystrix and zuul was removed across the release train projects.

对应地，Spring Cloud 团队推荐了用于替代的方案：

```
Netflix     替代方案                          说明

Ribbon	    Spring Cloud Loadbalancer	     Spring 自己的产品
Hystrix     
Zuul        Spring Cloud Gateway	         Spring 自己的产品


Hystrix	       sentinel Resilience4j	
Hystrix Dashboard / Turbine	Micrometer + Monitoring System	说白了，监控这件事交给更专业的组件去做


Archaius 1	   Spring Boot外部化配置 + Spring Cloud 配置   比Netflix实现的更好、更强大

```



## 总结

参考自：
1. [Spring Cloud和Dubbo的区别及各自的优缺点](http://c.biancheng.net/spring_cloud/)


