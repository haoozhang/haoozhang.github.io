---
layout:     post
title:      SSL Certificate Format
subtitle:   
date:       2020-09-15
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - SSL
---

在做Spring Boot + SSL时，了解到有不同格式的SSL证书。根据不同的服务器及版本，我们需要用到不同的证书格式，就市面上主流的服务器来说，大概有以下格式：

+ .DER .CER，二进制格式，只保存证书，不保存私钥。
+ .PEM，一般是文本格式，可保存证书，可保存私钥。
+ .CRT，可以是二进制格式，可以是文本格式，与 .DER 格式相同，不保存私钥。
+ .PFX .P12，二进制格式，同时包含证书和私钥，一般有密码保护。
+ .JKS，二进制格式，同时包含证书和私钥，一般有密码保护。

### DER

该格式是二进制文件内容，Java 和 Windows 服务器偏向于使用这种编码格式。

OpenSSL 查看

```
$ openssl x509 -in certificate.der -inform der -text -noout
```

DER 转换为 PEM 格式

```
$ openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
```

### PEM
Privacy Enhanced Mail，一般为文本格式，以 **-----BEGIN...** 开头，以 **-----END...** 结尾。中间的内容是 BASE64 编码。这种格式可以保存证书和私钥，有时我们也把PEM 格式的私钥的后缀改为 .key 以区别证书与私钥。具体可以看文件的内容。

OpenSSL 查看

```
$ openssl x509 -in certificate.pem -text -noout
```

PEM 转换为 DER 格式

```
$ openssl x509 -in cert.crt -outform der -out cert.der
```

### CRT

Certificate 的简称，有可能是 PEM 编码格式，也有可能是 DER 编码格式。

### PFX / P12

Predecessor of PKCS#12，这种格式是二进制格式，且证书和私钥存在一个文件中。一般用于 Windows 上的 IIS 服务器。文件一般会有一个密码用于保证私钥的安全。PFX格式和P12格式的文件可以通过直接该文件后缀来转换。

OpenSSL 查看

```
$ openssl pkcs12 -in for-iis.pfx
```

转换为 PEM

```
$ openssl pkcs12 -in for-iis.pfx -out for-iis.pem -nodes
```

pfx导出crt和key

```
$ openssl pkcs12 -in example.cn.ssl.pfx -nocerts -nodes -out example.key
$ openssl pkcs12 -in example.cn.ssl.pfx -clcerts -nokeys -out example.crt
```

crt和key合并为pfx

```
$ openssl pkcs12 -export -in certificate.crt -inkey privateKey.key -out certificate.pfx
```

### JKS

Java Key Storage，很容易知道这是 JAVA 的专属格式，利用 JAVA 的一个叫 keytool 的工具可以进行格式转换。一般用于 Tomcat 服务器。

P12/PFX 转换为 JKS

```
$ keytool -importkeystore -srckeystore mypfxfile.pfx -srcstoretype pkcs12 -destkeystore newkeystore.jks -deststoretype JKS
```

### 创建证书

创建证书的工具主要有openssl、keypool、mkcert等。我主要用的是mkcert工具，它可以创建自签名的证书。mkcert的具体安装使用可看之前[这篇文章](https://newbiecoder-hao.github.io/2020/09/09/Enable-HTTPS-in-Spring-Boot-Application/)的 **创建自签名证书** 章节。

参考自：
1. [SSL 证书格式普及](https://blog.freessl.cn/ssl-cert-format-introduce/)
2. [convert pfx format to p12](https://stackoverflow.com/questions/6819079/convert-pfx-format-to-p12)


