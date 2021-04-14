---
layout:     post
title:      1. SpringMVC Overview
subtitle:   
date:       2021-04-13
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring MVC
---

## 回顾 MVC

MVC 是 Model、View、Controller 的简写，是一种软件设计规范。通过将业务逻辑、数据、显示分离的方法组织代码，它降低了视图与业务逻辑间的双向耦合。

+ Model (模型)：数据模型，提供要展示的数据，因此包含数据和行为，可以认为是领域模型或JavaBean组件（包含数据和行为），不过现在一般都分离开来：Value Object（数据Dao） 和 服务层（行为Service）。也就是模型提供了模型数据查询和模型数据的状态更新等功能，包括数据和业务。

+ View (视图)：负责模型的展示，一般就是客户端见到的用户界面。

+ Controller (控制器)：接收用户请求，委托给模型进行处理（状态改变），处理完毕后把返回的模型数据返回给视图，由视图负责展示。也就是说控制器做了调度的工作。

最典型的MVC就是JSP + servlet + javabean的模式。

![img](/img/post/SpringMVC/mvc.png)

### Model 1 时代

在 Web 开发早期，通常采用 Model 1 架构。Model 1 架构主要分为两层：视图层和模型层。

![img](/img/post/SpringMVC/model1.png)

Model 1 的优点是架构简单，比较适合小型项目的开发；但缺点也显而易见，JSP 的职责不单一，后期不便于维护。

### Model 2 时代

Model 2 架构把一个项目分为三部分：视图、控制、模型。

![img](/img/post/SpringMVC/model2.png)

+ 用户发请求
+ Servlet接收请求数据，并调用对应的业务逻辑方法
+ 业务处理完毕，返回更新后的数据给servlet
+ servlet转向到JSP，由JSP来渲染页面
+ 响应给前端更新后的页面

Controller：控制器

取得表单数据

调用业务逻辑

转向指定的页面

Model：模型

业务逻辑

保存数据的状态

View：视图

显示页面

Model 2 不仅提高了代码的复用率与项目的扩展性，且大大降低了项目的维护成本。Model 1模式的实现比较简单，适用于快速开发小规模项目，Model1中JSP页面身兼View和Controller两种角色，将控制逻辑和表现逻辑混杂在一起，从而导致代码的重用性非常低，增加了应用的扩展性和维护的难度。Model2消除了Model1的缺点。

## 回顾 Servlet

1、新建 Maven 工程，并引入以下依赖

```xml
<dependencies>
   <dependency>
       <groupId>junit</groupId>
       <artifactId>junit</artifactId>
       <version>4.12</version>
   </dependency>
   <dependency>
       <groupId>org.springframework</groupId>
       <artifactId>spring-webmvc</artifactId>
       <version>5.1.9.RELEASE</version>
   </dependency>
   <dependency>
       <groupId>javax.servlet</groupId>
       <artifactId>servlet-api</artifactId>
       <version>2.5</version>
   </dependency>
   <dependency>
       <groupId>javax.servlet.jsp</groupId>
       <artifactId>jsp-api</artifactId>
       <version>2.2</version>
   </dependency>
   <dependency>
       <groupId>javax.servlet</groupId>
       <artifactId>jstl</artifactId>
       <version>1.2</version>
   </dependency>
</dependencies>
```

2、右键工程 Add Framework Support，添加 Web App 的支持

3、编写一个 Servlet 类用于处理用户的请求

```java
//实现Servlet接口
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 获取参数
        String method = req.getParameter("method");
        if (method.equals("add")) {
            req.getSession().setAttribute("msg", " execute add function");
        }
        if (method.equals("delete")) {
            req.getSession().setAttribute("msg", " execute delete function");
        }
        // 业务逻辑

        // 视图跳转
        req.getRequestDispatcher("/WEB-INF/jsp/hello.jsp").forward(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        doGet(req, resp);
    }
}
```

4、在 WEB-INF 目录下新建一个 jsp 文件夹，新建 hello.jsp

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
   <title>Kuangshen</title>
</head>
<body>
${msg}
</body>
</html>
```

5、在 web.xml 中注册 Servlet

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
        version="4.0">
   <servlet>
       <servlet-name>HelloServlet</servlet-name>
       <servlet-class>com.kuang.servlet.HelloServlet</servlet-class>
   </servlet>
   <servlet-mapping>
       <servlet-name>HelloServlet</servlet-name>
       <url-pattern>/user</url-pattern>
   </servlet-mapping>

</web-app>
```

6、配置 Tomcat

![img](/img/post/SpringMVC/tomcat.png)

点击 fix 自动部署打包，或者手动添加

![img](/img/post/SpringMVC/tomcat_deploy.png)

7、 启动测试

```
localhost:8080/user?method=add
localhost:8080/user?method=delete
```

MVC框架要做哪些事情?
1. 将url映射到java类或java类的方法 .
2. 封装用户提交的数据 .
3. 处理请求--调用相关的业务处理--封装响应数据 .
4. 将响应的数据进行渲染 .jsp / html 等表示层数据 .

## Introducing SpringMVC

Spring MVC 是 Spring Framework 的一部分，是基于 Java 实现 MVC 的轻量级 Web 框架。

+ 轻量级，简单易学
+ 高效，基于请求响应的MVC框架
+ 无缝兼容 Spring
+ 约定优于配置
+ 功能强大：RESTful、数据验证、格式化、本地化、主题等

Spring 的 web 框架围绕 DispatcherServlet [ 调度 Servlet ] 设计。
DispatcherServlet 的作用是将请求分发到不同的处理器。从 Spring 2.5开始，使用 Java 5 以上版本的用户可以采用基于注解形式进行开发，十分简洁；

### 中心控制器

Spring的web框架围绕DispatcherServlet设计。DispatcherServlet的作用是将请求分发到不同的处理器。从Spring 2.5开始，使用Java 5或者以上版本的用户可以采用基于注解的controller声明方式。

![img](/img/post/SpringMVC/DispatcherServlet.png)

Spring MVC 框架像许多其他 MVC 框架一样, 以请求为驱动，围绕一个中心Servlet分派请求及提供其他功能，DispatcherServlet 是一个实际的 Servlet (它继承自HttpServlet 基类)。

SpringMVC的原理如下图所示。
当发起请求时被前置的控制器拦截到请求，根据请求参数生成代理请求，找到请求对应的实际控制器，控制器处理请求，创建数据模型，访问数据库，将模型响应给中心控制器，控制器使用模型与视图渲染视图结果，将结果返回给中心控制器，再将结果返回给请求者。

![img](/img/post/SpringMVC/SpringMVC_principle.png)

### SpringMVC 执行流程

![img](/img/post/SpringMVC/SpringMVC_flow.png)

1、DispatcherServlet 表示前置控制器，是整个SpringMVC的控制中心。用户发出请求，DispatcherServlet接收请求并拦截请求。

我们假设请求的url为 : http://localhost:8080/SpringMVC/hello

如上url拆分成三部分：

http://localhost:8080服务器域名

SpringMVC部署在服务器上的web站点

hello表示控制器

通过分析，如上url表示为：请求位于服务器localhost:8080上的SpringMVC站点的hello控制器。

2、HandlerMapping为处理器映射。DispatcherServlet调用HandlerMapping,HandlerMapping根据请求url查找Handler。

3、HandlerExecution表示具体的Handler,其主要作用是根据url查找控制器，如上url被查找控制器为：hello。

4、HandlerExecution将解析后的信息传递给DispatcherServlet,如解析控制器映射等。

5、HandlerAdapter表示处理器适配器，其按照特定的规则去执行Handler。

6、Handler让具体的Controller执行。

7、Controller将具体的执行信息返回给HandlerAdapter,如ModelAndView。

8、HandlerAdapter将视图逻辑名或模型传递给DispatcherServlet。

9、DispatcherServlet调用视图解析器(ViewResolver)来解析HandlerAdapter传递的逻辑视图名。

10、视图解析器将解析的逻辑视图名传给DispatcherServlet。

11、DispatcherServlet根据视图解析器解析的视图结果，调用具体的视图。

12、最终视图呈现给用户。

## 总结

本篇我们首先回顾了 MVC 和 Servlet，然后简单介绍了 SpringMVC 及其执行原理。

参考自：
1. [狂神说SpringMVC01：什么是SpringMVC](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483970&idx=1&sn=352e571ee88957ce391e972344e2a3d7&scene=19#wechat_redirect)
