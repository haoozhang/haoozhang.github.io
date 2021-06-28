---
layout:     post
title:      Spring Cloud Circuit Breaker 服务熔断
subtitle:   以 Sentinel 实现服务熔断
date:       2021-06-17
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

## What is Circuit Breaker

多个微服务之间调用时，假设微服务 A 调用微服务 B 和 C，微服务 B 和 C 又调用其他的微服务，此时调用链路上某个微服务的调用响应时间过长或不可用，对微服务 A 的调用就会占用越来越多的系统资源，进而导致系统崩溃，这就是"雪崩"效应。

对于高流量应用来说，单一的后端依赖可能会导致所有服务器上的资源在几秒内饱和，导致整个系统发生更多的级联故障。因此我们需要对故障和延迟隔离管理，以达到单个依赖关系的失败不影响整个系统的目标。

## What is Sentinel

Sentinel，中文译为哨兵，是为微服务提供流量控制、熔断降级的功能，它和 Hystrix 提供的功能一样，可以有效解决微服务调用时可能产生的"雪崩"效应，为微服务系统提供了稳定性的解决方案。

随着 Hytrxi 进入了维护期，不再提供新功能，Sentinel 是一个不错的替代方案。通常情况，Hystrix 采用线程池对服务的调用进行隔离，Sentinel 采用了用户线程对接口进行隔离，二者相比，Hystrxi 是服务级别的隔离，Sentinel 提供了接口级别的隔离，Sentinel 隔离级别更加精细，另外 Sentinel 直接使用用户线程进行限制，相比 Hystrix 的线程池隔离，减少了线程切换的开销。另外 Sentinel DashBoard 提供了在线更改限流规则的配置，也更加方便。

从官方文档介绍，Sentinel 具有以下特征：
+ 丰富的应用场景： Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀 (即突发流量控制在系统容量可以承受的范围)、消息削峰填谷、实时熔断下游不可用应用等。
+ 完备的实时监控： Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。

+ 广泛的开源生态： Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Dubbo、gRPC 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。

+ 完善的 SPI 扩展点： Sentinel 提供简单易用、完善的 SPI 扩展点。您可以通过实现扩展点，快速的定制逻辑。例如定制规则管理、适配数据源等。

## Sentinel 实现熔断

### 下载 sentinel dashboard

Sentinel dashboard提供一个轻量级的控制台，它提供机器发现、单机资源实时监控、集群资源汇总，以及规则管理的功能。只需要对应用进行简单的配置，就可以使用这些功能。

sentinel dashboard 下载地址为https://github.com/alibaba/Sentinel/releases

下载完启动，设置端口为8001，命令如下：
```
java -Dserver.port=8001 -Dcsp.sentinel.dashboard.server=localhost:8001 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.1.jar
```

### 改造之前创建的 consumerFeign

在 pom.xml 文件引入 spring cloud sentinel 的依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    <version>2021.1</version>
</dependency>
```

配置 application.yml 文件，通过 feign.sentinel.enable 开启 Feign 和 sentinel 的自动适配，然后配置sentinel的dashboard的地址

```yml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8001
feign:
  sentinel:
    enabled: true
```

通过这样的配置，sentinel 就配置好了。\
依次启动 eureka、provider、consumerFeign，在浏览器上访问几次服务

打开 localhost:8001，登录 sentinel 的控制台，登录名和密码都是 sentinel

### 流量控制

对 PROVIDER 服务的 /dept/get 接口，增加一条流控规则：

![img](/img/SpringCloud/rate_limit.png)

+ 资源名：默认为请求路径
+ 针对来源：Sentinel 可以针对调用者进行限流，填写微服务名，默认default (不区分来源)
+ 阈值类型/单机阈值：
    + QPS (每秒钟的请求数量)：当调用该 api 的 QPS 达到阈值的时候，进行限流
    + 线程数：当调用该 api 的线程数达到阈值的时候，进行限流

阈值类型 QPS 和 线程数的区别：QPS 方式是达到阈值时直接挡在外面，线程数是有多少个线程在处理，先把请求放进来，有空闲线程就处理，没有空闲的就限流

上图我们设置 QPS 为 1，快速访问 http://localhost:8080/dept/get，当每秒超过1个请求时，就会有失败的情况

### 服务熔断

服务熔断是指，当复杂调用链路上的某一环不稳定，对不稳定的弱依赖服务调用进行熔断降级，暂时切断不稳定调用，避免局部不稳定因素导致整体的雪崩。熔断降级作为保护自身的手段，通常在客户端（调用端）进行配置。

Sentinel 提供如下熔断策略：
+ 慢调用比例 (SLOW_REQUEST_RATIO)：以慢调用比例作为阈值，需要设置允许的慢调用 RT (即最大响应时间)，请求的响应时间大于该值则统计为慢调用。当单位统计时长（statIntervalMs）内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求自动被熔断。\
经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。
+ 异常比例 (ERROR_RATIO)：当单位统计时长（statIntervalMs）内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。异常比率的阈值范围是 [0.0, 1.0]，代表 0% - 100%。
+ 异常数 (ERROR_COUNT)：当单位统计时长内的异常数目超过阈值之后会自动进行熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。

![img](/img/SpringCloud/rongduan.png)

参考自：
1. [SpringCloud学习笔记11-熔断与限流Sentinel](hhttps://saturnblog.space/2021/02/22/SpringCloud%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B011-%E7%86%94%E6%96%AD%E4%B8%8E%E9%99%90%E6%B5%81Sentinel/)

2. [SpringCloud 2020版本教程3：使用sentinel作为熔断器](https://www.fangzhipeng.com/springcloud/2021/04/04/sc-2020-sentinel.html)