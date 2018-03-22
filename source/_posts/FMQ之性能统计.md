---
title: FMQ之性能统计
date: 2017-03-12 19:31:18
tags:
- FMQ
- 消息队列
categories: 编程
---
*FMQ的性能统计包括基础的性能统计模块，客户端、proxy、server等实际进行自身的统计并存储到hbase，task对结果进行汇总和统计，用户可以在web端查询到数据和查看展示。*
<!--more-->

# 1 TP性能统计
## 1.1 类结构

```java
public class TPStatBuffer implements Serializable {
    protected static final int LENGTH = 256;
    protected static final int MAX_INDEX = LENGTH - 1;
    public static final int MAX_TIME = LENGTH * LENGTH - 1;
    // 存放每个耗时对应发生的次数
    protected AtomicReferenceArray<AtomicIntegerArray> timer = new AtomicReferenceArray<AtomicIntegerArray>(LENGTH);
    // 成功处理的记录条数
    protected AtomicLong totalCount = new AtomicLong(0);
    // 成功调用次数
    protected AtomicLong totalSuccess = new AtomicLong(0);
    // 失败调用次数
    protected AtomicLong totalError = new AtomicLong(0);
    // 数据大小
    protected AtomicLong totalSize = new AtomicLong(0);
    // 总时间
    protected AtomicLong totalTime = new AtomicLong(0);
    }
```
timer是一个 N X N 的矩阵，如果某次调用耗时 M 毫秒，则将矩阵 M 毫秒对应的值加一。其余参数含义见注释。

### 1.2 矩阵存储

为什么要用矩阵来存储呢？
我们要统计每次调用所耗费的时间，需要估计一个耗时上限，这个上限很难达到，但是一旦超过这个时间，我们的程序就不能精确统计，只能将其统计为上限值。为了避免经常出现这种情况，上限值必须远远大于实际的均值。
按照这个设计，我们在实际生产中，每次调用所花费的时间将大量分布在低值带附近，同时由于网络抖动，服务器GC，瞬时并发量，磁盘写入速度等影响，会出现较高的异常值。我们需要设计一种结构，可以巧妙的跳过大多数不存在的值，于是设计了矩阵进行存储。

```java
/**
     * 成功调用，增加TP计数
     *
     * @param count 处理的记录条数
     * @param size  数据包大小
     * @param time  所耗时间(毫秒)
     */
    public void success(final int count, final long size, final int time) {
        totalSuccess.incrementAndGet();
        if (count > 0) {
            totalCount.addAndGet(count);
        }
        if (size > 0) {
            totalSize.addAndGet(size);
        }
        if (time > 0) {
            totalTime.addAndGet(time);
        }
        int ptime = time;
        if (ptime > MAX_TIME) {
            ptime = MAX_TIME;
        }
        int i = ptime >> 8;
        int j = ptime & MAX_INDEX;
        AtomicIntegerArray v = timer.get(i);
        if (v == null) {
            v = new AtomicIntegerArray(LENGTH);
            if (!timer.compareAndSet(i, null, v)) {
                v = timer.get(i);
            }
        }
        v.addAndGet(j, 1);
    }
```

以上是调用成功后的性能统计方法。9-18行分别将调用总次数、成功次数、数据包大小、所花时间进行了累加。19-22行将万一有超出统计范围的时间，则按最大值进行处理。23-32行则对所耗时间计算出其在矩阵中的位置，并将对应位置的值进行加一。
我们可以注意到，如果某一列完全没有数据（对应整整256ms没有值，这在高值区间很正常），则对应的结构为null。再看我们统计的代码：

```java
  /**
     * 获取性能统计
     *
     * @return 性能统计
     */
    public TPStat getTPStat() {
        TPStat stat = new TPStat();
        stat.setSuccess(totalSuccess.get());
        stat.setError(totalError.get());
        stat.setCount(totalCount.get());
        stat.setTime(totalTime.get());
        stat.setSize(totalSize.get());

        if (stat.getSuccess() <= 0) {
            return stat;
        }

        int min = -1;
        int max = -1;
        int tp999 = (int) Math.ceil(stat.getSuccess() * 99.9 / 100);
        int tp99 = (int) Math.ceil(stat.getSuccess() * 99.0 / 100);
        int tp90 = (int) Math.ceil(stat.getSuccess() * 90.0 / 100);
        int tp50 = (int) Math.ceil(stat.getSuccess() * 50.0 / 100);

        int count;
        int prev = 0;
        int next = 0;
        int time;
        AtomicIntegerArray v;
        for (int i = 0; i < LENGTH; i++) {
            v = timer.get(i);
            if (v == null) {
                continue;
            }
            for (int j = 0; j < LENGTH; j++) {
                count = v.get(j);
                if (count <= 0) {
                    continue;
                }
                time = i * LENGTH + j;
                if (min == -1) {
                    min = time;
                }
                if (max == -1 || time > max) {
                    max = time;
                }
                next = prev + count;
                if (prev < tp50 && next >= tp50) {
                    stat.setTp50(time);
                }
                if (prev < tp90 && next >= tp90) {
                    stat.setTp90(time);
                }
                if (prev < tp99 && next >= tp99) {
                    stat.setTp99(time);
                }
                if (prev < tp999 && next >= tp999) {
                    stat.setTp999(time);
                }
                prev = next;
            }
        }
        stat.setMin(min);
        stat.setMax(max);
        return stat;
    }
```
统计的思路如下：

* 计算出tpxx需要成功调用多少次（20-23行）
* 遍历timer矩阵，其位置为M的元素值为N，则代表有N次调用耗时为M毫秒。累积调用次数，当超过tpxx的值时，说明tpxx的耗时就落在这个值上。同时统计出最大值和最小值
* 当某列为null时，可以直接跳过一列，说明其后连续256ms都没有值；当某个值为null，可以直接跳过此值的统计。

由此可见，对于性能统计的分布特性（大量分布在低值段），利用矩阵存储可以省去大量遍历的时间。

### 1.3 局限

* 存在统计的最大值，即矩阵能代表的最大毫秒数，超过这个值则统计不精确
* 必须遍历完矩阵才能得到最大值