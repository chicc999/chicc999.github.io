---
title: FMQ Proxy概要设计
date: 2017-12-11 14:40:24
tags:
- FMQ
- 消息队列
categories: 编程
---
*FMQ Proxy用来支持多语言接入FMQ，主要功能为在应用层提供HTTP协议到自定义协议的转换*
<!--more-->

# 1 需求背景

* FMQ目前采用自定义协议，如果要对多语言进行支持，序列化相对复杂。因此需要一个网关来提供Http协议到自定义协议的转换，以方便用户使用多种语言通过Http的方式接入系统。
* 同时为了简化客户端编程，需要一个代理来隐藏客户端的路由、重试等操作。
* 

# 2 需求分析


## 2.1 详细需求

### 2.1.1 验证功能

* 提供用户合法性验证
* 缓存认证结果，避免重复校验
* 缓存需要有时效性

### 2.1.2 路由

* 返回结果为用户的生产消费提供第一层路由（即用户端到代理层），以达到代理层可以水平扩展的目的（不更改域名解析的情况下）

### 2.1.3 生产

* 提供消息生产的功能
* 提供默认策略，优先向本机房broker生产

### 2.1.4 消费

* 提供消费功能
* 消费失败进入重试队列
* 消费成功进行ack

### 2.1.5 监控

* proxy日志级别修改。
* proxy业务级别监控，如流量、生产消费条数等。
* proxy机器级别监控，如进程状态、内存使用、网络IO等。

### 2.1.6 报警

* 对于proxy异常状况进行报警

### 2.1.7 安全

* Https支持


# 3 系统总体设计

# 4 系统功能设计

## 4.1 协议设计

### 4.1.1 URL

通过HTTP支持验证、生产消息、消费消息等方法，URL格式如下：

URL = http://[host]:[port]/[version]/[Method]

version部分为协议版本号，目前发布的为1.0，支持method:

* auth
* produce
* consume
* ack
* retry 

### 4.1.2 Request Header

所有方法均走post请求，header部分值如下：

| key | 描述|举例|
|:----------:|:-------------:|:------|
| User-Agent |客户端说明，FMQ-{客户端语言}|FMQ-PHP|
|Accept|指定客户端能够接受的内容类型，目前只支持返回text/plain|text/plain|
|Host|固定值|请求域名|
|Timestamp|发送时间|取发送请求当前时间|
|token|验证的令牌,除了auth方法，其余方法需要带此值||


### 4.1.3 Request Body


content部分为所需信息的json字符串。

| method | 描述      | content字段|response字段|
|:----------:|:-------------:|:------|:-----:|:-----:|
| auth |   用来进行服务认证  |  user</br>password</br>topic</br>app| status {code}</br>result {authid,servers}|
| produce |   生产消息    |  topic</br>app</br>messages|status {code}|
| consume| 获取消息|   topic</br>app | status {code}|
| ack |    消费成功确认   |  |
| retry  | 消费失败确认   |  |

### 4.1.4 response


## 4.2 验证设计

## 4.3 路由设计

路由设计主要考虑动态分配proxy的流量，一方面要让属于某个主题的流量尽可能集中在某些proxy上，另一方面兼顾到主题流量不均衡，允许特殊规则支持大流量主题。

### 4.3.1 普通路由分配规则

* 每个proxy需要绑定若干个业务分组，并可以提供此业务分组所有topic的代理服务
* 一个proxy可以代理多个业务分组，一个业务分组也可以被多个proxy代理
* 若主题对应业务分组未分配给此proxy代理，则添加按钮控制是否允许提供代理服务

### 4.3.2 特殊路由分配规则

* 指定某个主题、应用或者主题-应用绑定特定proxy,则只返回此proxy

### 4.3.3 



# 5.测试

## 5.1.1 验证部分

| 测试点 | 预期结果     |
|:----------|:-------------|
|域名正确，方法不存在|返回异常|



## 1 整体架构

## 2 协议实现


RISK_SMS_SEND_TASK_POC	postLoanCloud
### 2.1 



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
* 忽略超级用户user和app的匹配

### 3.2 ProxyProducerManager模块
* 根据topic,app唯一确定ProducerTrace，独享TransportManager
* ProducerTrace熟悉lastTime记录最后一次生产消息的时间
* 空闲管理，5min没有消息终止客户端

### 3.3 ProxyConsumerManager模块
* 根据topic,app唯一确定ConsumerTrace，独享TransportManager ***存在问题，MessageConsumer本身支持多个topic***
* 其余逻辑类似producer

### GroupClient

## 问题

1. 返回servers靠配置，如何进行负载均衡
	* auth用域名访问vip，后端挂所有proxy，验证完毕缓存共享auth信息
	* 返回所有proxy,固定访问某台/访问域名/自定义策略由用户自己决定
	
2. 机器资源不足
	* 虚拟机
	* 储备机器
	
	
### transport 层

关联channel，包括重连，断开等事件的处理(建立物理连接)

### brokerTransport 层

* addConnection（建立broker层面的逻辑连接），用户认证

### groupTransport 层

* socket存在读写2个通道
* 权限为整个group的权限

### MessageProducer/Consumer

* 缓存是否对某通道进行过add producer/consumer 操作
* 关闭时remove所有producer/consumer
* 消息出入口


### cluster（group-topic）层

* 存在权重
* 选举group

### broker_zip 变更

* 添加新broker
	* 判断是否在zk存活列表，在的话新增连接
	* 判断是否存在此group，不存在则新增
* 更改原有broker
	
	* 通知group变更权限
* 删除broker
	* 通知group变更权限
	* 移除原有broker
	
	
## 问题

* 没有集群信息时，返回code不对
* getMessage messages 为null
* channel中附加信息会不会无限膨胀
* 页面查看某个token对应的app

## 讨论

* 验证信息中是否带入topic
* 是否返回明确的topic-proxy映射关系，还是合起来一个list
* http业务发生错误（如参数校验失败）响应码是否200