---
layout:     post
title:      3. Mybatis ResultMap 及分页
date:       2021-04-08
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Mybatis
---

## 字段名与属性名不一致

我们之前的项目中，数据库的字段名和实体类中的属性名完全相同。当这两者不一致时，会出现什么问题呢？

### 搭建环境

1、之前的数据库的字段名
+ id, int(20)
+ name, varchar(30)
+ pwd, varchar(30)

2、实体类设计

```java
public class User {

   private int id;  //id
   private String name;   //姓名
   private String password;   //密码和数据库不一样！
   
   //构造
   //set/get
   //toString()
}
```

3、UserMapper 接口
```java
//根据id查询用户
User selectUserById(int id);
```

4、mapper映射文件
```xml
<select id="selectUserById" resultType="user">
  select * from user where id = #{id}
</select>
```

5、测试
```java
@Test
public void testSelectUserById() {
   SqlSession session = MybatisUtils.getSession();  //获取SqlSession连接
   UserMapper mapper = session.getMapper(UserMapper.class);
   User user = mapper.selectUserById(1);
   System.out.println(user);
   session.close();
}
```

结果显示为：User{id=1, name='张三', password='null'}，password 为空

分析：上述的 select 语句可以理解为 

```xml
select id,name,pwd from user where id = #{id}
```

Mybatis 会根据查询的列名去实体类中寻找对应的 set 方法，由于找不到 setPwd 方法，所以 password 为 null。

### 解决方法

#### 方案一：为列名指定别名, 别名和 java 实体类的属性名一致
```xml
<select id="selectUserById" resultType="User">
  select id , name , pwd as password from user where id = #{id}
</select>
```

#### 方案二：使用结果集映射 ResultMap [推荐]

```xml
<resultMap id="UserMap" type="User">
   <!-- id为主键 -->
   <id column="id" property="id"/>
   <!-- column是数据库表的列名 , property是对应实体类的属性名 -->
   <result column="name" property="name"/>
   <result column="pwd" property="password"/>
</resultMap>

<select id="selectUserById" resultMap="UserMap">
  select id , name , pwd from user where id = #{id}
</select>
```

如果世界总是这么简单就好了。数据库不可能永远是上面的简单样子，可能会存在一对多和多对一的情况，之后我们会使用到一些高级的结果集映射，如 association、collection 等。

## 日志工厂

思考：当测试 SQL 时，要是能在控制台输出 SQL ，是否就能够有更快的排错效率？

使用 Mybatis 基于接口和配置文件执行源代码，无法打印 SQL 语句。因此我们必须选择日志工厂来作为我们开发的工具。

Mybatis内置的日志工厂提供日志功能，具体的日志实现有以下几种工具：

+ SLF4J
+ Apache Commons Logging
+ Log4j 2
+ Log4j
+ JDK logging

具体选择哪个日志实现工具由 MyBatis 的内置日志工厂确定，它会按照优先级顺序查找，如果均未找到，日志功能就会被禁用。

### 标准日志实现

在核心配置文件中指定 MyBatis 应该使用哪个日志记录实现。

```xml
<settings>
    <setting name="logImpl" value="STDOUT_LOGGING"/>
</settings>
```

测试，可以看到控制台有大量的输出！我们可以通过这些日志来判断程序哪里出了 Bug

### Log4j

Log4j 是 Apache 的一个开源项目。我们可以使用它控制日志输送的目的地：控制台，文本，GUI组件等，也可以控制每一条日志的输出格式。

通过定义每一条日志信息的级别，我们能够更加细致地控制日志的生成过程。并且，这些可以通过一个配置文件来灵活地进行配置，而不需要修改应用的代码。

1、导入 log4j 的包

```xml
<dependency>
   <groupId>log4j</groupId>
   <artifactId>log4j</artifactId>
   <version>1.2.17</version>
</dependency>
```

2、配置文件编写

```xml
#将等级为DEBUG的日志信息输出到console和file这两个目的地，console和file的定义在下面的代码
log4j.rootLogger=DEBUG,console,file

#控制台输出的相关设置
log4j.appender.console = org.apache.log4j.ConsoleAppender
log4j.appender.console.Target = System.out
log4j.appender.console.Threshold=DEBUG
log4j.appender.console.layout = org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=[%c]-%m%n

#文件输出的相关设置
log4j.appender.file = org.apache.log4j.RollingFileAppender
log4j.appender.file.File=./log/kuang.log
log4j.appender.file.MaxFileSize=10mb
log4j.appender.file.Threshold=DEBUG
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=[%p][%d{yy-MM-dd}][%c]%m%n

#日志输出级别
log4j.logger.org.mybatis=DEBUG
log4j.logger.java.sql=DEBUG
log4j.logger.java.sql.Statement=DEBUG
log4j.logger.java.sql.ResultSet=DEBUG
log4j.logger.java.sql.PreparedStatement=DEBUG
```

3、setting设置日志实现

```xml
<settings>
   <setting name="logImpl" value="LOG4J"/>
</settings>
```

4、在程序中使用Log4j进行输出！

```java
static Logger logger = Logger.getLogger(MyTest.class);

@Test
public void selectUser() {
   logger.info("info：进入selectUser方法");
   logger.debug("debug：进入selectUser方法");
   logger.error("error: 进入selectUser方法");
   SqlSession session = MybatisUtils.getSession();
   UserMapper mapper = session.getMapper(UserMapper.class);
   List<User> users = mapper.selectUser();
   for (User user: users){
       System.out.println(user);
  }
   session.close();
}
```

5、测试，看控制台输出！

## 分页

思考：为什么需要分页？

查询大量数据的时候，我们往往使用分页进行查询，也就是每次处理小部分数据，这样对数据库压力就在可控范围内。

### Limit 实现分页

首先回顾一下 Limit 语法

```sql
SELECT * FROM table LIMIT stratIndex，pageSize

SELECT * FROM table LIMIT 5,10; // 检索记录行 6-15  

SELECT * FROM table LIMIT 5; //检索前 5 个记录行  
```

1、修改 Mapper 文件
```xml
<select id="selectUser" parameterType="map" resultType="user">
  select * from user limit #{startIndex}, #{pageSize}
</select>
```
2、Mapper 接口，参数为map
```java
// 选择全部用户实现分页
List<User> selectUser(Map<String,Integer> map);
```
3、在测试类中传入参数测试

```java
//分页查询 , 两个参数startIndex , pageSize
@Test
public void testSelectUser() {
   SqlSession session = MybatisUtils.getSession();
   UserMapper mapper = session.getMapper(UserMapper.class);

   int startIndex = 1;  //第几页
   int pageSize = 2;  //每页显示几个
   Map<String,Integer> map = new HashMap<String,Integer>();
   map.put("startIndex",startIndex);
   map.put("pageSize",pageSize);

   List<User> users = mapper.selectUser(map);

   for (User user: users){
       System.out.println(user);
  }

   session.close();
}
```

### RowBounds 分页

除了使用Limit在SQL层面实现分页，也可以使用RowBounds在Java代码层面实现分页，当然此种方式作为了解即可。

1、mapper接口
```java
//选择全部用户RowBounds实现分页
List<User> getUserByRowBounds();
```
2、mapper文件
```xml
<select id="getUserByRowBounds" resultType="user">
select * from user
</select>
```
3、测试类

在这里，我们需要使用RowBounds类
```java
@Test
public void testUserByRowBounds() {
   SqlSession session = MybatisUtils.getSession();

   int currentPage = 2;  //第几页
   int pageSize = 2;  //每页显示几个
   RowBounds rowBounds = new RowBounds((currentPage-1)*pageSize,pageSize);

   //通过session.**方法进行传递rowBounds，[此种方式现在已经不推荐使用了]
   List<User> users = session.selectList("com.kuang.mapper.UserMapper.getUserByRowBounds", null, rowBounds);

   for (User user: users){
       System.out.println(user);
  }
   session.close();
}
```

参考自：
1. [Mybatis Docs](https://mybatis.org/mybatis-3/zh/configuration.html)

2. [狂神说MyBatis03：ResultMap及分页](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247484043&idx=1&sn=94dbbcbca7ae17c50aa66fd78c8ecaa3&scene=19#wechat_redirect)
