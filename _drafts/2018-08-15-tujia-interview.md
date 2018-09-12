---
layout: post
title:  "记一次途家面试经历"
date:   2018-08-15 23:06:05
categories: 面试
tags: 面试
author: sqp
---

* content
{:toc}

# 记一次途家面试经历

前几天刚好有个途家的内推机会，正好现在待的公司情况不太好，去试一下好了。  
废话不多说。  

### 一面
一面比较简单，比较常规的问题
+ 限流的实现   
两个思路，单机和分布式，单机信号量控制，分布式用令牌桶和漏桶算法。
+ Dubbo  
主要考察dubbo中filter和重试机制，默认重试2次
+ spring  
Aop IOC DI 机制和实现原理
+ java  
Haspmap的实现原理
+ 项目  
项目介绍。负责的模块

### 二面
二面比一面有难度而且角度相对刁钻
+ java中String的实现原理，Char的默认编码是什么?  
String是由char数组实现的，char的编码是16位Unicode。





