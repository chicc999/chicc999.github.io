---
title: JDK源码剖析之线程
date: 2017-01-11 15:51:44
tags:
- java
- 线程
categories: 编程
comments: false
---
通过分析JDK线程的源码，了解线程及其中断的用法，以及适用场景。
<!--more-->
# 1 构造

```
public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) 
```


# 2 启动

# 3 中断

## 3.1 API

### 3.1.1 interrupt

```
public void interrupt()
```
表示中断对应的线程。
* 对应线程处于wait,sleep,join等阻塞状态，则抛出InterruptedException，并且清空状态位。
* 如果线程正在通过InterruptibleChannel进行IO操作，channel将被关闭，线程中断位状态置为true,并且抛出ClosedByInterruptException。
* 如果线程阻塞在Selector上则中断状态位被置为true并且立即返回。


### 3.1.2 isInterrupted

```
 public boolean isInterrupted()
```
返回线程的中断标志位

### 3.1.3 interrupted

```
public static boolean interrupted()
```
返回当前线程的中断标志位，并且将标志位进行重置（设置为false）
注意这是一个静态方法，只能设置当前线程

# 3.2 为什么要用interrupt
线程的stop被设置为不建议使用的方法。
一个线程的运行不应该被另一个线程去中止，stop方法在被调用的时候会释放所有的锁，从而导致对象处于不一致的状态。举个例子,在多线程下有两个方法doSomething和getMeasurement

```
public synchronized void doSomething() 
{
      count++;
      average = count/total;
}

public synchronized AverageAndCount getMeasurement() 
{
   return new AverageAndCount(average, count);
}
```
如果线程A在执行doSomething方法的count++时被stop了，随即释放锁，线程B调用getMeasurement，由于A线程没有能完整执行完方法就释放了锁，会导致此时对象的状态不一致，就可能出现很多奇怪的执行结果。

而interrupt() 并不能真正的中断线程，它仅仅给线程设置一个标志告诉线程应该结束了，最终到底中断还是继续运行，应该由被通知的线程自己处理。

# 3.3 interrupt用法

从api的注释来看，调用interrupt后，根据此时被调用线程的状态，主要有两种结果：

* 如果线程处于被阻塞状态（sleep, wait, join 等），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。
* 如果线程处于活动状态，那么会将该线程的中断标志设置为 true。被设置中断标志的线程将继续正常运行，不受影响。

由上可以看出，interrupt() 并不能真正的中断线程。其作用只是给线程一个信号，具体是中断还是继续，由此线程自行控制。

