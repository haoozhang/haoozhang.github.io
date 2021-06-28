---
layout:     post
title:      Spring Cloud Gateway 作为服务网关
subtitle:   
date:       2021-06-18
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

本文学习 Spring Cloud 中的路由网关，并简单使用 Spring Cloud Gateway。

## What is Spring Cloud Gateway

Spring Cloud Gateway 是 Spring Cloud 官方推出的第二代网关框架，来取代 Zuul 网关。网关作为流量的入口，在微服务系统中有着非常重要的作用，常见功能有路由转发、权限校验、限流控制等。

在之前的文章中，我们讲述了如何使用 Eureka 作为服务注册发现中心，使用 Feign 和 LoadBalancer 做服务调用和负载均衡，使用 Sentinel 替代 Hystrix 实现服务熔断。本文我们学习如何使用 Spring Cloud Gateway 作为服务网关。

## 工作流程

![img](/img/SpringCloud/gateway.png)

如上图所示，客户端向 Spring Cloud Gateway 发出请求。\
如果 Gateway Handler Mapping 确定请求与路由匹配 (这时用到 **Predicate**)，则将其发送到Gateway web handler 处理。Predict作为断言，它决定了请求会被路由到哪个 router \
Gateway web handler 处理请求时会经过一系列的过滤器链 (这时用到 **Filter**)。过滤器链被虚线划分的原因是过滤器链可以在发送代理请求之前或之后执行过滤逻辑。\
先执行所有 "pre" 过滤器逻辑，然后进行代理请求。在收到代理服务的响应之后执行 "post" 过滤器逻辑。在执行所有 "pre" 过滤器逻辑时，往往进行了鉴权、限流、日志输出等功能，以及请求头的更改、协议的转换；转发之后收到响应之后，会执行所有 "post" 过滤器的逻辑，在这里可以响应数据进行了修改，比如响应头、协议的转换等。

## Predicate

在上面的处理过程中，有一个重要的点就是匹配请求和路由，这时候就需要用到 Predicate，它是决定了一个请求走哪一个路由。

Spring Cloud Gateway 内置了许多 Predict，这些 Predict 的源码在 org.springframework.cloud.gateway.handler.predicate 包中。现在列举各种 Predicate 如下图：

![img](/img/SpringCloud/predicate.png)

上图中有很多类型的 Predicate，比如时间类型的 Predicate (Before/Between/After Route Predicate Factory)，只有满足特定时间要求的请求才会进入 Predicate，并交给对应 router；cookie类型的 Cookie Route Predicate Factory，指定的 cookie 满足正则匹配，才会进入此 router；以及 host、method、path、querparam、remoteaddr 类型的 Predicate，每一种 Predicate 都会对当前的客户端请求进行判断，是否满足当前的要求，如果满足则交给当前请求处理。如果一个请求满足多个 Predicate，则按照配置的顺序第一个生效。

### After Route Predicate Factory

After Route Predicate Factory 可配置一个时间，当请求的时间在配置时间之后，才交给 router 去处理，否则报错，不通过路由。

```yml
spring:
  profiles:
    active: after_route

--- 
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: http://httpbin.org:80/get
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
  profiles: after_route
```

在上面的配置文件中，配置 spring.profiles.active:after_route 指定了程序的 spring 的启动文件为 after_route 文件。在 application.yml 再建一个配置文件，语法是三个横线，通过 spring.profiles 来配置文件名，和 spring.profiles.active 一致，然后配置 spring cloud gateway 相关的配置，id 标签配置的是 router id，每个 router 都需要一个唯一的id，uri 配置的是将请求路由到哪里。

predicates： After=2017-01-20T17:42:47.789-07:00[America/Denver] 会被解析成PredicateDefinition 对象 （name =After ，args= 2017-01-20T17:42:47.789-07:00[America/Denver]）。在这里需要注意的是 predicates 的After这个配置，遵循约定大于配置的思想，它实际被 AfterRoutePredicateFactory 这个类所处理，这个 After 就是指定了它的 Gateway web handler 类为 AfterRoutePredicateFactory，同理，其他类型的 predicate 也遵循这个规则。

### Header Route Predicate Factory
Header Route Predicate Factory 需要2个参数，一个是 header 名，另外一个 header 值，该值可以是一个正则表达式。当此断言匹配了请求的 header 名和值时，断言通过，进入到 router 的规则中。

```yml
spring:
  profiles:
    active: header_route

---
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: http://httpbin.org:80/get
        predicates:
        - Header=X-Request-Id, \d+
  profiles: header_route
```

在上面的配置中，当请求的 Header 中有 X-Request-Id 的 header 名，且 header 值为数字时，请求会被路由到配置的 uri。

### Cookie Route Predicate Factory
Cookie Route Predicate Factory需要2个参数，一个时cookie名字，另一个时值，可以为正则表达式。它用于匹配请求中，带有该名称的cookie和cookie匹配正则表达式的请求。

```yml
spring:
  profiles:
    active: cookie_route

---
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: http://httpbin.org:80/get
        predicates:
        - Cookie=name, forezp
  profiles: cookie_route
```

在上面的配置中，请求带有 cookie 名为 name, cookie 值为 forezp 的请求将都会转发到 uri

### Host Route Predicate Factory
Host Route Predicate Factory需要一个参数即hostname，它可以使用. * 等去匹配host。这个参数会匹配请求头中的host的值，一致，则请求正确转发。

```yml
spring:
  profiles:
    active: host_route
---
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: http://httpbin.org:80/get
        predicates:
        - Host=**.fangzhipeng.com
  profiles: host_route
```

在上面的配置中，请求头中含有 Host 为 fangzhipeng.com 的请求将会被路由转发转发到配置的 uri。

### Method Route Predicate Factory

Method Route Predicate Factory 需要一个参数，即请求的类型。比如GET类型的请求都转发到此路由。
```yml
spring:
  profiles:
    active: method_route

---
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://httpbin.org:80/get
        predicates:
        - Method=GET
  profiles: method_route
```

在上面的配置中，所有的 GET 类型的请求都会路由转发到配置的 uri。

### Path Route Predicate Factory

Path Route Predicate Factory 需要一个参数: 一个spel表达式，应用匹配路径。
```yml
spring:
  profiles:
    active: path_route

---
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: http://httpbin.org:80/get
        predicates:
        - Path=/foo/{segment}
  profiles: path_route
```

在上面的配置中，所有的请求路径满足/foo/{segment}的请求将会匹配并被路由，比如/foo/1 、/foo/bar的请求，将会命中匹配，并成功转发。

### Query Route Predicate Factory

Query Route Predicate Factory 需要2个参数: 一个参数名和一个参数值的正则表达式。
```yml
spring:
  profiles:
    active: query_route
---
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: http://httpbin.org:80/get
        predicates:
        - Query=foo, ba.
  profiles: query_route
```

在上面的配置文件中，配置了请求中含有参数foo，并且foo的值匹配ba.，则请求命中路由，比如一个请求中含有参数名为foo，值的为bar，能够被正确路由转发。

Query Route Predicate Factory也可以只填一个参数，填一个参数时，则只匹配参数名，即请求的参数中含有配置的参数名，则命中路由。

## Filter

Predict 决定了请求由哪一个路由处理，在路由处理之前，需要经过 "pre" 类型的过滤器处理，处理返回响应之后，可以由 "post" 类型的过滤器处理。

filter 有着非常重要的作用，在“pre”类型的过滤器可以做参数校验、权限校验、流量监控、日志输出、协议转换等，在“post”类型的过滤器中可以做响应内容、响应头的修改，日志的输出，流量监控等。

### 作用

当我们有很多个服务时，比如 user-service、goods-service、sales-service 等服务，客户端请求各个服务的Api时，每个服务都需要做相同的事情，比如鉴权、限流、日志输出等。

对于这样重复的工作，有没有办法做的更好，答案是肯定的。在微服务的上一层加一个全局的权限控制、限流、日志输出的 Api Gateway 服务，然后再将请求转发到具体的业务服务层。这个 Api Gateway 服务就是起到一个服务边界的作用，外接的请求访问系统，必须先通过网关层。

### 生命周期

Spring Cloud Gateway 同 zuul 类似，有 "pre" 和 "post" 两种方式的filter。客户端的请求先经过 "pre" 类型的 filter，然后将请求转发到具体的业务服务，比如上图中的 user-service，收到业务服务的响应之后，再经过 "post" 类型的 filter 处理，最后返回响应到客户端。

![img](/img/SpringCloud/filter_lifecycle.png)

与 zuul 不同的是，filter 除了分为 "pre" 和 "post" 两种方式的 filter 外，在 Spring Cloud Gateway 中，filter 从作用范围可分为另外两种，一种是针对于单个路由的 gateway filter，它在配置文件中的写法同 predict 类似；另外一种是针对于所有路由的 global gateway filer。现在从作用范围划分的维度来讲解这两种 filter。

### gateway filter

过滤器允许以某种方式修改传入的HTTP请求或传出的HTTP响应。过滤器可以限定作用在某些特定请求路径上。 Spring Cloud Gateway 包含许多内置的 GatewayFilter 工厂。

GatewayFilter 工厂同上一篇介绍的 Predicate 工厂类似，都是在配置文件 application.yml 中配置，遵循了约定大于配置的思想，只需要在配置文件配置 GatewayFilter Factory 的名称，而不需要写全部的类名，比如 AddRequestHeaderGatewayFilterFactory 只需要在配置文件中写 AddRequestHeader，而不是全部类名。在配置文件中配置的 GatewayFilter Factory 最终都会相应的过滤器工厂类处理。

Spring Cloud Gateway 内置的过滤器工厂在 org.springframework.cloud.gateway.filter.factory 中。现在挑几个常见的过滤器工厂来学习。

![img](/img/SpringCloud/gateway_filter.png)

#### AddRequestHeader GatewayFilter Factory

```yml
spring:
  profiles:
    active: add_request_header_route

---
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: http://httpbin.org:80/get
        filters:
        - AddRequestHeader=X-Request-Foo, Bar
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
  profiles: add_request_header_route
```

在上述的配置中，配置文件为add_request_header_route，其中配置了 route id、路由地址、一个 Predicate 和 Filter。AddRequestHeader过滤器工厂会在请求头加上一对请求头，名称为X-Request-Foo，值为Bar，查看它的源码：

```java
public class AddRequestHeaderGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {

	@Override
	public GatewayFilter apply(NameValueConfig config) {
		return (exchange, chain) -> {
			ServerHttpRequest request = exchange.getRequest().mutate()
					.header(config.getName(), config.getValue())
					.build();

			return chain.filter(exchange.mutate().request(request).build());
		};
    }
}
```

由上面的代码可知，根据旧的 ServerHttpRequest 创建新的 ServerHttpRequest，在新的 ServerHttpRequest 加了一个请求头，然后创建新的 ServerWebExchange ，提交过滤器链继续过滤。

#### RewritePath GatewayFilter Factory

在Nginx服务启中有一个非常强大的功能就是重写路径，Spring Cloud Gateway默认也提供了这样的功能，这个功能是Zuul没有的。在配置文件中加上以下的配置：

```yml
spring:
  profiles:
    active: rewritepath_route
---
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://blog.csdn.net
        predicates:
        - Path=/foo/**
        filters:
        - RewritePath=/foo/(?<segment>.*), /$\{segment}
  profiles: rewritepath_route
```
所有的/foo/\*\*开始的路径都会命中配置的router，并执行过滤器的逻辑，此工厂将/foo/(?.*)重写为{segment}，然后转发到https://blog.csdn.net。比如在网页上请求localhost:8081/foo/forezp，此时会将请求转发到https://blog.csdn.net/forezp的页面

#### 自定义过滤器

在 Spring Cloud Gateway 中，过滤器需要实现 GatewayFilter 和 Ordered2 个接口。写一个 RequestTimeFilter，代码如下：
```java
public class RequestTimeFilter implements GatewayFilter, Ordered {

    private static final Log log = LogFactory.getLog(GatewayFilter.class);
    private static final String REQUEST_TIME_BEGIN = "requestTimeBegin";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

        exchange.getAttributes().put(REQUEST_TIME_BEGIN, System.currentTimeMillis());
        return chain.filter(exchange).then(
                Mono.fromRunnable(() -> {
                    Long startTime = exchange.getAttribute(REQUEST_TIME_BEGIN);
                    if (startTime != null) {
                        log.info(exchange.getRequest().getURI().getRawPath() + ": " + (System.currentTimeMillis() - startTime) + "ms");
                    }
                })
        );

    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

在上面的代码中，Ordered中的int getOrder()方法是来给过滤器设定优先级别的，值越大则优先级越低。还有有一个filterI(exchange,chain)方法，在该方法中，先记录了请求的开始时间，并保存在ServerWebExchange 中，此处是一个 "pre" 类型的过滤器，然后再 chain.filter 的内部类中的run()方法中相当于 "post" 过滤器，在此处打印了请求所消耗的时间。然后将该过滤器注册到 router中，代码如下：
```java
@Bean
public RouteLocator customerRouteLocator(RouteLocatorBuilder builder) {
    // @formatter:off
    return builder.routes()
            .route(r -> r.path("/customer/**")
                    .filters(f -> f.filter(new RequestTimeFilter())
                            .addResponseHeader("X-Response-Default-Foo", "Default-Bar"))
                    .uri("http://httpbin.org:80/get")
                    .order(0)
                    .id("customer_filter_router")
            )
            .build();
    // @formatter:on
}
```

### global filter

Spring Cloud Gateway根据作用范围划分为GatewayFilter和GlobalFilter，二者区别如下：

+ GatewayFilter : 需要通过spring.cloud.routes.filters 配置在具体路由下，只作用在当前路由上或通过spring.cloud.default-filters配置在全局，作用在所有路由上

+ GlobalFilter : 全局过滤器，不需要在配置文件中配置，作用在所有的路由上，最终通过GatewayFilterAdapter包装成GatewayFilterChain可识别的过滤器，它为请求业务以及路由的URI转换为真实业务服务的请求地址的核心过滤器，不需要配置，系统初始化时加载，并作用在每个路由上。

Spring Cloud Gateway框架内置的GlobalFilter如下：

![img](/img/SpringCloud/global_filter.png)

上图中每一个GlobalFilter都作用在每一个router上，能够满足大多数的需求。但是如果遇到业务上的定制，可能需要编写满足自己需求的GlobalFilter。在下面的案例中将讲述如何编写自己GlobalFilter，该GlobalFilter会校验请求中是否包含了请求参数“token”，如何不包含请求参数“token”则不转发路由，否则执行正常的逻辑。代码如下：

```java
public class TokenFilter implements GlobalFilter, Ordered {

    Logger logger=LoggerFactory.getLogger( TokenFilter.class );
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getQueryParams().getFirst("token");
        if (token == null || token.isEmpty()) {
            logger.info( "token is empty..." );
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```

在上面的TokenFilter需要实现GlobalFilter和Ordered接口，这和实现GatewayFilter很类似。然后根据ServerWebExchange获取ServerHttpRequest，然后根据ServerHttpRequest中是否含有参数token，如果没有则完成请求，终止转发，否则执行正常的逻辑。

然后需要将 TokenFilter 在工程的启动类中注入到 Spring IoC 容器中，代码如下：

```java
@Bean
public TokenFilter tokenFilter(){
        return new TokenFilter();
}
```

## Gateway 实战

在介绍完 Spring Cloud Gateway 的 Predicate 和 Filter 之后，我们通过修改之前的工程来实战体验 Spring Cloud Gateway 的功能。

1、新建 gateway 模块，并引入以下依赖，sc gateway 用于实现网关功能，eureka 用于该服务的注册和发现

```xml
<dependencies>
    <!-- sc gateway -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
    <!-- eureka -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        <version>3.0.3</version>
    </dependency>
    <!-- 完善监控信息 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

2、配置文件中添加相应配置

```yml
server:
  port: 5000

spring:
  application:
    name: gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false
          lower-case-service-id: true

eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    instance-id: springcloud-gateway-5000  # 修改默认的描述信息

# actuator/info 配置
info:
  app.name: springcloud-gateway
  company.name: hao.blog
```

可以看到，除了 spring.cloud.gateway 以外，其它均为常见配置。在 gateway 配置中，我们设置 enabled 为 false，表示不能通过 DiscoveryClient 自动根据服务 ID 创建路由；设置 lower-case-service-id 为 true，表示使用小写的服务 ID。

3、创建路由配置类，来添加网关的路由

```java
@Configuration
public class RouteConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                // 增加 path 匹配，以 /customize/dept/** 开头的请求都在此路由
                .route(r -> r.path("/customize/dept/**")
                        // 跳过第一个前缀，剩下的路径与 uri 拼接
                        .filters(f -> f.stripPrefix(1)
                                // 添加响应首部
                                .addResponseHeader("extendTag", "gateway-" + System.currentTimeMillis()))
                        // 指定匹配服务 provider，lb 是 load balance 的意思
                        .uri("lb://PROVIDER"))
                .build();
    }
}
```

之前我们都是在配置文件中添加路由，这里我们通过代码实现，可以看到添加了一个 path 匹配的路由，并使用 filter 添加了响应的首部

4、创建主启动类

```java
@SpringBootApplication
@EnableEurekaClient
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

5、依次启动 eureka、provider、provider2、gateway，通过 eureka 页面可以看到服务都已注册

![img](/img/SpringCloud/gateway_eureka.png)

然后我们访问网关 http://localhost:5000/customize/dept/get/1，可以得到如下输出

```
{"id":1,"name":"develop","dbSource":"cloud"}
```

多次刷新访问，可以看到负载均衡的效果，输出在 cloud 和 cloud02 两个数据库获取数据

观察返回的响应首部，可以看到我们添加的信息

![img](/img/SpringCloud/gateway_output.png)

参考自：
1. [极速体验SpringCloud Gateway](https://blog.csdn.net/boling_cavalry/article/details/94907172)
2. [Spring Cloud Gateway 之Predict篇](https://www.fangzhipeng.com/springcloud/2018/12/05/sc-f-gateway2.html)
3. [Spring Cloud Gateway之Filter篇](https://www.fangzhipeng.com/springcloud/2018/12/05/sc-f-gateway2.html)


