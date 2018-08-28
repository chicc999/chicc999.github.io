---
title: FMQ之客户端血泪升级史
date: 2018-07-24 15:02:53
tags:
- 中间件
- 消息队列
- failover
categories: 编程
comments: true
---

本文介绍了FMQ的生产者自动故障切换的原理，以及在生产环境中遇到的问题及改进方案。从自动容灾、重试、重连、心跳检测等多方面，结合生产环境中实际遇到的问题，介绍了升级的过程。
<!--more-->

# 1 failover方案

FMQ对于一组broker采用master-slave结构，slave是对master的Full Backup。在正常情况下，master处于active状态，slave处于standby状态。当主从其中一台宕机时，另一台切换为active状态。但由于slave只有写权限，所以对于后续的消息写入操作，此组broker不再提供写入。这样对于写入操作，可用性就很差。为了解决这个问题，我们会给主题配备多组broker（一般会跨机房），并在故障时通过权重控制，将流量打到无故障的broker上，本节对此做简要介绍。

## 1.1 基于权重的容灾

### 1.1.1 路由接口

在客户端进行路由选择时，FMQ设计上提供了如下接口

```java
public interface LoadBalance {
    /**
     * 选取broker
     *
     * @param transports    分组列表
     * @param errTransports 发送异常的分组
     * @param message       消息
     * @param dataCenter    当前客户端所在数据中心
     * @param config        主题配置
     * @return 指定的分组
     */
    <T extends ElectTransport> T electTransport(List<T> transports, List<T> errTransports, Message message, byte dataCenter, TopicConfig config) throws
            FMQException;

    /**
     * 选择queueId 0表示随机
     *
     * @param transport  当前发送分组
     * @param message    消息 如果发送多条消息，取的第一条消息
     * @param dataCenter 当前客户端所在数据中心
     * @param config     主题配置
     * @return
     * @throws FMQException
     */
    <T extends ElectTransport> short electQueueId(T transport, Message message, byte dataCenter, TopicConfig config) throws 	FMQException;
}
```

electTransport方法的返回值是选择的broker对应的transport，electQueueId的返回值则表示你要将消息生产到哪个队列中。入参提供了消息实体、连接信息、数据中心信息和主题配置，以方便用户可以实现此接口，根据入参自行制定路由策略。

### 1.1.2 轮盘赌（Roulette）算法

借鉴遗传算法中选择下一代的策略，我们实现了根据权重来选择broker的轮盘赌算法。假设一个topic分配了N个broker分组，分组i 的权重为𝒘(𝒙_{_𝒊}),则此producer选择分组i的概率为

x^2

### 1.1.3 影响权重的因素


### 1.1.4 故障切换

## 1.2 重试

# 2 超时？超时！

## 2.1 原因分析

## 2.2 解决方案


# 3 翻车了？再升级

## 3.1 线上故障描述

在做了以上容灾以后，生产环境还是在某一天出现了问题，并且完全恢复用了15分40秒。

* 上午10:56:05，线上硬件故障导致broker不可用，生产成功率从100%掉到93%左右，于11:12:00成功率恢复到100%。
* 故障期间，客户端对于出问题的连接，一直没有断开以及后续的重连操作；同时一直有流量打到此broker，导致成功率下降。
* 在无人操作的情况下，生产环境于15分40秒检测到连接异常，开始尝试重连，同时更新路由表不再向此机器有流量，成功率恢复。
* 故障期间连接状态为ESTABLISHED。

## 3.2 故障排查

首先由于日志显示未进行重连，连接也没有异常;tcp状态显示连接为建立状态，由此可以推断应用层并没有感知到对端宕机。在此次故障中，主要有以下几个问题需要解决。

1. 对端宕机为何tcp没有发现连接异常，还处于ESTABLISHED状态？
2. 之前也有做过宕机以及网络物理断开（拔网线）的测试，为何没有出现长时间的成功率下降？
3. 应用层最终是如何感知到连接故障并恢复的？
4. 为何应用层写数据没有出现错误？
5. 为何应用层的心跳没有起到应有的作用？

带着以上疑惑，我进行了问题的排查，并成功复现线上场景，解答了所有疑惑。

### 3.2.1 复现

首先尝试复现生产环境的问题。

* 对于远端物理机，shutdown -h now 以及拔网线均能复现场景。
* 对于本地物理机，shutdown -h now 的情况下无法复现；物理断开连接可以复现。

通过tcpdump以及用wireshark进行分析发现，本地 shutdown -h now 的情况下，会收到FIN，自然应用层能感知到。而复杂网络跨机房的情况下，远端机器宕机的情况下客户端并没有收到FIN。

同时抓包发现，当服务器宕机或者网络连接被断开，客户端进入tcp的超时重传模式，如下图。

![超时重传](http://ovor60v7j.bkt.clouddn.com/%E8%B6%85%E6%97%B6%E9%87%8D%E4%BC%A0.png)

在超时重传过程中，连接始终处于ESTABLISHED。查看本机的tcp超时重传次数设置，在/proc/sys/net/ipv4下找到tcp_retries2文件打开：

> 15

可以看到超时重传的最大重试次数为15。
查询生产环境内核版本为2.6.32，参考net/ipv4/tcp_input.c中的tcp_rtt_estimator和tcp_set_rto,RTO最大最小值如下

>  #define TCP_RTO_MAX     ((unsigned)(120*HZ))  
>  #define TCP_RTO_MIN     ((unsigned)(HZ/5))

由于RTO采用指数回退，15次重试可以计算出超时重试间隔分别为

> 0.2 0.6 1.4 3.0 6.2 12.7 25.5 51.2 102.6 120 120 120 120 120 120 

通过抓包与理论计算，恢复时间约为15分钟23秒。由此可知最终恢复是因为tcp超时重传达到最大重试次数以后，重置连接，应用层接收到网络异常的信号后，由上述介绍的自动容灾模块做了故障切换。  
为什么在之前的测试中，能很快恢复呢？  
对本地测试机做了抓包分析。原来是本地用的OSX和WINDOWS系统，最大超时重试次数都很少，所以很快就能发现连接的异常。
由此，3.2节中五个问题的1-3已经有了解答，下面来看剩下的2个疑问。

### 3.2.2 真假难辨 write & flush

我们可以看 AbstractChannelHandlerContext 的 writeAndFlush 方法，分别进行了 write 和 flush 。

#### 3.2.2.1 write

我们先看 write 阶段，调用私有的 write 方法如下：

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
        AbstractChannelHandlerContext next = findContextOutbound();
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeWrite(msg, promise);
            if (flush) {
                next.invokeFlush();
            }
        } else {
            AbstractWriteTask task;
            if (flush) {
                task = WriteAndFlushTask.newInstance(next, msg, promise);
            }  else {
                task = WriteTask.newInstance(next, msg, promise);
            }
            safeExecute(executor, task, promise, msg);
        }
    }
    
```
如果当前线程是IO线程（inEventLoop）,则直接执行，否则包装成task扔到IO线程的任务队列中。我们继续看 write 部分：

```java
 private void invokeWrite(Object msg, ChannelPromise promise) {
        try {
            ((ChannelOutboundHandler) handler()).write(this, msg, promise);
        } catch (Throwable t) {
            notifyOutboundHandlerException(t, promise);
        }
    }
```
线程会从责任链的tailHandler开始，依次经过业务 handler ，调用 write 方法，直到 headHandler。我们忽略掉业务类型的 handler 直接往下走：

```java
final class DefaultChannelPipeline implements ChannelPipeline {

……

public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            unsafe.write(msg, promise);
        }
……

}


public abstract class AbstractChannel extends DefaultAttributeMap implements Channel {

……

        public final void write(Object msg, ChannelPromise promise) {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                // 如果outboundBuffer为null说明channel已经关闭，需要设置 promise失败
                // 如果不为null的话会在flush0()进行处理
                // See https://github.com/netty/netty/issues/2362
                safeSetFailure(promise, CLOSED_CHANNEL_EXCEPTION);
                // 释放资源
                ReferenceCountUtil.release(msg);
                return;
            }

            int size;
            try {
                msg = filterOutboundMessage(msg);
                size = estimatorHandle().size(msg);
                if (size < 0) {
                    size = 0;
                }
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                ReferenceCountUtil.release(msg);
                return;
            }

            outboundBuffer.addMessage(msg, size, promise);
        }
        
        ……
        
}
```

由此可见write出去的数据实际被投递到了outboundBuffer中，我们继续跟进addMessage方法

```java
 /**
     * Add given message to this {@link ChannelOutboundBuffer}. The given {@link ChannelPromise} will be notified once
     * the message was written.
     */
    public void addMessage(Object msg, int size, ChannelPromise promise) {
        Entry entry = Entry.newInstance(msg, size, total(msg), promise);
        if (tailEntry == null) {
            flushedEntry = null;
            tailEntry = entry;
        } else {
            Entry tail = tailEntry;
            tail.next = entry;
            tailEntry = entry;
        }
        if (unflushedEntry == null) {
            unflushedEntry = entry;
        }

        // increment pending bytes after adding message to the unflushed arrays.
        // See https://github.com/netty/netty/issues/1619
        incrementPendingOutboundBytes(size, false);
    }
```

ChannelOutboundBuffer内部维护了一个Entry链表，将msg封装成Entry里放入链表。其中tailEntry 指向链表尾部，flushedEntry 指向链表待执行flushed的Entry，unflushedEntry指向下一个待标记为不可取消的Entry。此链表变化结构如下：

![Entry链表](http://ovor60v7j.bkt.clouddn.com/blog/FMQ%E4%B9%8B%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%A1%80%E6%B3%AA%E5%8D%87%E7%BA%A7%E5%8F%B2/EntryList.png)

如果一次后续的flush能将所有Entry都写出去，则链表恢复初始状态（无数据）；如果只写出了一部分Entry，就又进行了addMessage操作，那么指针情况如下：

![Entry链表](http://ovor60v7j.bkt.clouddn.com/blog/FMQ%E4%B9%8B%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%A1%80%E6%B3%AA%E5%8D%87%E7%BA%A7%E5%8F%B2/addMessage.png)


由此可见，msg以及promise最终被包装成了entry，添加到了未flush的的链表里，而没有真正写出去。
同时注意由于没有对链表长度进行控制，是否意味着这是一个无界链表？如果因为某些原因一直没有flush会不会引起内存的持续增加？incrementPendingOutboundBytes方法看上去是进行流控，是否能生效呢？我们后续再考虑这些问题，接着看 HeadContext 的 flush 方法。

#### 3.2.2.2 flush


```java
public final void flush() {
            assertEventLoop();

            ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null) {
                return;
            }

            outboundBuffer.addFlush();
            flush0();
        }
```

flush() 分别执行了 outboundBuffer的 addFlush 方法和自身的 flush0 方法。其中addFlush()方法主要是将已经被取消发送的msg资源进行释放，以及设置promise使得不能够再被取消（快要真正写出了），代码如下：

```java
 /**
     * Add a flush to this {@link ChannelOutboundBuffer}. This means all previous added messages are marked as flushed
     * and so you will be able to handle them.
     */
    public void addFlush() {
        // 后续逻辑主要是把entry标记为不可取消的，但是一旦被标记，就没有必要在后续此方法被调用的时候再进行重复检测
        // 所以在标记成功以后，将 unflushedEntry 标记为null，并在这里进行检测以避免重复标记
        // See https://github.com/netty/netty/issues/2577
        Entry entry = unflushedEntry;
        if (entry != null) {
            if (flushedEntry == null) {
                // 如果flushedEntry为null说明队列已经清空，所有前序都已被写出，flushed指针指到unflushedEntry（头节点），后续此Entry到tailEntry会被标记为不可取消
                flushedEntry = entry;
            }
            do {
                flushed ++;
                if (!entry.promise.setUncancellable()) {
                    // Was cancelled so make sure we free up memory and notify about the freed bytes
                    int pending = entry.cancel();
                    decrementPendingOutboundBytes(pending, false, true);
                }
                entry = entry.next;
            } while (entry != null);

            // 所有entry都被标记了，重置unflushedEntry
            unflushedEntry = null;
        }
    }
```

继续看flush0（）方法

```

        protected void flush0() {
            if (inFlush0) {
                // 避免重入,由于netty4的线程模型，此时应该只会由IO线程单线程执行，不明白为何会有这个判断
                return;
            }

            final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
            if (outboundBuffer == null || outboundBuffer.isEmpty()) {
                return;
            }

            inFlush0 = true;

            // 连接不活跃时，给future设置失败标志及原因
            if (!isActive()) {
                try {
                    if (isOpen()) {
                        outboundBuffer.failFlushed(NOT_YET_CONNECTED_EXCEPTION, true);
                    } else {
                    
                        outboundBuffer.failFlushed(CLOSED_CHANNEL_EXCEPTION, false);
                    }
                } finally {
                    inFlush0 = false;
                }
                return;
            }

            try {
                doWrite(outboundBuffer);
            } catch (Throwable t) {
                if (t instanceof IOException && config().isAutoClose()) {
                    /**
                     * Just call {@link #close(ChannelPromise, Throwable, boolean)} here which will take care of
                     * failing all flushed messages and also ensure the actual close of the underlying transport
                     * will happen before the promises are notified.
                     *
                     * This is needed as otherwise {@link #isActive()} , {@link #isOpen()} and {@link #isWritable()}
                     * may still return {@code true} even if the channel should be closed as result of the exception.
                     */
                    close(voidPromise(), t, false);
                } else {
                    outboundBuffer.failFlushed(t, true);
                }
            } finally {
                inFlush0 = false;
            }
        }
```

用inFlush0来避免重入不知道有何意义，按照netty4的线程模型，此处只有channel绑定的IO线程会去执行，有理解的可以探讨下。如果发生了异常，则对异常进行处理并且promise置为失败，然后调用promise中的回调。正常的话进行真正的写入，我们来看下 doWrite 方法，以 NioSocketChannel 的实现为例。

```java
    @Override
    protected void doWrite(ChannelOutboundBuffer in) throws Exception {
        for (;;) {
            int size = in.size();
            if (size == 0) {
                // 所有数据都已经写出，清理interestOps中的写标志位
                clearOpWrite();
                break;
            }
            long writtenBytes = 0;
            boolean done = false;
            boolean setOpWrite = false;

            // ChannelOutboundBuffer中的Entry转换成buffer数组
            ByteBuffer[] nioBuffers = in.nioBuffers();
            // Entry数量
            int nioBufferCnt = in.nioBufferCount();
            // ChannelOutboundBuffer中待写的字节数
            long expectedWrittenBytes = in.nioBufferSize();
            SocketChannel ch = javaChannel();

            // Always us nioBuffers() to workaround data-corruption.
            // See https://github.com/netty/netty/issues/2761
            switch (nioBufferCnt) {
                case 0:
                    // We have something else beside ByteBuffers to write so fallback to normal writes.
                    super.doWrite(in);
                    return;
                case 1:
                    // Only one ByteBuf so use non-gathering write
                    ByteBuffer nioBuffer = nioBuffers[0];
                    for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                        final int localWrittenBytes = ch.write(nioBuffer);
                        if (localWrittenBytes == 0) {
                            setOpWrite = true;
                            break;
                        }
                        expectedWrittenBytes -= localWrittenBytes;
                        writtenBytes += localWrittenBytes;
                        if (expectedWrittenBytes == 0) {
                            done = true;
                            break;
                        }
                    }
                    break;
                default:
                    for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                        final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt);
                        if (localWrittenBytes == 0) {
                            setOpWrite = true;
                            break;
                        }
                        expectedWrittenBytes -= localWrittenBytes;
                        writtenBytes += localWrittenBytes;
                        if (expectedWrittenBytes == 0) {
                            done = true;
                            break;
                        }
                    }
                    break;
            }

            // Release the fully written buffers, and update the indexes of the partially written buffer.
            in.removeBytes(writtenBytes);

            if (!done) {
                // Did not write all buffers completely.
                incompleteWrite(setOpWrite);
                break;
            }
        }
    }
```

可以看到 doWrite 方法中，先将 ChannelOutboundBuffer 中的Entry写入网络，总共分为三种情况：

* 写入网络全部成功，done==true,setOpWrite==false
* 写入网络次数达到了SpinCount,跳出循环,done==flase,setOpWrite==false
* 写的过程中网络缓冲区已经满了，返回了0字节,done==false,setOpWrite==true

无论哪种情况，都需要执行 in.removeBytes 更改已经写出去的 Entry 链表，并执行回调。我们先看此方法及其调用链：

```java
 public void removeBytes(long writtenBytes) {
        for (;;) {
            Object msg = current();
            if (!(msg instanceof ByteBuf)) {
                assert writtenBytes == 0;
                break;
            }

            final ByteBuf buf = (ByteBuf) msg;
            final int readerIndex = buf.readerIndex();
            final int readableBytes = buf.writerIndex() - readerIndex;

            if (readableBytes <= writtenBytes) {
                if (writtenBytes != 0) {
                    progress(readableBytes);
                    writtenBytes -= readableBytes;
                }
                remove();
            } else { // readableBytes > writtenBytes
                if (writtenBytes != 0) {
                    buf.readerIndex(readerIndex + (int) writtenBytes);
                    progress(writtenBytes);
                }
                break;
            }
        }
        clearNioBuffers();
    }
    
     /**
     * 处理写出进度
     */
    public void progress(long amount) {
        Entry e = flushedEntry;
        assert e != null;
        ChannelPromise p = e.promise;
        if (p instanceof ChannelProgressivePromise) {
            long progress = e.progress + amount;
            e.progress = progress;
            ((ChannelProgressivePromise) p).tryProgress(progress, e.total);
        }
    }
    
    /**
     * 移除当前messgae，设置promise为true，调用回调
     */
    public boolean remove() {
        Entry e = flushedEntry;
        if (e == null) {
            clearNioBuffers();
            return false;
        }
        Object msg = e.msg;

        ChannelPromise promise = e.promise;
        int size = e.pendingSize;

        removeEntry(e);

        if (!e.cancelled) {
            // 释放资源，调用回调
            ReferenceCountUtil.safeRelease(msg);
            safeSuccess(promise);
            decrementPendingOutboundBytes(size, false, true);
        }

        // 循环使用Entry
        e.recycle();

        return true;
    }
    
    /**
     * 移除当前Entry
     */
        private void removeEntry(Entry e) {
        if (-- flushed == 0) {
            // processed everything
            flushedEntry = null;
            if (e == tailEntry) {
                tailEntry = null;
                unflushedEntry = null;
            }
        } else {
            flushedEntry = e.next;
        }
    }
```
由代码可以看出，removeBytes主要是对已经写成功（写入网络缓冲区）的字节数进行操作，主要分为以下两个操作：

* 更新写出进度progress，此方法一般用于大包拆成小包的同时对小包写出进度添加监听，本例中没有用，在 ChunkedWriteHandler 中会有使用，为每个小数据包注册一个listener。
* 对于已经完整写出的数据包
	* 移除Entry
	* 处理promise并且调用回调(safeSuccess(promise)）。

此过程无论是哪种情况都会被执行，因为后两种情况下也有可能有一部分Entry写成功，需要进行处理。如果完全写成功，写入算是已经完成了。如果还有未写成功的呢？我们继续看incompleteWrite方法。

```
protected final void incompleteWrite(boolean setOpWrite) {
        // Did not write completely.
        if (setOpWrite) {
            setOpWrite();
        } else {
            // Schedule flush again later so other tasks can be picked up in the meantime
            Runnable flushTask = this.flushTask;
            if (flushTask == null) {
                flushTask = this.flushTask = new Runnable() {
                    @Override
                    public void run() {
                        flush();
                    }
                };
            }
            eventLoop().execute(flushTask);
        }
    }
```

如果缓冲区已满（setOpWrite==true），则去注册对write感兴趣；如果是flush次数到了，则添加一个任务到task，后续再执行，避免线程因为一直flush而耽误其他任务。  

#### 3.2.2.3 结论

由上述分析，问题4的答案也很明显了。当网络缓冲区还没满时，消息可以正常写出，并且回调也被调用；当网络缓冲区满后，消息会堆积在ChannelOutboundBuffer中。由于系统不会出现错误，只会包装成flush任务进行重试，所以应用层对于数据没有写出这件事一直没有感知。

### 3.3.3 形同摆设的心跳

现在我们还剩最后一个疑问，为什么心跳形同摆设，没有起作用呢？我们回看心跳代码，用的是netty自带的实现IdleStateHandler。首先来看此类的源码。

#### 3.3.3.1 IdleStateHandler

此类有READ、WRITE、ALL三种触发方式，分别对应读空闲、写空闲、读写空闲（无读且无写触发），实现几乎一样，我们以ALL为例。

```java
private final class AllIdleTimeoutTask implements Runnable {

        private final ChannelHandlerContext ctx;

        AllIdleTimeoutTask(ChannelHandlerContext ctx) {
            this.ctx = ctx;
        }

        @Override
        public void run() {
            if (!ctx.channel().isOpen()) {
                return;
            }

            long nextDelay = allIdleTimeNanos;
            if (!reading) {
                nextDelay -= System.nanoTime() - Math.max(lastReadTime, lastWriteTime);
            }
            if (nextDelay <= 0) {
                // 已经需要触发，下一次延迟allIdleTimeNanos以后调度
                allIdleTimeout = ctx.executor().schedule(
                        this, allIdleTimeNanos, TimeUnit.NANOSECONDS);
                try {
                    IdleStateEvent event;
                    if (firstAllIdleEvent) {
                        firstAllIdleEvent = false;
                        event = IdleStateEvent.FIRST_ALL_IDLE_STATE_EVENT;
                    } else {
                        event = IdleStateEvent.ALL_IDLE_STATE_EVENT;
                    }
                    channelIdle(ctx, event);
                } catch (Throwable t) {
                    ctx.fireExceptionCaught(t);
                }
            } else {
                // 尚未触发idle
                allIdleTimeout = ctx.executor().schedule(this, nextDelay, TimeUnit.NANOSECONDS);
            }
        }
    }
```

这个任务每次执行时，会判断是否需要触发idle事件。

* 如果nextDelay小于0说明已经大于设定的时间没有数据流入流出，立即触发idle事件，并且在设定的间隔后再次调度；
* 如果nextDelay大于0，说明还要经过nextDelay才到设定触发idle的时间。所以在nextDelay后再次进行判断。

```java
  @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        if (readerIdleTimeNanos > 0 || allIdleTimeNanos > 0) {
            lastReadTime = System.nanoTime();
            reading = false;
        }
        ctx.fireChannelReadComplete();
    }
    
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        // Allow writing with void promise if handler is only configured for read timeout events.
        if (writerIdleTimeNanos > 0 || allIdleTimeNanos > 0) {
            promise.addListener(writeListener);
        }
        ctx.write(msg, promise);
    }
    
     private final ChannelFutureListener writeListener = new ChannelFutureListener() {
        @Override
        public void operationComplete(ChannelFuture future) throws Exception {
            lastWriteTime = System.nanoTime();
            firstWriterIdleEvent = firstAllIdleEvent = true;
        }
    };
```
lastReadTime在每次channelReadComplete被调用的时候刷新，lastWriteTime则会在writeListener被回调的时候刷新。由我们之前write过程的分析，显然会在写进网络缓冲区以后被调用。

#### 3.3.3.2 Event.IDLE传递

我们继续看idle事件触发后是怎么传递到后续handler的。

```java
  protected void channelIdle(ChannelHandlerContext ctx, IdleStateEvent evt) throws Exception {
        ctx.fireUserEventTriggered(evt);
    }
    
   @Override
    public ChannelHandlerContext fireUserEventTriggered(final Object event) {
        if (event == null) {
            throw new NullPointerException("event");
        }

        final AbstractChannelHandlerContext next = findContextInbound();
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeUserEventTriggered(event);
        } else {
            executor.execute(new OneTimeTask() {
                @Override
                public void run() {
                    next.invokeUserEventTriggered(event);
                }
            });
        }
        return this;
    }
    
      private AbstractChannelHandlerContext findContextInbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;
        } while (!ctx.inbound);
        return ctx;
    }
```

channelIdle中调用了ChannelHandlerContext的fireUserEventTriggered方法，此方法会通过findContextInbound的实现，找到下一个InboundHandler,然后调用其userEventTriggered方法。我们再来看业务类中userEventTriggered的实现。

```
 @Override
        public void userEventTriggered(final ChannelHandlerContext ctx, final Object evt) throws Exception {
            if (evt instanceof IdleStateEvent) {
                IdleStateEvent event = (IdleStateEvent) evt;
                if (event.state().equals(IdleState.ALL_IDLE)) {
                    if (heartbeatHandler != null) {
                        heartbeatHandler.heartbeat(ctx);
                    }
                    eventManager.add(new NettyEvent(NettyEvent.NettyEventType.IDLE, ctx.channel()));
                }
            }
            super.userEventTriggered(ctx, evt);
        }
    }
    
 /**
	 * 默认心跳处理器
	 */
	protected static class DefaultHeartbeatHandler implements HeartbeatHandler {
		@Override
		public void heartbeat(final ChannelHandlerContext ctx) {
			// 心跳不用应答
			Header header = new Header(HeaderType.REQUEST, Command.HEARTBEAT);
			header.setAcknowledge(Acknowledge.ACK_NO);
			ctx.writeAndFlush(new Heartbeat(header)).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
		}
	}
```

很明显，在触发idle以后，业务类写出了一个约定的心跳包，并广播了此消息给其它组件。如果写发生异常，则关闭连接（ChannelFutureListener.CLOSE_ON_FAILURE）。其它组件是否处理此事件不在考虑范围内（依赖各个组件自己实现而非框架）。

#### 3.3.3.3 问题所在

分析到这里，问题已经显而易见。在tcp超时重传以后，刚开始网络缓冲区没有满，lastWriteTime则写入成功后刷新。当网络缓冲区满了以后，成功触发idle。由于idle只是向对方发一个心跳包，而不做其余处理，此时心跳包也被放在channelOutboundBuffer中不会抛出异常，所以连接状态一直正常，直至tcp超时重传达到最大值。

### 3.3.4 减少拷贝的小彩蛋

```java
  public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        ByteBuf buf = null;
        try {
            if (acceptOutboundMessage(msg)) {
                @SuppressWarnings("unchecked")
                I cast = (I) msg;
                buf = allocateBuffer(ctx, cast, preferDirect);
                try {
                    encode(ctx, cast, buf);
                } finally {
                    ReferenceCountUtil.release(cast);
                }

                if (buf.isReadable()) {
                    ctx.write(buf, promise);
                } else {
                    buf.release();
                    ctx.write(Unpooled.EMPTY_BUFFER, promise);
                }
                buf = null;
            } else {
                ctx.write(msg, promise);
            }
        } catch (EncoderException e) {
            throw e;
        } catch (Throwable e) {
            throw new EncoderException(e);
        } finally {
            if (buf != null) {
                buf.release();
            }
        }
    }
```

```java
    /**
     * Allocate a {@link ByteBuf} which will be used as argument of {@link #encode(ChannelHandlerContext, I, ByteBuf)}.
     * Sub-classes may override this method to returna {@link ByteBuf} with a perfect matching {@code initialCapacity}.
     */
    protected ByteBuf allocateBuffer(ChannelHandlerContext ctx, @SuppressWarnings("unused") I msg,
                               boolean preferDirect) throws Exception {
        if (preferDirect) {
            return ctx.alloc().ioBuffer();
        } else {
            return ctx.alloc().heapBuffer();
        }
    }
```

## 3.3 解决方案

### 3.3.1 验证结论

首先复现此问题，观察日志发现，第一次失败发生在 14:28:04 requestId=615 ,并且于 14:30:54 恢复，共持续170s。
同时wireshark进行抓包，发现最后一次RTO发生在故障出现144.9s后，并且于20s后触发RST，与日志恢复时间比较一致，如下图所示。

![RST](http://ovor60v7j.bkt.clouddn.com/blog/FMQ%E4%B9%8B%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%A1%80%E6%B3%AA%E5%8D%87%E7%BA%A7%E5%8F%B2/RST.png)

同时查看日志，14:28:37 requestID=724为最后一个立即调用写出成功回调的请求，14:28:04 开始到目前为止总共有请求110个。由于配置了2台机器，按概率平分，每台机器预计55个请求（数据量小可能会上下波动）。通过debug观察到每个包大小为136个字节，网络缓冲区配置为8K，60个请求会将缓冲区写满，从而应用层再得不到写入成功回调。

![putMessageSize](http://ovor60v7j.bkt.clouddn.com/blog/FMQ%E4%B9%8B%E5%AE%A2%E6%88%B7%E7%AB%AF%E8%A1%80%E6%B3%AA%E5%8D%87%E7%BA%A7%E5%8F%B2/putMessageSize.png)

以上结果与源码阅读完的推测现象完全一致。

### 3.3.2 解决

将读写idle进行分离，同时读idle设置默认值为70s，写idle默认值为30s。当触发读idle时，可以认为对端机器出问题或者网络问题（还有种可能是对端用的并非本框架），直接关闭连接。

```
public void userEventTriggered(final ChannelHandlerContext ctx, final Object evt) throws Exception {
			if (evt instanceof IdleStateEvent) {
				IdleStateEvent event = (IdleStateEvent) evt;
				if (event.state().equals(IdleState.WRITER_IDLE)) {
					if (heartbeatHandler != null) {
						heartbeatHandler.heartbeat(ctx);
					}
					logger.debug("Channel write idled: {}", ctx.channel());
					eventManager.add(new NettyEvent(NettyEvent.NettyEventType.WRITE_IDLE, ctx.channel()));
				}else if(event.state().equals(IdleState.READER_IDLE)){
					//读超时时间默认值大于写超时，读超时说明对端未发送心跳或者网络问题，断开连接
					logger.warn("Channel read idled: {} , channel will be closed", ctx.channel());
					ctx.channel().close();
					eventManager.add(new NettyEvent(NettyEvent.NettyEventType.READER_IDLE, ctx.channel()));
				}
			}
			super.userEventTriggered(ctx, evt);
		}
```

### 3.3.3 验证

>ERROR - 2018-08-14 15:32:09 [main]xxx.ApiManualProducer.sendOne(ApiManualProducer.java:155) -- 消息发送失败 error:1 time: 0

>DEBUG - 2018-08-14 15:33:13 [IoLoopGroup - 2]xxx.NettyTransport$ConnectionHandler.userEventTriggered(NettyTransport.java:845) -- Channel write idled: [id: 0x3437c1b8, L:/10.13.70.166:57296 - R:/10.9.3.59:50088]

>WARN  - 2018-08-14 15:33:18 [IoLoopGroup - 2]xxx.NettyTransport$ConnectionHandler.userEventTriggered(NettyTransport.java:849) -- Channel read idled: [id: 0x3437c1b8, L:/10.13.70.166:57296 - R:/10.9.3.59:50088] , channel will be closed

* 通过以上日志能看到，网络15:32:09出问题，并于15:33:18被检测到，经过了69s，符合设置预期。
* 写超时于15:33:13触发，由于设置的写超时是30s，可见15:32:43才没有写请求进网络缓冲区。
* 15:32:09 - 15:32:43 期间总共34s，由于单线程发送，且另一台正常服务器耗时可以忽略不计，共向问题机器发送请求60+，每个请求136字节，网络缓冲区8K。所有实际数据与源码分析结果吻合。