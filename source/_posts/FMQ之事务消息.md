---
title: FMQ之事务消息
date: 2018-04-08 14:40:13
tags:
- 中间件
- 消息队列
categories: 编程
comments: true
---
*事务消息是指帮用户实现类似XA的分布式事务功能。*
<!--more-->

# 1 概述

## 1.1 分布式事务

如果事务的参与者、支持事务的服务器及资源分别位于不同的分布式系统的不同节点之上，那就称为分布式事务。一个事务可能由很多操作构成，这些小的操作分布在不同的服务器上，且属于不同的应用，分布式事务需要保证这些小操作要么全部成功，要么全部失败。

## 1.2 产生

分布式事务的核心在于保证不同系统的数据一致性。比较应用SOA化以后，各个模块都有自己的数据库。分布式事务就是保证不同数据库的数据一致性。

## 1.3 解决方案

一般分布式事务的解决方案主要采用X/Open DTP模型，此方案主要问题是协议和实现都比较复杂，且效率较低。
eBay提供了一个分布式系统一致性问题的解决方案。它的核心思想是将分布式事务变成 本地事务+异步事件通知。将需要分布式处理的任务通过消息队列来异步执行。


# 2 基于消息的分布式事务

## 2.1 业务方实现

一些消息队列本身不支持事务消息（Kafka,rabbitMQ），但只要消息队列保证消息可靠，业务方就可以通过MQ自己实现分布式事务。自己实现的话步骤如下：

* 将涉及分布式系统的的操作A和操作B所在的应用，分别定义成某个主题的生产者与消费者。
* 在生产者的数据库中加一个事务状态表，将本地操作与此表状态的更新放在同一个本地事务中，这样就维持了数据的一致性。其中一个字段记录事务状态，一个字段记录消息的投递状态。
* 对于本地回滚的事务，不进行投递。
* 对于本地commit的事务，则向消息队列投递消息，且必须得到消息队列投递成功的响应，然后更改数据库中的消息投递状态。
* 对于已经commit的事务，如果消息投递状态还没改变，则起线程循环投递，直到成功。
* 其它数据库的更新操作则订阅对应的消息，由于消息队列保证不丢消息，所以一定能保证最终一致性。注意消费者需要自己保证幂等。


## 2.2 事务消息

另一个方案就是采取自身支持事务消息的MQ。事务消息的基本思路如下图所示。

![分布式事务](http://ovor60v7j.bkt.clouddn.com/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1.png)

显然，这个方案和2.1中的方案原理上并没有太大区别，只是把需要业务自己做的轮询变成了服务端的回查。

* 对于业务方来说，减少了开发难度。
* 对于多个需要事务消息的应用，不用重复造轮子。
* 对于服务端，交互次数增多，增加了部分压力。

## 3 设计与实现

### 3.1 交互流程

![分布式事务](https://docs-aliyun.cn-hangzhou.oss.aliyun-inc.com/cn/ons/0.1.06/assets/pic/sdk/mq_trans.png)


事务消息交互流程如下：

1. 发送方向 MQ 服务端发送消息；
2. MQ Server 将消息持久化成功之后，向发送方 ACK 确认消息已经发送成功，此时消息为半消息。
3. 发送方开始执行本地事务逻辑。
4. 发送方根据本地事务执行结果向 MQ Server 提交二次确认（Commit 或是 Rollback），MQ Server 收到 Commit 状态则将半消息标记为可投递，订阅方最终将收到该消息；MQ Server 收到 Rollback 状态则删除半消息，订阅方将不会接受该消息。
5. 在断网或者是应用重启的特殊情况下，上述步骤4提交的二次确认最终未到达 MQ Server，经过固定时间后 MQ Server 将对该消息发起消息回查。
6. 发送方收到消息回查后，需要检查对应消息的本地事务执行的最终结果。
7. 发送方根据检查得到的本地事务的最终状态再次提交二次确认，MQ Server 仍按照步骤4对半消息进行操作。

其中1-4属于正常流程，而5-7属于异常情况下的回查机制（后文称为Feedback机制）。

### 3.2 客户端设计

客户端设计比较简单，只需解决以下问题

* 如何设计生产接口
* 如何设计回查的接口
* 如何保证事务ID的唯一性
* 宕机了如何知道哪些事务没有提交

### 3.2.1 生产接口设计

为了减少业务方的自由度和工作量，生产的流程的prepare,commit,rollback操作由提供出去的SDK自己定义。所以提供回调供业务方实现本地事务的业务逻辑。调用方式如下(此部分设计参考RocketMQ):


```
producer.sendMessageInTransaction(msg, new LocalTransactionExecuter() {
                                @Override
                                public TransactionState executeLocalTransactionBranch(Message msg) {
                                    //本地事务，返回值为事务成功或失败
                                }
                            })
```
接口的返回值为事务的成功或失败，定义如下

```
public enum TransactionState {
    //事务准备
    PREPARE,
    //事务已提交
    COMMITTED,
    //事务已回滚
    ROLLBACK,
    //未知状态
    UNKNOWN;
}
```

### 3.2.2 回查接口设计

回查接口是服务端发现事务处于不确定的状态后，向客户端进行查询。

```
public interface TxStatusQuerier {
    /**
     * 查询事务状态
     *
     * @param txId    事务ID
     * @param queryId 查询ID
     * @return 事务状态
     */
    TransactionState queryStatus(String txId, String queryId);
}
```
返回值为事务的状态，缺点是要像业务方暴露事务的状态。

## 3.2.3 事务ID

事务ID作为唯一标识，支持业务方自定义。
如果用户不进行定义则为  “客户端版本 - 客户端IP - 客户端启动时间 - 进程号 - 连接自增数 - 事务自增数”。

## 3.2.4 宕机恢复




## Prepare

### store.beginTransaction

* 判断事务是否超出限制个数
* brokerPrepare刷盘（不包含消息内容）
* 判断事务（id）是否存在，如果不存在添加id到事务的映射关系，并将此topic事务数+1
* 准备BrokerPreMessage
* putBrokerPreMessage
	* 选择队列
	* append preMessage
	* 建立BrokerRefMessage
	* 内存prepare添加refMessage
* 归档



## Commit

* store.commitTransaction
	* refMessage刷盘
	* commitMessage刷盘
	* 事务ID移除

## RollBack

* BrokerRollback刷盘
* 事务ID移除


## Feedback

### client

#### TopicFeedback

* 每个app+topic唯一对应一个TopicFeedback
* TopicFeedback里保存map<BrokerGroup,GroupFeedback>
* GroupFeedback
	* 独立线程，按照100ms间隔不断向服务端发TxFeedback命令（5s超时）。如果更新成功，则变更TxFeedback状态。
	* 如果feedback带事务ID，根据状态进行commit或者rollback。
	* 锁定并获取未提交的事务返回给客户端
	* 没有锁定的事务则挂起（长轮询）


## 疑问

* 宕机了如何维护内存的prepare
* 刷盘不同消息做哪些操作
	* dealBrokerMessage
		* 刷盘包括消息
		* 更新索引文件，刷盘
	* dealPrepare
		* 刷盘brokerPrepare，没有消息体
	* dealBrokerPreMessage
		* 刷盘BrokerPreMessage，包括消息体
	* dealBrokerRefMessage
		* 判断和BrokerPreMessage还是不是一个文件
		* 判断消息大小是否小于 事务需要建索引消息下限，小于需要建立索引
		* 如果与pre不处于同一个文件 || 文件剩余空间不能写下 ，也需要建立索引
		* 如果需要建索引
			* 复制一份原来的消息
			* 当做TYPE_TX_MESSAGE消息进行刷盘
		* 如果不需要建索引，ref_message刷盘
		* 异步更新索引文件，刷盘
	* dealCommit
		* commitMessage刷盘
		* rollBack刷盘
* 由于prepare阶段没有建立索引，如果2天还没有commit，原始文件被删除了怎么办
* 宕机了如果恢复事务管理器
* 是否支持消息回溯

## RocketMQ问题

* table & redo log是否同步刷盘，对于效率的影响
* 修改commit log导致脏页的问题