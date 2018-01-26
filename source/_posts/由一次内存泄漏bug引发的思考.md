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
```
创建完成以后，对象的引用关系如下：
![申请堆外内存完毕](http://ovor60v7j.bkt.clouddn.com/%E7%94%B3%E8%AF%B7%E5%A0%86%E5%A4%96%E5%86%85%E5%AD%98.png)

first是Cleaner类的静态变量，相当于Clener链表的根指针。Cleaner对象在初始化时会被添加到Clener链表中。

### 堆外内存的释放

如果该DirectByteBuffer对象在一次GC中被回收了，在下一次FGC时，Cleaner对象被放入到ReferenceQueue中，并触发clean方法。如下图所示：

![申请堆外内存完毕](http://ovor60v7j.bkt.clouddn.com/%E5%9B%9E%E6%94%B6%E5%A0%86%E5%A4%96%E5%86%85%E5%AD%98.png)

Cleaner对象的clean方法主要有两个作用：

* 把自身从Clener链表删除，从而在下次GC时能够被回收
* 执行unsafe.freeMemory(address)，回收这块堆外内存。

如果该DirectByteBuffer对象在一次GC中被回收了，Cleaner对象会在合适的时候执行unsafe.freeMemory(address)，从而回收这块堆外内存。
由以上分析可知，小文件上传时，在触发FGC的情况下，泄漏的内存最终是能被回收的

### 虚拟机宕机原因

在上传大文件时，由于每个请求的body（在堆外）要远远大于其在堆内的指针，而如果在堆内没有触发FGC时堆外内存就耗尽了，就会引发虚拟机宕机。当然，对于此种场景，JDK也做了处理。在申请堆外内存时，如果内存不足，会触发System.gc()主动进行FGC。
之所以我们系统还是宕机了，是因为开启了-XX:+DisableExplicitGC，导致了System.gc()等于一个空函数。