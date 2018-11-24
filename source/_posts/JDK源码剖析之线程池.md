---
title: JDK源码剖析之线程池
date: 2017-01-17 19:54:42
tags:
- java
- 线程
categories: 编程
comments: false
---
通过对JDK源码的分析，学习线程池的实现。
<!--more-->

# 1 体系结构

JDK的线程池将线程的调度流程与被调度对象分开，对于调度流程的相关的接口与类，设计如下图所示：

<img src=http://cyblog.oss-cn-hangzhou.aliyuncs.com/threadpool/ThreadPool.png width="50%" height="50%"/>


## 1.1 接口

Executor和ExecutorService接口定义了执行器最基础的方法，方法列表如下图。

<img src=http://cyblog.oss-cn-hangzhou.aliyuncs.com/threadpool/interface.png width="50%" height="50%"/>


# 2 基本参数