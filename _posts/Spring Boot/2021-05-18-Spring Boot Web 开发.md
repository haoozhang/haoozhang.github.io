---
layout:     post
title:      5. Spring Boot Web 开发
subtitle:   
date:       2021-05-18
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SpringBoot
---

接下来，我们学习 Spring Boot 的 Web 开发。Spring Boot 最大的特点就是自动装配，所以我们使用 Spring Boot 非常简单，主要是如下几个步骤：
1. 创建 Spring Boot 应用，选择需要的模块，Spring Boot 会自动帮我们装配好
2. 手动在配置文件中配置，就可以运行起来了
3. 专注编写业务代码，无需花太多心思在配置上

## 静态资源处理

我们像之前搭建一个 Hello World 的 Spring Boot 项目，然后我们就要写接口请求了。在之前的 Web 开发中我们写接口请求，并引入前端的 css，js 等资源，放在 webapp 目录下。但在 Spring Boot 中我们如何处理呢？其实，Spring Boot 有约定好的静态资源放置位置。

在 Spring Boot 中，SpringMVC 的相关配置都在 WebMvcAutoConfiguration 这个自动配置类中，我们查看该类源码，有很多配置方法，其中有一个 addResourceHandlers 方法添加资源处理

```java
protected void addResourceHandlers(ResourceHandlerRegistry registry) {
    super.addResourceHandlers(registry);
    if (!this.resourceProperties.isAddMappings()) {
        // 禁用默认的资源处理
        logger.debug("Default resource handling disabled");
    } else {
        ServletContext servletContext = this.getServletContext();
        // webjars 映射规则
        this.addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
        // 静态资源映射规则
        this.addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
            registration.addResourceLocations(this.resourceProperties.getStaticLocations());
            if (servletContext != null) {
                registration.addResourceLocations(new Resource[]{new ServletContextResource(servletContext, "/")});
            }
        });
    }
}
```

### webjars

webjars 本质就是以 jar 包的形式导入静态资源。之前我们导入静态资源，如学习 Ajax 时的 jQuery.js，直接导入到目录即可。现在在 Spring Boot 中要使用 jQuery，只需要在 webjars 官网上找到 jQuery 依赖，并引入 jQuery 对应版本的 pom 依赖即可。

```xml
<dependency>
    <groupId>org.webjars</groupId>
    <artifactId>jquery</artifactId>
    <version>3.4.1</version>
</dependency>
```

引入依赖之后，查看 webjars 目录结构，我们访问 Jquery.js 文件

![img](/img/post/SpringBoot/jquery_webjars.png)

```
localhost:8080/webjars/jquery/3.4.1/jquery.js
```

### 静态资源

项目中自己使用的静态资源如何映射呢？我们查看 **mvcProperties.getStaticPathPattern()** 方法，发现返回 **/\*\***；查看
**resourceProperties.getStaticLocations** 的源码，发现以下代码：

```java
private static final String[] CLASSPATH_RESOURCE_LOCATIONS = new String[]{
    "classpath:/META-INF/resources/", 
    "classpath:/resources/", 
    "classpath:/static/", 
    "classpath:/public/"};
private String[] staticLocations;

public Resources() {
    this.staticLocations = CLASSPATH_RESOURCE_LOCATIONS;
    ....
}
```

所以，项目识别存放在以上四个目录中的静态资源。我们可以在 resources 目录下新建对应的文件夹，存放静态资源文件。例如，我们访问 *http://localhost:8080/1.js* , 他就会去这些文件夹中寻找对应的静态资源文件

### 自定义静态资源路径

我们也可以自己通过配置文件来指定静态资源的存放位置。一旦自定义设置，按照源码中的 if 判断，默认的自动配置就失效了，一般不建议使用这种方式。

```
spring.resources.static-locations=classpath:/a/,classpath:/b/
```

## 首页处理

处理完静态资源，我们处理首页。默认情况下，Spring Boot 首页无法访问，我们继续看 WebMvcAutoConfiguration 类的源码，发现欢迎页的映射方法

```java
@Bean
// 欢迎页处理映射
public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext applicationContext, FormattingConversionService mvcConversionService, ResourceUrlProvider mvcResourceUrlProvider) {
    WelcomePageHandlerMapping welcomePageHandlerMapping = new WelcomePageHandlerMapping(
        new TemplateAvailabilityProviders(applicationContext),
        applicationContext, 
        this.getWelcomePage(),  // 获取欢迎页
        this.mvcProperties.getStaticPathPattern());
    welcomePageHandlerMapping.setInterceptors(this.getInterceptors(mvcConversionService, mvcResourceUrlProvider));
    welcomePageHandlerMapping.setCorsConfigurations(this.getCorsConfigurations());
    return welcomePageHandlerMapping;
}
```

继续看 getWelcomePage() 方法。

```java
private Resource getWelcomePage() {
    // 上面的四个目录
    String[] var1 = this.resourceProperties.getStaticLocations();
    int var2 = var1.length;
    for(int var3 = 0; var3 < var2; ++var3) {
        String location = var1[var3];
        // 四个目录下获取 indexHtml
        Resource indexHtml = this.getIndexHtml(location);
        if (indexHtml != null) {
            return indexHtml;
        }
    }
    ServletContext servletContext = this.getServletContext();
    if (servletContext != null) {
        return this.getIndexHtml((Resource)(new ServletContextResource(servletContext, "/")));
    } else {
        return null;
    }
}
private Resource getIndexHtml(String location) {
    return this.getIndexHtml(this.resourceLoader.getResource(location));
}
private Resource getIndexHtml(Resource location) {
    try {
        // index.html
        Resource resource = location.createRelative("index.html");
        if (resource.exists() && resource.getURL() != null) {
            return resource;
        }
    } catch (Exception var3) {
    }
    return null;
}
```

所以，首页其实就是上面四个目录下的 index.html 文件，被 /\*\* 映射。
例如我们访问 http://localhost:8080/，就会找静态资源文件夹下的 index.html。

## Thymeleaf 模板引擎

前端交给我们的页面是 html 页面，以前开发我们会转换为 jsp 页面实现数据的显示和交互。
但现在 Spring Boot 以 jar 方式打包，默认不支持 jsp，而是推荐使用模板引擎。
模板引擎的作用就是我们来写一个页面模板，把模板和后台封装的数据都交给模板引擎，它会自动填充数据、解析表达式，最终生成一个我们想要的界面。

### 引入 Thymeleaf

1、通过 starter 引入

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

2、如果上面引入的依赖版本默认为2.x，则需要通过以下方式引入 3.0 版本的

```xml
<dependency>
    <groupId>org.thymeleaf</groupId>
    <artifactId>thymeleaf-spring5</artifactId>
</dependency>
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-java8time</artifactId>
</dependency>
```

### Thymeleaf 分析

引入 Thymeleaf 之后，如何使用呢？我们查看它的自动配置类：ThymeleafProperties

```java
@ConfigurationProperties(
    prefix = "spring.thymeleaf"
)
public class ThymeleafProperties {
    private static final Charset DEFAULT_ENCODING;
    public static final String DEFAULT_PREFIX = "classpath:/templates/";
    public static final String DEFAULT_SUFFIX = ".html";
    ....
```

可以看到默认的前缀和后缀，所以我们只需要将 html 页面放在 templates 目录下，thymeleaf 就可以帮我们自动渲染了

### Thymeleaf 测试

1、编写 Testcontroller 和接口

```java
@Controller
public class TestController {

    @GetMapping("/test")
    public String test() {
        return "test";
    }
}
```

2、编写一个测试页面  test.html 放在 templates 目录下

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>

<h1>
    test
</h1>

</body>
</html>
```

3、启动项目，访问请求测试

### Thymeleaf 语法

学习语法最好参考官方文档 https://www.thymeleaf.org/，我们做个简单练习

1、修改测试请求，增加数据传输；
```java
@RequestMapping("/t1")
public String test1(Model model){
    // 存入数据
    model.addAttribute("msg","Hello,Thymeleaf");
    // classpath:/templates/test.html
    return "test";
}
```

2、要使用 thymeleaf，需要在 html 文件中导入命名空间的约束，方便提示。我们可以去官方文档的#3中看一下命名空间拿来过来：
```
xmlns:th="http://www.thymeleaf.org"
```
3、我们去编写下前端页面
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<h1>测试页面</h1>

<!--th:text就是将div中的内容设置为它指定的值，和之前学习的Vue一样-->
<div th:text="${msg}"></div>
</body>
</html>
```

4、启动测试！

5、我们再测试非转义字符和循环

```java
@RequestMapping("/t2")
public String test2(Map<String,Object> map){
    //存入数据
    map.put("msg","<h1>Hello</h1>");
    map.put("users", Arrays.asList("aaa","bbb"));
    //classpath:/templates/test.html
    return "test";
}
```
6、对应的前端页面取出数据
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<h1>测试页面</h1>

<div th:text="${msg}"></div>
<!--不转义-->
<div th:utext="${msg}"></div>

<!--遍历数据-->
<!--th:each每次遍历都会生成当前这个标签：官网#9-->
<h4 th:each="user :${users}" th:text="${user}"></h4>

</body>
</html>
```
7、启动项目测试

## MVC 自动配置原理

在项目编写前，我们还需要了解一个点，就是 Spring Boot 对 SpringMVC 做了哪些配置，我们如何扩展它。在 Spring Boot 的官方文档[这里](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-developing-web-applications)中，我们可以看到如下片段

![img](/img/post/SpringBoot/springboot_mvc_autoconfig.png)

### ContentNegotiatingViewResolver 内容协商视图解析器 

自动配置了 ViewResolver，即就是我们之前学习的 SpringMVC 的视图解析器，它根据方法的返回值取出视图对象 (View)，然后由视图对象决定如何渲染 (转发，重定向)。

我们看看这里的源码：找到 WebMvcAutoConfiguration， 然后搜索ContentNegotiatingViewResolver，找到如下方法

```java
@Bean
@ConditionalOnBean({ViewResolver.class})
@ConditionalOnMissingBean(
    name = {"viewResolver"},
    value = {ContentNegotiatingViewResolver.class}
)
public ContentNegotiatingViewResolver viewResolver(BeanFactorybeanFactory) {
    ContentNegotiatingViewResolver resolver = new ContentNegotiatingViewResolver();
    resolver.setContentNegotiationManager((ContentNegotiationManager)beanFactory.getBean(ContentNegotiationManager.class));
    resolver.setOrder(-2147483648);
    return resolver;
}
```

进一步查看 ContentNegotiatingViewResolver 类，找到对应的解析视图的代码

```java
@Nullable
public View resolveViewName(String viewName, Locale locale) throwsException {
    RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
    Assert.state(attrs instanceof ServletRequestAttributes, "No current ServletRequestAttributes");
    List<MediaType> requestedMediaTypes = this.getMediaTypes(((ServletRequestAttributes)attrs).getRequest());
    if (requestedMediaTypes != null) {
        // 获取候选的视图
        List<View> candidateViews = this.getCandidateViews(viewName, locale, requestedMediaTypes);
        // 选择最好的视图，返回
        View bestView = this.getBestView(candidateViews, requestedMediaTypes, attrs);
        if (bestView != null) {
            return bestView;
        }
    }
    ....
}
```

如何获取候选视图呢？在 getCandidateViews 方法中可以看到它是把所有的视图解析器拿来，while 循环逐个解析。

```java
....
Iterator var5 = this.viewResolvers.iterator();
while(var5.hasNext()) {
....
```

因此得出结论：ContentNegotiatingViewResolver 这个视图解析器就是用来组合所有的视图解析器的。

我们再探索一下它的组合逻辑，有个属性 viewResolvers，看看是在哪里赋值的

```java
protected void initServletContext(ServletContext servletContext) {
        Collection<ViewResolver> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(this.obtainApplicationContext(), ViewResolver.class).values();
        ViewResolver viewResolver;
        if (this.viewResolvers == null) {
            this.viewResolvers = new ArrayList(matchingBeans.size());
            ....
```

既然它是在容器中去找视图解析器实例，那我们是否可以自己实现一个视图解析器并添加为容器组件，这个类自动把它组合进来呢？

1、编写一个视图解析器

```java
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {

    public MyViewResolver getMyViewResolver() {
        return new MyViewResolver();
    }
    
    // 自定义的视图解析器
    private static class MyViewResolver implements ViewResolver {
        @Override
        public View resolveViewName(String s, Locale locale) throws Exception {
            return null;
        }
    }
}
```

2、怎么看我们自己写的视图解析器有没有起作用呢？我们给 DispatcherServlet 类中的 doDispatch方法加个断点调试一下，因为所有的请求都会走到这个方法中

3、启动项目，看看断点的 debug 信息，找到 this.viewResolver，我们就可以看到我们自己实现的视图解析器了

因此，如果我们想要扩展视图解析器，我们只需要向容器中添加这个组件就可以了，Spring Boot 会自动帮我们做剩下的事情

### 转换器和格式化器

```java
@Bean
public FormattingConversionService mvcConversionService() {
    Format format = this.mvcProperties.getFormat();
    WebConversionService conversionService = new WebConversionService((new DateTimeFormatters()).dateFormat(format.getDate()).timeFormat(format.getTime()).dateTimeFormat(format.getDateTime()));
    this.addFormatters(conversionService);
    return conversionService;
}
```

可以看到在 Properties 文件中，我们可以进行自动配置它

### 扩展 SpringMVC

SpringBoot在自动配置很多组件的时候，先看容器中有没有用户自己配置的 (如果用户自己配置 @bean)，如果有就用用户配置的，如果没有就用自动配置的；如果有些组件可以存在多个，比如我们的视图解析器，就将用户配置的和自己默认的组合起来

如果我们要扩展 SpringMVC，只需要编写一个 @Configuration 注解类，并且类型要为 WebMvcConfigurer，注意不能标注 @EnableWebMvc 注解，这是为什么呢？

1、我们点进去 @EnableWebMvc 注解，可以看到它导入了一个类

```java
@Import({DelegatingWebMvcConfiguration.class})
public @interface EnableWebMvc {
}
```

2、点进该类，看到

```java
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
....
```

3、而我们回顾一下 WebMvcAutoConfiguration 类

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
// 容器中没有这个组件时，该自动配置类才生效
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})
@AutoConfigureOrder(-2147483638)
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
public class WebMvcAutoConfiguration {
....
```

可以看到这个自动配置类生效的前提是没有 WebMvcConfigurationSupport 类，而 @EnableWebMvc 将 WebMvcConfigurationSupport 组件导入进来了，它只是 SpringMVC 最基本的功能。

## 总结

本篇我们以 HttpEncodingAutoConfiguration 为例学习了自动配置的原理。

参考自：
1. [狂神说SpringBoot10：Web开发静态资源处理](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483796&idx=1&sn=ea13e2858328a582338e89c3459021c1&scene=19#wechat_redirect)

2. [狂神说SpringBoot11：Thymeleaf模板引擎](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483807&idx=1&sn=7e1d5df51cdeb046eb37dec7701af47b&scene=19#wechat_redirect)

3. [狂神说SpringBoot12：MVC自动配置原理](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483819&idx=1&sn=b9009aaa2a9af9d681a131b3a49d8848&scene=19#wechat_redirect)
