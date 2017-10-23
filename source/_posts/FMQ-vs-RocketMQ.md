---
title: FMQ vs RocketMQ
date: 2017-10-18 10:10:52
tags:
- 中间件
- 消息队列
categories: 编程
comments: true
---

*消息队列一直是中间件中核心的一环，承担着应用解耦，削峰填谷等重任。目前市场上已经有很多成熟的系统珠玉在前，比如ActiveMQ，Kafka,RabbitMQ等。本文选取的比较对象为阿里的RocketMQ，旨在全方位对比京东金融使用的FMQ与其差异。以下分析基于Apache RocketMQ 4.2.0，FMQ 2.1.5。*
<!--more-->

# 1 底层架构分析
## RocketMQ
![RocketMQ架构](http://cdn.infoqstatic.com/statics_s2_20171017-0336/resource/news/2017/02/RocketMQ-future-idea/zh/resources/1.png)
RocketMQ架构如图所示，主要包括client(producer，consumer)，NameServer，Broker三部分。非核心组件包括用于过滤消息的FilterServer。  

* client
	* producer  
	消息的生产者,从NameServer获取路由信息，并进行负载均衡发送至broker。
	* consumer   
	消息的消费者，从NameServer获取路由信息，从broker消费消息。支持PUSH和PULL两种消费模式，支持集群消费和广播消息。
* NameServer  
	提供服务发现的功能，每个NameServer存有路由信息，供客户端获取。同时NameServer每台都拥有全量信息，可以横向扩容。
* broker  
	负责消息的实际存储和分发。 
* filterServer

每个broker启动时，将自己的信息以及元数据注册到NameServer，元信息包括broker所拥有的topic信息。client启动的时候，向NameServer查阅所需的topic信息，NameServer返回此topic对应的broker信息（broker 名称到地址的映射），client直接和broker通信。

## FMQ
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
### 2.1.1 高吞吐
#### 存储特性
#### 网络特性
#### 传输协议
| Tables        | RocketMQ           | FMQ  |
| ------------- |:-------------:| -----:|
| 传输层协议      | TCP | TCP |
| 应用层协议      | 自定义协议      |   自定义协议 |
| 额外协议支持 | jms(社区简单支持)      |    无 |

### 2.1.2 高可用
#### 自动负载均衡
#### 从消费 
### 2.1.3 高可靠
#### 同步刷盘
#### 同步复制
### 2.1.4 低延迟
#### 长轮询
## 2.2 功能特性
### 2.2.1 消息过滤

#### RocketMQ
通过FilterServer进行过滤，将过滤规则由客户端传至FilterServer，本质上是用cpu资源换网络带宽。（代码待看）


#### FMQ

### 2.2.2 顺序消息
无论RocketMQ还是FMQ，默认发送都是无序的，如图所示。
![RocketMQ发送](https://pic1.zhimg.com/50/v2-612e7eaad6734189fdb6c927a9a1d520_hd.png)
消息以轮询的方式发向各个队列。但是在每个队列中，不考虑重试的情况下，是可以保证FIFO的。那如何保证消息有序呢？

#### RocketMQ

RocketMQ有序队列的示例代码如下，用户可以自定义一个队列选择器。客户端获取了所有的队列数量，并将id按照队列数量取模，这样在队列数量不变的情况下，id相同的消息一定会被投递到同一个队列。由于单个队列是FIFO的，就能保证有序。

```
SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                        Integer id = (Integer) arg;
                        int index = id % mqs.size();
                        return mqs.get(index);
                    }
                }, orderId);

```
如何保证顺序消费？消费者的监听器实现MessageListenerOrderly接口。

#### FMQ

### 2.2.3 重试
#### RocketMQ
* 生产端重试  
	生产出现异常后会一直重试直到超时或者达到配置的最大重试次数。
* 消费端重试  
	* 服务端如果未收到客户端消费失败的ack，会一直进行重发。
	* 客户端消费失败，消息被发往重试队列（%RETRY%ConsumerGroup），按consumerGroup组织。
	* 重试一定次数仍然失败，消息被发往私信队列（%DLQ%ConsumerGroup）
	
#### FMQ

### 2.2.4 事务消息
事务消息是指本地操作与发送消息两个操作，在操作成功的同时要保证消息一定发送成功，操作失败的情况下消息一定不能发送。采用两阶段提交的方式流程如下图：
![2PC](http://ovor60v7j.bkt.clouddn.com/blog/FMQVSRMQ/2pc.png)

1. 客户端先发送消息，此时消息在broker中处于prepared状态;
2. 服务端收到消息，存入内存中（或磁盘的prepared队列），回复ack；
3. 客户端接收到ack消息以后，执行本地操作；如果操作成功，发送commit歇息，如果操作失败，发送rollback消息；
4. 服务端处理对应的commit or rollback消息，回复客户端ack。

以上是一个标准的两阶段提交过程。
#### RocketMQ
![事务消息](http://ovor60v7j.bkt.clouddn.com/blog/FMQVSRMQ/%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF.png)
RocketMQ事务消息过程如图所示。

1. 客户端先发送消息，此时消息在broker中处于prepared状态;
2. 服务端收到消息，持久化到磁盘，此时消息处于prepared状态，回复ack；
3. 客户端接收到ack消息以后，执行本地操作
	* 在数据库中添加一个字段表明本地操作是否成功；
	* 将本地操作与更新数据库做成一个事务，如果本地操作成功，则状态一定被更新
4. 如果操作成功，发送commit歇息，如果操作失败，发送rollback消息；
5. 服务端处理对应的commit or rollback消息，回复客户端ack。
6. 服务端如果发现有超期的消息处于prepared阶段，则启动回查机制，查询本地状态表，根据状态表的状态来觉得commit还是rollback。


#### FMQ

### 2.2.5 延迟消费
#### RocketMQ
RocketMQ的延迟消费需要在broker上配置延迟级别，默认值如下：
> messageDelayLevel=1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h  


> 这个配置项配置了从1级开始，各级延时的时间，可以修改这个指定级别的延时时间；   
> 时间单位支持：s、m、h、d，分别表示秒、分、时、天；

1.broker对于每个DelayLevel建立自己的队列
2.有延迟消息到来的时候，暂时保存其真实的topic和queueId,将主题设置为 SCHEDULE_TOPIC ，队列设置为DelayLevel所对应的队列中（逻辑见如下引用代码）；

```
 				 topic = ScheduleMessageService.SCHEDULE_TOPIC;
                queueId = ScheduleMessageService.delayLevel2QueueId(msg.getDelayTimeLevel());

                // Backup real topic, queueId
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_TOPIC, msg.getTopic());
                MessageAccessor.putProperty(msg, MessageConst.PROPERTY_REAL_QUEUE_ID, String.valueOf(msg.getQueueId()));
                msg.setPropertiesString(MessageDecoder.messageProperties2String(msg.getProperties()));

                msg.setTopic(topic);
                msg.setQueueId(queueId);
```

3.启动线程模拟成SCHEDULE_TOPIC的消费者，取队列头部消息比较是否已经满足投递时间，如果满足，则将消息按原来的topic和queueId投向原有队列。

```
 ConsumeQueue cq = ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(SCHEDULE_TOPIC,delayLevel2QueueId(delayLevel));
                    
```

### 2.2.6 消息轨迹
### 2.2.7 消息回溯
* FMQ支持按照offset和timestamp进行回溯

### 2.2.8 并行消费

## 2.3 运维特性
### 2.3.1 配置
### 2.3.2 监控
### 2.3.3 归档
### 2.3.4 报警

# 3 性能对比
## 单机性能对比
## 集群性能对比

# 4 友好性对比
## 4.1 文档完备性
## 4.2 快速入门
## 4.3 部署难度
## 4.4 社区活跃度