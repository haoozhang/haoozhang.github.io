---
layout:     post
title:      Spring Cloud 通过 REST API 消费服务
subtitle:   
date:       2021-06-10
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

本文通过搭建服务提供者 (Provider) 和消费者 (Comsumer) 的环境学习 Spring Cloud 通过 REST 调用服务，顺便复习 Spring、Mybatis、Maven 等知识。

## Spring Cloud 版本选择

由于 Spring Cloud 依赖 Spring Boot 开发，因此两者的版本要对应，否则会出现一些莫名其妙的问题。打开 Spring Cloud 某个版本的官方文档，首部会出现当前版本支持的 Spring Boot 版本，如下图：

![img](/img/SpringCloud/cloud_boot_version.png)

这里我们选择 Spring Cloud 最新的稳定版本 2020.0.3，其对应的 Spring Boot 版本为 2.4.6。

## 搭建实体类微服务环境

为了更好的服务拆分，我们将实体类抽离出来作为一个微服务。

1、新建 Maven 项目，删除 src 目录，将它作为父项目\
使用依赖管理添加如下依赖

```xml
<dependencyManagement>
    <dependencies>
        <!-- Spring Cloud -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${springcloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${springboot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <!-- MySQL 连接 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>
        <!-- Druid 数据源 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.6</version>
        </dependency>
        <!-- MyBatis -->
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>2.1.4</version>
        </dependency>
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.20</version>
        </dependency>
        <!-- Log4j -->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>1.2.12</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

使用 properties 标签定义上面依赖的版本

```xml
<properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <!-- 查看 Spring Cloud 和 Spring Boot 的对应版本 -->
    <springcloud.version>2020.0.3</springcloud.version>
    <springboot.version>2.4.6</springboot.version>
    <mysql.version>8.0.23</mysql.version>
</properties>
```

2、在父项目中新建 Api 模块，并引入这个模块需要的依赖，这里我们只引入 Lombok 方便实体类的编写

```xml
<!-- 当前 Module 需要的依赖 -->
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

3、新建 cloud 数据库，新建 Dept 表，表示部门信息，并添加如下列和数据

```sql
insert into dept (name, db_source) values ('develop', DATABASE())
insert into dept (name, db_source) values ('market', DATABASE())
insert into dept (name, db_source) values ('recruit', DATABASE())
insert into dept (name, db_source) values ('finance', DATABASE())

id      name        db_source
1       develop     cloud
2       market      cloud
3       recruit     cloud
4       finance     cloud
```

4、IDEA 连接数据库，并新建对应的实体类

```java
@Data
@NoArgsConstructor
@Accessors(chain = true)  // 链式写法
public class Dept implements Serializable {

    private Long id;
    private String name;
    private String dbSource;

    public Dept(String name) {
        this.name = name;
    }
}
```

实体类的微服务就完成了，接下来我们搭建服务提供者的环境

## 服务提供者

1、新建 Maven 模块，并引入以下依赖，我们从零开始搭建一个 Spring Boot 项目

```xml
<dependencies>
    <!-- 需要实体类 -->
    <dependency>
        <groupId>com.zhao</groupId>
        <artifactId>api</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
    </dependency>
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
    </dependency>
</dependencies>
```

2、新建 application.yml，配置文件中配置数据源等信息

```yml
server:
  port: 8081

mybatis:
  type-aliases-package: com.zhao.pojo
  mapper-locations: classpath:mybatis/mapper/*.xml
  config-location: classpath:mybatis-config.xml

spring:
  application:
    name: provider
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/cloud?useUnicode=true&characterEncoding=utf8
    username: root
    password: 12345678
```

在 resources 目录下新建对应的 mybatis/mapper 目录，并新建编写 Mybatis 的核心配置文件 mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <settings>
        <!-- 开启驼峰命名 -->
        <setting name="cacheEnabled" value="true"/>
        <!-- 数据库中的'下划线命名'转换为'驼峰命名' -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>

</configuration>
```

3、编写 Mybatis 的 Mapper 接口和 xml 文件

```java
@Mapper
@Repository
public interface DeptMapper {

    boolean addDept(Dept dept);

    Dept selectDeptById(Long id);

    List<Dept> selectAllDept();
}
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.zhao.mapper.DeptMapper">

    <insert id="addDept" parameterType="Dept">
        insert into dept (name, db_source)
        values (#{name}, DATABASE());
    </insert>

    <select id="selectDeptById" parameterType="Long" resultType="Dept">
        select * from dept where id = #{id};
    </select>

    <select id="selectAllDept" resultType="Dept">
        select * from dept;
    </select>

</mapper>
```

4、编写 Service 层

```java
public interface DeptService {

    boolean addDept(Dept dept);

    Dept selectDeptById(Long id);

    List<Dept> selectAllDept();
}
```

```java
@Service
public class DeptServiceImpl implements DeptService {

    @Autowired
    private DeptMapper deptMapper;

    @Override
    public boolean addDept(Dept dept) {
        return deptMapper.addDept(dept);
    }

    @Override
    public Dept selectDeptById(Long id) {
        return deptMapper.selectDeptById(id);
    }

    @Override
    public List<Dept> selectAllDept() {
        return deptMapper.selectAllDept();
    }
}
```

5、编写 Controller 层

```java
@RestController
public class DeptController {

    @Autowired
    private DeptService deptService;

    @PostMapping("/dept/add")
    public boolean addDept(@RequestBody Dept dept) {
        return deptService.addDept(dept);
    }

    @GetMapping("/dept/get/{id}")
    public Dept getDept(@PathVariable("id") Long id) {
        return deptService.selectDeptById(id);
    }

    @GetMapping("/dept/get")
    public List<Dept> getAllDept() {
        return deptService.selectAllDept();
    }
}
```

这里写的接口，我们就需要消费者来调用它们。

6、编写 Spring 的启动类

```java
@SpringBootApplication
public class DeptProvider {

    public static void main(String[] args) {
        SpringApplication.run(DeptProvider.class, args);
    }
}
```

配置消费者的环境之前，我们先测试访问 localhost:8081 确保在提供者本地接口调用成功

## 服务消费者

1、同样地，新建 Maven 模块，并引入依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.zhao</groupId>
        <artifactId>api</artifactId>
        <version>1.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
    </dependency>
</dependencies>
```

2、编写 controller 的接口

```java
@RestController
public class DeptController {

    @GetMapping("/dept/get/{id}")
    public Dept getDept(@PathVariable("id") Long id) {
    }

    @GetMapping("/dept/get")
    public List<Dept> getAllDept() {
    }

    @RequestMapping("/dept/add")
    public boolean addDept(Dept dept) {
    }
}
```

作为消费者，我们不应该调用 Service 层，而是要考虑如何远程调用提供的服务。这里我们使用 RestTemplate。\
Spring Cloud 抛弃了 Dubbo 的 RPC 通信方式，改用基于 Http 的 REST API 调用方式，而 RestTemplate 是封装好的服务调用模板，用于访问远程 Http 服务，我们查看其源码可以看到如下的调用方法：

![img](/img/SpringCloud/restTemplate.png)

因此，我们使用 RestTemplate 的这些方法调用服务提供者的接口

```java
@RestController
public class DeptController {

    // 消费者不应有 Service 层，那么如何调用提供者的服务呢？Spring 支持 RestFul 风格
    // Spring Cloud 中抛弃 Dubbo 的 RPC 通信，基于 Http 的 REST API 通信
    // RestTemplate 是 RestFul 服务的模板，用于访问远程 Http 服务器
    @Autowired
    private RestTemplate restTemplate;

    private static final String REST_URL_PREFIX = "http://localhost:8081";

    @GetMapping("/dept/get/{id}")
    public Dept getDept(@PathVariable("id") Long id) {
        return restTemplate.getForObject(REST_URL_PREFIX+"/dept/get/"+id, Dept.class);
    }

    @GetMapping("/dept/get")
    public List<Dept> getAllDept() {
        return restTemplate.getForObject(REST_URL_PREFIX+"/dept/get", List.class);
    }

    @RequestMapping("/dept/add")
    public boolean addDept(Dept dept) {
        Assert.notNull(dept, "dept must not be null!");
        return restTemplate.postForObject(REST_URL_PREFIX+"/dept/add", dept, Boolean.class);
    }

}
```

3、但是，查看 RestTemplate 源码我们并没有发现它被注册到容器中，所以我们手动的将它添加为 bean 实例

```java
@Configuration
public class BeanConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

4、编写 Spring 的启动类

```java
@SpringBootApplication
public class DeptConsumer {

    public static void main(String[] args) {
        SpringApplication.run(DeptConsumer.class, args);
    }
}
```

5、依次启动提供者、消费者项目，访问如下的路径测试远程调用服务，可以看到对应的输出

```
$ http://localhost:8080/dept/get
[{"id":1,"name":"develop","dbSource":"cloud"},{"id":2,"name":"market","dbSource":"cloud"},{"id":3,"name":"recruit","dbSource":"cloud"},{"id":4,"name":"finance","dbSource":"cloud"}]

$ http://localhost:8080/dept/get/1
{"id":1,"name":"develop","dbSource":"cloud"}

$ http://localhost:8080/dept/add?name=aaa
true (查看数据库的变化)
```

需要注意的是，这里我们是通过消费者的8080端口通过 RestTemplate 调用远程服务提供者的服务，而非本地的调用。可以与 Dubbo+Zookeeper 的远程调用服务做一个比较和思考。



