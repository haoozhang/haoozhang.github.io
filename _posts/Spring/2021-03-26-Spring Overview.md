---
layout:     post
title:      1. Spring Overview
subtitle:   
date:       2021-03-26
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring
---

## Introducing Spring

今天我们站在前人的肩膀上，觉得 Spring 框架是 Java 程序员的基本功，但为什么会提出这个框架呢？前人帮我们踩过哪些坑呢？我们要先梳理一下 Spring 框架的由来。

从 Java 这门语言诞生以来，因为其语法简单、面向对象、跨平台等特性迅速在业界发展，基于 Java 的各种应用、服务、框架和第三方工具包都爆炸性增长。一方面，这代表了 Java 在业界的欢迎程度，但另一方面，也在不断增加程序员的开发成本、代码量也越来越大，这就带来了很多问题。最突出的有如下几个：

+ 分层架构中，上层调用与下层实现形成代码级关联，增加了设计的耦合度；
+ 第三方框架强制继承指定父类或实现指定接口，导致框架"侵入性"地绑死在应用上；
+ 散布在应用各个模块中的非功能性重复模块，没有得到充分且灵活的复用；

这些问题导致 Java 开发越发臃肿复杂，直到 Rod Johnson 发表《Expert One-On-One J2EE Development Without EJB》一书，阐述了轻量级框架的开发理念，并推出 interface21 框架，这是之后 Spring 框架的雏形。

## What is Spring

**Spring 是一种轻量级的控制反转 (IoC) 和面向切面编程 (AOP) 的容器框架。** 它最初的设计目标就是整合支持现有的框架技术和应用工具，让开发人员在实现过程中感觉就像使用简单的 JavaBean 一样。所以，Spring 并不是打造一个大而全的新框架，而是一个 "大杂烩"。

Spring 官网: http://spring.io/

GitHub: https://github.com/spring-projects

### 优点

1. Spring 是一个开源免费的框架；
2. Spring是一个轻量级的、非侵入式的容器；
3. 控制反转 IOC，面向切面编程 AOP；
4. 事务支持，框架的支持；

## Spring 组成

![img](/img/post/Spring/spring_components.png)

## Spring Boot 与 Spring Cloud

前面我们说到，因为 Java 各种框架太过臃肿，所以提出轻量级的 Spring 框架；但随着 Spring 越往后发展，配置文件越发复杂，所以就提出 Spring Boot 来简化文件的配置，Spring Boot 是 Spring 的一套快速配置脚手架，可以使开发人员快速部署单个微服务；随着微服务越来越多，就需要协调和编排它们，于是提出 Spring Cloud。

以上三者是逐渐发展的关系，具体可用下图所示。

![img](/img/post/Spring/spring_everything.png)

## 总结

本篇我们简单介绍了 Spring 的由来，先大体上了解其概貌，之后篇章详细学习各个部分。

