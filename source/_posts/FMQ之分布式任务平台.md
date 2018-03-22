---
title: FMQ之分布式任务平台
date: 2018-03-01 16:09:44
tags:
- java
- 消息队列
categories: 编程
comments: false
---
*FMQ系统的分布式任务平台，主要以插件的形式提供各种辅助任务的执行。*
<!--more-->
## 1 总体架构
分布式任务平台主要用于执行一些额外的单次任务或者守护任务，包括但不限于进行系统的数据统计，将其他队列的消息转移到FMQ，或者进行数据转移等。其总体设计如下：
![Task总体架构](http://ovor60v7j.bkt.clouddn.com/task.png)

其中zookeeper中用到三个节点，分别提供主从选举，存活检测，以及任务发现的功能。
任务平台主要分成三个模块：

* registry负责跟zookeeper交互；
* 在成为主节点后，taskDispatcher负责进行所有任务的调度工作，以保证任务负载的均衡；
* taskService负责起不同的任务实例，执行具体的任务。

具体的任务以插件的形式提供给平台，实现对应接口以后，在控制台进行配置写入数据库，此任务即能被任务平台所调度和执行。



## 2 Zookeeper & Registry
任务平台使用Zookeeper来进行主从选举，存活检测，以及任务发现的功能。其中registry模块负责将Zookeeper的事件转化为任务平台感兴趣的事件，以触发后续流程。如下图所示。  

![ZK结点](http://ovor60v7j.bkt.clouddn.com/zk%E7%BB%93%E7%82%B9.png)

### 2.1 存活检测
每个task实例启动以后，registry模块在task/live路径下注册EPHEMERAL类型的结点，结点名称为实例所在机器的“ip_prot“。  
如果有实例宕机、网络不通或者无法提供服务，则会断开与zookeeper连接，临时结点在心跳超时后消失;又或者有新机器加入，路径下新增了结点。此时因为路径下数据变动，触发master实例在此结点上的回调，通知其它模块，进行任务的负载均衡调度或者其他操作。

### 2.2 主从选举
每个实例的registry模块启动后向task/leader路径注册PERSISTENT_SEQUENTIAL类型的结点，名字为“member”+自动递增编号。每个实例获取此路径下所有结点，如果自身的编号为其中编号最小值，则表示自己抢到了锁，即为master。
但是以上方案有两个明显的问题：

* 在分布式锁的竞争中，所有实例都要获取所有的结点的编号并做判断。这样就限制了分布式系统的规模（机器数上升会导致选主时间大量增加）。
* 对于所有结点，只有一个可以成为主结点。其余结点进行了大量通信和比较，最终都是无用功。

所以对于原方案，进行些许改动，以避免羊群效应：

* registry获取到task/leader下所有的子结点后，如果判断自身不是master，watcher不再注册到整个路径的变化上，而是只关注序号最近的比自己恰好小的结点。
* 如果watcher被触发，要么说明已经没有比自己更小的结点，自身可以成为master；要么代表关注的结点出现了问题，将watcher重新监听到新的比自己恰好小的结点上。

### 2.3 任务发现
task/executor结点主要用来存储任务的调度信息。由master对所有任务进行统一的调度和分发，以“实例名：任务ID1，任务ID2“的形式将结果写在此结点。
其余实例watch这个结点，在数据变化时，找到自己需要执行的任务编号，加载插件，执行对应任务。

## 3 TaskDispatcher

如果某个实例成为master，则尤其启动TaskDispatcher模块，负责整个系统任务的调度以及负载均衡。
registry在通过zk进行选举，当自身所在机器成为master以后，即启动TaskDispatcher模块，对任务进行分布式的调度。如下图所示，在机器数量变更时（新增或减少），通过监控zk live结点的数据得到通知，触发LoadBalance类型的调度；平时则固定时间(间隔IDLE)进行Minimum类型的调度。

![dispatch](http://ovor60v7j.bkt.clouddn.com/dispatch.png)

### 3.1 任务分配

任务分配过程包括以下任务

* 在上一轮中已经被分配过的任务，且执行器依然存活
* 上一次分配完成后新增的任务
* 在上一轮中已经被分配过的任务，但是被分配的执行器已经不再存活
* 执行失败且需要重试的任务（一般为执行结果非幂等的任务，如守护任务等）

在每次执行中，对于某个任务遵循以下逻辑。  

![任务分配](http://ovor60v7j.bkt.clouddn.com/%E4%BB%BB%E5%8A%A1%E5%88%A4%E6%96%AD.png)

### 3.2 负载均衡

负载均衡仅在执行器数量变更的时候被执行（非变更时正常分配就会分配给任务最少的执行器），按照以下逻辑：

* 执行器按任务数升序排序，任务按照状态和优先级降序排序
* 对于任务多的执行器（任务数大于平均值），只要此任务不是被指定分配给此结点，则将此任务分配到此时任务最少的执行器上

### 3.3 移除任务

每个执行器启动的时候配置了最多同时执行的并发任务，所以如果分配的任务超出并发的最大值，只能将多余的任务进行删除操作。删除遵循以下逻辑：

* 按照任务状态进行排序，一旦任务数量少于并发度，即停止删除
* 优先删除普通任务，再删除守护任务

此处设计不合理，守护任务可以延迟调度但不应该被删除。后续修改。普通任务因为是幂等任务，只影响时效性，不会对最终结果产生影响。


## 4 TaskService

TaskService是具体执行任务的执行器，其会根据数据库配置的任务类型，去找寻对应的ServicePlugin的实现。采用了SPI模式对任务进行扩展，即为接口寻找具体服务的实现类。


>SPI使用：当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。

 

![类结构](http://ovor60v7j.bkt.clouddn.com/%E7%B1%BB%E7%BB%93%E6%9E%84.png)

### 4.1 Service

Service提供了某种功能的接口。在分布式调度平台上，为顶层为ServicePlugin接口，子接口TaskExecutor定义了task的生命周期，子类AbstractTaskExecutor定义了每个任务的流程如下：

```java
 public void execute(final TaskContext context) throws Exception {
        this.context = context;
        try {
            initialize(context);
            doExecute();
            clear();
        } catch (Exception e) {
            stop();
            throw e;
        }
    }
```

### 4.2 Service Loader

Service Loader负责发现和加载classpath中所有的Service实现。

```java
ServiceLoader<T> loader = ServiceLoader.load(clazz,clazz.getClassLoader());
        for (T plugin : loader) {
            plugins.add(plugin);
        }
```

### 4.3 Serivce Provider

提供特定service接口的实现,在本例中，就是AbstractTaskExecutor的各种子类。目前的task中主要提供了三大类型的任务。

* update任务，此任务主要是将数据库里的数据更新到zk上，以通知broker配置信息变更；
* transfer任务，此任务主要用来做其他MQ到FMQ的数据迁移；
* message任务，此种任务类型主要用于FMQ内部的各个模块之间（数据库重试模块，磁盘存储模块，hbase归档模块）进行数据转移；
* statistics任务，此种任务主要是来进行每分钟/小时/天的业务数据统计以及归并；
* clean任务，此任务主要是用来清理数据库中过期的数据。