---
layout:     post
title:      Install OpenCV for Java on Mac
subtitle:   
date:       2020-12-5
author:     Hao
header-img: img/post/post_bg_coffee.jpg
catalog: true
mathjax: true
tags:
    - OpenCV
    - Mac
---


![img](/img/post/early_exit.png)

在算法执行过程中，每个点需要记录到达该点的前一个点位置 (父节点)。一旦到达终点，便可以从终点开始，反过来顺着父节点的顺序找到起点，由此就构成了一条路径。

### Dijkstra 算法


参考自：
1. [路径规划之 A* 算法](https://paul.pub/a-star-algorithm/)
