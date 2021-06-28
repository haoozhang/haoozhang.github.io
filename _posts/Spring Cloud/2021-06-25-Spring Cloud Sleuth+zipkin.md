---
layout:     post
title:      Spring Cloud Sleuth + zipkin 实现链路追踪
subtitle:   
date:       2021-06-25
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

本文学习 Spring Cloud Sleuth 来实现链路追踪。

## What is Spring Cloud Sleuth + zipkin

Spring Cloud Sleuth 主要功能就是在分布式系统中提供追踪解决方案，并且兼容支持了 zipkin，只需要在pom文件中引入相应的依赖即可。

微服务架构上通过业务来划分服务的，通过 REST 调用，对外暴露的一个接口，可能需要很多个服务协同才能完成这个接口功能，如果链路上任何一个服务出现问题或者网络超时，都会形成导致接口调用失败。随着业务的不断扩张，服务之间互相调用会越来越复杂。

![img](/img/SpringCloud/sleuth1.png)

本文讲述如何使用 sleuth 和 zipkin 来构建微服务的链路追踪。

## 实战 Spring Cloud Sleuth + zipkin

### 下载 zipkin server 并启动

```
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

访问 zipkin 的 ui 界面，地址为 localhost:9411

### 改造工程

在工程 provider\consumer\gateway 的 pom 文件加上以下的依赖：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

工程的配置文件 application.yml 加上以下的配置：

```yml
spring:
  application:
    name: consumer
  zipkin: # sleuth + zipkin 配置
    sender:
      type: web
    base-url: http://localhost:9411/
    service:
      name: consumer
  sleuth:
    sampler:
      probability: 1
```

zipkin server支持mq方式收集链路信息，同时支持多种数据存储方式，比如es\mysql等，更多收集数据和存储方式见：https://github.com/openzipkin/zipkin/tree/master/zipkin-server

启动三个工程，在浏览器上访问：localhost:5000/customize/dept/get/

访问 zipkin server 的 api ，可以看到如下的服务依赖：

![img](/img/SpringCloud/zipkin_ui.png)


参考自：
1. [使用 spring cloud sleuth + zipkin 实现链路追踪](hhttps://www.fangzhipeng.com/springcloud/2021/04/05/sc-2020-sleuth.html)


