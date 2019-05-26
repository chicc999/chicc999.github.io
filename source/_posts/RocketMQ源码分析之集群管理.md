---
title: RocketMQ源码分析之集群管理
date: 2018-02-11 15:06:47
tags:
- java
- 消息队列
- RocketMQ
categories: 编程
comments: true
---
*本文为RocketMQ NameServer源码解析，主要包括服务启动，broker注册，路由管理等功能。*
<!--more-->

# 1 服务启动与停止

NameServer的主函数在NamesrvStartup类，整体逻辑比较简单，只做了四件事：

* 读取配置文件，设置环境变量。
* NamesrvController调用初始化方法initialize()。
* 注册系统钩子，在虚拟机关闭的时候调用NamesrvController的shutdown函数。
* 启动NamesrvController（启动网络监听）。

所以启动过程的主干逻辑在类NamesrvController中，忽略读取配置文件，看其他逻辑。

## 1.1 初始化

在启动逻辑的4件事情中，其余3件都为流程化的事情，初始化则和此模块直接相关。

* 加载配置与初始化网络模块remotingServer。
* 初始化了一个业务线程池remotingExecutor，用来在网络模块对请求做初步的处理后，处理此模块相关的业务。
* 定时任务线程池scheduledExecutorService，扫描broker是否存活和打印配置。

```java
public boolean initialize() {

        // 加载配置
        this.kvConfigManager.load();
        // 初始化网络模块服务端
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
        // 初始化业务线程池
        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
        // 注册业务处理器
        this.registerProcessor();

        // 定时扫描任务，处理非激活态broker
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);

        // 定时任务，按一定周期打印配置
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);

        return true;
    }
```

## 1.2 注册钩子

```java
 Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
                @Override
                public Void call() throws Exception {
                    controller.shutdown();
                    return null;
                }
            }));
```

注册钩子，在虚拟机退出的时候调用NamesrvController的shutdown函数，释放资源。

## 1.3 启动与关闭

代码如下：

```java

 public void start() throws Exception {
        this.remotingServer.start();
    }

    public void shutdown() {
        this.remotingServer.shutdown();
        this.remotingExecutor.shutdown();
        this.scheduledExecutorService.shutdown();
    }

```

* NamesrvController的start方法实际是去启动了remotingServer，没有其它逻辑。remotingServer的逻辑各个模块共用，详见网络模块的源码分析。
* 关闭逻辑在虚拟机被关闭的时候被调用，主要就是用来释放资源，调用了1.1中remotingServer，remotingExecutor，scheduledExecutorService三个类的shutdown方法进行优雅关闭。

# 2 路由管理

路由管理功能的实现逻辑主要在类RouteInfoManager中，其中除了读写锁，还维持了5个map用来存放路由数据。

| 变量名        | 类型(map)     | 含义|
|:----------:|:-------------:|:-------------:|
| topicQueueTable|  String/\*topic\*/, List&lt;QueueData&gt; | 保存&lt;主题,队列信息&gt;映射的集合|
|brokerAddrTable| String/\*brokerName\*/, BrokerData |保存&lt;broker地址,broker配置&gt;映射关系集合|
|clusterAddrTable| String/\*clusterName\*/, Set&lt;String/\*brokerName\*/&gt; |保存&lt;集群名称，集群中broker集合&gt;的映射关系|
|brokerLiveTable|String/\*brokerAddr\*/, BrokerLiveInfo|保存&lt;broker地址,broker存活信息&gt;映射关系集合|
|filterServerTable|String\*brokerAdd\*/, List&lt;String&gt;\*Filter Serve\*/|保存&lt;broker地址,过滤服务器名称&gt;映射关系集合|

## 2.1 broker注册

broker启动的时候，会给NameServer发送REGISTER_BROKER指令，其带有的信息如下表：

| 变量名        | 含义     | 
|:----------:|:-------------:|
| brokerName|  broker名称   | 
|brokerAddr|broker地址|
|clusterName|集群名称|
|haServerAddr|ha服务的地址|
|brokerId|集群中brokerid,0为master其余为slave|
|topicConfigTable|主题配置|
|dataVersion|数据版本|
|filterServerList|过滤服务器列表|

其中过滤服务器等消息过滤模块再做解读。注册代码如下：

```java
public RegisterBrokerResult registerBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final Channel channel) {
        RegisterBrokerResult result = new RegisterBrokerResult();
        try {
            try {
                this.lock.writeLock().lockInterruptibly();

                // 添加broker到其对应的集群中
                Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
                if (null == brokerNames) {
                    brokerNames = new HashSet<String>();
                    this.clusterAddrTable.put(clusterName, brokerNames);
                }
                brokerNames.add(brokerName);

                boolean registerFirst = false;
                // 初始化brokerData对象
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null == brokerData) {
                    registerFirst = true;
                    brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                    this.brokerAddrTable.put(brokerName, brokerData);
                }
                // 4行之前似乎没有必要设置为true,此部分加了写锁，如果brokerData为新增，则此处oldAddr为null，会在135行置为true
                String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
                // 新增broker的话将registerFirst置为true
                registerFirst = registerFirst || (null == oldAddr);

                if (null != topicConfigWrapper
                    && MixAll.MASTER_ID == brokerId) {
                    if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                        || registerFirst) {
                        // 如果是主题信息发生变更或者首次注册，且注册的是master节点，则变更数据
                        ConcurrentMap<String, TopicConfig> tcTable =
                            topicConfigWrapper.getTopicConfigTable();
                        if (tcTable != null) {
                            for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                                // 遍历变更队列数据
                                this.createAndUpdateQueueData(brokerName, entry.getValue());
                            }
                        }
                    }
                }

                // 更新或添加broker存活信息表
                BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                    new BrokerLiveInfo(
                        System.currentTimeMillis(),
                        topicConfigWrapper.getDataVersion(),
                        channel,
                        haServerAddr));
                if (null == prevBrokerLiveInfo) {
                    log.info("new broker registerd, {} HAServer: {}", brokerAddr, haServerAddr);
                }

                // 更新过滤服务器列表
                if (filterServerList != null) {
                    if (filterServerList.isEmpty()) {
                        this.filterServerTable.remove(brokerAddr);
                    } else {
                        this.filterServerTable.put(brokerAddr, filterServerList);
                    }
                }

                // 如果是slave发送过来的注册请求，将master地址填充到response中
                if (MixAll.MASTER_ID != brokerId) {
                    String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                    if (masterAddr != null) {
                        BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                        if (brokerLiveInfo != null) {
                            result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                            result.setMasterAddr(masterAddr);
                        }
                    }
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("registerBroker Exception", e);
        }

        return result;
    }
```

在注册时，broker会上报主题的配置数据TopicConfig，NameServer将其管理在本地。同时对每一个broker的存活状态BrokerLiveInfo进行维护。BrokerLiveInfo的结构如下：

```java
class BrokerLiveInfo {
    // 最后更新时间
    private long lastUpdateTimestamp;
    // 数据版本
    private DataVersion dataVersion;
    // 通道
    private Channel channel;
    // ha服务地址
    private String haServerAddr;
 }
```
存活信息什么时候更新呢？在1.1启动流程中，我们注意到有个定时线程来做这件事。

## 2.2 存活信息更新

broker在启动时，会每隔30s发送一次2.1中提到的注册信息。

```java
 this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false);
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
        }, 1000 * 10, 1000 * 30, TimeUnit.MILLISECONDS);

```

在NameServer执行registerBroker中，会更新BrokerLiveInfo中的时间。同时NameServer中的定时任务检测最后更新的时间，如果超过阈值，则断开连接。


```java
 public void scanNotActiveBroker() {
        Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, BrokerLiveInfo> next = it.next();
            long last = next.getValue().getLastUpdateTimestamp();
            // 如果BROKER_CHANNEL_EXPIRED_TIME没有更新，则断开连接
            if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
                RemotingUtil.closeChannel(next.getValue().getChannel());
                it.remove();
                log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
                this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
            }
        }
    }
```

注意此处最后一行

```java
this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
```

closeChannel会触发close事件，在close事件的回调中代码如下：

```java
public class BrokerHousekeepingService implements ChannelEventListener {
    ……

    @Override
    public void onChannelClose(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
    ……
    }
```

重复对资源进行了释放与清理。在exception和idle时也有同样的问题。onChannelDestroy的逻辑与2.3注销比较类似，就是去清理内存的各种数据。有个可以优化的地方就是将brokerAddress直接绑定在channel，避免加锁去遍历集合。

## 2.3 注销

注销部分代码如下，相对比较简单。不过有一个问题是，是否有必要将队列数据汇报给NameServer?感觉只上报broker数据组成集群信息进行路由，而队列部分的数据则交给Broker自身进行管理更为合理一些。

```java
public void unregisterBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId) {
        try {
            try {
                this.lock.writeLock().lockInterruptibly();
                // 从broker存活列表中删除对应broker
                BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.remove(brokerAddr);
                log.info("unregisterBroker, remove from brokerLiveTable {}, {}",
                    brokerLiveInfo != null ? "OK" : "Failed",
                    brokerAddr
                );

                // 过滤器中删除broker
                this.filterServerTable.remove(brokerAddr);

                // 从brokerData中删除注销的broker
                boolean removeBrokerName = false;
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null != brokerData) {
                    String addr = brokerData.getBrokerAddrs().remove(brokerId);
                    log.info("unregisterBroker, remove addr from brokerAddrTable {}, {}",
                        addr != null ? "OK" : "Failed",
                        brokerAddr
                    );

                    if (brokerData.getBrokerAddrs().isEmpty()) {
                        this.brokerAddrTable.remove(brokerName);
                        log.info("unregisterBroker, remove name from brokerAddrTable OK, {}",
                            brokerName
                        );

                        removeBrokerName = true;
                    }
                }

                // 如果cluster中所有broker都注销，则去更新cluster表，移除整个cluster
                if (removeBrokerName) {
                    Set<String> nameSet = this.clusterAddrTable.get(clusterName);
                    if (nameSet != null) {
                        boolean removed = nameSet.remove(brokerName);
                        log.info("unregisterBroker, remove name from clusterAddrTable {}, {}",
                            removed ? "OK" : "Failed",
                            brokerName);

                        if (nameSet.isEmpty()) {
                            this.clusterAddrTable.remove(clusterName);
                            log.info("unregisterBroker, remove cluster from clusterAddrTable {}",
                                clusterName
                            );
                        }
                    }
                    // 移除主题中和此broker相关的队列数据
                    this.removeTopicByBrokerName(brokerName);
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("unregisterBroker Exception", e);
        }
    }
```

# 3 信息获取

这一部分主要是对客户端的接口包括以下指令：

| 指令        | 含义     | 
|:----------:|:-------------:|
|PUT（GET/DELETE）_KV_CONFIG|添加/查询/删除配置|
| GET_ROUTEINTO_BY_TOPIC|  获取主题的路由信息   | 
|GET_BROKER_CLUSTER_INFO|获取broker的集群信息|
|WIPE_WRITE_PERM_OF_BROKER|屏蔽broker写权限|
|GET_ALL_TOPIC_LIST_FROM_NAMESERVER|获取所有主题|
|DELETE_TOPIC_IN_NAMESRV|删除主题|
|GET_KVLIST_BY_NAMESPACE|获取某个命名空间下所有KV配置|
|GET_TOPICS_BY_CLUSTER|获取某个cluster下所有主题信息|
|GET_SYSTEM_TOPIC_LIST_FROM_NS|获取系统主题列表|
|GET_UNIT_TOPIC_LIST|未知|
|GET_HAS_UNIT_SUB_TOPIC_LIST|未知|
|GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST|未知|
|UPDATE（GET）_NAMESRV_CONFIG|更新（获取）nameserver配置|

大部分指令都是单纯的返回内存的值，选一些有实际逻辑指令分析下。

## 3.1 GET_ROUTEINTO_BY_TOPIC

这个指令用来获取主题的路由信息。

```java
TopicRouteData topicRouteData = this.namesrvController.getRouteInfoManager().pickupTopicRouteData(requestHeader.getTopic());

        if (topicRouteData != null) {
            if (this.namesrvController.getNamesrvConfig().isOrderMessageEnable()) {
                String orderTopicConf =
                    this.namesrvController.getKvConfigManager().getKVConfig(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG,
                        requestHeader.getTopic());
                topicRouteData.setOrderTopicConf(orderTopicConf);
            }
```

先是通过主题获取路由信息，然后再添加顺序消息相关的配置。获取路由信息的代码如下：

```java
 try {
                this.lock.readLock().lockInterruptibly();
                // 获取此主题所有队列
                List<QueueData> queueDataList = this.topicQueueTable.get(topic);
                if (queueDataList != null) {
                    topicRouteData.setQueueDatas(queueDataList);
                    foundQueueData = true;

                    Iterator<QueueData> it = queueDataList.iterator();
                    while (it.hasNext()) {
                        // 遍历所有队列，获取相关broker进行保存
                        QueueData qd = it.next();
                        brokerNameSet.add(qd.getBrokerName());
                    }

                    for (String brokerName : brokerNameSet) {
                        // 获取broker相关路由信息（含过滤服务器）
                        BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                        if (null != brokerData) {
                            BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap<Long, String>) brokerData
                                .getBrokerAddrs().clone());
                            brokerDataList.add(brokerDataClone);
                            foundBrokerData = true;
                            for (final String brokerAddr : brokerDataClone.getBrokerAddrs().values()) {
                                List<String> filterServerList = this.filterServerTable.get(brokerAddr);
                                filterServerMap.put(brokerAddr, filterServerList);
                            }
                        }
                    }
                }
            } finally {
                this.lock.readLock().unlock();
            }
```

做了以下三件事：

* 选出和主题相关的队列，添加到响应中
* 选出和这些队列相关的broker,将这些broker的路由信息添加到响应中
* 添加过滤服务器




# 4 总结

NameServer的设计总体来说还是一个比较简单的路由服务，总体设计成由broker进行上报信息，由以下好处：

* 避免了nameServer进行数据存储与集群间同步
* NameServer本身无中心架构，可以横向扩展

但同时也带来了一个问题：

* 通过broker进行上报而非中心化分配，那例如创建主题这些指令，是如何来选择合适的broker呢？

带着这个疑问，继续看源码其他部分。