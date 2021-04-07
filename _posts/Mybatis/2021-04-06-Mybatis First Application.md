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

MyBatis 是一款优秀的 **持久层** 框架，它支持自定义 SQL、存储过程以及高级映射。\
MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。\
MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 POJO 类为数据库中的记录。

## Mybatis 第一个程序

### 搭建实验数据库

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

insert  into `user`(`id`,`name`,`pwd`) values 
(1,'狂神','123456'),
(2,'张三','abcdef'),
(3,'李四','987654');
```

### 导入 Mybatis 相关 jar 包

```xml
<dependency>
   <groupId>org.mybatis</groupId>
   <artifactId>mybatis</artifactId>
   <version>3.5.2</version>
</dependency>

<dependency>
   <groupId>mysql</groupId>
   <artifactId>mysql-connector-java</artifactId>
   <version>5.1.47</version>
</dependency>
```

### 编写 Mybatis 核心配置文件

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
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
       <mapper resource="com/kuang/dao/userMapper.xml"/>
  </mappers>
</configuration>
```

4、编写 Mybatis 工具类

```java
public class MybatisUtils {
 
    private static SqlSessionFactory  sqlSessionFactory;
 
    static {
        try {
            // 核心配置文件路径
            String resource = "mybatis-config.xml";
            InputStream inputStream = Resources. getResourceAsStream(resource);
            // 创建 SqlSessionFactory 实例
            sqlSessionFactory = new  SqlSessionFactoryBuilder().build (inputStream);
       } catch (IOException e) {
            e.printStackTrace();
       }
    }
 
    //获取 SqlSession 连接
    public static SqlSession getSession(){
        return sqlSessionFactory.openSession();
    }
 
}
```

### 创建实体类

```java
public class User {
   
   private int id;  //id
   private String name;   //姓名
   private String pwd;   //密码
   
   //构造,有参,无参
   //set/get
   //toString()
}
```

### 编写 Mapper 接口类

```java
import com.kuang.pojo.User;
import java.util.List;

public interface UserMapper {
   List<User> selectUser();
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
<mapper namespace="com.kuang.dao.UserMapper">
 <select id="selectUser" resultType="com.kuang.pojo.User">
  select * from user
 </select>
</mapper>
```

### 测试

```java
public class MyTest {
    @Test
    public void selectUser() {
        SqlSession session = MybatisUtils. getSession();
        //方法一: 不推荐使用
        //List<User> users = session.selectList ("com.kuang.mapper.UserMapper.selectUser");
        //方法二:
        UserMapper mapper = session.getMapper (UserMapper.class);
        List<User> users = mapper.selectUser();
 
        for (User user: users){
            System.out.println(user);
        }
        session.close();
    }
}
```

### 可能遇到的问题

由于 Mapper.xml 配置文件在 src/main 目录下，可能会出现 Maven 打包时过滤静态资源的问题。

```xml
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
```

## 总结

本篇我们通过第一个程序基本了解了 Mybatis。

参考自：
1. [Mybatis Docs](https://mybatis.org/mybatis-3/zh/getting-started.html)

2. [狂神说MyBatis01：第一个程序](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247484031&idx=1&sn=948869263f6dd06ccfb494cc5f07c7c4&scene=19#wechat_redirect)
