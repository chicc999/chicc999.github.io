---
title: GC算法之标记清除
date: 2017-11-14 14:03:40
tags:
- 读书笔记
- GC
categories: 编程
comments: false
---
*《垃圾回收的算法与实现》的读书笔记与总结。本文介绍世界上首个值得纪念的GC算法— Mark Sweep GC。*
<!--more-->
## 1 算法简介
GC 标记- 清除算法由标记阶段和清除阶段构成,伪代码如下：

``` 
mark_sweep(){  mark_phase()  sweep_phase()}
```

### 1.1 标记阶段 
核心就是“遍历标记对象”。从根域出发，遍历所有的对象，把所有活动的对象打上标记。

```
mark_phase(){    for(r : $roots)  	mark(*r)}
```
``` 
mark(obj){  if(obj.mark == FALSE)    obj.mark = TRUE	for(child : children(obj)){		mark(*child)	}}
```
在标记阶段中，程序会标记所有活动对象。显然，标记所花费的时间与“活动对象的总数”成正比。
比较一下内存使用量（已存储的对象数量）就可以知道，深度优先搜索比广度优先搜索更能压低内存使用量。因此我们在标记阶段经常用到深度优先搜索。
### 1.2 清除阶段
在清除阶段中，collector 会遍历整个堆，回收没有打上标记的对象（即垃圾），使其能再次得到利用。
```
sweep_phase(){	sweeping = $heap_start;	while(sweeping < $heap_end)		if(sweeping.mark == TRUE){			sweeping.mark = FALSE;		}else{			sweeping.next = $free_list;			$free_list = sweeping;			}		sweeping += sweeping.size;}
```
sweeping是从堆首地址$heap_start开始，按顺序一个个遍历对象的标志位。如果是活动对象，就取消标志位，等待下一次标记；不如不是标志位

### 1.3 分配
在清除阶段已经把垃圾对象连接到空闲链表了。搜索空闲链表并寻找大小合适的分块，这项操作就叫作分配。
```
new_obj(size){	chunk = pickup_chunk(size, $free_list);	if(chunk != NULL)		return chunk;	else		allocation_fail();}
```
pickup_chunk() 函数用于遍历$free_list，寻找大于等于size 的分块。如果此函数没有找到合适的分块，则会返回NULL。

#### 分配算法

* First - fit
	最初发现大于等于size 的分块时就会立即返回该分块;如果它找到比size 大的分块，则会将其分割成size 大小的分块和去掉size 后剩余大小的分块，并把剩余的分块返回空闲链表。
* Best - fit
	有遍历空闲链表，返回大于等于size 的最小分块.
* Worst - fit
	找出空闲链表中最大的分块，将其分割成mutator 申请的大小和分割后剩余的大小，目的是将分割后剩余的分块最大化。

### 1.4 合并
根据分配策略的不同可能会产生大量的小分块。如果它们是连续的，就能把所有的小分块连在一起形成一个大分块。合并是在清除阶段进行的。加入合并功能的清扫函数如下，只在发现当前内存需要被清理时，判断是否和上一块连续，如果连续则合并成同一块。
```
sweep_phase(){	sweeping = $heap_start;	while(sweeping < $heap_end){		if(sweeping.mark == TRUE)			sweeping.mark = FALSE;		else {			if(sweeping == $free_list + $free_list.size)				$free_list.size += sweeping.size;			else {
				sweeping.next = $free_list;				$free_list = sweeping;
            }		sweeping += sweeping.size;	}}```
## 2 优缺点
### 2.1 优点

* 实现简单
* 与保守式GC算法（对象不能被移动)兼容

### 2.2 缺点

#### 2.2.1 碎片化 
由分配算法可知标记-清除法或多或少会发生碎片化。如果发生碎片化，那么即使堆中分块的总大小够用，也会因为一个个的分块都太小而不能执行分配。
	
#### 2.2.2 分配速度
GC标记- 清除算法中分块不是连续的，因此每次分配都必须遍历空闲链表，找到足够大的分块。最糟的情况就是每次进行分配都得把空闲链表遍历到最后。

#### 2.2.3 与copy-on-write不兼容
写时复制技术的优势在于，fork进程的时候，内存空间是引用而不是复制。这样fork后的两个进程的内存指向的是同一片区域，在进程执行不同的操作时，才在其他空闲内存区域写入一个新的对象并改写原指针。如果两个程序的大部分内存数据都是相同的，那么这个技术能省下非常多的内存空间。
而标记清除算法每次gc，都会修改对象头部的标志位。“copy-on-write”会判断对象已经被改动，从空闲内存中写入新的对象并改写原指针，相当于每一个对象都被复制了一遍。

## 3 改进
### 3.1 多个空闲链表
是利用分块大小不同的空闲链表，即创建只连接大分块的空闲链表和只连接小分块的空闲链表。假设对象中的每个成员变量都占用Field_Size个字节。

```
new_obj(size){  index = size / Field_Size  if(index <= 100){    if($free_list[index] != NULL){	  chunk = $free_list[index]      $free_list[index] = $free_list[index].next	  return chunk
	}  }else{	chunk = pickup_chunk(size, $free_list[101])	if(chunk != NULL)		return chunk
  }allocation_fail()}
```

```
sweep_phase(){  for(i : 2..101)	$free_list[i] = NULL
	sweeping = $heap_start
while(sweeping < $heap_end){  if(sweeping.mark == TRUE){	sweeping.mark = FALSE  }else{    index = size / Field_Size    if(index <= 100){	  sweeping.next = $free_list[index]	  $free_list[index] = sweeping
	}else{	  sweeping.next = $free_list[101]	  $free_list[101] = sweeping
	}	sweeping += sweeping.size
  }}
```

### 3.2 BIBOP
BiBOP核心是“将大小相近的对象整理成固定大小的块进行管理“。
此方法原本是为了消除碎片化，提高堆使用效率而采用。但在多个块中分散残留着同样大小的对象，反而会降低堆使用效率。
### 3.3 位图标记法
位图标记法通过标记的时候，不在对象的头里置位，而是在特定的位置设置标志位。

![位图标记法](http://ovor60v7j.bkt.clouddn.com/blog/GC%E4%B9%8B%E6%A0%87%E8%AE%B0%E6%B8%85%E9%99%A4%E4%BD%8D%E5%9B%BE%E6%A0%87%E8%AE%B0.png)

mark() 函数如代码如下：

```
mark(obj){obj_num = (obj - $heap_start) / WORD_LENGTHindex = obj_num / WORD_LENGTHoffset = obj_num % WORD_LENGTHif(($bitmap_tbl[index] & (1 << offset)) == 0)$bitmap_tbl[index] |= (1 << offset)for(child : children(obj))mark(*child)}
```

WORD_LENGTH代表一个字的位宽，每个字需要一个位来标记是否使用。obj_num表示从位图前面数起，obj的标志位在第几个。由于每个“字”可以存储WORD_LENGTH个“位“”，所以用obj_num 除以WORD_LENGTH 得到的商index 以及余数offset 来分别表示落在位图的哪个字，以及这个字的哪个位上。

#### 与写时复制技术兼容

位图标记在单独的区域而不是对象中，可以与写时复制技术兼容。

#### 清除操作更高效

位图标记的sweep_phase()函数如下：

```
sweep_phase(){sweeping = $heap_startindex = 0offset = 0while(sweeping < $heap_end){  if($bitmap_tbl[index] & (1 << offset) == 0){	sweeping.next = $free_list	$free_list = sweeping
	}  index += (offset + sweeping.size) / WORD_LENGTH  offset = (offset + sweeping.size) % WORD_LENGTH  sweeping += sweeping.size}
for(i : 0..(HEAP_SIZE / WORD_LENGTH - 1))$bitmap_tbl[i] = 0}
```

用sweeping 指针遍历整个堆，如果对应的位图没有被标记，则加入空闲列表。最后将整个位图清零。

注意，在堆有多个，对象地址不连续的情况下，我们无法用单纯的位运算求出标志位的位置。因此，在堆为多个的情况下，一般会为每个堆都准备一个位图表格。

### 3.4 延迟清除法

延迟清除法在标记操作结束后，不一并进行清除操作，而是延迟清除。从而缩减因清除操作而导致的应用最大暂停时间。

#### 3.4.1 new_obj() 函数
延迟清除法中的new_obj() 函数的定义如下：

```
new_obj(size){chunk = lazy_sweep(size)if(chunk != NULL)return chunkmark_phase()chunk = lazy_sweep(size)if(chunk != NULL)return chunkallocation_fail()}
```

在分配时直接调用lazy_sweep() 函数，进行清除操作。如果它能用清除操作来分配分块，就会返回分块；如果不能分配分块，就会执行标记操作。当lazy_sweep() 函数返回NULL时，也就是没有找到分块时，会调用mark_phase() 函数进行一遍标记操作，再调用lazy_sweep() 函数来分配分块。在这里没能分配分块也就意味着堆上没有分块，分配失败。

#### 3.4.2 lazy_sweep() 函数

```
lazy_sweep(size){  while($sweeping < $heap_end){	if($sweeping.mark == TRUE){	  $sweeping.mark = FALSE
	  }	else if($sweeping.size >= size){	  chunk = $sweeping	  $sweeping += $sweeping.size	  return chunk	  }	$sweeping += $sweeping.size	}	$sweeping = $heap_start	return NULL}
```

lazy_sweep() 函数会一直遍历堆，直到找到大于等于所申请大小的分块为止。如果分块已经被标记，说明不能使用，清空标志位等待下一轮标记。如果没有被标记且符合所需要的大小，则直接返回使用。

注意：

* $sweeping 变量是全局变量。也就是说，遍历的开始位置位于上一次清除操作中发现的分块的右边。
* 当lazy_sweep() 函数遍历到堆最后都没有找到分块时，会返回NULL。
延迟清除法不是一下遍历整个堆，它只在分配时执行必要的遍历，所以可以压缩因清除操作而导致的应用的暂停时间。
#### 延迟清除法的问题
![垃圾分布不均匀](http://ovor60v7j.bkt.clouddn.com/%E5%9E%83%E5%9C%BE%E5%88%86%E5%B8%83%E4%B8%8D%E5%9D%87%E5%8C%80.png)
程序在清除垃圾较多的部分时能马上获得分块，所以能减少应用的暂停时间。然而一旦程序开始清除活动对象周围，就怎么也无法获得分块了，这样就增加了应用的暂停时间。导致某些分配特别慢。
