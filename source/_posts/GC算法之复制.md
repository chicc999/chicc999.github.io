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