---
title: GC算法之引用计数
date: 2017-12-01 14:48:19
tags:
- 读书笔记
- GC
categories: 编程
comments: false
---
*《垃圾回收的算法与实现》的读书笔记与总结。本文介绍记录各个对象“人气指数”的GC算法— Reference Counting。*
<!--more-->
## 1 算法简介
引用计数法中引入了“计数器”的概念，表示有多少程序引用了这个对象（被引用数）。引用计数法中的对象如下图所示。
![引用计数](http://cyblog.oss-cn-hangzhou.aliyuncs.com/GC%E4%B9%8B%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0.png)


## 2 引用计数的算法
### 2.1 new_obj() 函数

```
new_obj(size){  obj = pickup_chunk(size, $free_list)  if(obj == NULL){    allocation_fail()
  }  else{    obj.ref_cnt = 1    return obj
  }}
```

### 2.2 update_ptr() 函数

```
update_ptr(ptr, obj){  inc_ref_cnt(obj)  dec_ref_cnt(*ptr)  *ptr = obj}
```
update_ptr() 函数做了以下操作，来更新指针指向另一个对象:

* 将目标对象引用数加1
* 原指针指向对象引用数减1
* 改变指针指向。

![update_ptr()](http://cyblog.oss-cn-hangzhou.aliyuncs.com/GC%E4%B9%8B%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0/update_ptr%E6%89%A7%E8%A1%8C%E6%83%85%E5%86%B5.png)


注意一定要先调用inc_ref_cnt，再调用dec_ref_cnt，来避免*ptr和obj是同一个对象时，由于减少引用计数导致obj被提前回收的bug。其中inc_ref_cnt() 函数只是单纯增加引用数，代码如下：

```
inc_ref_cnt(obj){  obj.ref_cnt++}
```

dec_ref_cnt() 函数则相对复杂，伪代码如下：

```
dec_ref_cnt(obj){  obj.ref_cnt--  if(obj.ref_cnt == 0){	for(child : children(obj)){	  dec_ref_cnt(*child)
	}	reclaim(obj)  }}
```

* obj的引用计数减一
* 如果引用计数为0，说明此对象是垃圾，需要被回收,对此对象的子对象递归减少引用计数
* 通过reclaim() 函数将obj 连接到空闲链表

## 3 优劣
###  3.1 优点

#### 可即刻回收垃圾
在引用计数法中，被引用数的值为0时，对象马上就会把自己作为空闲空间连接到空闲链表。也就是说，各个对象在变成垃圾的同时就会立刻被回收。
在其他的GC算法中，直到GC执行之前都无法判断一个对象是否是垃圾，所以有一部分内存空间会被垃圾占用。
#### 最大暂停时间短
在引用计数法中，只有更新指针时程序才会执行垃圾回收，且只回收因这次指针变动而产生的垃圾。所以最大暂停时间会明显缩短。
#### 没有必要沿指针查找
引用计数法，没必要由根沿指针查找。当我们想减少沿指针查找的次数时，就很合适。
比如，在分布式环境中，如果要沿各个计算节点之间的指针进行查找，成本就会增大，因此需要极力控制沿指针查找的次数。典型的参考RMI的DGC（ Distributed Garbage Collection）。
### 3.2 缺点
#### 计数器值的增减处理繁重
在引用计数法中，每当指针更新时，计数器的值都会随之更新。由于指针更新频繁，值的增减处理必然会变得繁重，尤其是根指针的更新。
#### 计数器需要占用很多位
用于引用计数的计数器最大必须能数完堆中所有对象的引用数。
#### 循环引用无法回收
在两个及两个以上的对象互相循环引用形成对象组的情况下，即使这些对象组都成了垃圾，程序也无法将它们回收。

## 4 延迟引用计数法

### 4.1 原理

由于计数器值增减处理繁重的原因之一是从根的引用变化频繁，因此延迟引用计数法的核心思想是让从根引用的指针的变化不反映在计数器上。
因为引用没有反映到计数器上，所以各个对象的计数器没有正确表示出对象本身的被引用数。为了解决这一问题，使用ZCT（Zero Count Table）。ZCT是一个表，它会事先记录下计数器值在dec_ref_cnt() 函数的作用下变为0的对象，如下图所示。
![ZCT](http://cyblog.oss-cn-hangzhou.aliyuncs.com/GC%E4%B9%8B%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0/ZCT.png)
### 4.2 dec_ref_cnt()函数
因为计数器值为0 的对象不一定都是垃圾，所以暂时先将这些对象保留。我们必须修正dec_ref_cnt() 函数，使其适应延迟引用计数法。
```
dec_ref_cnt(obj){  obj.ref_cnt--  if(obj.ref_cnt == 0){	if(is_full($zct) == TRUE){	  scan_zct()
	}	push($zct, obj)
  }}```
当obj的计数器值为0（也就是说obj可能是垃圾）时，把obj 添加到ZCT表中。如果ZCT表已经满了，则通过scan_zct()来判断真正需要被回收的对象。

### 4.3 new_obj() 函数

```
new_obj(size){  obj = pickup_chunk(size, $free_list)  if(obj == NULL){	scan_zct()	obj = pickup_chunk(size, $free_list)	if(obj == NULL){	  allocation_fail()	}  }	obj.ref_cnt = 1	return obj}
```
如果第一次分配没有成功，就意味着空闲链表中没有了大小合适的分块。此时搜索一遍$zct以回收资源，再次分配分块。如果这样还没有合适的分块，分配就失败了。

### 4.4 scan_zct() 函数

```
scan_zct(){  for(r : $roots){	(*r).ref_cnt++
  }  for(obj : $zct){	if(obj.ref_cnt == 0){	  remove($zct, obj)	  delete(obj)	 }
  }  for(r : $roots){	(*r).ref_cnt--
  }}
```

* 把所有通过根直接引用的对象的计数器都进行增量
* 遍历ZCT，将所有引用为0的对象移出
* 对子对象的计数器进行减量操作，并且回收自身
* 把所有通过根直接引用的对象的计数器都减一

其中第三步对于delete() 函数功能，代码如下

```
delete(obj){  for(child : children(obj){	(*child).ref_cnt--	if((*child).ref_cnt == 0){	  delete(*child)
	}
  }	    reclaim(obj)}
```
### 4.5 优点

延迟了根引用的计数，减轻了因根引用频繁发生变化而导致的计数器增减所带来的额外负担。

### 4.6 缺点

* 垃圾不可即使回收
* scan_zct() 函数导致最大暂停时间延长了。执行scan_zct() 函数所花费的时间与$zct 的大小成正比；如果缩小$zct降低最大暂停时间，调用scan_zct() 函数的频率就增加了，也压低了吞吐量。

## 5 Sticky 引用计数法
## 6 1位引用计数法
## 7 部分标记- 清除算法




