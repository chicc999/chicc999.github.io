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

version部分目前为1.0版本，所有方法均走post请求，content部分为所需信息的json字符串。

| method | 描述       | header字段 |content字段|response字段
|:----------:|:-------------:|:------|:-----:|:-----:|
| auth |   用来进行服务认证  | | user</br>password</br>topic</br>app| status {code}</br>result {authid,servers}|
| produce |   生产消息    |    authid |topic</br>app</br>messages|status {code}|
| consume| 获取消息|    authid | topic</br>app | status {code}|
| ack |    消费成功确认   |   authid | |
| retry  | 消费失败确认      | authid | |


### 2.1.1 auth

auth方法主要用来进行服务认证。





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


## 问题

1. 返回servers靠配置，如何进行负载均衡
	* auth用域名访问vip，后端挂所有proxy，验证完毕缓存共享auth信息
	* 返回所有proxy,固定访问某台/访问域名/自定义策略由用户自己决定
	
2. 机器资源不足
	* 虚拟机
	* 储备机器