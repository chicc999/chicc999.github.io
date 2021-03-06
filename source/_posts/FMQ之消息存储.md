---
title: FMQ之消息存储
date: 2017-04-21 14:41:28
tags:
- 中间件
- 消息队列
categories: 编程
comments: true
---

消息队列如果要做到高可靠且有一定积压能力，则必须要进行持久化。持久化对性能的影响很大，于是如何设计一个可靠而高效的存储结构就成为了关键。本文主要介绍FMQ消息存储模块设计时考虑的问题，以及解决的思路。
<!--more-->

# 1 介绍

## 1.1 消息队列与持久化

## 1.2 存储模型浅析

# 2 FMQ存储逻辑模型

# 3 FMQ存储结构设计

## 3.1 文件结构设计

### 3.1.1 日志文件设计

日志文件分为header和body两部分，固定总大小为128M。其中头部组成如下图。

![日志文件结构](http://cyblog.oss-cn-hangzhou.aliyuncs.com/FMQ%E4%B9%8B%E6%B6%88%E6%81%AF%E5%AD%98%E5%82%A8/%E6%97%A5%E5%BF%97%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84.png)

# 4 其他工程优化