---
layout:     post
title:      Spring Cloud Feign 
subtitle:   声明式 REST 客户端
date:       2021-06-16
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

之前文章讲述了如何通过 RestTemplate 来消费服务，本文学习 Spring Cloud 中的另一个负载均衡工具 Feign，并通过它来消费服务

## What is Feign

Feign 是声明式的 Web Service 客户端。它简化了微服务之间的调用，之前我们使用 RestTemplate 时通过微服务名字来消费服务，使用 Feign 之后我们可以通过类似 controller 调用 service 的方式消费服务。Feign 默认集成了负载均衡。

在 Feign 的实现下，我们只需要创建一个接口并添加注解配置，即完成了对服务提供放的接口绑定，然后消费方的 controller 就以接口的方式调用远程的服务，简化了使用 RestTemplate 时封装服务调用客户端的开发量。

## Feign 消费服务

首先我们新建 consumerFeign 的模块，模块 src 目录下的代码和配置文件与原来的 consumer 一样，我们只需要简单修改和配置

1、添加 Feign 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2、配置文件中修改应用名和描述信息

```yml
spring:
  application:
    name: consumerFeign

eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    instance-id: springcloud-consumerFeign-8080  # 修改默认的描述信息

# actuator/info 配置
info:
  app.name: springcloud-consumerFeign-dept
  company.name: hao.blog
```

3、创建接口和注解

```java
@FeignClient(value = "PROVIDER")  // 指定服务提供方
public interface FeignService {

    @PostMapping("/dept/add")  // 对应的服务提供方访问路径
    boolean addDept(Dept dept);  // 提供的服务

    @GetMapping("/dept/get/{id}")
    Dept selectDeptById(@PathVariable("id") Long id);

    @GetMapping("/dept/get")
    List<Dept> selectAllDept();
}
```

4、修改 consumer 的 controlller 层，以接口方式调用

```java
@RestController
public class DeptController {

    @Autowired
    private FeignService feignService;

    @GetMapping("/dept/get/{id}")
    public Dept getDept(@PathVariable("id") Long id) {
        return feignService.selectDeptById(id);
    }

    @GetMapping("/dept/get")
    public List<Dept> getAllDept() {
        return feignService.selectAllDept();
    }

    @RequestMapping("/dept/add")
    public boolean addDept(Dept dept) {
        return feignService.addDept(dept);
    }
}
```

5、在主启动类添加 @EnableFeignClients 注解，开启扫描创建的 Feign 接口

```java
....
@EnableFeignClients  // 开启扫描 Feign 接口
public class DeptConsumerFeign {
    ....
}
```

6、依次启动 Eureka、provider、provider2、consumerFeign 项目，测试访问服务，可看到服务访问和负载均衡的效果

参考自：
1. [[狂神说Java]SpringCloud最新教程IDEA版](https://www.bilibili.com/video/BV1jJ411S7xr?p=13)
2. [SpringCloud教程第3篇：feign（F版本）](https://www.fangzhipeng.com/springcloud/2018/08/03/sc-f3-feign.html)