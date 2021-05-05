---
layout:     post
title:      2. SpringBoot 运行原理探究
subtitle:   
date:       2021-05-05
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SpringBoot
---

上一篇我们创建了 SpringBoot 的第一个项目 HelloSpringBoot，那它到底是怎么运行的呢？这里我们一探究竟，我们先从 Maven 项目的 pom.xml 文件看起。

## pom.xml 文件

### 父依赖

pom.xml 文件最开始有一个父依赖，点进去可以查看它的内容，根据其描述："Parent pom providing dependency and plugin management for applications built with Maven" 可知，它主要提供依赖和插件的管理。

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.4.5</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

我们可以看到上面这个文件中还有一个父依赖，这个才是真正管理 Spring Boot 依赖的地方，是 Spring Boot 的版本控制中心。

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.4.5</version>
</parent>
```

以后我们导入 jar 包默认不需要指定版本，Spring Boot 会根据这个依赖文件自动配置；但如果导入的包没有被该依赖管理，则仍需要我们手动配置版本。

### 启动器

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

spring-boot-boot-starter-xxx：spring-boot的某个场景启动器，例如上面的依赖就是 web 场景启动器，Spring Boot 会帮我们导入 web 模块运行所依赖的组件。

Spring Boot 将所有的功能场景抽取出来，封装成了一个个 starter，我们只需要在项目中引入这些 starter，所有相关的依赖就会导入进来。未来我们也可以自定义 starter。

Spring Boot 封装的 starter 可以在官网的[这里](https://docs.spring.io/spring-boot/docs/current/reference/html/using-spring-boot.html#using-boot-starter)查看。

## 主启动类

### 默认的主启动类

```java
@SpringBootApplication
public class HelloSpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(HelloSpringBootApplication.class, args);
    }
}
```

### @SpringBootApplication

这个注解标注在类上，说明这个类是 Spring Boot 的主配置类，Spring Boot 应该运行这个类的 main 方法启动整个应用。

进入这个注解，可以看到还有很多其他注解

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
```

#### @ComponentScan
这个注解用于自动扫描 bean 实例，并把它加载到 IOC 容器中

#### @SpringBootConfiguration

标注在某个类上，表示这是一个SpringBoot的配置类；我们继续进去这个注解查看

```java
@Configuration
```
@Configuration 注解说明，这个类是一个配置类，配置类就是对应于 Spring 的 XML 配置文件。

我们继续回到 @SpringBootApplication 注解中查看

#### @EnableAutoConfiguration

见文知意，这个注解开启了自动配置。以前我们要手动配置的东西，现在 Spring Boot 可以自动配置。点进去继续查看

```java
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
```

**@AutoConfigurationPackage** 注解中，包含了以下内容：
```java
@Import({Registrar.class})
```
Spring 底层注解 @import 给容器中导入一个组件，Registrar.class 作用是将主启动类的所在包及包下面所有子包里面的所有组件扫描到 Spring 容器；

**@Import({AutoConfigurationImportSelector.class}):**

AutoConfigurationImportSelector: 自动配置导入选择器，我们查看它导入了哪些组件的选择器。

1、源码中有这个方法

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    // 这里的 getSpringFactoriesLoaderFactoryClass() 返回的就是之前的 EnableAutoConfiguration 注解类
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

2、该方法又调用了 SpringFactoriesLoader.loadFactoryNames 方法

```java
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    ClassLoader classLoaderToUse = classLoader;
    if (classLoader == null) {
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
    }
    String factoryTypeName = factoryType.getName();
    return (List)loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
}
```

3、又调用了 loadSpringFactories 方法

```java
private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
    Map<String, List<String>> result = (Map)cache.get(classLoader);
    if (result != null) {
        return result;
    } else {
        HashMap result = new HashMap();
        try {
            // 获取资源：META-INF/spring.factories
            Enumeration urls = classLoader.getResources("META-INF/spring.factories");
            // 遍历资源，封装为 Properties
            while(urls.hasMoreElements()) {
                URL url = (URL)urls.nextElement();
                UrlResource resource = new UrlResource(url);
                Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                Iterator var6 = properties.entrySet().iterator();
                while(var6.hasNext()) {
                    Entry<?, ?> entry = (Entry)var6.next();
                    String factoryTypeName = ((String)entry.getKey()).trim();
                    String[] factoryImplementationNames = StringUtils.commaDelimitedListToStringArray((String)entry.getValue());
                    String[] var10 = factoryImplementationNames;
                    int var11 = factoryImplementationNames.length;
                    for(int var12 = 0; var12 < var11; ++var12) {
                        String factoryImplementationName = var10[var12];
                        ((List)result.computeIfAbsent(factoryTypeName, (key) -> {
                            return new ArrayList();
                        })).add(factoryImplementationName.trim());
                    }
                }
            }
            result.replaceAll((factoryType, implementations) -> {
                return (List)implementations.stream().distinct().collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
            });
            cache.put(classLoader, result);
            return result;
        } catch (IOException var14) {
            throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var14);
        }
    }
}
```

4、我们查看这个文件：META-INF/spring.factories

#### spring.factories

双击 shift 全局搜索打开该文件，看到了很多自动配置的文件，这就是自动配置根源所在！

![img](/img/post/SpringBoot/spring.factories.png)

所以，自动配置的真正实现是从 classpath 中搜寻所有的 META-INF/spring.factories 配置文件，并将其中对应的 org.springframework.boot.autoconfigure. 包下的配置项，通过反射实例化为对应标注了 @Configuration 的 JavaConfig 形式的 IOC 容器配置类，然后将这些都汇总成为一个实例并加载到IOC容器中。

## SpringApplication

```java
public static void main(String[] args) {
    SpringApplication.run(HelloSpringBootApplication.class, args);
}
```

SpringApplication.run 方法做了两件事情：SpringApplication 的实例化，和 run 方法执行

### SpringApplication 实例化

主要做了以下四件事情：
+ 推断应用的类型是普通的项目还是Web项目
+ 查找并加载所有可用初始化器，设置到initializers属性中
+ 找出所有的应用程序监听器，设置到listeners属性中
+ 推断并设置main方法的定义类，找到运行的主类

```java
public SpringApplication(ResourceLoader resourceLoader, Class... primarySources) {
    // ......
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.setInitializers(this.getSpringFactoriesInstances();
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

### run方法流程分析

![img](/img/post/SpringBoot/run.png)

## 总结

本篇我们了解了 SpringBoot 项目的运行原理。

参考自：
1. [狂神说SpringBoot02：运行原理初探](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483743&idx=1&sn=431a5acfb0e5d6898d59c6a4cb6389e7&scene=19#wechat_redirect)
