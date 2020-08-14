---
layout:     post
title:      Spring Cloud Gateway
subtitle:   
date:       2020-08-14
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - Spring Cloud
---

#### A Kubernetes Story (基本概念)

![img](/img/post/post_gateway.png)


1. 如上图所示，客户端向Spring Cloud Gateway发出请求。  
2. 如果Gateway Handler Mapping确定请求与路由匹配（这时候就用到predicate）, 则将其发送到Gateway web handler处理。Predict作为断言，它决定了请求会被路由到哪个router 
3. Gateway web handler处理请求时会经过一系列的过滤器链。 过滤器链被虚线划分的原因是过滤器链可以在发送代理请求之前或之后执行过滤逻辑。 先执行所有“pre”过滤器逻辑，然后进行代理请求。在收到代理服务的响应之后执行“post”过滤器逻辑。这跟zuul的处理过程很类似。在执行所有“pre”过滤器逻辑时，往往进行了鉴权、限流、日志输出等功能，以及请求头的更改、协议的转换；转发之后收到响应之后，会执行所有“post”过滤器的逻辑，在这里可以响应数据进行了修改，比如响应头、协议的转换等。

参考自：
1. [递归公式的理解](https://leetcode-cn.com/problems/yuan-quan-zhong-zui-hou-sheng-xia-de-shu-zi-lcof/solution/nan-dian-shi-di-gui-gong-shi-de-li-jie-by-piao-yi-/)


