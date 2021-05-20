---
layout:     post
title:      6. Spring Boot Web 开发实战项目
subtitle:   
date:       2021-05-19
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SpringBoot
---

上一篇我们通过分析源码明白了 Web 开发中遇到的常见问题该如何解决，例如静态资源怎么访问、首页怎么定制、模板引擎以及如何扩展 MVC。这一篇我们通过一个实战项目正式学习 Spring Boot Web 开发。

## 准备工作

### 静态资源准备

我们先从[这里](https://github.com/haozhangms/SpringBoot_blog/tree/main/SpringBoot%20Web%20Template)下载好静态资源文件，主要包括几个页面和一些静态资源。

新建 Spring Boot 项目，将上面文件夹中的 html 页面放到项目工程的 template 目录下，将 assets 文件夹下的内容放到 static 目录下，这样静态资源就导入完毕了，之后我们再根据需求详细修改这些静态资源。

### 数据准备

暂时我们还没整合数据库，所以我们伪造一些数据库的数据和操作。具体地，我们先新建两个 pojo 类

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Department {

    private Integer departmentId;
    private String departmentName;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class Employee {

    private Integer id;
    private String name;
    private String email;
    private Integer gender;  // female: 0, male: 1
    private Department department;
    private Date birth;
}
```

可以看到，我们已经使用了 Lombok 自动化生成 get、set 方法。接下来我们再创建 mapper 层，首先新建 DepartmentMapper

```java
@Repository
public class DepartmentMapper {

    // 暂时模拟数据库数据
    private static Map<Integer, Department> departmentMap = null;
    static {
        departmentMap = new HashMap<>();
        departmentMap.put(1, new Department(1, "教学部"));
        departmentMap.put(2, new Department(2, "科研部"));
        departmentMap.put(3, new Department(3, "市场部"));
        departmentMap.put(4, new Department(4, "后勤部"));
        departmentMap.put(5, new Department(5, "运营部"));
    }

    // 数据库操作
    public Collection<Department> selectAllDepartment() {
        return departmentMap.values();
    }

    public Department selectDepartmentById(int id) {
        return departmentMap.get(id);
    }
}
```

这里我们先模拟生成了数据库中的数据，并用 map 保存；然后定义了两个方法操作 "数据库" 中的数据；最后将这个类注册到容器中。类似地，新建 EmployeeMapper

```java
@Repository
public class EmployeeMapper {

    // 暂时模拟数据库数据
    private static Map<Integer, Employee> employeeMap = null;
    static {
        employeeMap = new HashMap<>();
        employeeMap.put(1, new Employee(1, "aa", "aa@qq.com", 1, new Department(1, "教学部"), new Date()));
        employeeMap.put(2, new Employee(2, "bb", "bb@qq.com", 0, new Department(2, "科研部"), new Date()));
        employeeMap.put(3, new Employee(3, "cc", "cc@qq.com", 1, new Department(3, "市场部"), new Date()));
        employeeMap.put(4, new Employee(4, "dd", "dd@qq.com", 0, new Department(4, "后勤部"), new Date()));
        employeeMap.put(5, new Employee(5, "ee", "ee@qq.com", 1, new Department(5, "运营部"), new Date()));
    }

    // 数据库操作
    public Collection<Employee> selectAllEmployee() {
        return employeeMap.values();
    }

    public Employee selectEmployeeById(int id) {
        return employeeMap.get(id);
    }

    // 主键自增
    private static int initId = 6;

    public void addEmployee(Employee employee) {
        if (employee.getId() == null) {
            employee.setId(initId++);
        }
        employeeMap.put(employee.getId(), employee);
    }
}
```

这样 pojo 和 mapper 数据就准备好了，之后正式开始功能的实现

## 首页处理

实现首页有两种方式，第一种方式是通过 controller 实现，我们新建一个 IndexController 类，编写如下接口

```java
@GetMapping({"/", "/index.html"})
public String index() {
    return "index";
}
```

第二种方式是通过扩展 SpringMVC，我们上一篇学习到，可以通过新建 @Configuration 类再实现 WebMvcConfigurer 接口来扩展 MVC。

```java
@Configuration
public class MyMvcConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        // 首页映射
        registry.addViewController("/").setViewName("index");
        registry.addViewController("/index.html").setViewName("index");
    }
}
```

通过以上任一种方式都可实现首页，我们推荐第二种。当我们启动项目测试访问时，会发现一个问题，首页的静态资源没有加载进来，这是因为使用 Thymeleaf 之后，我们还没有把 index.html 页面改为 Thymeleaf 支持的格式

1、先添加 Thymeleaf 的命名空间

```html
xmlns:th="http://www.thymeleaf.org"
```

2、然后更改几个 html 页面中的超链接，通过查看 Thymeleaf 文档超链接使用 "@{}" 符号，以 index.html 为例

![img](/img/post/SpringBoot/index_th.png)

3、为了防止之前的被缓存导致我们修改的不生效，我们在配置文件汇总禁用 Thymeleaf 缓存

```yml
spring:
  thymeleaf:
    cache: false  # 关闭模板引擎的缓存
```

## 国际化

有时我们的网页涉及到多语言的切换，这时我们就需要学习国际化了。我们首先要统一 IDEA 中的文件编码问题

![img](/img/post/SpringBoot/fileEncoding.png)

### 编写 i18n 配置文件

1、在 resources 目录下新建 i18n 目录，存放国际化配置文件

2、新建 login.properties 文件，再新建 login_zh_CN.properties 文件，发现 IDEA 自动识别了我们做的国际化操作，文件夹有变化

3、我们可以在它上面新建英文的配置文件

![img](/img/post/SpringBoot/newResourceBundle.png)
![img](/img/post/SpringBoot/add_en_US.png)

4、接下来我们编写配置，可以在 Resource Bundle 中同步可视化编写以下内容

![img](/img/post/SpringBoot/i18n_properties.png)

默认的 login.properties
```
login.btn=登录
login.password=密码
login.remember=记住我
login.tip=请登录
login.username=用户名
```
英文
```
login.btn=Sign in
login.password=Password
login.remember=Remember me
login.tip=Please sign in
login.username=Username
```
中文
```
login.btn=登录
login.password=密码
login.remember=记住我
login.tip=请登录
login.username=用户名
```

### 配置文件生效探究

我们探究一下 Spring Boot 对国际化的自动配置，这里又涉及到一个自动配置类：MessageSourceAutoConfiguration

我们可以通过源码看到，通过 spring.messages 前缀配置相关的属性
```java
@ConfigurationProperties(prefix = "spring.messages")
```

并且在 messageSource 方法中，Spring Boot 已经帮我们自动配置好了国际化的组件

```java
@Bean
public MessageSource messageSource(MessageSourceProperties properties) {
    ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
    // 获取 properties 绑定的配置文件的值进行判断
    // 从 MessageSourceProperties 源码中可以看到 basename 默认为 messages
    if (StringUtils.hasText(properties.getBasename())) {
        // 设置国际化文件的基础名 (去掉语言国家代码的)
        messageSource.setBasenames(StringUtils.commaDelimitedListToStringArray(StringUtils.trimAllWhitespace(properties.getBasename())));
    }
    if (properties.getEncoding() != null) {
        messageSource.setDefaultEncoding(properties.getEncoding().name());
    }
    messageSource.setFallbackToSystemLocale(properties.isFallbackToSystemLocale());
    Duration cacheDuration = properties.getCacheDuration();
    if (cacheDuration != null) {
        messageSource.setCacheMillis(cacheDuration.toMillis());
    }
    messageSource.setAlwaysUseMessageFormat(properties.isAlwaysUseMessageFormat());
    messageSource.setUseCodeAsDefaultMessage(properties.isUseCodeAsDefaultMessage());
    return messageSource;
}
```

这里我们是放在 i18n 目录下，所以我们在配置文件中配置 messages 的路径

```yml
spring:
  messages:
    basename: i18n.login  # 国际化配置
```

### 配置页面国际化值

现在我们已经写好了国际化的配置文件，并且让 Spring Boot 自动配置，接下来我们要在页面上获取和显示国际化的值

查看 Thymeleaf 的文档，找到 message 取值操作为：#{...}，我们去页面获取国际化的值，IDEA 已经智能的帮我们做好了提示

![img](/img/post/SpringBoot/i18n_thymeleaf.png)

启动项目测试，发现已经可以识别为中文了

![img](/img/post/SpringBoot/ch_index.png)

### 点击按钮自动切换中英文

但是我们想要更好，想根据按钮自动切换中英文！

Spring 中有一个用于国际化解析的对象 LocaleResolver，我们在 webmvc 自动配置类中查找，可以看到 Spring Boot 的自动配置

```java
@Bean
@ConditionalOnMissingBean(
    name = {"localeResolver"}
)
public LocaleResolver localeResolver() {
    // 如果容器中有用户配置的，就用用户配置的
    if (this.webProperties.getLocaleResolver() == org.springframework.boot.autoconfigure.web.WebProperties.LocaleResolver.FIXED) {
        return new FixedLocaleResolver(this.webProperties.getLocale());
    } else if (this.mvcProperties.getLocaleResolver() == org.springframework.boot.autoconfigure.web.servlet.WebMvcProperties.LocaleResolver.FIXED) {
        return new FixedLocaleResolver(this.mvcProperties.getLocale());
    } else {
        // 否则用默认的
        AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
        Locale locale = this.webProperties.getLocale() != null ? this.webProperties.getLocale() : this.mvcProperties.getLocale();
        localeResolver.setDefaultLocale(locale);
        return localeResolver;
    }
}
```

在 AcceptHeaderLocaleResolver 类中有一个方法

```java
public Locale resolveLocale(HttpServletRequest request) {
    Locale defaultLocale = this.getDefaultLocale();
    // 默认配置：根据请求头带来的区域信息获取 Locale 进行国际化
    if (defaultLocale != null && request.getHeader("Accept-Language") == null) {
        return defaultLocale;
    } else {
        Locale requestLocale = request.getLocale();
        List<Locale> supportedLocales = this.getSupportedLocales();
        if (!supportedLocales.isEmpty() && !supportedLocales.contains(requestLocale)) {
            Locale supportedLocale = this.findSupportedLocale(request, supportedLocales);
            if (supportedLocale != null) {
                return supportedLocale;
            } else {
                return defaultLocale != null ? defaultLocale : requestLocale;
            }
        } else {
            return requestLocale;
        }
    }
}
```

这里 AcceptHeaderLocaleResolver 实现了 LocaleResolver 接口，所以我们也可以定义一个实现了该接口的类，用于自己配置我们的 LocaleResolver

我们去写一个自己的 LocaleResolver，可以在链接上携带区域信息，我们修改一下前端页面的跳转连接：

```html
<a class="btn btn-sm" th:href="@{/index.html(lang='zh_CN')}">中文</a>
<a class="btn btn-sm" th:href="@{/index.html(lang='en_US')}">English</a>
```

```java
public class MyLocaleResolver implements LocaleResolver {
    @Override
    public Locale resolveLocale(HttpServletRequest httpServletRequest) {
        // 获取请求的语言参数
        String language = httpServletRequest.getParameter("lang");

        Locale locale = Locale.getDefault();
        if (!StringUtils.isEmpty(language)) {
            String[] split = language.split("_");
            locale = new Locale(split[0], split[1]);
        }
        return locale;
    }

    @Override
    public void setLocale(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Locale locale) {

    }
}
```

为了让自定义的 MyLocaleResolver 生效，我们要把它注册到容器中。我们在自定义的 MyMvcConfig 下添加 bean

```java
// 自定义国际化, 方法名必须为 localeResolver
// 因为 DispatcherServlet 类的 initLocaleResolver 方法中， getBean 指定了 beanName 为 localeResolver
@Bean
public LocaleResolver localeResolver() {
    return new MyLocaleResolver();
}
```

重启项目访问，发现可以通过按钮实现切换中英文，搞定！

## 登录功能

接下里我们实现登录功能，我们首先修改 index 页面的登录按钮，让它绑定到指定的接口，并给两个文本框指定一个名字

![img](/img/post/SpringBoot/index_thaction.png)

然后新建 LoginController，编写对应的接口

```java
@Controller
public class LoginController {

    @GetMapping("/login")
    public String login(@RequestParam("username") String username,
                        @RequestParam("password") String password,
                        Model model) {
        if ("admin".equals(username) && "123456".equals(password)) {
            // 登录成功，跳转到主界面
            return "dashboard";
        } else {
            // 登录失败，回传错误信息
            model.addAttribute("msg", "用户名或密码有误");
            return "index";
        }
    }
}
```

我们已经实现了登录功能，接下来我们做一些改进。我们想要在登录失败后，给用户一个 "用户名或密码有误" 的提示，我们在 index 页面对应位置添加一个标签

![img](/img/post/SpringBoot/index_thif.png)

这里我们指定，如果登录失败，即 msg 不为空，我们就显示这个标签，我们使用 th:if 操作

当我们登录成功后，我们发现 url 有些问题，如下图所示。

![img](/img/post/SpringBoot/redirect_main0.png)

我们想把页面映射过去，这里我们首先在扩展 SpringMVC 的 MyMvcConfig 类中添加映射，然后在 login 接口中重定向过去

![img](/img/post/SpringBoot/redirect_main1.png)

![img](/img/post/SpringBoot/redirect_main2.png)

启动项目测试，发现登录成功后重定向到了 main 页面

## 登录前拦截

上面我们实现了登录功能，并在登录成功后重定向到 main 页面。但这里有一个问题，即便我们不登录直接访问 main 页面，这时也可以访问到。这不合理，我们要在登录前拦截这种操作，这里就用到之前学习的拦截器 Interceptor。

我们先修改 login 接口，当登录成功后，在 session 中 添加一个 loginUsername 属性来表示用户已经登录成功

```java
@GetMapping("/login")
public String login(@RequestParam("username")String username,
                    @RequestParam("password") String password,
                    Model model,
                    HttpSession session) {
    if ("admin".equals(username) && "123456".equals(password)) {
        session.setAttribute("loginUsername", username);
        // 登录成功，跳转到主界面
        return "redirect:main.html";
    } else {
        // 登录失败，回传错误信息
        model.addAttribute("msg", "用户名或密码有误");
        return "index";
    }
}
```

然后我们自定义一个拦截器类，在覆盖的方法中判断 session 是否有 loginUsername 属性，如果没有就说明没有登录，我们转发到 index 页面提示先登录

```java
public class LoginInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // 登录成功后有用户的 session
        Object loginName = request.getSession().getAttribute("loginUsername");
        if (loginName == null) {
            request.setAttribute("msg", "无权限，请先登录");
            request.getRequestDispatcher("/index.html").forward(request, response);
            return false;
        } else {
            return true;
        }
    }
}
```

我们还需要在扩展 SpringMVC 的类中添加自定义的拦截器，可以设置拦截那些请求，不拦截哪些请求

```java
@Override
public void addInterceptors(InterceptorRegistryregistry) {
    registry.addInterceptor(new LoginInterceptor())
            .addPathPatterns("/**")
            .excludePathPatterns("/", "/index.html", "/login", "static/**");
}
```

最后启动项目测试，我们可以先访问 main 页面，发现被转发到登录页面，拦截成功！

## 展示列表

接下来完成核心功能：增删改查。我们先展示员工列表

### 公共页面代码提取和复用

我们观察 dashboard 和 list 两个页面可以看到，它们的顶边栏和侧边栏都是一样的，所以我们可以提取复用这部分公共的页面代码。首先，我们新建 commons/commons.html 文件，然后使用 th:fragment 提取以下代码片段

![img](/img/post/SpringBoot/th_fragment.png)

然后在 dashbaord 和 list 页面的顶栏和侧边栏位置使用 th:replace 引用这个片段，这样就实现了代码的复用

![img](/img/post/SpringBoot/th_replace.png)

为了可以在点击侧边栏按钮时实现高亮，我们在 th:replace 时传入参数，如上图所示。然后在渲染侧边栏时接受参数并判断

![img](/img/post/SpringBoot/active.png)

### 列表数据循环展示

列表循环展示数据就比较简单了。我们之前已经在 /emps 接口返回了数据，现在我们把对应的侧边栏绑定该接口，然后在 list 页面的主体部分填充数据即可，这里主要用到了 th:each

```html
<main role="main" class="col-md-9 ml-sm-auto col-lg-10 pt-3 px-4">
	<h2>Section title</h2>
	<div class="table-responsive">
		<table class="table table-striped table-sm">
			<thead>
				<tr>
					<th>id</th>
					<th>name</th>
					<th>email</th>
					<th>gender</th>
					<th>department</th>
					<th>birth</th>
					<th>操作</th>
				</tr>
			</thead>
			<tbody>
				<tr th:each="emp:${emps}">
					<td th:text="${emp.getId()}"></td>
					<td th:text="${emp.getName()}"></td>
					<td th:text="${emp.getEmail()}"></td>
					<td th:text="${emp.getGender()==0?'女':'男'}"></td>
					<td th:text="${emp.getDepartment().getDepartmentName()}"></td>
					<td th:text="${#dates.format(emp.getBirth(), 'yyyy-MM-dd HH:mm:ss')}"></td>
					<td>
						<button class="btn btn-sm btn-primary">编辑</button>
						<button class="btn btn-sm btn-danger">删除</button>
					</td>
				</tr>
			</tbody>
		</table>
	</div>
</main>
```

## 增加员工

### 员工信息表单提交界面

为了新增一个员工，我们需要先写一个提交员工信息的表单界面，我们复制 list.html 名为 add.html，然后只需要在主体界面修改为表单即可

```html
<main role="main" class="col-md-9 ml-sm-auto col-lg-10 pt-3 px-4">
	<form th:action="@{/addEmp}" method="post">
		<div class="form-group">
			<label>Name</label>
			<input type="text" name="name" class="form-control" placeholder="海绵宝宝">
		</div>
		<div class="form-group">
			<label>Email</label>
			<input type="email" name="email" class="form-control" placeholder="1176244270@qq.com">
		</div>
		<div class="form-group">
			<label>Gender</label><br>
			<div class="form-check form-check-inline">
				<input class="form-check-input" type="radio" name="gender" value="1">
				<label class="form-check-label">男</label>
			</div>
			<div class="form-check form-check-inline">
				<input class="form-check-input" type="radio" name="gender" value="0">
				<label class="form-check-label">女</label>
			</div>
		</div>
		<div class="form-group">
			<label>Department</label><br>
			<!-- 这里提交的是 department 的 id，所以 name 要对应 -->
			<select class="form-control" name="department.departmentId">
				<option th:each="dept:${departments}" th:text="${dept.getDepartmentName()}" th:value="${dept.getDepartmentId()}"></option>
			</select>
		</div>
		<div class="form-group">
			<label>Birth</label><br>
			<input type="text" name="birth" class="form-control" placeholder="2020/1/2">
		</div>
		<button type="submit" class="btn btn-primary">添加</button>
	</form>
</main>
```

### 接口

界面写好之后，我们需要写接口来控制跳转到表单界面和表单数据提交到后台。首先，我们实现跳转到表单界面，我们在展示页面 list.html 中添加一个新增按钮，当他被点击就会跳转到 add 页面。

![img](/img/post/SpringBoot/toAdd.png)

对应地，我们实现 /addEmp 接口。这里我们查询了所有部门并返回给前端，因为前端的 add 页面需要显示所有部门供用户选择。

```java
@GetMapping("/addEmp")
public String toAdd(Model model) {
    Collection<Department> departments = departmentMapper.selectAllDepartment();
    model.addAttribute("departments", departments);
    return "emp/add";
}
```

在 add 页面，我们点击提交按钮之后，就会调用 /addEmp post 方法接口，把表单数据传递给后台。我们再实现 /addEmp post 方法接口

```java
@PostMapping("/addEmp")
public String addEmp(Employee employee) {
    // 添加员工操作
    employeeMapper.addEmployee(employee);
    return "redirect:/emps";
}
```

从 add.html 中可以看到，表单的部门信息传递给后台的是部门的 id，所以它的 name 也要对应于 departmentId。并且，我们实现的 employeeMapper 中的添加员工的方法，也要先根据部门 id 查询到 Department 对象，再设置到 Employee 对象中

```java
// 主键自增
private static int initId = 6;
public void addEmployee(Employee employee) {
    if (employee.getId() == null) {
        employee.setId(initId++);
    }
    // 传入的对象中只有 department 的 id
    Integer departmentId = employee.getDepartment().getDepartmentId();
    employee.setDepartment(departmentMapper.selectDepartmentById(departmentId));
    employeeMap.put(employee.getId(), employee);
}
```

启动项目，测试添加员工！

最后还需要注意的一点是，birth 格式目前必须为 yyyy/MM/dd 格式，这是因为 Spring Boot 默认的 dateFormatter 默认为该格式，你也可以在配置文件中修改。

## 修改员工

修改员工的逻辑和新增员工差不多。我们在 list 页面的 "编辑" 位置指定一个接口，当点击该按钮时，我们就执行这个接口的方法，把要修改的员工 id 传给后端
```html
<a class="btn btn-sm btn-primary" th:href="@{/updateEmp/}+${emp.id}">编辑</a>
```

我们在 controller 中实现这个接口，因为链接是 Restful 风格，所以接口参数也要是 Restful 风格。接口内查询要修改的员工信息，方便显示在更新页面；并且把部门信息也查询了，方便在页面上显示

```java
@GetMapping("/updateEmp/{id}")
public String toUpdate(@PathVariable("id") Integer id, Model model) {
    // 查询原来的数据
    Employee employee = employeeMapper.selectEmployeeById(id);
    model.addAttribute("emp", employee);
    // 查询部门信息
    Collection<Department> departments = departmentMapper.selectAllDepartment();
    model.addAttribute("departments", departments);
    return "emp/update";
}
```

接下来编写一个 update 页面方便跳转，这里是复制了 add.html 然后修改主体部分内容

```html
<main role="main" class="col-md-9 ml-sm-auto col-lg-10 pt-3 px-4">
	<form th:action="@{/updateEmp}" method="post">
        <!-- id 作为隐藏域传递 -->
		<input type="hidden" name="id" th:value="${emp.getId()}">
		<div class="form-group">
			<label>Name</label>
            <!-- 使用 th:value 显示后台传递的员工 emp 的名字 -->
			<input type="text" name="name" class="form-control" th:value="${emp.getName()}">
		</div>
		<div class="form-group">
			<label>Email</label>
            <!-- 使用 th:value 显示后台传递的员工 emp 的 email -->
			<input type="email" name="email" class="form-control" th:value="${emp.getEmail()}">
		</div>
		<div class="form-group">
			<label>Gender</label><br>
			<div class="form-check form-check-inline">
                <!-- 使用 th:checked 显示哪一个性别被选择 -->
				<input th:checked="${emp.getGender()==1}" class="form-check-input" type="radio" name="gender" value="1">
				<label class="form-check-label">男</label>
			</div>
			<div class="form-check form-check-inline">
				<input th:checked="${emp.getGender()==0}" class="form-check-input" type="radio" name="gender" value="0">
				<label class="form-check-label">女</label>
			</div>
		</div>
		<div class="form-group">
			<label>Department</label><br>
			<!-- 这里提交的是 department 的 id，所以 name 要对应 -->
			<select class="form-control" name="department.departmentId">
                <!-- 使用 th:selected 显示和员工部门相同的部门 -->
				<option th:selected="${emp.getDepartment().getDepartmentId()==dept.getDepartmentId()}" th:each="dept:${departments}" th:text="${dept.getDepartmentName()}" th:value="${dept.getDepartmentId()}"></option>
			</select>
		</div>
		<div class="form-group">
			<label>Birth</label><br>
            <!-- 使用 th:value 显示 birth，并用 #dates.format 格式化 -->
			<input th:value="${#dates.format(emp.getBirth(), 'yyyy/MM/dd')}" type="text" name="birth" class="form-control" placeholder="2020/1/2">
		</div>
		<button type="submit" class="btn btn-primary">更新</button>
	</form>
</main>
```

在提交按钮时，绑定 /updateEmp 接口，我们实现这个接口

```java
@PostMapping("/updateEmp")
public String updateEmp(Employee employee) {
    System.out.println("update--> " + employee);
    // 更新员工操作
    employeeMapper.addEmployee(employee);
    return "redirect:/emps";
}
```

## 删除员工

删除员工比较简单，只需要在 list 页面的删除按钮出绑定一个接口，然后实现这个接口，删除员工即可

```html
<a class="btn btn-sm btn-danger" th:href="@{/deleteEmp/}+${emp.id}">删除</a>
```

```java
@GetMapping("/deleteEmp/{id}")
public String deleteEmp(@PathVariable("id") Integer id) {
    employeeMapper.deleteEmployee(id);
    return "redirect:/emps";
}
```

## 404 处理

Spring Boot 对 404 的配置非常简单，只需要在 templates 目录下新建一个 error 目录，然后放入 404.html 即可

## 注销用户

我们还需要实现注销用户的小功能。具体地，在 commons.html 页面的注销位置绑定一个接口，然后实现这个接口

```html
<a class="nav-link" th:href="@{/logout}">Sign out</a>
```

```java
@GetMapping("/logout")
public String logout(HttpSession session) {
    session.invalidate();
    return "redirect:/index";
}
```

## 总结

本篇我们在了解了 Web 开发源码的基础上，实战练习了一个员工管理系统的 Web 开发项目，主要包括前期的一些准备工作和后面的 CRUD 业务功能。

参考自：
1. [【狂神说Java】SpringBoot最新教程IDEA版通俗易懂](https://www.bilibili.com/video/BV1PE411i7CV?p=28)
