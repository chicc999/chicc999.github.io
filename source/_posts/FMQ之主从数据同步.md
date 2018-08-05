---
title: FMQ之主从数据同步
date: 2017-08-21 16:45:00
tags:
- java
- 消息队列
categories: 编程
comments: true
---
*FMQ每组broker分为主从两个角色，支持一主多从。目前版本主从角色固定，本文介绍其主从同步数据的逻辑。*
<!--more-->

# 1. 启动

# 2. 数据复制

## 2.1 日志复制

* slave和master成功建立连接。
* slave向master发送getOffset请求（请求中包含本地日志文件的最大偏移量）。
* master根据slave发送过来的最大偏移量，返回一个合适的日志复制起始位置。master回复response包含起始复制offset和日志最大偏移量journalMaxOffset。

## 2.1 索引日志重建

主从同步时只会对日志文件和消费者索引文件同步，而不会对topic的队列索引文件同步。队列索引文件可以通过日志文件进行重建，所以在数据复制过程中，还伴随索引重建的过程。


