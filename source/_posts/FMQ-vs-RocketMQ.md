---
title: FMQ vs RocketMQ
date: 2017-10-18 10:10:52
tags:
- 中间件
- 消息队列
categories: 编程
comments: true
---

*消息队列一直是中间件中核心的一环，承担着应用解耦，削峰填谷等重任。目前市场上已经有很多成熟的系统珠玉在前，比如ActiveMQ，Kafka,RabbitMQ等。本文选取的比较对象为阿里的RocketMQ，旨在全方位对比京东金融使用的FMQ与其差异。*
<!--more-->

# 1 底层架构分析
## RocketMQ
![RocketMQ架构](http://cdn.infoqstatic.com/statics_s2_20171017-0336/resource/news/2017/02/RocketMQ-future-idea/zh/resources/1.png)
RocketMQ架构如图所示，主要包括client(producer，consumer)，NameServer，Broker三部分。非核心组件包括用于过滤消息的FilterServer。  

* client
	* producer  
	消息的生产者
	* consumer   
	消息的消费者
* NameServer  
	提供服务发现的功能，每个NameServer存有路由信息，供客户端获取。同时NameServer每台都拥有全量信息，可以横向扩容。
* broker  
	负责消息的实际存储。 
* filterServer
## FMQ 2.0
![RocketMQ架构](http://ovor60v7j.bkt.clouddn.com/blog/FMQVSRMQFMQ%E6%9E%B6%E6%9E%84.png)
FMQ 2.0的架构如图所示，分为核心组件（必须部署）和可选组件（提供消息队列以外额外功能）。  
### 自研核心组件  
* client
	* producer
	* consumer
* broker
* task
* web
### 第三方核心组件
* mysql
* zookeeper
### 自研可选组件
* alarm
* agent
### 第三方可选组件
* R2M（redis）
* UNC
* hbase

# 2 功能对比
## 2.1 系统特性
### 高吞吐
#### 存储特性
#### 网络特性
#### 自定义协议
### 高可用
#### 自动负载均衡
#### 从消费 
### 高可靠
#### 同步刷盘
#### 同步复制
### 低延迟
#### 长轮询
## 2.2 功能特性
### 消息过滤
### 顺序消息
### 重试
### 事务消息
### 延迟消费
### 消息轨迹
### 消息回溯
* FMQ支持按照offset和timestamp进行回溯
## 2.3 运维特性
### 配置
### 监控
### 归档
### 报警

# 3 性能对比
## 单机性能对比
## 集群性能对比

# 4 友好性对比
## 4.1 文档完备性
## 4.2 快速入门
## 4.3 部署难度
## 4.4 社区活跃度