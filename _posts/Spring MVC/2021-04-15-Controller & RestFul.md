---
layout:     post
title:      3. Controller & RestFul
subtitle:   
date:       2021-04-15
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring MVC
---

上一篇中我们编写了 SpringMVC 的第一个程序，并通过注解方式简化了相关的配置。这里我们进一步理解其中的 Controller。

## Controller

Controller 负责提供访问应用程序的行为，通常通过实现接口或注解定义两种方法实现。内部的主要处理步骤就是解析用户的请求并将其转换为一个模型。

### 实现 Controller 接口

Controller 是一个接口，在 org.springframework.web.servlet.mvc 包下，接口中只有一个方法

```java
//实现该接口的类获得控制器功能
public interface Controller {
   //处理请求且返回一个模型与视图对象
   ModelAndView handleRequest(HttpServletRequest req, HttpServletResponse resp) throws Exception;
}
```

上一篇中我们已经通过实现 Controller 接口编写了 SpringMVC 程序。当时我们在 springmvc-servlet.xml 文件中指定了 **处理器映射器**、**处理器适配器**和 **视图解析器**，其实SpringMVC 已经默认加载了 **处理器映射器**、**处理器适配器**，因此我们可以不用显式地指定。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- 这里没有再指定 handlerMapping 和 handleAdapter -->

    <!--视图解析器:DispatcherServlet给他的ModelAndView-->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" id="InternalResourceViewResolver">
        <!--前缀-->
        <property name="prefix" value="/WEB-INF/jsp/"/>
        <!--后缀-->
        <property name="suffix" value=".jsp"/>
    </bean> 

    <bean id="/hello" class="com.zhao.controller.HelloController"/>

</beans>
```

实现 Controller 接口来定义控制器是较老的办法。其缺点是一个控制器中只有一个方法，如果要多个方法则需要定义多个 Controller类，比较麻烦。

### 使用 @Controller 注解

@Controller 注解用于声明 Spring 类的实例是一个控制器。Spring 可以使用组件扫描机制来找到应用程序中所有注解的控制器类，为了保证Spring能扫描到控制器，需要在配置文件中声明组件扫描。

```xml
<!-- 自动扫描指定的包，下面所有注解类交给IOC容器管理 -->
<context:component-scan base-package="com.zhao.controller"/>
```

上一篇我们已经使用 @Controller 注解实现 SpringMVC 了。注解方式是推荐使用的方式！


## RequestMapping

@RequestMapping 注解用于映射url到控制器类或其中一个特定的方法。用于类上时，表示类中的所有响应请求的方法都是以该地址作为父路径。

注解在方法上面

```java
@Controller
public class TestController {
   @RequestMapping("/h1")
   public String test(){
       return "test";
  }
}
```
访问路径：http://localhost:8080/项目名/h1

同时注解类与方法
```java
@Controller
@RequestMapping("/admin")
public class TestController {
   @RequestMapping("/h1")
   public String test(){
       return "test";
  }
}
```
访问路径：http://localhost:8080/项目名/admin/h1 需要先指定类的路径再指定方法的路径

## RestFul 风格

Restful 是一种资源定位及资源操作的风格。基于这种风格设计的软件可以更简洁，更有层次，更易于实现缓存等机制。

传统方式操作资源：通过不同的参数来实现不同的效果！方法单一
+ http://127.0.0.1/item/queryItem.action?id=1 GET
+ http://127.0.0.1/item/saveItem.action POST
+ http://127.0.0.1/item/updateItem.action POST
+ http://127.0.0.1/item/deleteItem.action?id=1 DELETE

RESTFul 操作资源：可以通过不同的请求方式来实现不同的效果！如下，请求地址一样，但是功能可以不同！
+ http://127.0.0.1/item/1 GET
+ http://127.0.0.1/item POST
+ http://127.0.0.1/item PUT
+ http://127.0.0.1/item/1 DELETE

### 测试

1、新建一个类 RestFulController
```java
@Controller
public class RestFulController {
}
```

2、Spring MVC 中可以使用 @PathVariable 注解，让方法参数的值对应绑定到一个URL模板变量上。
```java
@Controller
public class RestFulController {

   //映射访问路径
   @RequestMapping("/commit/{p1}/{p2}")
   public String index(@PathVariable int p1, @PathVariable int p2, Model model){
       int result = p1+p2;
       //Spring MVC会自动实例化一个Model对象用于向视图中传值
       model.addAttribute("msg", "结果："+result);
       //返回视图位置
       return "test";    
  }
}
```

3、测试请求

![img](/img/post/SpringMVC/restful1.png)

4、思考：使用路径变量的好处？

+ 安全；
+ 使路径变得更加简洁；
+ 获得参数更加方便，框架会自动进行类型转换。
+ 通过路径变量的类型可以约束访问参数，如果类型不一样，则访问不到对应的请求方法，如这里访问是的路径是/commit/1/a，则路径与方法不匹配

5、修改对应的参数类型，再次测试
```java
//映射访问路径
@RequestMapping("/commit/{p1}/{p2}")
public String index(@PathVariable int p1, @PathVariable String p2, Model model){

   String result = p1+p2;
   //Spring MVC会自动实例化一个Model对象用于向视图中传值
   model.addAttribute("msg", "结果："+result);
   //返回视图位置
   return "test";
}
```
![img](/img/post/SpringMVC/restful2.png)

### 使用 method 属性指定请求类型

1、增加一个方法
```java
//映射访问路径,必须是POST请求
@RequestMapping(value = "/hello",method = {RequestMethod.POST})
public String index2(Model model){
   model.addAttribute("msg", "hello!");
   return "test";
}
```

2、访问默认是Get请求，会报错405

![img](/img/post/SpringMVC/restful3.png)

3、如果将 POST 修改为 GET 则正常了

```java
//映射访问路径,必须是Get请求
@RequestMapping(value = "/hello", method = {RequestMethod.GET})
public String index2(Model model){
   model.addAttribute("msg", "hello!");
   return "test";
}
```

![img](/img/post/SpringMVC/restful4.png)

我们也可以使用以下的变体来代表上面的 @RequestMapping 注解：
+ @GetMapping
+ @PostMapping
+ @PutMapping
+ @DeleteMapping

## 总结

本篇我们在第一个 SpringMVC 程序的基础上，了解了 Controller 的用法，顺便了解了 RestFul 风格。

参考自：
1. [狂神说SpringMVC03：RestFul和控制器](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483993&idx=1&sn=abdd687e0f360107be0208946a7afc1d&scene=19#wechat_redirect)
