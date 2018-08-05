---
title: FMQ 从节点消费
date: 2018-05-21 16:51:14
tags:
- java
- 消息队列
categories: 编程
comments: true
---
*FMQ每组broker分为主从两个角色，目前默认读写都在主节点。但是一旦主节点宕机，写入的高可用由权重路由算法保证，读取的高可用则必须支持从节点消费。*
<!--more-->

# 1 介绍

FMQ每组broker分为主从两个角色，目前默认读写都在主节点。但是一旦主节点宕机，如何保证高可用就是一个问题。写入的高可用可以由权重路由算法保证，读取的高可用则要求从节点要能进行消息消费，而不仅仅是一个消息备份的作用。
要设计从消费功能，主要要解决以下问题：

* 从消费功能的触发条件是什么，即何时开启从消费功能？
* 消费者客户端如何能正确并且及时的知道现在的消费模式，和对应的服务端建立通信？
* 主从之间的信息（数据文件、消费索引文件）如何进行同步？
围绕以上三个问题，我们来介绍从节点消费功能的详细设计。

在此之前先介绍下启动流程。


## 1.1 角色判断

* 采取固定角色策略
	* 管理员在web控制台配置好每一组的主从broker，存放在数据库中，由task写入zookeeper。
	* broker启动时从zk获取集群信息，判断自身角色，根据角色启动HAMaster或者HASlave。
	
## 1.2 主从数据同步

slave主动连接master，建立数据同步的连接。同步的数据包括日志数据以及消费者的偏移数据。
master接收到生产消息的请求后，会将写请求持续的转发到从节点，在从节点存储日志数据成功后，再返回成功。
	

# 2 路由规则

从消费要解决的一个关键问题是，如何让客户端及时、准确的建立连接？
现有客户端是采用定时轮询的方式获取broker的权限以及其它信息，无论怎样都会有30s以上的时间窗口。加上客户端缓存设置，配置拉取不成功直接采用原有配置，想做到“及时“的话，只能通过服务端权限控制配合客户端自动重连实现。

## 2.1 服务端权限控制

通过服务端自身进行读写权限控制，当此节点没有读权限时，对AddConsumer请求返回false。

```
if (!broker.isReadable()) {
            // 机器被禁用消费
            throw new Exception(broker.getName() + " is not readable", NO_PERMISSION.getCode());
        }
        
 public boolean isReadable() {
        return role != ClusterRole.NONE && getPermission().contain(Permission.READ);
    }
```

同时客户端接收到非success的响应以后，直接关闭连接，配合2.2中的自动重连实现目标服务器的切换。

## 2.2 客户端自动重连

* 客户端在连接关闭后，会触发重连。
* 重连获取地址策略采用按权限排序，逐个尝试（FULL,WRITE,READ,NONE）

由此，只要控制好服务端的权限，保证每组同时只有一个Broker有读权限，即可做到客户端自动进行路由切换。

## 2.3 消费过程中权限变化

* 在每次拉取消息/消费确认时，都对权限进行检测，一旦权限变更，则返回“无权限”。
* 客户端收到“无权限”的响应，触发链路的健康检查。
* 健康检查会再次确认权限，如果权限校验失败，则断开连接，触发2.2中的重连。

## 2.4 权限控制正确性

* 非从消费情况下，主权限为所配置的权限，其余权限为NONE。
* 从消费下，主权限为NONE，从权限为原始权限与Read的交集。
* 是否处于从消费状态，由zk保证。


# 3 触发条件

先考虑场景，以下何种条件需要触发从消费？

* 1 主被禁用或禁止读权限
* 2 主宕机
* 3 主和zk断开连接，但和从有连接

目前只考虑主宕机的情况，即master与zk和slave同时断开连接。其中场景3无需考虑切换，权限不做变更。场景1也不做主从切换，认为是管理员手动干涉的一种情况。  
场景2具体触发条件必须要分为master和slave两方面来进行考虑。

## 3.1 master

目前场景只考虑Master宕机的情况下切换为从消费，所以不用考虑master触发条件。
但此时存在一个问题，如果master跟zk和slave同时断开，slave切换为从消费模式（添加读权限）。与此同时如果master与某个消费者存在网络问题，消费者路由到从节点。此时主从同时存在读权限。
为了避免这种情况，在发生以上问题的时候，直接取消master的读权限。

```
 if (isCurrentRoleMaster()) {
                // master孤独条件：没有slave，且ZK连接不上
                if (hasNoneSlave() && !zookeeper.isConnected()) {
                    clusterManager.removeBrokerPermissionRead();
                }
            }
```

此处有个问题：

* 连不上zk和slave，读权限移交给了slave,是否还允许其被写入？

考虑到我们broker采用多机房部署，几乎不存在所有broker组同时宕机的情况，实际实现时考虑移除master所有权限。

```
 if (isCurrentRoleMaster()) {
                // master孤独条件：没有slave，且ZK连接不上
                if (hasNoneSlave() && !zookeeper.isConnected()) {
                    clusterManager.setPermission(Permission.None);
                }
            }
```


## 3.2 slave

Slave节点需要自行判断是否需要触发从消费。首先启动一个定时线程，每秒检测一次，当以下条件连续同时满足一定检测次数时，切换为从消费：

* 自身能连接zk且master节点与zk断开
* 与master断开连接
* master拥有读写权限
* slave自身拥有读权限

以上第一点与第二点保障master宕机或者类似宕机（孤岛）。同时slave自身一定要能连zk才触发从消费，否则会导致slave和master、zk断开连接时自行切换到从消费，从而发生主从都有Read权限的问题。第四点的话则保证如果slave本身就没有读权限，就不应该进行从消费的切换。

切换从消费slave主要逻辑如下：

* 在zk建立从消费节点（方便cluster在路由阶段做权限过滤）
* 如果从节点配置有读权限，给从节点赋予读权限（非从消费模式从节点即使配置了读权限，实际权限也为None）

示例代码如下：

```
            String slaveConsumePath = PathUtil.makePath(config.getSlaveConsumePath(), brokerGroup.getGroup());
            zookeeper.createLive(slaveConsumePath, JSON.toJSONString(brokerGroup).getBytes());
```

但此处存在问题，当从消费一段时间后宕机。主节点恢复时，会辨认不出从消费状态（zk上临时节点，从宕机后消失），从而让消费者重复消费。此处暂时不考虑重复消费，由业务自身进行去重。


# 4 数据同步

## 4.1 consume offset同步

![consume offset同步](http://ovor60v7j.bkt.clouddn.com/getConsumeOffset.png)

* slave：启动定时任务，将自身的ConsumeOffset、是否处于从消费模式两个字段序列化后发给master。
* master：如果slave处于从消费模式，则master中的ConsumeOffset更新为slave内存中的值。
* master：如果master判断自身也处于从消费模式，则切换为主消费模式。
* slave：无条件更新自身的consumeOffset为master的值。

# 4.2 master恢复时数据同步

* cluster监听zk slave_consume下所有值。存储在map中，计算权限时只赋予该group的master主节点写权限，不会有流量打到新启动的master上。
* master在slave连上来以后，如果slave处于从消费模式，自身offset更新为slave的值。取消自身从消费模式。
* slave连上master,在进行完一次offset同步以后，取消从消费状态，删除zk从消费节点。
* 所有watch此节点的cluster得到通知，路由部分添加master读权限。


此处存在一个问题：

* master自身权限没做限制，在启动期间客户端如果和从节点发生闪断，连上来也能读取。

根据以上问题，改进了流程，在从节点取消读权限之前master不给于读权限，在zk发生变更以后再做修改：

master崩溃后启动时序图如下所示：
![consume offset同步](http://ovor60v7j.bkt.clouddn.com/master%E5%90%AF%E5%8A%A8%E6%97%B6%E5%BA%8F.png)

* master启动，向zk注册存活，并且检查是否处于从消费状态。
* 发现处于从消费状态，将自身权限置为NONE。
* 等到slave连接完成，同步offset。
* slave同步offset完成，开始退出从消费状态
	* 删除zk上的从消费节点
	* 关闭消费连接
	* 给master发送同步consumerOffset的命令
* master收到consumerOffset
	* 将自身的consumerOffset更新成slave的
	* 结束从消费状态
	* 回复响应
* zk上从消费节点变更，主从得到通知，master从zk更新为正常权限。

其中最后一步有可能在绿色框内的任意时间点发生。所以当其在master进行updateConsumerOffset之前进行触发，则会过早的给予master读权限，导致客户端重复消费。由于客户端会进行去重，暂时不做解决。

## 5 问题

因为在设计里master和zk、slave同时断开连接就进入从消费模式，但是在实际场景中，未必真的处于从消费模式，也就造成了zk或者slave重新连接上以后，master无法恢复为正常模式。考虑以下两个场景：

### 5.1 场景描述

* 场景1：slave启动之前，master因为与zk断开连接切换为从消费模式；slave启动后由于zk不存在从消费节点，无法触发master进行权限更新（将NONE置为配置的权限）。
* 场景2：slave不存在（未在数据库配置），master与zk发生网络异常断开，master切换为从消费。一段时间后master与zk连接恢复，master权限应该恢复但按照上文设计并未恢复。
* 场景3：slave配置了但是未曾启动，master与zk发生网络异常断开，master切换为从消费。一段时间后master与zk连接恢复，master权限应该恢复但按照上文设计并未恢复。
* 场景4：slave起始与master成功连接，但是在master切换为从消费时自身未切换未从消费（在此之前就宕机或者与zk断开连接），并且在master恢复后slave任然未恢复

### 5.2 方案修复

### 5.2.1 场景1

在HAMaster模块中，slave连接上来以后加上如下代码


```java
  if(slaveConsumeManager.isSlaveConsume()) {
                if(!getConsumeOffset.isSlaveConsume()){
                    //如果master处于从消费而slave不是从消费，有可能是master与zk、slave失联导致
                    //此时不会由zk从消费节点删除触发权限更新，主动更新权限
                    slaveConsumeManager.updatePermissionFromRegistry();
                }
                slaveConsumeManager.cancelSlaveConsume();
            }
```

如果master自身处于从消费状态，且slave不处于从消费状态，则让master强制从zk上更新权限信息。


### 5.2.2 场景2

slave不存在时（没在数据库配置），master改为不会切换为从消费。在切换前先去判断是否分配了从节点。

```
/**
		 * 该master分配了从节点
		 * @return true 节点角色为主且group中broker数量大于1
		 */
		private boolean hasSlaveAssigned(){
			return clusterManager.getBroker().getRole().equals(ClusterRole.MASTER) 
					&& clusterManager.getBrokerGroup().getBrokers().size() >1;
		}
		
…
if (isCurrentRoleMaster()) {
				if (hasSlaveAssigned() && brokerMonitor.hasNoneSlave() && !registry.isConnected()) {
					// slave分配但未连接，且zk连接不上,取消master所有权限，写转移到其它group
					SwitchToSlaveConsume();
				}
			}
…

```

### 5.2.3 场景3

此种情况下增加一个参数，表明是否曾经有slave连上来过。如果从未有slave连上来过，则master也不会切换为从消费。

```
if (isCurrentRoleMaster()) {
				if (hasSlaveAssigned() && slaveOnceConnected && brokerMonitor.hasNoneSlave() && !registry.isConnected()) {
					// slave分配但当前未连接，且曾经连接过，且zk连接不上,取消master所有权限，写转移到其它group
					SwitchToSlaveConsume();
				}
			}
```

### 5.2.4 场景4

由于此种状况下slave与master同时出问题，且slave一直未恢复，此种状态不进行恢复，由人工干预进行恢复操作。



