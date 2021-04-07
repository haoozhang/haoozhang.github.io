---
layout:     post
title:      2. Mybatis CRUD 操作及配置解析
date:       2021-04-07
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Mybatis
---

## CRUD

### select

需求：根据id查询用户

1、在UserMapper中添加对应方法

```java
public interface UserMapper {
   //查询全部用户
   List<User> selectUser();
   //根据id查询用户
   User selectUserById(int id);
}
```

2、在UserMapper.xml中添加Select语句

```xml
<select id="selectUserById" resultType="com.kuang.pojo.User">
select * from user where id = #{id}
</select>
```

3、测试

```java
@Test
public void tsetSelectUserById() {
   SqlSession session = MybatisUtils.getSession();  //获取SqlSession连接
   UserMapper mapper = session.getMapper(UserMapper.class);
   User user = mapper.selectUserById(1);
   System.out.println(user);
   session.close();
}
```

需求扩展：根据 密码 和 名字 查询用户

思路一：直接在方法中传递参数

1、在接口方法的参数前加 @Param属性

2、Sql语句编写的时候，直接取@Param中设置的值即可，不需要单独设置参数类型

```java
//通过密码和名字查询用户
User selectUserByNP(@Param("username") String username,@Param("pwd") String pwd);

/*
   <select id="selectUserByNP" resultType="com.kuang.pojo.User">
     select * from user where name = #{username} and pwd = #{pwd}
   </select>
*/
```

思路二：使用万能的Map

1、在接口方法中，参数直接传递Map；

```java
User selectUserByNP2(Map<String,Object> map);
```

2、编写sql语句的时候，需要传递参数类型，参数类型为map

```xml
<select id="selectUserByNP2" parameterType="map" resultType="com.kuang.pojo.User">
select * from user where name = #{username} and pwd = #{pwd}
</select>
```

3、在使用方法的时候，Map的 key 为 sql中取的值即可，没有顺序要求！

```java
Map<String, Object> map = new HashMap<String, Object>();
map.put("username","小明");
map.put("pwd","123456");
User user = mapper.selectUserByNP2(map);
```

如果参数过多，我们可以考虑直接使用Map实现，如果参数比较少，直接传递参数即可

### insert

需求：给数据库增加一个用户

1、在UserMapper接口中添加对应的方法

```java
//添加一个用户
int addUser(User user);
```

2、在UserMapper.xml中添加insert语句

```xml
<insert id="addUser" parameterType="com.kuang.pojo.User">
    insert into user (id,name,pwd) values (#{id},#{name},#{pwd})
</insert>
```

3、测试

```java
@Test
public void testAddUser() {
   SqlSession session = MybatisUtils.getSession();
   UserMapper mapper = session.getMapper(UserMapper.class);
   User user = new User(5,"王五","zxcvbn");
   int i = mapper.addUser(user);
   System.out.println(i);
   session.commit(); //提交事务,重点!不写的话不会提交到数据库
   session.close();
}
```

**注意：增、删、改操作需要提交事务！**

### update

需求：修改用户的信息

1、同理，编写接口方法
```java
//修改一个用户
int updateUser(User user);
```
2、编写对应的配置文件SQL
```xml
<update id="updateUser" parameterType="com.kuang.pojo.User">
  update user set name=#{name},pwd=#{pwd} where id = #{id}
</update>
```
3、测试
```java
@Test
public void testUpdateUser() {
   SqlSession session = MybatisUtils.getSession();
   UserMapper mapper = session.getMapper(UserMapper.class);
   User user = mapper.selectUserById(1);
   user.setPwd("asdfgh");
   int i = mapper.updateUser(user);
   System.out.println(i);
   session.commit(); //提交事务,重点!不写的话不会提交到数据库
   session.close();
}
```

### delete

需求：根据id删除一个用户

1、同理，编写接口方法
```java
//根据id删除用户
int deleteUser(int id);
```
2、编写对应的配置文件SQL
```xml
<delete id="deleteUser" parameterType="int">
  delete from user where id = #{id}
</delete>
```
3、测试
```java
@Test
public void testDeleteUser() {
   SqlSession session = MybatisUtils.getSession();
   UserMapper mapper = session.getMapper(UserMapper.class);
   int i = mapper.deleteUser(5);
   System.out.println(i);
   session.commit(); //提交事务,重点!不写的话不会提交到数据库
   session.close();
}
```

小结：
+ 所有的增删改操作都需要提交事务！
+ 接口所有的普通参数，尽量都写上@Param参数，尤其是多个参数时，必须写上！
+ 有时候根据业务的需求，可以考虑使用map传递参数！
+ 为规范操作，在SQL的配置文件中，我们尽量将Parameter参数和resultType都写上！

### 思考：模糊查询like

第1种：在Java代码中添加sql通配符。

```xml
string wildcardname = “%smi%”;
list<name> names = mapper.selectlike(wildcardname);

<select id=”selectlike”>
select * from foo where bar like #{value}
</select>
```

第2种：在sql语句中拼接通配符，会引起sql注入

```xml
string wildcardname = “smi”;
list<name> names = mapper.selectlike(wildcardname);

<select id=”selectlike”>
    select * from foo where bar like "%"#{value}"%"
</select>
```

## 配置解析

上面我们已经基本学习了 Mybatis 的 CURD 操作，接下来我们深入到配置文件，看看 Mybatis 的配置文件都包括哪些内容。

### 核心配置文件

MyBatis 的配置文件包含了会深深影响 MyBatis 行为的设置和属性信息。配置文档的顶层结构如下：

configuration（配置）
+ properties（属性）
+ settings（设置）
+ typeAliases（类型别名）
+ typeHandlers（类型处理器）
+ objectFactory（对象工厂）
+ plugins（插件）
+ environments（环境配置）
+ environment（环境变量）
+ transactionManager（事务管理器）
+ dataSource（数据源）
+ databaseIdProvider（数据库厂商标识）
+ mappers（映射器）

**注意元素节点的顺序！顺序不对会报错**

### 环境配置 environments

MyBatis 可以配置适应多种环境，例如，开发、测试和生产环境需要有不同的配置。尽管如此，每个 SqlSessionFactory 实例只能选择一种环境。

MyBatis 有两种类型的事务管理器（也就是 type="[JDBC|MANAGED]"）

MyBatis 有三种内建的数据源类型（也就是 type="[UNPOOLED|POOLED|JNDI]"）
+ UNPOOLED – 这个数据源的实现会每次请求时打开和关闭连接
+ POOLED – 利用池化的概念将 JDBC 连接对象组织起来，避免了每次创建新连接的开销
+ jndi - 这个数据源的实现是为了能在如 Spring 或应用服务器这类容器中使用，容器可以集中或在外部配置数据源，然后放置一个 JNDI 上下文的引用

### 属性 properties

配置文件中的数据库属性都是可外部配置且可动态替换的，可以在典型的 Java 属性文件中配置，亦可通过 properties 元素的子元素来传递。

我们来优化我们的配置文件

第一步：在资源目录下新建一个db.properties
```xml
driver=com.mysql.jdbc.Driver
url=jdbc:mysql://localhost:3306/mybatis?useSSL=true&useUnicode=true&characterEncoding=utf8
username=root
password=123456
```
第二步：将文件导入properties 配置文件
```xml
<configuration>
   <!--导入properties文件-->
   <properties resource="db.properties"/>

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
       <mapper resource="mapper/UserMapper.xml"/>
   </mappers>
</configuration>
```

### 类型别名 typeAliases

类型别名是为 Java 类型设置一个短的名字。它只和 XML 配置有关，存在的意义仅在于用来减少类完全限定名的冗余。

```xml
<typeAliases>
   <typeAlias type="com.kuang.pojo.User" alias="User"/>
</typeAliases>
```

这样，User 可以用在任何使用 com.kuang.pojo.User 的地方。

也可以指定一个包名，MyBatis 会在包名下面搜索需要的 Java Bean，比如:
```xml
<typeAliases>
   <package name="com.kuang.pojo"/>
</typeAliases>
```
每一个在包 com.kuang.pojo 中的 Java Bean，在没有注解的情况下，会使用 Bean 的首字母小写的非限定类名来作为它的别名。

若有注解，则别名为其注解值。见下面的例子：

```java
@Alias("user")
public class User {
  ...
}
```

### 设置 settings

阅读[官方文档](https://mybatis.org/mybatis-3/zh/configuration.html#settings)

### 映射器 mappers

```xml
<mappers>
    <!-- 方式一: 引用资源 【推荐使用】-->
    <mapper resource="org/mybatis/builder/UserMapper.xml"/>
   
    <!-- 方式二: class; 需要配置文件名称和接口名相同, 并位于同一目录下 -->
    <mapper class="org.mybatis.builder.UserMapper"/>
   
    <!-- 方式三: 扫描包; 需要配置文件名称和接口名相同, 并位于同一目录下 -->
    <mapper class="org.mybatis.builder.UserMapper"/>
</mappers>
```

## 生命周期和作用域

### SqlSessionFactoryBuilder

一旦创建了 SqlSessionFactory，就不再需要它了。 因此该实例的最佳作用域是局部变量。 

### SqlSessionFactory

SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例。使用 SqlSessionFactory 的最佳实践是在应用运行期间不要重复创建多次，最简单的就是使用单例模式或者静态单例模式。

### SqlSession

相当于访问连接池的一个请求。SqlSession 的实例不是线程安全的，因此不能被共享，它的最佳的作用域是方法作用域。绝对不能将 SqlSession 实例的引用放在一个类的静态域。

如果你现在正在使用一种 Web 框架，考虑将 SqlSession 放在一个和 HTTP 请求相似的作用域中。 换句话说，每次收到 HTTP 请求，就可以打开一个 SqlSession，返回一个响应后，就关闭它。 这个关闭操作很重要，为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 finally 块中。 下面的示例就是一个确保 SqlSession 关闭的标准模式：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  // 你的应用逻辑代码
}
```

在所有代码中都遵循这种使用模式，可以保证所有数据库资源都能被正确地关闭。

### 映射器实例

映射器是一些绑定映射语句的接口。映射器接口的实例是从 SqlSession 中获得的。

映射器实例应该在调用它们的方法中被获取，使用完毕之后即可丢弃。就像下面的例子一样：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  // 你的应用逻辑代码
}
```

## 总结

本篇我们先学习了 Mybatis 基本的 CURD 操作，然后深入到配置文件中了解了各个配置选项。最后，我们介绍了各个实例的作用域。

参考自：
1. [Mybatis Docs](https://mybatis.org/mybatis-3/zh/configuration.html)

2. [狂神说MyBatis02：CRUD操作及配置解析](https://mp.weixin.qq.com/s?__biz=Mzg2NTAzMTExNg==&mid=2247484037&idx=1&sn=cc4bda4e560702e8f7b3e74d3838929f&scene=19#wechat_redirect)