---
title: RocketMQ之主从复制
date: 2018-10-21 13:14:54
tags:
- java
- 消息队列
categories: 编程
comments: true
---
*本文为RocketMQ消息复制部分的源码解析，同时关注FMQ后续设计跨机房主从复制时有没有可以借鉴与改进的点。*
<!--more-->

# 1 时序模型

# 2 源码剖析

### 2.1 写入与等待

我们只看与复制相关的写流程。

![调用链](http://cyblog.oss-cn-hangzhou.aliyuncs.com/mqReplication/%E8%B0%83%E7%94%A8%E9%93%BE.png)

wrotePostion的更新调用链如上图所示，在MappedFile的appendMessagesInner方法中做了更新。也就是如果判断当前文件可写，就会将消息写入内存映射文件中。然后更新 wrotePosition,代码如下：

```java
 public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb) {
        assert messageExt != null;
        assert cb != null;

        int currentPos = this.wrotePosition.get();

        if (currentPos < this.fileSize) {
            // 如果文件未写满则写入
            ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
            byteBuffer.position(currentPos);
            AppendMessageResult result = null;
            if (messageExt instanceof MessageExtBrokerInner) {
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt);
            } else if (messageExt instanceof MessageExtBatch) {
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt);
            } else {
                return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
            }
            
            // 更新 wrotePosition
            this.wrotePosition.addAndGet(result.getWroteBytes());
            this.storeTimestamp = result.getStoreTimestamp();
            return result;
        }
        log.error("MappedFile.appendMessage return null, wrotePosition: {} fileSize: {}", currentPos, this.fileSize);
        return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
    }
```

在写入内存映射文件以后，处理putMessage请求的线程（后文统一简称为生产线程）调用了handleDiskFlush和handleHA方法等待写磁盘和复制的完成。我们来看handleHA方法,重点方法如注释。

```java
 public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        // 判断角色是否为同步master，如果是异步的话不再等待
        if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
            HAService service = this.defaultMessageStore.getHaService();
            // 判断消息是否需要等待存储完成
            if (messageExt.isWaitStoreMsgOK()) {
                // 如果有slave存在，且当前消息写入位置与已经同步的位置差值小于主从允许的最大差值
                if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
                    GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                    
                    // 存入groupTransferService
                    service.putRequest(request);
                    
                    // 唤醒所有等待的复制线程（HAConnection）开始复制数据
                    service.getWaitNotifyObject().wakeupAll();

                    // 等待复制成功或者超时（用的是同步刷盘的超时）
                    boolean flushOK =
                        request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                    if (!flushOK) {
                        log.error("do sync transfer other node, wait return, but failed, topic: " + messageExt.getTopic() + " tags: "
                            + messageExt.getTags() + " client address: " + messageExt.getBornHostNameString());
                        putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
                    }
                }
                // Slave problem
                else {
                    // Tell the producer, slave not available
                    putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
                }
            }
        }

    }
```

生产线程先判断了是否需要等待同步完成，如果需要的话，则：

* 构建GroupCommitRequest加入到groupTransferService（这个服务下文再说）。
* 通知负责复制的线程，有新的数据写入，唤醒它们工作
* await(SyncFlushTimeout),等待超时或者复制完成被唤醒。

此处有个问题就是，handleDiskFlush与handleHA是串行执行，在极端情况下，可能等待2个SyncFlushTimeout时间才最终超时，与设计的参数解释不符合。即使不是极端情况，也不是一个精确的时间，依赖于当时的写磁盘以及复制状况。

## 2.2 复制

### 2.2.1 HA服务

RocketMQ的HA服务模块，其中包含了3个子服务模块，分别是用于接受从新建连接的AcceptSocketService模块，GroupTransferService，HAClient。除了这三个模块以及启动停止等方法，我们看下重点业务相关方法。

![调用链](http://cyblog.oss-cn-hangzhou.aliyuncs.com/mqReplication/HAService.png)

##### isSlaveOK


```java
/**
     * 如果有slave存在，且当前消息写入位置与已经同步的位置差值小于主从允许的最大差值，则认为slave正常。
     * 否则为异常。
     *
     * @param masterPutWhere
     * @return
     */
    public boolean isSlaveOK(final long masterPutWhere) {
        boolean result = this.connectionCount.get() > 0;
        result =
            result
                && ((masterPutWhere - this.push2SlaveMaxOffset.get()) < this.defaultMessageStore
                .getMessageStoreConfig().getHaSlaveFallbehindMax());
        return result;
    }
```

判断是否有slave正常工作的方法，slave首先要有连接，其次复制的差值小于配置的容忍的主从最大差值，才能被认为slave是正常工作的。

##### notifyTransferSome

```java
    /**
     * 更新slave的复制位置，如果更新成功，唤醒线程恢复响应
     * 
     * @param offset
     */
    public void notifyTransferSome(final long offset) {
        for (long value = this.push2SlaveMaxOffset.get(); offset > value; ) {
            boolean ok = this.push2SlaveMaxOffset.compareAndSet(value, offset);
            if (ok) {
                this.groupTransferService.notifyTransferSome();
                break;
            } else {
                value = this.push2SlaveMaxOffset.get();
            }
        }
    }
```

如上述代码，当slave有offset更新被汇报到master时，调用这个方法，并且将保存的slave复制成功的最大位置更新。如果更新成功，通知groupTransferService去检查能否回复响应。
注意此处实现有个问题在于，当挂载多个slave时，只要有一个成功更新offset，push2SlaveMaxOffset即被更新，客户端无法对消息重要性分级，做到指定的副本数量更新成功才返回响应。


### 2.2.2 Accept服务

AcceptSocketService逻辑比较简单，这个服务先是想selector注册了对accept事件感兴趣，然后不断处理accept事件。循环处理的代码如下：

```java
public void run() {
            log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                try {
                    this.selector.select(1000);
                    Set<SelectionKey> selected = this.selector.selectedKeys();

                    if (selected != null) {
                        for (SelectionKey k : selected) {
                            if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                                SocketChannel sc = ((ServerSocketChannel) k.channel()).accept();

                                if (sc != null) {
                                    HAService.log.info("HAService receive new connection, "
                                        + sc.socket().getRemoteSocketAddress());

                                    try {
                                        HAConnection conn = new HAConnection(HAService.this, sc);
                                        conn.start();
                                        HAService.this.addConnection(conn);
                                    } catch (Exception e) {
                                        log.error("new HAConnection exception", e);
                                        sc.close();
                                    }
                                }
                            } else {
                                log.warn("Unexpected ops in select " + k.readyOps());
                            }
                        }

                        selected.clear();
                    }
                } catch (Exception e) {
                    log.error(this.getServiceName() + " service has exception.", e);
                }
            }

            log.info(this.getServiceName() + " service end");
        }
```
当线程没有被终止的时候，一直执行select操作，如果有需要accpet的连接，则创建HAConnection对象并添加引用到连接集合里。注意在这里select(timeout)操作是为了能及时感知线程被终止，个人感觉做成select() + wakeup(或者线程interrupt)更合理一些，避免空转。

### 2.2.3 GroupTransfer服务

在写入流程的时候我们看到，请求被加入到GroupTransferService以后，生产线程就一直在等待被唤醒或者超时。可见如果复制完成，是由GroupTransferService去通知生产线程请求返回的。GroupTransferService的类结构如下图所示。

![GroupTransferService](http://cyblog.oss-cn-hangzhou.aliyuncs.com/mqReplication/GroupTransferService.png)

此类设计的成员变量有三个：

* notifyTransferObject，同步信号量，用于线程间协作。
* requestsWrite && requestsRead ，链表实现的双缓冲区，避免入队/出队的同步开销

主要方法有以下几个：

##### run

主要包括waitForRunning和doWaitTransfer两个方法。前者在没有请求的时候，会await()让出cpu。下面看下doWaitTransfer的实现。

##### doWaitTransfer

doWaitTransfer主要是判断队列中的请求能否进行响应。其先遍历队列中元素，对每一个GroupCommitRequest判断其对应的消息写入位置有没有被从节点成功复制（从节点成功复制则更新push2SlaveMaxOffset）。如果尚未完成则循环判断5次，每次失败等待1s（对应5s超时间隔）。最后则将复制的结果（成功或者超时）填入GroupCommitRequest，并唤醒生产线程。

```java
    private void doWaitTransfer() {
            synchronized (this.requestsRead) {
                if (!this.requestsRead.isEmpty()) {
                    for (CommitLog.GroupCommitRequest req : this.requestsRead) {
                        // 判断复制是否完成
                        boolean transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
                        for (int i = 0; !transferOK && i < 5; i++) {
                            // 如果首条都没有完成复制，则重试5次（此处可能导致生产线程已经返回，这边却还在判断）
                            this.notifyTransferObject.waitForRunning(1000);
                            transferOK = HAService.this.push2SlaveMaxOffset.get() >= req.getNextOffset();
                        }

                        if (!transferOK) {
                            log.warn("transfer messsage to slave timeout, " + req.getNextOffset());
                        }

                        // 释放请求
                        req.wakeupCustomer(transferOK);
                    }

                    this.requestsRead.clear();
                }
            }
        }
```

这里我认为存在两个缺陷：
	
* 超时作为一个可配置的参数，在这里进行判断的时候，被写死认为是5s
* 按这里的逻辑GroupCommitRequest的失败只能是因为超时，其它错误（如网络断开、slave磁盘满）无法反应到GroupCommitRequest中，导致上层逻辑及客户端不能知道错误原因。
* 由上面两个问题引申，一旦发生其它错误不能被立即发现并回复响应，只能等到超时。

##### putRequest

```java
public synchronized void putRequest(final CommitLog.GroupCommitRequest request) {
            synchronized (this.requestsWrite) {
                this.requestsWrite.add(request);
            }
            if (hasNotified.compareAndSet(false, true)) {
                waitPoint.countDown(); // notify
            }
        }
```

添加请求到当前使用的缓冲区，并且通知线程开始执行。hasNotified 在有请求时被改成true并立即唤醒线程，线程起来以后重新将其置为false表示等待下次唤醒，并且执行waitEnd做等待期间的收尾工作，然后执行实际的业务。
注意hasNotified置为false要先于onWaitEnd，这样如果在putRequest方法的add和hasNotified.compareAndSet(false,true)中间执行了hasNotified.(true,false）和swapRequests(),也只是触发了一次空转。

##### swapRequests

交换缓冲区

### 2.2.4 HAClient

HAClient只有当角色为slave时，才会启动此模块。类结构如下图所示：

![HAClient](http://cyblog.oss-cn-hangzhou.aliyuncs.com/mqReplication/HAClient.png)

| 变量名/方法名        | 作用       |
|:----------:|:-------------:|
// TODO

##### run

线程核心方法，主要功能是处理读事件，以及上报已经复制的位置，代码及关键部分注释如下：

```java
 public void run() {
            log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                try {
                    if (this.connectMaster()) {
                        // 和master有连接
                        if (this.isTimeToReportOffset()) {
                            // 满足时间汇报已复制的offset
                            boolean result = this.reportSlaveMaxOffset(this.currentReportedOffset);
                            if (!result) {
                                // 写失败断开连接
                                this.closeMaster();
                            }
                        }

                        this.selector.select(1000);

                        // 等待读事件，有数据则试图处理（未必是完整的数据包）
                        boolean ok = this.processReadEvent();
                        if (!ok) {
                            // 发生异常则断开连接
                            this.closeMaster();
                        }

                        // 如果需要，向master上报新写成功的位置
                        if (!reportSlaveMaxOffsetPlus()) {
                            continue;
                        }

                        // 计算读空闲，如果大于配置的值则关闭连接
                        long interval =
                            HAService.this.getDefaultMessageStore().getSystemClock().now()
                                - this.lastWriteTimestamp;
                        if (interval > HAService.this.getDefaultMessageStore().getMessageStoreConfig()
                            .getHaHousekeepingInterval()) {
                            log.warn("HAClient, housekeeping, found this connection[" + this.masterAddress
                                + "] expired, " + interval);
                            this.closeMaster();
                            log.warn("HAClient, master not response some time, so close connection");
                        }
                    } else {
                        this.waitForRunning(1000 * 5);
                    }
                } catch (Exception e) {
                    log.warn(this.getServiceName() + " service has exception. ", e);
                    this.waitForRunning(1000 * 5);
                }
            }

            log.info(this.getServiceName() + " service end");
        }
```

在一个循环里，先是检查是否和master连接，如果有连接，则处理读事件。在处理完读事件以后，再一次检测是否需要向master汇报写入进度。最后再做一次应用层面的心跳超时检测判断是否连接有问题。
这里存在以下问题：

* 对于数据量大的请求，由于业务层面的一个包可能被tcp拆成多个包发送，每个子包都会触发读事件不可避免（假设read的时候下一个包还没来）。但是由于每个循环里带上了其它逻辑，导致其它逻辑（例如是否需要汇报数据）也被循环执行，浪费了一定cpu资源。
* 可以改成每个包实际写body成功再进行一次触发。在dispatchReadRequest中也有类似代码，那么此处就无需再做判断。

#### 处理读事件

```java
private boolean processReadEvent() {
            int readSizeZeroTimes = 0;
            while (this.byteBufferRead.hasRemaining()) {
                // 还有可读的字节
                try {
                    int readSize = this.socketChannel.read(this.byteBufferRead);
                    if (readSize > 0) {
                        // 更新时间戳，长久收不到读事件会触发closeMaster
                        lastWriteTimestamp = HAService.this.defaultMessageStore.getSystemClock().now();
                        readSizeZeroTimes = 0;
                        // 派发读请求
                        boolean result = this.dispatchReadRequest();
                        if (!result) {
                            log.error("HAClient, dispatchReadRequest error");
                            return false;
                        }
                    } else if (readSize == 0) {
                        if (++readSizeZeroTimes >= 3) {
                            break;
                        }
                    } else {
                        log.info("HAClient, processReadEvent read socket < 0");
                        return false;
                    }
                } catch (IOException e) {
                    log.info("HAClient, processReadEvent read socket exception", e);
                    return false;
                }
            }

            return true;
        }

        private boolean dispatchReadRequest() {
            final int msgHeaderSize = 8 + 4; // phyoffset + size
            int readSocketPos = this.byteBufferRead.position();

            while (true) {
                int diff = this.byteBufferRead.position() - this.dispatchPostion;
                if (diff >= msgHeaderSize) {
                    // 可读的字节数至少要大于协议头，才尝试处理
                    long masterPhyOffset = this.byteBufferRead.getLong(this.dispatchPostion);
                    int bodySize = this.byteBufferRead.getInt(this.dispatchPostion + 8);

                    long slavePhyOffset = HAService.this.defaultMessageStore.getMaxPhyOffset();
                    // 如果master push的字节位置与当前写入位置不相等，则发生异常
                    if (slavePhyOffset != 0) {
                        if (slavePhyOffset != masterPhyOffset) {
                            log.error("master pushed offset not equal the max phy offset in slave, SLAVE: "
                                + slavePhyOffset + " MASTER: " + masterPhyOffset);
                            return false;
                        }
                    }

                    if (diff >= (msgHeaderSize + bodySize)) {
                        // 有body处理body，半包消息忽略

                        byte[] bodyData = new byte[bodySize];
                        this.byteBufferRead.position(this.dispatchPostion + msgHeaderSize);
                        this.byteBufferRead.get(bodyData);

                        HAService.this.defaultMessageStore.appendToCommitLog(masterPhyOffset, bodyData);

                        this.byteBufferRead.position(readSocketPos);
                        this.dispatchPostion += msgHeaderSize + bodySize;

                        // 完整的处理了一次复制包后尝试汇报当前进度
                        if (!reportSlaveMaxOffsetPlus()) {
                            return false;
                        }

                        continue;
                    }
                }

                if (!this.byteBufferRead.hasRemaining()) {
                    this.reallocateByteBuffer();
                }

                break;
            }

            return true;
        }
```

以上代码对每一次读事件被触发，都试图去解析整个包。如果满足协议，则反序列化整个包后进行刷盘操作，然后汇报当前刷盘进度。如果数据量不足，则返回等待后续数据。

##### reallocateByteBuffer && swapByteBuffer


//TODO 暂时不明

### 2.2.5 HAConnection

HAConnection是master用来管理主从复制的服务，其主要类说明如下：

| 变量/方法        | 作用       |
|:----------:|:-------------:|
|haService|提供master和slave之间的复制接口|
|socketChannel|实际传输数据的管道|
|writeSocketService|读取写入位置，主动将数据push给slave|
|readSocketService|处理slave汇报的复制数据|
|slaveRequestOffset|记录slave连接上来的初始请求位置|
|slaveAckOffset|记录slave汇报的ack位置|

### 2.2.6 ReadSocketService

这个服务主要用于处理slave上报的复制位置的请求，以及做业务层面的心跳检测。run方法很简单不做分析，我们来看处理读事件的方法。

````java
 private boolean processReadEvent() {
            int readSizeZeroTimes = 0;

            // 没有剩余字节，清理已读部分
            if (!this.byteBufferRead.hasRemaining()) {
                this.byteBufferRead.flip();
                this.processPostion = 0;
            }

            while (this.byteBufferRead.hasRemaining()) {
                try {
                    // 读取数据至byteBufferRead
                    int readSize = this.socketChannel.read(this.byteBufferRead);
                    if (readSize > 0) {
                        readSizeZeroTimes = 0;
                        this.lastReadTimestamp = HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now();
                        if ((this.byteBufferRead.position() - this.processPostion) >= 8) {
                            // 读优化，多个offset直接合并，只读最后一个完成的offset
                            int pos = this.byteBufferRead.position() - (this.byteBufferRead.position() % 8);
                            long readOffset = this.byteBufferRead.getLong(pos - 8);
                            this.processPostion = pos;

                            // 更新slaveAckOffset
                            HAConnection.this.slaveAckOffset = readOffset;
                            if (HAConnection.this.slaveRequestOffset < 0) {
                                HAConnection.this.slaveRequestOffset = readOffset;
                                log.info("slave[" + HAConnection.this.clientAddr + "] request offset " + readOffset);
                            }

                            // 更新groupTransferService的push2SlaveMaxOffset
                            HAConnection.this.haService.notifyTransferSome(HAConnection.this.slaveAckOffset);
                        }
                    } else if (readSize == 0) {
                        if (++readSizeZeroTimes >= 3) {
                            break;
                        }
                    } else {
                        log.error("read socket[" + HAConnection.this.clientAddr + "] < 0");
                        return false;
                    }
                } catch (IOException e) {
                    log.error("processReadEvent exception", e);
                    return false;
                }
            }

            return true;
        }
    }
````

逻辑见代码注释，需要注意的是这里对读offset做了优化，只读了最后一个合法的offset。但是存在2个优化点：

* 是否可以在发送端先做一次优化，即发送端有多个offset的时候直接发送最后一个？（当然在接收端也可以保留读取优化，兼容发送端连续发送）
* buffer.flip（）的触发条件是没有剩余未读字节，如果buffer的size不是消息包大小的倍数（目前分别是1M和8字节），可能导致buffer的指针一直没法更新。

### 2.2.7 writeSocketService


writeSocketService的主干方法就是不断的检测有没有可写的数据，并且不断的进行写出。这里要注意如果从节点没有数据的话，只会从当前写入位置所在的文件开始复制，而不会复制历史日志。

```java
 public void run() {
            HAConnection.log.info(this.getServiceName() + " service started");

            while (!this.isStopped()) {
                try {
                    // 由于在构造函数里注册了对读事件感兴趣，所以除非写缓冲区满，否则会立即返回
                    this.selector.select(1000);

                    // slave第一个请求都没来，等待slave更新初始位置
                    if (-1 == HAConnection.this.slaveRequestOffset) {
                        Thread.sleep(10);
                        continue;
                    }

                    if (-1 == this.nextTransferFromWhere) {
                        if (0 == HAConnection.this.slaveRequestOffset) {
                            // slaveRequestOffset非-1表示已经同步过数据，进入这里说明slave中没有数据
                            long masterOffset = HAConnection.this.haService.getDefaultMessageStore().getCommitLog().getMaxOffset();
                            masterOffset =
                                masterOffset
                                    - (masterOffset % HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                                    .getMapedFileSizeCommitLog());

                            if (masterOffset < 0) {
                                masterOffset = 0;
                            }
                            // 复制位置设置为当前文件的起始位置
                            this.nextTransferFromWhere = masterOffset;
                        } else {
                            // 曾经同步过数据
                            this.nextTransferFromWhere = HAConnection.this.slaveRequestOffset;
                        }

                        log.info("master transfer data from " + this.nextTransferFromWhere + " to slave[" + HAConnection.this.clientAddr
                            + "], and slave request " + HAConnection.this.slaveRequestOffset);
                    }

                    // 是否所有的数据都已经写完
                    if (this.lastWriteOver) {
                        // 没有数据可写，这里只是为了触发心跳
                        long interval =
                            HAConnection.this.haService.getDefaultMessageStore().getSystemClock().now() - this.lastWriteTimestamp;

                        if (interval > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig()
                            .getHaSendHeartbeatInterval()) {

                            // Build Header
                            this.byteBufferHeader.position(0);
                            this.byteBufferHeader.limit(headerSize);
                            this.byteBufferHeader.putLong(this.nextTransferFromWhere);
                            // 触发心跳的包body长度为0
                            this.byteBufferHeader.putInt(0);
                            this.byteBufferHeader.flip();

                            this.lastWriteOver = this.transferData();
                            if (!this.lastWriteOver)
                                continue;
                        }
                    } else {
                        // 已经读取的数据没写完，持续不断的写
                        this.lastWriteOver = this.transferData();
                        if (!this.lastWriteOver)
                            continue;
                    }

                    SelectMappedBufferResult selectResult =
                        HAConnection.this.haService.getDefaultMessageStore().getCommitLogData(this.nextTransferFromWhere);
                    if (selectResult != null) {
                        // 有未写出去的数据则写出
                        int size = selectResult.getSize();
                        if (size > HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize()) {
                            size = HAConnection.this.haService.getDefaultMessageStore().getMessageStoreConfig().getHaTransferBatchSize();
                        }

                        long thisOffset = this.nextTransferFromWhere;
                        this.nextTransferFromWhere += size;

                        selectResult.getByteBuffer().limit(size);
                        this.selectMappedBufferResult = selectResult;

                        // Build Header
                        this.byteBufferHeader.position(0);
                        this.byteBufferHeader.limit(headerSize);
                        this.byteBufferHeader.putLong(thisOffset);
                        this.byteBufferHeader.putInt(size);
                        this.byteBufferHeader.flip();

                        this.lastWriteOver = this.transferData();
                    } else {
                        // 没有数据的话等待
                        HAConnection.this.haService.getWaitNotifyObject().allWaitForRunning(100);
                    }
                } catch (Exception e) {

                    HAConnection.log.error(this.getServiceName() + " service has exception.", e);
                    break;
                }
            }

            if (this.selectMappedBufferResult != null) {
                this.selectMappedBufferResult.release();
            }

            this.makeStop();

            readSocketService.makeStop();

            haService.removeConnection(HAConnection.this);

            SelectionKey sk = this.socketChannel.keyFor(this.selector);
            if (sk != null) {
                sk.cancel();
            }

            try {
                this.selector.close();
                this.socketChannel.close();
            } catch (IOException e) {
                HAConnection.log.error("", e);
            }

            HAConnection.log.info(this.getServiceName() + " service end");
        }
```

存在的问题：

* 注册对写事件感兴趣，如果业务里不进行挂起操作，接近于cpu空转。
* slave每次启动后会等到第一次触发心跳5s才汇报当前复制进度，在此期间master一直无法提供服务。
* 为了能维持心跳，线程需要一直不停的在await(）一定时间后恢复，浪费了通知机制。
* 如果slave断开连接时间比较长，master已经删除了slave未同步数据，master获取slave复制位置的selectResult返回null，将会导致一直无法复制。（当然slave如果启动时自动清理过期数据可以避免这个问题）

设计优势与借鉴：

* nextTransferFromWhere为写线程单独维护，不依赖读线程中slave汇报的位置。这样主动push模式对于跨机房的传输较为友好，节省延迟。
* 判断了（如写缓冲区满）多次写入失败的情况，避免线程一直尝试写造成cpu使用率太高。我们实现的时候如果采用netty会在网络缓冲上层用一个无界的堆外内存链表，容易打爆内存，需要针对这种情况做好流控或者异常处理。


# 3 问题

* selector作用？
* 异步模式，master写盘未成功宕机，复制成功，导致slave存在master没有的消息
