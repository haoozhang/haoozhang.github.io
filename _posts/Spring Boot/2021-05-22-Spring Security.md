---
layout:     post
title:      8. Spring Security
subtitle:   
date:       2021-05-23
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SpringBoot
---

Spring Security 是一个功能强大且高度可定制的身份验证和访问控制框架，是保护基于 Spring 的应用程序的标准。

## 环境搭建

1、新建 Spring Boot 项目，导入 web 模块和 Thymeleaf

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- thymeleaf -->
<dependency>
    <groupId>org.thymeleaf</groupId>
    <artifactId>thymeleaf-spring5</artifactId>
</dependency>
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-java8time</artifactId>
</dependency>
```

2、导入静态资源，静态资源见[这里]()

3、编写 controller 接口控制跳转

```java
@Controller
public class RouteController {

    @GetMapping({"/","/index"})
    public String index() {
        return "index";
    }

    @GetMapping({"/toLogin"})
    public String toLogin() {
        return "views/login";
    }

    @GetMapping("/level1/{idx}")
    public String level1(@PathVariable("idx") int idx) {
        return "views/level1/" + idx;
    }

    @GetMapping("/level2/{idx}")
    public String level2(@PathVariable("idx") int idx) {
        return "views/level2/" + idx;
    }

    @GetMapping("/level3/{idx}")
    public String level3(@PathVariable("idx") int idx) {
        return "views/level3/" + idx;
    }

}
```

4、启动项目测试环境是否搭建成功

## 认证和授权

要使用 Spring Security，仅需要引入 spring-boot-starter-security 模块，进行少量的配置，即可实现强大的安全管理。
Spring Security的两个主要目标是 *认证* 和 *授权*，认证就是验证登录的身份，授权就是登录成功后授予的访问权限

1、引入 Spring Security 依赖

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

2、编写配置类

```java
@EnableWebSecurity
public class MySecurityConfig extends WebSecurityConfigurerAdapter {

    // 授权
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        
    }
}
```

3、配置一些授权的规则

```java
@Override
protected void configure(HttpSecurity http) throwsException {
    // 首页都可以访问，但功能页有权限的人才可访问
    http.authorizeRequests()
            .antMatchers("/").permitAll()
            .antMatchers("/level1/**").hasRole("vip1")
            .antMatchers("/level2/**").hasRole("vip2")
            .antMatchers("/level3/**").hasRole("vip3");  // 链式编程
}
```

4、启动项目测试，发现我们除了首页，其他页面都进入不了。因为此时我们还没有登录的角色，访问其他页面要有对应权限的角色访问才可以

5、在 configure() 方法中加入开启自动配置的登录功能

```
// 没有权限就跳到自动配置的登录页
http.formLogin();
```

6、启动项目测试，发现没有权限会跳转到 Spring Security 自动配置的登录页

7、进入 http.formLogin 方法，查看方法上的注释。我们可以通过重写 configure(AuthenticationManagerBuilder auth) 方法定义认证的规则

```java
// 认证
@Override
protected void configure(AuthenticationManagerBuilder auth) throwsException {
    auth.inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
            .withUser("admin").password("123456").roles("vip2", "vip3")
            .and()
            .withUser("root").password("123456").roles("vip1", "vip2", "vip3")
            .and()
            .withUser("guest").password("123456").roles("vip1");
}
```

8、启动项目测试登录，发现会报 PassEncoder 相关的错误。这是因为前端传过来的明文密码需要加密，因此我们按照提示使用 PassEncoder

 ```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throwsException {
    // PasswordEncoder 明文密码加密
    auth.inMemoryAuthentication().passwordEncoder(new BCryptPasswordEncoder())
            .withUser("admin").password(new BCryptPasswordEncoder().encode("123456")).roles("vip2", "vip3")
            .and()
            .withUser("root").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1", "vip2", "vip3")
            .and()
            .withUser("guest").password(new BCryptPasswordEncoder().encode("123456")).roles("vip1");
}
```

9、再次启动项目，测试登录功能，可以正常登录和访问自己用户权限的内容，因此认证和授权就完成了。

## 注销和权限控制

1、开启自动配置的注销功能

```java
// 开启注销功能
http.logout();
```

2、在前端页面 index.html 中增加一个注销的按钮

```html
<a class="item" th:href="@{/logout}">
    <i class="sign-out icon"></i> 注销
</a>
```

3、测试发现，注销后会跳转到登录页面，我们可以让他跳转到首页，通过以下方式实现

```java
// 开启注销功能，跳到登录页面
http.logout().logoutSuccessUrl("/");
```

4、接下来再来一个新的需求：当没有登录时，要求导航栏只出现登录按钮；登录后，要求导航栏出现用户和角色，以及注销按钮。并且，登录后页面只显示自己有权限的界面，例如 guest 用户拥有 vip1 权限，所以只显示 vip1 对应的 level1 部分，不显示其他两部分。这该怎么实现呢？

这里我们要结合 thymeleaf 中的一些功能

5、导入 thymeleaf-spring security5 依赖

```xml
<!-- thymeleaf-spring security5 -->
<dependency>
    <groupId>org.thymeleaf.extras</groupId>
    <artifactId>thymeleaf-extras-springsecurity5</artifactId>
    <version>3.0.4.RELEASE</version>
</dependency>
```

6、修改前端页面，通过 sec：authorize="isAuthenticated()" 判断是否认证登录

导入命名空间
```html
xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity4"
```
修改导航栏，添加登录的判断

```html
<div class="ui segment" id="index-header-nav" th:fragment="nav-menu">
    <div class="ui secondary menu">

        <a class="item"  th:href="@{/index}">首页</a>
        <!--登录注销-->
        <div class="right menu">

            <!--未登录, 显示登录按钮-->
            <div sec:authorize="!isAuthenticated()">
                <a class="item" th:href="@{/toLogin}">
                    <i class="address card icon"></i> 登录
                </a>
            </div>

            <!--已登录, 显示用户名和注销按钮-->
            <div sec:authorize="isAuthenticated()">
                <a class="item">
                   用户名: <span sec:authentication="name"></span>
                   角色: <span sec:authentication="principal.authorities"></span>
                </a>
                <a class="item" th:href="@{/logout}">
                    <i class="sign-out icon"></i> 注销
                </a>
            </div>

        </div>
    </div>
</div>
```

7、启动项目测试，观察登录前后的导航栏变化。

如果注销出现404，是因为它默认防止 csrf 跨站请求伪造，会产生安全问题，可以在配置中关闭 csrf 功能

```java
http.csrf().disable();
http.logout().logoutSuccessUrl("/");
```

8、继续修改下面的角色功能模块

```html
<div class="ui three column stackable grid">
    <!-- 根据用户角色权限显示菜单 -->
    <div class="column" sec:authorize="hasRole('vip1')">
        <div class="ui raised segment">
            <div class="ui">
                <div class="content">
                    <h5 class="content">Level 1</h5>
                    <hr>
                    <div><a th:href="@{/level1/1}"><i class="bullhorn icon"></i> Level-1-1</a></div>
                    <div><a th:href="@{/level1/2}"><i class="bullhorn icon"></i> Level-1-2</a></div>
                    <div><a th:href="@{/level1/3}"><i class="bullhorn icon"></i> Level-1-3</a></div>
                </div>
            </div>
        </div>
    </div>

    <div class="column" sec:authorize="hasRole('vip2')">
        <div class="ui raised segment">
            <div class="ui">
                <div class="content">
                    <h5 class="content">Level 2</h5>
                    <hr>
                    <div><a th:href="@{/level2/1}"><i class="bullhorn icon"></i> Level-2-1</a></div>
                    <div><a th:href="@{/level2/2}"><i class="bullhorn icon"></i> Level-2-2</a></div>
                    <div><a th:href="@{/level2/3}"><i class="bullhorn icon"></i> Level-2-3</a></div>
                </div>
            </div>
        </div>
    </div>

    <div class="column" sec:authorize="hasRole('vip3')">
        <div class="ui raised segment">
            <div class="ui">
                <div class="content">
                    <h5 class="content">Level 3</h5>
                    <hr>
                    <div><a th:href="@{/level3/1}"><i class="bullhorn icon"></i> Level-3-1</a></div>
                    <div><a th:href="@{/level3/2}"><i class="bullhorn icon"></i> Level-3-2</a></div>
                    <div><a th:href="@{/level3/3}"><i class="bullhorn icon"></i> Level-3-3</a></div>
                </div>
            </div>
        </div>
    </div>
</div>
```

9、启动测试，注销功能和权限控制成功



## 记住我

当前情况下，登录之后只要关闭页面，就需要再次登录。我们需要实现一个记住密码的功能。

1、开启记住我功能，增加如下的配置

```java
// 开启"记住我"功能，通过 cookie 实现
http.rememberMe();
```

2、启动测试，发现多了一个记住我的功能。我们登录之后关闭页面，再次打开发现登录的用户信息仍然存在。这是如何实现的呢？

其实非常简单，就是在浏览器中储存了一个 cookie。当我们注销的时候，可以发现 Spring Security 帮我们自动删除了这个 cookie

## 首页定制

现在的登录页面是 Spring Security 自动配置的，我们怎么使用我们自己的 login 页面呢？

1、在刚才开启的登录页配置后面指定 loginPage

```java
// 没有权限就跳到自动配置的登录页
// 定制登录页 .loginPage
http.formLogin().loginPage("/toLogin");
```

2、前端的 index 页面也需要修改为 "toLogin"

```html
<a class="item" th:href="@{/toLogin}">
    <i class="address card icon"></i> 登录
</a>
```

3、login 页面中，当点击登录按钮后，绑定 "toLogin"

```html
<form th:action="@{/toLogin}" method="post">
    <div class="field">
        <label>Username</label>
        <div class="ui left icon input">
            <input type="text" placeholder="Username" name="username">
            <i class="user icon"></i>
        </div>
    </div>
    <div class="field">
        <label>Password</label>
        <div class="ui left icon input">
            <input type="password" name="password">
            <i class="lock icon"></i>
        </div>
    </div>
    <div>
        <input type="checkbox" name="rememberMe"/> 记住我
    </div>
    <input type="submit" class="ui blue submit button"/>
</form>
```

4、请求提交上来，我们还需要验证处理，怎么做呢？我们可以查看 formLogin() 方法的源码，配置接收登录的用户名和密码的参数

```java
http.formLogin()
  .usernameParameter("username")
  .passwordParameter("password")
  .loginPage("/toLogin");
```

5、登录页增加记住我的多选框
```html
<input type="checkbox" name="rememberMe"> 记住我
```

6、后端验证处理！
```java
//定制记住我的参数！
http.rememberMe().rememberMeParameter("rememberMe");
```

7、启动项目测试

## 总结

本篇我们学习了 Spring Security，了解了认证和授权两大功能如何配置，以及控制权限、定制首页、注销和记住我等功能。

参考自：
1. [狂神说SpringBoot18: 集成SpringSecurity](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483957&idx=1&sn=fc30511490b160cd1519e7a7ee3d4ed0&scene=19#wechat_redirect)
