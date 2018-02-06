---
title: 由一次内存泄漏bug引发的思考
date: 2017-01-25 16:42:12
tags:
- java
- JVM
categories: 编程
comments: false
---
由一次堆外内存泄漏引发的思考，为什么在JVM可以回收内存的情况下，还要主动进行堆外内存回收？
<!--more-->

# 问题描述
文件系统的上传功能由于SDK是JAVA实现的，其它语言没法使用，所以用Netty写了个proxy，实现HTTP协议到自定义协议的转换，以支持其它语言上传文件。在测试中，发现有以下现象：

* 对于小文件的上传，内存短时间内持续上升，但是触发FULL GC以后能被正确回收。可以平稳运行。
* 对于大文件的上传，堆内存持续上升，尤其堆外内存。最终导致内存耗尽，虚拟机宕机。

# 问题解决

这是很明显的内存泄漏问题。由于在proxy对于http的解码模块，使用了netty的HttpPostRequestDecoder，此对象直接使用堆外内存存储http请求的body部分。而在proxy的实现过程中，并没有在用完以后释放资源。忽略了类注释中的“You MUST call destroy() after completion to release all resources.“
这样在处理源源不断的请求过程中，堆外内存的使用不断上涨也就有了解释。但是仍然有个问题无法解释，为什么小文件和大文件的表现并不一样，后者会导致jvm宕机？

# 问题引申

首先我们看第一个问题，小文件上传时，为什么可以回收泄漏的堆外内存？首先我们来看下堆外内存的自动回收机制。

### 堆外内存的申请
JDK中使用DirectByteBuffer对象来表示堆外内存。
每个DirectByteBuffer对象在初始化时，都会调用unsafe.allocateMemory分配一段堆外内存，然后用返回的内存地址创建一个对应的Cleaner对象。如以下代码所示。

```java
long base = 0;
try {
	base = unsafe.allocateMemory(size);
	} catch (OutOfMemoryError x) {
    Bits.unreserveMemory(size, cap);
    throw x;
    }
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
```
创建完成以后，对象的引用关系如下：
![申请堆外内存完毕](http://ovor60v7j.bkt.clouddn.com/%E7%94%B3%E8%AF%B7%E5%A0%86%E5%A4%96%E5%86%85%E5%AD%98.png)

```java
public class Cleaner extends PhantomReference<Object> {
    private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue();
    private static Cleaner first = null;
    private Cleaner next = null;
    private Cleaner prev = null;
    private final Runnable thunk;
    … …
    }
```
Cleaner的构造中可以看出有个类的静态变量first，相当于Clener链表的根指针。有个静态的ReferenceQueue，被所有的cleaner实例共享。Cleaner对象在初始化时会被添加到Clener链表中。
注意DirectByteBuffer构造函数的倒数第二行，创建了一个Cleaner对象并且注册了回调，同时传入了this指针。来看下回调函数，这是一个Runnable对象。

```java
        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address);
            address = 0;
            Bits.unreserveMemory(size, capacity);
        }
```
可以看到回调的实现就是在释放堆外内存。那什么时候这段代码被执行呢？

### 堆外内存的释放

如果该DirectByteBuffer对象在一次GC中被回收了，Cleaner失去了强引用。如下图所示：

![申请堆外内存完毕](http://ovor60v7j.bkt.clouddn.com/%E5%9B%9E%E6%94%B6%E5%A0%86%E5%A4%96%E5%86%85%E5%AD%98.png)

于是在下一次FGC时，Cleaner对象被垃圾回收器放入到pending链表中。
在Reference中起了一个守护线程，一直在执行tryHandlePending，以下为实现

```java
 static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }

        // Fast path for cleaners
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }
```
这个方法一个做了三件事情：

* 更改指针，将pending链表的头指针指向下一个
* 如果引用对象是cleaner类型，执行clean方法
* 如果非cleaner对象且自己带有ReferenceQueue，则把对象加入ReferenceQueue

再来看Cleaner的clean方法

```
    public void clean() {
        if(remove(this)) {
            try {
                this.thunk.run();
            } catch (final Throwable var2) {
            … …
        }
    }
```
Cleaner对象的clean方法主要有两个作用：

* Cleaner对象的实例先将自己从静态的链表里去除（即断开和静态变量first的连接）
* 执行thunk的run方法。thunk就是在创建Cleaner对象时传入的回调，在DirectByteBuffer中的实现就是执行unsafe.freeMemory(address)，回收这块堆外内存。

由以上分析可知，小文件上传时，在触发FGC的情况下，泄漏的内存最终是能被回收的

### 虚拟机宕机原因

在上传大文件时，由于每个请求的body（在堆外）要远远大于其在堆内的指针，而如果在堆内没有触发FGC时堆外内存就耗尽了，就会引发虚拟机宕机。当然，对于此种场景，JDK也做了处理。在申请堆外内存时，如果内存不足，会触发System.gc()主动进行FGC。
之所以我们系统还是宕机了，是因为开启了-XX:+DisableExplicitGC，导致了System.gc()等于一个空函数。