---
title: FMQ Proxy设计
date: 2017-12-11 14:40:24
tags:
- FMQ
- 消息队列
categories: 编程
---
*FMQ Proxy用来支持多语言接入FMQ，主要功能为在应用层提供HTTP协议到自定义协议的转换*
<!--more-->

## 1 整体架构

## 2 协议实现

Proxy提供HTTP协议到自定义协议的转换。

### 2.1 HTTP协议
通过HTTP支持验证、生产消息、消费消息等方法，URL格式如下：
URL = http://host:port/version/Method

支持method:

* auth
* produce
* consume
* ack
* retry 


## 3 模块分析
### 3.1 验证模块

#### auth命令
* 验证用户名、密码是否合法
* 验证主题、app是否为空
* 生成tokenMD5(prefix + username + password + app + suffix)）
存入redis
* 返回所有proxy的列表

#### put/get/ack/retry命令

* 验证app是否为authid对应的app

### 3.2 ProxyProducerManager模块
* 根据topic,app唯一确定ProducerTrace，独享TransportManager
* ProducerTrace熟悉lastTime记录最后一次生产消息的时间
* 空闲管理，5min没有消息终止客户端

### 3.3 ProxyConsumerManager模块
* 根据topic,app唯一确定ConsumerTrace，独享TransportManager ***存在问题，MessageConsumer本身支持多个topic***
* 其余逻辑类似producer