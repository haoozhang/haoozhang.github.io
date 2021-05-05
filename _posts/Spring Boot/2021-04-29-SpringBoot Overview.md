---
layout:     post
title:      1. SpringBoot Overview
subtitle:   
date:       2021-04-29
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SpringBoot
---

## What is SpringBoot

Spring 是一个轻量级的 Java 开发框架，是为了解决企业级应用开发的复杂性而诞生的。它采用了以下四种关键策略来降低 Java 开发的复杂性：

+ 所有 pojo 都注册为 bean 实例；
+ 控制反转 IOC 和面向接口编程实现低耦合；
+ 切面编程 AOP 和声明式编程；
+ 通过切面和模板 xxTemplate 减少样式代码；

而随着 Spring 的不断发展，项目开发需要配置越来越多的配置文件，慢慢变成了 *配置地狱*，这违背了当初设计的理念。SpringBoot 正是在这样的一个背景下被抽象出来的开发框架，目的是为了让大家更容易的使用 Spring、更容易的集成各种常用的中间件、开源软件。

SpringBoot 基于 Spring 开发，用于快速、敏捷地开发新一代基于 Spring 框架的应用程序。它并不是用来替代 Spring，而是和 Spring 框架紧密结合以提升 Spring 开发者的体验。以约定大于配置的核心思想，默认帮我们进行了很多设置，多数 SpringBoot 应用只需要几行 Spring 配置。同时它集成了大量常用的第三方库配置（例如 Redis、MongoDB、Jpa、RabbitMQ、Quartz 等），Spring Boot 应用中这些第三方库几乎可以零配置的开箱即用。

SpringBoot 的主要优点：
+ 更快入门搭建 Spring 应用；
+ 开箱即用，各种默认配置来简化项目配置；
+ 内嵌式容器简化 Web 项目；
+ 无需生成冗余代码和XML配置；

## Hello SpringBoot

接下来我们创建第一个 SpringBoot 应用。官方提供了两种快速构建应用的方式：
+ 通过 https://start.spring.io/ 生成项目，然后下载导入到本地 IDEA 开发环境中；
+ 直接通过 IDEA Spring Initializr 创建；

创建好之后会自动生成一下项目结构：
+ 程序的主启动类
+ resources 目录下的 application.properties 配置文件
+ 测试类
+ pom.xml

### pom 文件分析

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    
    <groupId>com.zhao</groupId>
    <artifactId>hellospringboot</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>HelloSpringBoot</name>
    <description>Demo project for Spring Boot</description>
    
    <properties>
        <java.version>1.8</java.version>
    </properties>
    
    <dependencies>

        <!-- web 容器：  -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 单元测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <!-- 打包插件 -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### 编写接口

在主程序相同目录下新建 controller 包，包中新建 HelloController 类

```java
@RestController
public class HelloController {

    @GetMapping("/hello")
    public String hello() {
        return "hello, springboot";
    }
}
```

然后启动项目测试，可以从 console 窗口中看到项目自动启动了 Tomcat、加载了 dispatcherServlet，我们访问 localhost 测试。

### 项目打包

我们使用 maven 的 package 命令打包整个项目，打包成功后会在 target 目录下生成一个 jar 包，我们就可以在任何地方运行了！

## 总结

本篇我们了解了 SpringBoot 并搭建了 SpringBoot 的第一个应用，下一节我们将深入其中探究其原理。

参考自：
1. [狂神说SpringBoot01：Hello,World！](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483724&idx=1&sn=77ce80187dbfdbaaafa0366f6a0c9151&scene=19#wechat_redirect)
