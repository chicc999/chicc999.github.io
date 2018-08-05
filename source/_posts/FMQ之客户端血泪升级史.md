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

## 1.1 基于权重的容灾

### 1.1.1 负载均衡 -- 轮盘赌算法

### 1.1.2 故障切换

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

* 1. 对端宕机为何tcp没有发现连接异常，还处于ESTABLISHED状态？
* 2. 之前也有做过宕机以及网络物理断开（拔网线）的测试，为何没有出现长时间的成功率下降？
* 3. 应用层最终是如何感知到连接故障并恢复的？
* 4. 为何应用层写数据没有出现错误？
* 5. 为何应用层的心跳没有起到应有的作用？

带着以上疑惑，我进行了问题的排查，并成功复现线上场景，解答了所有疑惑。

### 3.2.1 复现

首先尝试复现生产环境的问题。

* 对于远端物理机，shutdown -h now 以及拔网线均能复现场景。
* 对于本地物理机，shutdown -h now 的情况下无法复现；物理断开连接可以复现。

通过tcpdump以及用wireshark进行分析发现，本地 shutdown -h now 的情况下，会收到FIN，自然应用层能感知到。而复杂网络跨机房的情况下，远端机器宕机的情况下客户端并没有收到FIN。

同时抓包发现，当服务器宕机或者网络连接被断开，客户端进入tcp的超时重传模式，如下图。

[超时重传](http://ovor60v7j.bkt.clouddn.com/%E8%B6%85%E6%97%B6%E9%87%8D%E4%BC%A0.png)

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
对本地测试机做了抓包分析。原来是本地用的OSX和WINDOWS系统，最大超时重试次数都很少，所以分别在几秒以及十几秒就能发现连接的异常。
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

```
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

ChannelOutboundBuffer内部维护了一个Entry链表，将msg封装成Entry里放入链表。其中tailEntry 指向链表尾部，flushedEntry 指向链表待执行flushed的Entry，unflushedEntry指向下一个待标记为不可取消的Entry（详见3.2.2.2）。
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

分别执行了 outboundBuffer的 addFlush 方法和自身的 flush0 方法。其中addFlush()方法主要是将已经被取消发送的msg资源进行释放，以及设置promise使得不能够再被取消（快要真正写出了），代码如下：

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

用inFlush0来避免重入不知道有何意义，按照netty4的线程模型，此处只有channel绑定的IO线程会去执行，有理解的可以探讨下。如果发生了异常，则对异常进行处理并且promise置为失败，然后调用promise中的回调。正常的话进行真正的写入，我们来看下 write 方法，以 NioSocketChannel 的实现为例。

```
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

            // Ensure the pending writes are made of ByteBufs only.
            ByteBuffer[] nioBuffers = in.nioBuffers();
            int nioBufferCnt = in.nioBufferCount();
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

### 3.3.3 形同摆设的心跳

```
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        // Allow writing with void promise if handler is only configured for read timeout events.
        if (writerIdleTimeNanos > 0 || allIdleTimeNanos > 0) {
            promise.addListener(writeListener);
        }
        ctx.write(msg, promise);
    }
```

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

