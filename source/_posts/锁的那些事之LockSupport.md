---
title: 锁的那些事之LockSupport
date: 2017-09-02 13:17:27
tags: 
- java
- 锁
categories: 编程
comments: false
---
LockSupport是用来创建锁和其他同步类的基本线程阻塞原语。

## API说明

```
public static void park() 
```
为了线程调度，禁用当前线程，除非许可可用。  
如果许可可用，则使用该许可，并且该调用立即返回；否则，为线程调度禁用当前线程，此线程将处于休眠状态，除非遇到下面三种情况之一：
	* 其它线程调用unpark以此线程为目标
	* 其它线程中断此线程
	* 虚假唤醒


```
public static unpark(Thread thread) 
```
向指定线程给予许可
 <br /> 
这俩是LockSupport提供的核心方法，其余接口则是在这两种方法之上的变形。unpark函数为线程提供“许可(permit)”，线程调用park函数则等待“许可”。但是要注意以下2点：
	* 这种“许可”是一次性的。比如线程B连续调用了三次unpark函数，当线程A调用park函数就使用掉这个“许可”，如果线程A再次调用park，则进入等待状态  
	* unpark函数可以先于park调用  


## 示例  
``` java
class FIFOMutex {
	private final AtomicBoolean locked = new AtomicBoolean(false);
	private final Queue<Thread> waiters
			= new ConcurrentLinkedQueue<Thread>();

	public void lock() {
		boolean wasInterrupted = false;
		Thread current = Thread.currentThread();
		waiters.add(current);

		// Block while not first in queue or cannot acquire lock
		while (waiters.peek() != current ||
				!locked.compareAndSet(false, true)) {
			LockSupport.park(this);
			if (Thread.interrupted()) {
				System.out.println("current thread " + Thread.currentThread().getName() + " receive interrupt signal");
				wasInterrupted = true;
			}
		}
		System.out.println("current thread " + Thread.currentThread().getName() + " get Lock");
		waiters.remove();
		if (wasInterrupted) {
			System.out.println("current thread " + Thread.currentThread().getName() + " will interrupt itself");
			current.interrupt();
		}
	}

	public void unlock() {
		locked.set(false);
		System.out.println("current thread " + Thread.currentThread().getName() + " will release Lock");
		LockSupport.unpark(waiters.peek());
	}
}
```
以上为LockSupport的官方示例，用队列和一个原子变量实现了一个先进先出 (FIFO) 非重入锁类的框架。
* Lock阶段：
	* 将当前线程加入等待队列。
	* 如果当前线程不处于队列头部，或者CAS操作无法成功，则将自身park住。注意此时要将条件判断放在whlie循环中，以防止虚假唤醒等问题。在此过程中，如果线程被中断，则记录下状态，但不响应中断。恢复当前线程中断状态位。
	* 如果退出循环，则当前线程处于队列头部且CAS操作成功。如果其它线程发出要求当前线程中断信号，则相应中断。
* UnLock阶段：
	* 将原子变量复原
	* 唤醒队列头部的线程
	
在以上程序中，如果当前线程在获取锁过程中收到中断信号，是否依旧会执行unLock？写个测试程序如下：
``` java  
public static void main(String[] args) {
	final FIFOMutex lock = new FIFOMutex();

	final CountDownLatch countDownLatch = new CountDownLatch(1);

	Thread t1 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				lock.lock();
			} finally {
				//通知主线程中断t2
				countDownLatch.countDown();

				long cur = System.currentTimeMillis();

					try {
						//停止1s释放锁，让线程t2先被中断
						TimeUnit.MILLISECONDS.sleep(1000);
					} catch (InterruptedException e) {
						//当前线程睡眠被中断，暂时不处理，中断信号复位
						System.out.println("current thread" + Thread.currentThread().getName() + " sleep interrupted");
					}

				lock.unlock();


			}
		}
	});
	Thread t2 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				lock.lock();
			} finally {
				lock.unlock();
			}
		}
	});

	t1.start();
	t2.start();

	try {
		countDownLatch.await();
	} catch (InterruptedException e) {
		e.printStackTrace();
	}

	//收到信号，中断t2
	t2.interrupt();

}
```


测试结果为：
```
current thread Thread-0 get Lock
current thread Thread-1 receive interrupt signal
current thread Thread-0 will release Lock
current thread Thread-1 get Lock
current thread Thread-1 will interrupt itself
current thread Thread-1 will release Lock
```
	
由此如果调用者正确编码，unLock位于finally当中，当前线程中断自身后unlock依旧会被执行。
