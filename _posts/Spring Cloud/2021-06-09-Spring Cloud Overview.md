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

![img](/img/post/SpringCloud/sc_def.png)

## Spring Cloud 和 Spring Boot 的关系

+ Spring Boot 专注于快速构建和开发单个微服务
+ Spring Cloud 是关注全局的微服务协调框架，它整合 Spring Boot 开发的一个个微服务，为各个微服务提供：服务发现、配置管理、断路器、路由等服务
+ Spring Boot 可离开 Spring Cloud 独立使用和开发项目；但 Spring Cloud 依赖 Spring Boot

## Spring Cloud 和 Dubbo 对比

我们先从 Nginx 说起，最初的服务化解决方案是给相同服务提供一个统一的域名，然后服务调用者向这个域发送 HTTP 请求，由 Nginx 负责请求的分发和跳转。

![img](/img/post/SpringCloud/nginx.png)

这种架构存在很多问题：Nginx 作为中间层，在配置文件中耦合了服务调用的逻辑，这削弱了微服务的完整性，也使得 Nginx 在一定程度上变成了一个重量级的 ESB。

服务的信息分散在各个系统，无法统一管理和维护。每一次的服务调用都是一次尝试，服务消费方并不知道有哪些实例在给他们提供服务。这带来了一些问题：
+ 无法直观地看到服务提供方和服务消费方当前的运行状况与通信频率；
+ 消费方的失败重发、负载均衡等都没有统一策略，加大了开发每个服务的难度，不利于快速演化。

为了解决上面的问题，我们需要一个现成的中心组件对服务进行整合，将每个服务的信息汇总，包括服务的组件名称、地址、数量等。

服务的调用方在请求某项服务时首先通过中心组件获取提供服务的实例信息（IP、端口等），再通过默认或自定义的策略选择该服务的某一提供方直接进行访问，所以考虑引入 Dubbo。

下图为 Dubbo 的基本架构图，使用 Dubbo 构建的微服务已经可以较好地解决上面提到的问题：
+ 调用中间层变成了可选组件，消费方可以直接访问服务提供方；
+ 服务信息被集中到 Registry 中，形成了服务治理的中心组件；
+ 通过 Monitor 监控系统，可以直观地展示服务调用的统计信息；
+ 服务消费者可以进行负载均衡、服务降级的选择。

![img](/img/post/SpringCloud/dubbo.png)

但是对于微服务架构而言，Dubbo 也有一些缺陷，比如：
+ Registry 严重依赖第三方组件（ZooKeeper 或者 Redis），当这些组件出现问题时，服务调用很快就会中断。
+ Dubbo 只支持 RPC 调用。这使得服务提供方与调用方在代码上产生了强依赖，服务提供方需要不断将包含公共代码的 Jar 包打包出来供消费方使用。一旦打包出现问题，就会导致服务调用出错。

Dubbo 的定位始终是一款 RPC 框架，而 Spring Cloud 的目标是微服务架构下的一站式解决方案。作为新一代的服务框架，Spring Cloud 提出的口号是开发“面向云的应用程序”，它为微服务架构提供了更加全面的技术支持。结合我们一开始提到的微服务的诉求，参见表 1，把Spring Cloud 与 Dubbo 进行一番对比。

![img](/img/post/SpringCloud/sc_vs_dubbo.png)

Spring Cloud 抛弃了 Dubbo 的 RPC 通信，采用的是基于 HTTP 的 REST 方式。严格来说，这两种方式各有优劣。虽然从一定程度上来说，后者牺牲了服务调用的性能，但也避免了上面提到的原生 RPC 带来的问题。而且 REST 相比 RPC 更为灵活，服务提供方和调用方，不存在代码级别的强依赖，这在强调快速演化的微服务环境下显得更加合适。

## 参考网站

+ https://spring.io/projects/spring-cloud
+ https://www.springcloud.cc/

## 总结

参考自：
1. [Spring Cloud和Dubbo的区别及各自的优缺点](http://c.biancheng.net/spring_cloud/)


