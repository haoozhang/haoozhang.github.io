---
layout:     post
title:      Enable HTTPS in Spring Boot Application
subtitle:   
date:       2020-09-09
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Boot
---

默认情况下Spring Boot应用提供基于HTTP的访问方式，例如最常见的http://127.0.0.1:8080。本文中我们实现用HTTPS访问Spring Boot应用的接口，配置方式分为在application.yaml中配置和在code中配置两种，具体如下。

### 编写示例程序

首先，我们编写一个简单的Spring Boot示例应用，如果不想手动写，代码可以从[这里]()下载。我们写一个health check接口用于验证。

```java
@RestController
public class HealthController {

    @RequestMapping("/health")
    public String health(){
        return "hello, I'm healthy! " + 
                new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date());
    }
}
```

### 创建自签名证书

实现HTTPS访问需要SSL证书，使用JDK自带的keytool工具生成的证书由于未通过第三方认证，通过本地项目浏览器访问时会出现警告。为了调试方便，可以使用mkcert工具生成本地安全的SSL证书。

如果想了解mkcert项目和官方文档请点击[这里](https://github.com/FiloSottile/mkcert)，我是直接在[release](https://github.com/FiloSottile/mkcert/releases)下载可执行文件mkcert-v1.4.1-windows-amd64.exe，然后命令行中运行这个可执行文件即可。

先使用mkcert安装本次CA

```
$ mkcert-v1.4.1-windows-amd64 -install
```

使用mkcert生成本机识别的证书，添加 **-pkcs12** 参数生成 **.p12** 文件\
默认配置 **alias=1，password=changeit**

```
$ mkcert-v1.4.1-windows-amd64 -p12-file self-signed-cert.p12 -pkcs12 127.0.0.1 localhost ::1
```

可以通过keypool工具修改配置信息

```
$ keytool -changealias -alias 1 -destalias tomcat -keystore self-signed-cert.p12
Enter keystore password: (type your password here)
$ keytool -storepasswd -new password -keystore self-signed-cert.p12
Enter keystore password: (type your password here)
```

通过keytool将p12文件转换为JKS文件

```
$ keytool -importkeystore 
        \ -srckeystore self-signed-cert.p12 -srcstoretype pkcs12 -srcalias 1 
        \ -destkeystore self-signed-cert.jks -deststoretype jks -deststorepass changeit -destalias 1
```

### 配置应用属性

将生成的证书复制到工程的resource目录下，只需要p12文件或JKS文件的一种就行，这里两种文件配置都展示。按照不同的证书格式在application.yaml中配置如下。

```yaml
server:
  ssl:
    key-alias: 1
    key-password: changeit
    key-store-password: changeit
    # for JKS format
    #key-store: classpath:self-signed-cert.jks
    #key-store-type: JKS
    # for PKCS12 format
    key-store: classpath:self-signed-cert.p12
    key-store-type: PKCS12
```

### 在浏览器中导入证书

在浏览器中HTTPS访问之前，还需要导入之前生成的证书。以Chrome为例，打开 **设置-隐私设置和安全-安全-管理证书**，可以看到新弹出的窗口中有不同角色证书，点击到 **受信任的根证书颁发机构**，点击 **导入**，选择刚才生成的证书（也即resource目录下的p12文件，这里只能导入p12格式的文件）确定导入，可以看到 **导入成功**的提示框。

![img](/img/post/post_import_cert.png)

### 测试HTTPS访问

运行Spring Boot应用，在应用启动过程中可以看到如下信息输出，证明HTTPS初始化成功。

```
: Tomcat initialized with port(s): 8443 (https)
```

打开浏览器访问health cheak接口 **https://localhost/\<port>/health**，如果没有在系统配置文件中额外设置监听端口，那么端口号默认为8080。我这里设置为监听8443。可以看到类似如下信息输出，并且url前面没有not secure的警告，证明HTTPS配置成功。

```
hello, I'm healthy! 2020-09-09 16:41:06
```

### 在代码中配置

除了在application.yaml文件中配置，我们有时遇到在代码中动态配置的需求。这时我们可以在代码中添加相关的属性配置，来代替在application.yaml文件中的属性配置。先删除在application.yaml文件中的配置，也即上面 **配置应用属性** 小节中的配置。然后我们在工程的main函数中添加如下配置。

```java
public static void main(String[] args) {
	//SpringApplication.run(SpringBootSslApplication.class, args);
	
	// set server.ssl properties program
	Properties properties = new Properties();
	properties.put("server.ssl.key-alias", "1");
	properties.put("server.ssl.key-password", "changeit");
	properties.put("server.ssl.key-store", "classpath:self-signed-cert.p12");
	properties.put("server.ssl.key-store-password", "changeit");
	properties.put("server.ssl.key-store-type", "PKCS12");
	new SpringApplicationBuilder(SpringBootSslApplication.class).properties(properties).run(args);
	
}
```

上述代码在应用初始化时添加.p12证书的属性配置，等同于在配置文件中配置的效果，且不会覆盖在配置文件中配置的属性，你可以运行测试一下。

### 重定向HTTP到HTTPS

为了把流向HTTP的流量全部重定向到HTTPS，我们可以设置两个端口8080和8443分别监听HTTP和HTTPS，具体设置如下。

先在application.yaml文件中指定HTTPS监听的端口

```yaml
server:
  port: 8443
```

然后在main函数文件中注入以下bean实例

```java
@Bean
public ServletWebServerFactory servletContainer() {
	TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {
		@Override
		protected void postProcessContext(Context context) {
			SecurityConstraint securityConstraint = new SecurityConstraint();
			securityConstraint.setUserConstraint("CONFIDENTIAL");
			SecurityCollection collection = new SecurityCollection();
			collection.addPattern("/*");
			securityConstraint.addCollection(collection);
			context.addConstraint(securityConstraint);
		}
	};
	tomcat.addAdditionalTomcatConnectors(redirectConnector());
	return tomcat;
}
private Connector redirectConnector() {
	Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
	connector.setScheme("http");
	connector.setPort(8080);
	connector.setSecure(false);
	connector.setRedirectPort(8443);
	return connector;
}
```

### 小结

本文我们通过application.yaml和代码动态配置实现Spring Boot应用的HTTPS访问，并且将HTTP的流量全部重定向到HTTPS。

参考自：
1. [Spring Boot SSL Https Examples](https://mkyong.com/spring-boot/spring-boot-ssl-https-examples/)
2. [使用mkcert生成本地安全的SSL证书](https://www.jianshu.com/p/5064fef8c577)

