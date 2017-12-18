---
title: GC算法之复制
date: 2017-12-04 20:19:22
tags:
- 读书笔记
- GC
categories: 编程
comments: false
---
*《垃圾回收的算法与实现》的读书笔记与总结。本文介绍GC复制算法 — Copying GC。*
<!--more-->
## 1 算法简介
GC 复制算法（Copying GC）就是只把某个空间里的活动对象复制到其他空间，把原空间里的所有对象都回收掉。在此，我们将复制活动对象的原空间称为From 空间，将粘贴活动对象的新空间称为To 空间。### 1.1 算法描述
GC 复制算法是利用From 空间进行分配的。当From 空间被完全占满时，GC 会将活动对象全部复制到To 空间。当复制完成后，该算法会把From 空间和To 空间互换，GC 也就结束了。算法描述如下：
```
copying(){  $free = $to_start  for(r : $roots){	*r = copy(*r)  }
	  swap($from_start, $to_start)}```
将 $free 设置在 To 空间的开头，然后复制能从根引用的对象。copy()函数将作为参数传递的对象 \*r 复制的同时，也将其子对象进行递归复制。复制结束后返回的指针指向的是*r 所在的新空间的对象。在GC 复制算法中，在GC 结束时，原空间的对象会作为垃圾被回收。因此，由根指向原空间对象的指针也会被重写成指向返回值的新对象的指针。最后把From 空间和To 空间互换，GC 就结束了。
### 1.2 copy() 函数
copy() 函数将作为参数给出的对象复制，再递归复制其子对象。
```
copy(obj){  if(obj.tag != COPIED){	copy_data($free, obj, obj.size)	obj.tag = COPIED	obj.forwarding = $free	$free += obj.size  }  for(child : children(obj.forwarding)){	 *child = copy(*child)
  }  return obj.forwarding}
```
### 1.3 new_obj() 函数
```
new_obj(size){  if($free + size > $from_start + HEAP_SIZE/2){	copying()	if($free + size > $from_start + HEAP_SIZE/2){	  allocation_fail()	}
  }	    obj = $free  obj.size = size  $free += size  return obj}
```
* 如果当前已分配的地址加上待分配的大小小于最大可分配的地址，则执行复制操作
* 如果执行完复制操作后，空间依然不够，返回分配失败
* 执行分配操作，直接向后移动指针
### 1.4 执行过程
GC初始状态
![GC初始状态](http://ovor60v7j.bkt.clouddn.com/blog/GC%E4%B9%8B%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95/%E5%88%9D%E5%A7%8B%E7%8A%B6%E6%80%81.png)

复制引用B
![复制引用B](http://ovor60v7j.bkt.clouddn.com/blog/GC%E4%B9%8B%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95/B%E8%A2%AB%E5%A4%8D%E5%88%B6.png)

B被完整复制
![B被完整复制](http://ovor60v7j.bkt.clouddn.com/blog/GC%E4%B9%8B%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95/B%E8%A2%AB%E5%AE%8C%E6%95%B4%E5%A4%8D%E5%88%B6.png)

GC结束
![GC结束](http://ovor60v7j.bkt.clouddn.com/blog/GC%E4%B9%8B%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95/GC%E7%BB%93%E6%9D%9F.png)


## 2 优缺点
### 2.1 优点
#### 优秀的吞吐量

* GC 标记-清除算法消耗的吞吐量 = 搜索活动对象（标记阶段）所花费的时间 + 搜索整体堆（清除阶段）所花费的时间
* GC 复制算法只搜索并复制活动对象，不用遍历整个堆，能在较短时间完成gc。
* 在清除阶段所花费的时间不会随着堆增大而增大，仅仅和活动对象的数量有关系。

#### 可实现高速分配
可用分块是一个连续的内存空间，移动$free 指针就可以进行分配；不用在空闲列表里寻找可用空间。
