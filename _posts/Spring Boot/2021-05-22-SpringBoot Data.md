---
layout:     post
title:      7. Spring Boot Data
subtitle:   
date:       2021-05-22
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SpringBoot
---

对于数据访问层，不管是 SQL (关系型数据库) 还是 NOSQL (非关系型数据库)，Spring Boot 底层都是采用 Spring Data 统一处理，Spring Data 也是 Spring 中与 Spring Boot、Spring Cloud 等齐名的知名项目。

## 整合 JDBC

1、新建 Spring 项目，引入 web、JDBC、MySQL 驱动基础模块

2、项目建好之后，发现 pom.xml 文件中自动包含以下依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- MySQL -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- JDBC -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

3、配置文件中编写数据库连接信息

```yml
spring:
  datasource:
    username: root
    password: 12345678
    # 注意配置时区
    url: jdbc:mysql://localhost:3306/mybatis?useSSL=true&serverTimezone=UTC&useUnicode=true&characterEncoding=utf8
    driver-class-name: com.mysql.cj.jdbc.Driver
```

4、然后我们就可以直接使用了，因为 Spring Boot 默认帮我们自动配置好了。我们可以在测试类中测试

```java
@SpringBootTest
class SpringDataApplicationTests {

    @Autowired
    DataSource dataSource;

    @Test
    void contextLoads() throws SQLException {
        // 查看默认数据源：hikari
        System.out.println(dataSource.getClass());

        Connection connection = dataSource.getConnection();
        System.out.println(connection);
        connection.close();
    }

}
```

从输出中可以看到默认的数据源为 HikariDataSource

我们搜索数据源的自动配置文件：DataSourceAutoConfiguration，可以看到如下代码

```java
@Import({ DataSourceConfiguration.Hikari.class, 
          DataSourceConfiguration.Tomcat.class,
		  DataSourceConfiguration.Dbcp2.class, 
          DataSourceConfiguration.OracleUcp.class,
		  DataSourceConfiguration.Generic.class, 
          DataSourceJmxConfiguration.class })
protected static class PooledDataSourceConfiguration {
}
```

可以看到该版本 Spring Boot 默认支持 HikariDataSource 数据源。我们也可以在配置文件中使用 spring.datasource.type 指定其他数据源类型

### JdbcTemplate

有了 Hikari 数据源，拿到了 Connection 数据库连接，我们就可以使用 JDBC 语句操作数据库了。Spring Boot 对 JDBC 做了轻量级的封装，即 JdbcTemplate

JdbcTemplate 包含了数据库操作的所有 CRUD 方法，主要提供以下几类方法：
+ execute: 用于执行任何SQL语句，一般用于执行 DDL (数据库模式定义，create/alter/drop/truncate) 语句
+ update/batchUpdate: update 用于新增、更新、删除语句，batchUpdate 用于批处理相关语句
+ query/queryForXXX: 执行查询相关语句
+ call: 执行存储过程、函数相关语句

Spring Boot 不仅提供了默认的数据源，而且默认把 JdbcTemplate 注册到容器中，只需注入即可使用。JdbcTemplate 的自动配置是依赖于 JdbcTemplateConfiguration 类

### CRUD 测试

新建一个 Controller，注入 JdbcTemplate，编写测试方法进行访问测试

```java
@RestController
public class JdbcController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    // 查询数据库所有信息
    @GetMapping("/listUser")
    public List<Map<String, Object>> listUser() {
        String sql = "select * from user";
        // List 中的一个 Map 对应一行数据，Map 的 key 为字段名，value 为具体值
        List<Map<String, Object>> mapList = jdbcTemplate.queryForList(sql);
        return mapList;
    }

    // 添加用户
    @GetMapping("/addUser")
    public String addUser() {
        String sql = "insert into mybatis.user(id, name, pwd) values (10, 'xiaoming', '12345678')";
        jdbcTemplate.update(sql);
        return "add-ok";
    }

    // 更新用户
    @GetMapping("/updateUser/{id}")
    public String updateUser(@PathVariable("id") int id) {
        String sql = "update mybatis.user set name = ?, pwd = ? where id = " + id;
        Object[] objects = new Object[2];
        objects[0] = "xiaoming2";
        objects[1] = "4321";
        jdbcTemplate.update(sql, objects);
        return "update-ok";
    }

    // 删除用户
    @GetMapping("/deleteUser/{id}")
    public String deleteUser(@PathVariable("id") int id) {
        String sql = "delete from mybatis.user where id = ?";
        jdbcTemplate.update(sql, id);
        return "delete-ok";
    }
}
```

测试请求，结果正常，使用 JDBC 的 CRUD 就搞定了。

## 整合 Druid

Java 代码不可避免要操作数据库，为了提高数据库访问性能，不得不使用数据库连接池。Druid 是阿里巴巴开源的一个数据库连接池实现，它结合了 C3P0、DBCP 等数据库连接池的优点，并加入了日志监控，是针对监控而生的数据库连接池。这里我们学习 Spring Boot 整合 Druid，实现数据库监控。

### 配置数据源

1、pom.xml 添加 Druid 数据源依赖

```xml
<!-- Druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.6</version>
</dependency>
```

2、切换数据源，上面说过可以通过 spring.datasource.type 指定数据源

```yml
spring:
  datasource:
    username: root
    password: 12345678
    # 注意配置时区
    url: jdbc:mysql://localhost:3306/mybatis?useSSL=true&serverTimezone=UTC&useUnicode=true&characterEncoding=utf8
    driver-class-name: com.mysql.cj.jdbc.Driver
    # 指定使用 Druid
    type: com.alibaba.druid.pool.DruidDataSource
```

3、测试类中注入 DataSource，测试输出看是否切换成功

```java
@SpringBootTest
class SpringDataApplicationTests {

    @Autowired
    DataSource dataSource;

    @Test
    void contextLoads() throws SQLException {
        // 查看默认数据源：hikari
        System.out.println(dataSource.getClass());

        Connection connection = dataSource.getConnection();
        System.out.println(connection);
        connection.close();
    }

}
```

![img](/img/post/SpringBoot/druidDS.png)

4、切换成功后，就可以在配置文件中设置相关参数
```yml
spring:
  datasource:
    username: root
    password: 12345678
    # 注意配置时区
    url: jdbc:mysql://localhost:3306/mybatis?useSSL=true&serverTimezone=UTC&useUnicode=true&characterEncoding=utf8
    driver-class-name: com.mysql.cj.jdbc.Driver
    # 指定使用 Druid
    type: com.alibaba.druid.pool.DruidDataSource

    # 上面是 DataSourceProperties 的属性，下面是 DruidDataSource 的父类 DruidAbstractDataSource 的属性

    # Spring Boot 默认不注入这些属性值，需要自己绑定
    # druid 数据源专有配置
    initialSize: 5
    minIdle: 5
    maxActive: 20
    maxWait: 60000
    timeBetweenEvictionRunsMillis: 60000
    minEvictableIdleTimeMillis: 300000
    validationQuery: SELECT 1 FROM DUAL
    testWhileIdle: true
    testOnBorrow: false
    testOnReturn: false
    poolPreparedStatements: true

    # 配置监控统计拦截的 filters，stat:监控统计、log4j：日志记录、wall：防御sql注入
    # 需要导入 Log4j
    filters: stat,wall,log4j
    maxPoolPreparedStatementPerConnectionSize: 20
    useGlobalDataSourceStat: true
    connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500
```

![img](/img/post/SpringBoot/druidDSconfig.png)

5、导入 Log4j 依赖

```xml
<!-- Log4j -->
<dependency>
    <groupId>log4j</groupId>
    <artifactId>log4j</artifactId>
    <version>1.2.12</version>
</dependency>
```

6、虽然切换为 Druid 数据源之后，Spring Boot 会自动生成 DruidDataSource 并放入容器中供程序员使用，但是它并不会自动绑定配置文件的参数。因此需要程序员自己为 DruidDataSource 绑定全局配置文件中的参数，再添加到容器中，而不再使用 Spring Boot 自动生成的。我们需要自己添加 DruidDataSource 组件到容器中，并绑定全局配置文件的属性

```java
@Configuration
public class DruidConfig {

    /*
     * 自定义 Druid 数据源并添加到容器中，不再让 Spring Boot 自动创建
     * @ConfigurationProperties 作用是将配置文件中相同前缀的属性值，映射到 DruidDataSource 的同名参数中
     */
    @Bean
    @ConfigurationProperties("spring.datasource")
    public DataSource druidDataSource() {
        return new DruidDataSource();
    }
}
```

7、可以在测试类中测试一下

```java
@Test
void contextLoads() throws SQLException {
    // 查看默认数据源：hikari
    System.out.println(dataSource.getClass());
    Connection connection = dataSource.getConnection();
    System.out.println(connection);
    DruidDataSource druidDataSource = (DruidDataSource) dataSource;
    System.out.println(druidDataSource.getMaxActive());
    System.out.println(druidDataSource.getInitialSize());
    connection.close();
}
```

可以看到，输出为我们刚才在配置文件中配置的值，自定义的参数已经生效

![img](/img/post/SpringBoot/druidDStest.png)

### 配置数据源监控

Druid 数据源自动具有监控的功能，并提供了一个 web 界面方便用户后台查看。所以第一步需要设置 Druid 的后台管理页面，比如登录账号、密码等

```java
/*
 * Spring Boot 内置了 Servlet 容器，所以没有 web.xml，我们用替代类 ServletRegistrationBean
 */
@Bean
public ServletRegistrationBean statViewServlet() {
    ServletRegistrationBean<StatViewServlet> bean = new ServletRegistrationBean<>(new StatViewServlet(), "/druid/*");
    // key 参数可以在 StatViewServlet 的父类 ResourceServlet 中查看
    HashMap<String, String> initParams = new HashMap<>();
    // 设置后台监控界面的账户密码
    initParams.put("loginUsername", "admin");
    initParams.put("loginPassword", "123456");
    // 允许谁可以访问
    initParams.put("allow", "");
    // 禁止谁访问
    // initParams.put("deny", )
    bean.setInitParameters(initParams);
    return bean;
}
```

配置完毕后，我们可以选择访问 ：http://localhost:8080/druid/login.html

![img](/img/post/SpringBoot/druidLogin.png)

![img](/img/post/SpringBoot/druidMonitor.png)

接下来，我们配置 Druid 监控的过滤器。过滤器的作用就是统计 web 应用请求中所有的数据库信息，比如发出的 sql 语句，sql 执行的时间、请求次数、请求的 url 地址、以及seesion 监控、数据库表的访问次数等

```java
@Bean
public FilterRegistrationBean webStatFilter() {
    FilterRegistrationBean bean = new FilterRegistrationBean();
    bean.setFilter(new WebStatFilter());
    HashMap<String, String> initParams = new HashMap<>();
    // 这些不统计
    initParams.put("exclusions", "*.js,*.css,/druid/*");
    bean.setInitParameters(initParams);
    return bean;
}
```

## 整合 Mybatis

1、导入 MyBatis 所需要的依赖

```xml
<!-- Mybatis 整合 Spring Boot -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.4</version>
</dependency>
```

2、配置数据库连接信息（不变）

3、测试数据库连接成功

4、导入 lombok，创建实体类

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Teacher {

    private int id;
    private String name;
}
```

5、创建 Mapper 和对应的 Mapper 映射文件

```java
@Mapper  // 表示这是一个 Mybatis 的 mapper 类
@Repository
public interface TeacherMapper {

    List<Teacher> selectAllTeacher();

    Teacher selectTeacherById(int id);

    int addTeacher(Teacher teacher);

    int updateTeacher(Teacher teacher);

    int deleteTeacher(int id);
}
```

这里我们在 resources 目录下创建 mybatis/mapper 目录，将 Mapper 映射文件放在这里

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.zhao.mapper.TeacherMapper">

  <select id="selectAllTeacher" resultType="Teacher">
    select * from Teacher
  </select>

  <select id="selectTeacherById" resultType="Teacher">
      select * from Teacher where id = #{id}
  </select>

  <insert id="addTeacher" parameterType="Teacher">
      insert into Teacher(id, name) values (#{id}, #{name})
  </insert>

  <update id="updateTeacher" parameterType="Teacher">
      update Teacher set name = #{name} where id = #{id}
  </update>

  <delete id="deleteTeacher" parameterType="int">
      delete from Teacher where id = #{id}
  </delete>

</mapper>
```

然后在配置文件中要指定 Teacher 的别名，和 mapper 映射文件的位置

```yml
# 整合 Mybatis
mybatis:
  type-aliases-package: com.zhao.pojo
  mapper-locations:
    - classpath:mybatis/mapper/*.xml
```

6、避免 Maven 过滤静态资源问题

```xml
<build>
    <resources>
       <resource>
           <directory>src/main/java</directory>
           <includes>
               <include>**/*.properties</include>
               <include>**/*.xml</include>
           </includes>
           <filtering>false</filtering>
       </resource>
       <resource>
           <directory>src/main/resources</directory>
           <includes>
               <include>**/*.properties</include>
               <include>**/*.xml</include>
           </includes>
           <filtering>false</filtering>
       </resource>
    </resources>
</build>
```

7、编写 controller 测试

```java
@RestController
public class TeacherController {

    @Autowired
    private TeacherMapper teacherMapper;

    @GetMapping("/listTeacher")
    public List<Teacher> listTeacher() {
        return teacherMapper.selectAllTeacher();
    }

    @GetMapping("/listTeacher/{id}")
    Teacher selectUserById(@PathVariable int id) {
        return teacherMapper.selectTeacherById(id);
    }

    @GetMapping("addTeacher")
    int addTeacher() {
        Teacher teacher = new Teacher(2, "zhanghao");
        return teacherMapper.addTeacher(teacher);
    }

    @GetMapping("updateTeacher")
    int updateTeacher() {
        Teacher teacher = new Teacher(2, "张老师");
        return teacherMapper.updateTeacher(teacher);
    }

    @GetMapping("/deleteTeacher/{id}")
    int deleteTeacher(@PathVariable int id) {
        return teacherMapper.deleteTeacher(id);
    }

}
```

启动项目，测试访问 localhost:8080/ 下的 CURD 请求。

## 总结

本篇我们学习了 Spring Boot 整合数据访问层的几种方式。首先，Spring Boot 会原生的 JDBC 做了轻量级的封装，可通过 JdbcTemplate 直接操作数据库；然后，Druid 数据源配置可用于监控数据库的操作和状态；最后学习了如何整合 Mybatis。

参考自：
1. [狂神说SpringBoot07: 整合JDBC](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483785&idx=1&sn=cbf46019c14be7129bcd39002ab16706&scene=19#wechat_redirect)

2. [狂神说SpringBoot08: 整合Druid](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483786&idx=1&sn=f5f4ca792611af105140752eb67ce820&scene=19#wechat_redirect)

3. [狂神说SpringBoot09: 整合MyBatis](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247483788&idx=1&sn=aabf8cf31d7d45be184cc59cdb75258c&scene=19#wechat_redirect)
