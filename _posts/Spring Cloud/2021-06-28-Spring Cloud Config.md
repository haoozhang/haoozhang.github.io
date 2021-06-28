---
layout:     post
title:      Spring Cloud Config 实现分布式配置
subtitle:   
date:       2021-06-28
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

本文学习 Spring Cloud Config 来实现分布式配置。

## 分布式系统面临的配置问题

微服务架构意味着要将单体应用中的业务拆分为一个个子服务，每个服务的粒度较小，因此系统中会出现大量的服务。由于每个服务都需要必要的配置信息才能运行，所以需要一套集中式的、动态配置管理方案。Spring Cloud 提供了 Config Server 来解决此问题。

## What is Spring Cloud Config

Spring Cloud Config 为系统中的微服务提供了集中化的外部配置支持，它的运行流程如下：

![img](/img/SpringCloud/config_process.png)

Spring Cloud Config 分为 server 和 client 两部分：
+ 服务端也即分布式配置中心，它是一个独立的微服务应用，用于连接配置服务器并为客户端提供获取配置信息的接口
+ 客户端通过指定的配置中心管理配置信息，并在启动时从配置中心获取和加载配置信息。配置服务器默认采用 git 存储配置信息，有助于对环境配置进行版本管理。

## 配置信息存储

我们先创建一个 git 仓库来保存配置信息，其中只需要包含 .yml 配置文件即可，可参考[这里](https://github.com/haozhangms/SpringCloudConfig)

```yml
# application.yml
spring:
  profiles:
    active: dev

---
spring:
  application:
    name: config-dev
  config:
    activate:
      on-profile: dev

---
spring:
  application:
    name: config-test
  config:
    activate:
      on-profile: test
```

存储好配置信息后，我们创建 Config Server 来连接这个仓库，并获取配置信息

## Config Server

1、新建 Maven 模块，引入以下依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- eureka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>3.0.3</version>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

2、创建配置文件，并添加如下配置：

```yml

server:
  port: 3344

spring:
  application:
    name: configServer
  cloud:
    config:
      server:
        git:
          uri: https://github.com/haozhangms/SpringCloudConfig.git  # repo 地址
          default-label: main  # Github 默认分支改为 main，但这里默认仍为 master
          skip-ssl-validation: true

# Eureka 配置
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

端口和 Eureka 的配置不再赘述，通过 uri 字段指定了配置信息存储的仓库，default-label 字段指定了获取的分支，skip-ssl-validation 字段跳过了 Github SSL 验证，可加快 Github 的访问速度

3、创建主启动类，并添加 @EnableConfigServer 注解

```java
@SpringBootApplication
@EnableConfigServer  // 开启 Spring Cloud Config
public class ConfigServer {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServer.class, args);
    }
}
```

4、启动项目测试，因为我们在 git 仓库中存储了 application.yml 配置文件，并指定了 dev 和 test 两个环境，这里我们访问 http://localhost:3344/application-dev.yml，可获取到如下配置信息：

```yml
spring:
  profiles:
    active: dev
  application:
    name: config-dev
  config:
    activate:
      on-profile: dev
```

## Config Client

创建好 Config Server 之后，我们可以创建 Config Client 来实现从 Config Server 中加载配置信息

1、新建  Maven 模块，引入 config-client 的依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- eureka -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <version>3.0.3</version>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

2、创建配置文件 application.yml，添加配置信息

```yml
spring:
  application:
    name: configClient
```

这里我们只指定了应用的名称，那我们如何指定从 Config Server 中加载配置信息呢？
我们创建 bootstrap.yml 文件，这个文件也可以被识别为 Spring Boot 的配置文件，用于系统级别的配置，我们添加如下配置信息

```yml
spring:
  cloud:
    config:
      uri: http://localhost:3344  # 从 config 服务器获取
      name: config-client  # 需要读取的资源名称
      profile: test
      label: main
```

uri 字段指定了 Config Server 的地址，name 指定了 要获取的配置文件名称，profile 是指对应的环境，label 指定分支

为了启用 bootstrap，我们添加如下依赖

```xml
<!-- 启用 bootstrap，Spring Cloud 2020 版默认禁用 bootstrap -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

我们在存储配置信息的 git 仓库中添加如下信息的文件

```yml
# config-client.yml
spring:
  profiles:
    active: dev

---
server:
  port: 8201

spring:
  application:
    name: provider
  config:
    activate:
      on-profile: dev
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka

---
server:
  port: 8202

spring:
  application:
    name: provider
  config:
    activate:
      on-profile: test
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
```

3、编写 controller 接口获取配置信息

```java
@RestController
public class ConfigClientController {

    @Value("${spring.application.name}")
    private String appName;

    @Value("${eureka.client.service-url.defaultZone}")
    private String eurekaServer;

    @Value("${server.port}")
    private String port;

    @GetMapping("/getConfig")
    public String getConfig() {
        return "appName: " + appName +
               "\neurekaServer: " + eurekaServer +
               "\nport: " + port;
    }
}
```

4、编写主启动类

```java
@SpringBootApplication
public class ConfigClient {

    public static void main(String[] args) {
        SpringApplication.run(ConfigClient.class, args);
    }
}
```

5、启动项目测试，因为我们在 bootstrap.yml 文件中指定了 test 环境，所以可以看到该工程在 8202 端口启动，访问 http://localhost:8202/getConfig，可看到如下信息，说明成功从 Config Server 中加载配置信息

```
appName: provider eurekaServer: http://localhost:7001/eureka port: 8202
```

参考自：
1. [[狂神说Java] SpringCloud最新教程IDEA版](https://www.bilibili.com/video/BV1jJ411S7xr?p=18)


