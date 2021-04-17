---
layout:     post
title:      1. Mybatis First Application
subtitle:   
date:       2021-04-06
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Mybatis
---

## What is Mybatis

一款优秀的 **持久层** 框架，它支持自定义 SQL、存储过程以及高级映射。\
**简化 JDBC**：免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。\
可通过简单的 **XML或注解** 来配置和映射原始类型、接口和 POJO 类为数据库中的记录。

## Mybatis 第一个程序

### 搭建实验数据库

建库建表，创建一个简单的 user 表

```sql
CREATE DATABASE `mybatis`;

USE `mybatis`;

DROP TABLE IF EXISTS `user`;

CREATE TABLE `user` (
`id` int(20) NOT NULL,
`name` varchar(30) DEFAULT NULL,
`pwd` varchar(30) DEFAULT NULL,
PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

insert into `user`(`id`,`name`,`pwd`) values 
(1,'张三','123456'),
(2,'李四','abcdef'),
(3,'王五','987654');
```

### 导入 Mybatis 相关 jar 包

除了 mybatis jar 包外，还需要连接数据库的包，以及 junit 测试包

```xml
<dependencies>

    <!-- Mybatis -->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.2</version>
    </dependency>

    <!-- MySQL 数据库连接 -->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.23</version>
    </dependency>

    <!-- Junit -->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.13.2</version>
        <scope>test</scope>
    </dependency>

</dependencies>
```

### 编写 Mybatis 核心配置文件 mybatis-config.xml

该文件包括 Mybatis 的所有配置，这里我们先配置一下数据源，以及之后要注册绑定 Mapper.xml 文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useSSL=true&amp;useUnicode=true&amp;characterEncoding=utf8"/>
                <property name="username" value="root"/>
                <property name="password" value="12345678"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="com/zhao/mapper/UserMapper.xml"/>
    </mappers>

</configuration>
```

### 编写 Mybatis 工具类

```java
public class MybatisUtils {

    private static SqlSessionFactory sqlSessionFactory;

    static {
        try {
            // 核心配置文件路径
            String resource = "mybatis-config.xml";
            InputStream inputStream = Resources.getResourceAsStream(resource);
            // 创建 SqlSessionFactory 实例
            sqlSessionFactory = new SqlSessionFactoryBuilder().build (inputStream);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // 获取 SqlSession 连接
    public static SqlSession getSession(){
        return sqlSessionFactory.openSession();
    }
}
```

### 创建实体类

```java
public class User {
   
   private int id; 
   private String name;   
   private String pwd; 

   // 构造函数：有参，无参
   // set/get
   // toString()
}
```

### 编写 Mapper 接口类

```java
public interface UserMapper {

    List<User> selectAllUsers();

}
```

### 编写 Mapper.xml 配置文件

Mapper.xml 可以理解为 Mapper 接口的实现类 \
Namespace 不要写错，要准确对应 Mapper 接口的完整包名

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.zhao.mapper.UserMapper">

    <select id="selectAllUsers" resultType="com.zhao.pojo.User">
        select * from mybatis.user
    </select>

</mapper>
```

### 测试

```java
public class MyTest {

    @Test
    public void testSelectAllUsers() {
        SqlSession session = MybatisUtils.getSession();
        UserMapper mapper = session.getMapper(UserMapper.class);
        List<User> userList = mapper.selectAllUsers();
        for (User user : userList) {
            System.out.println(user);
        }
    }
}
```

### 可能遇到的问题

由于 Mapper.xml 配置文件在 src/main 目录下，可能会出现 Maven 打包时过滤静态资源的问题。错误信息为

![img](/img/post/Mybatis/not_found_UserMapper.png)

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

## 总结

本篇我们通过第一个程序基本了解了 Mybatis。

参考自：
1. [Mybatis Docs](https://mybatis.org/mybatis-3/zh/getting-started.html)

2. [狂神说MyBatis01：第一个程序](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247484031&idx=1&sn=948869263f6dd06ccfb494cc5f07c7c4&scene=19#wechat_redirect)
