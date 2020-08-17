---
title: JVM堆的对象布局
date: 2020/8/17 10:20:00
categories:
- JAVA
- jvm
- 锁
---
> > 本文章探讨的是HotSpot虚拟机下JVM堆中的对象布局

JVM堆的对象布局，主要如下图所示

<img src="https://blog-1256602811.file.myqcloud.com/16d189761607da75" style="zoom:50%;" />

## 对象头

- **Mark Word（标记字段）**：在32位及64位虚拟机存储长度分别为32bit，64bit；

  其中主要存储对象自身的运行时数据，根据对象状态复用自己的存储空间根据锁标志位的变化而变化

- **Klass Point（类型指针）**：指向类型元数据指针，通过该指针可以确定这个对象是哪个类的实例

- **数组长度**：如果该对象为数组，那么将会专门保存该数组的长度



## Mark Word
是**synchronized**实现线程同步的关键，该字段中的锁标志位，级别从低到高分别为：

- 无锁
- 偏向锁
- 轻量级锁
- 重量级锁

在64位的下的结构是这样的

<img src="https://blog-1256602811.file.myqcloud.com/16d189761d997c70" style="zoom:50%;" />

## Monitor（管程）
**synchronized**就是在Java中对Monitor的MESA模型的实现；在Java中每一个对象都是天生的Monitor，在HotSpot虚拟机中，**Monitor是由ObjectMonitor实现的**，其中关键的结构如下

```c++
class ObjectMonitor() {
    // 记录个数，用来支持可重入
    _count = 0;
    // 处于Wait状态的线程，会被加入到_WaitSet
    _WaitSet = NULL;
    // 处于阻塞(Block)状态的线程，会被加入到该列表
    _EntrySet = NULL;
    // 持有Monitor的线程
    _owner = NULL;
}
```

