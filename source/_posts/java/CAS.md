---
title: CAS原理及应用
date: 2020/8/3 07:03:00
categories:
- JAVA
---
## 介绍

CAS全称**Compare and Swap**（比较并交换），是一种无锁算法；在不使用锁的情况下保证线程安全

## 实现

使用的是一种**乐观锁**的思想，即：假设共享资源不会发生冲突，所以**不会加锁**，因此线程将不断执行，当碰到冲突时，就**重试当前操作**直到没用冲突为止

![](https://www.jianguoyun.com/c/tblv2/yel2sklqlILV9uS83DySlK95V-eJSWsaU4Qk__lORBxHysIBEFdHk--HKTt3FiRygPgLI1Is/S7SdiLsHJRO3h6zHswUiug/l)

### 如何鉴别冲突

CAS算法一般涉及三个参数：

1. 准备更新的变量V

2. 期望值E

3. 准备更新的值N

   仅当V的值 `==`E时，CAS通过**原子方式**用新值N来更新V的值（比较+更新整体是一个原子操作）；否则不会进行任何操作



## 无锁好处

如果多个线程同时使用CAS操作一个变量时，只有一个线程能够修改成功；其余线程均会失败，并继续重试

因为其他线程没有被挂起，所以不会发生线程上下文切换带来的开销

适用于**读操作多**的场景，不加锁的特点使其读操作的性能大幅提升

## JAVA中实现

在包`java.util.concurrent`下的原子类，就是通过CAS来实现了乐观锁，如下是`AtomicInteger`的源码：

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2018b/feda866e.png)

- unsafe：位于`sun.misc`包下的一个类，可以像C语言一样操作内存空间，主要用来调用CAS方法

- valueOffset：`value`地址偏移量
- value：`AtomicInteger`实例值，并且使用`volatile`关键字保证线程可见性

相关源码：

```java
// ------------------------- JDK 8 -------------------------
// AtomicInteger 自增方法
public final int incrementAndGet() {
  return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// Unsafe.class
public final int getAndAddInt(Object var1, long var2, int var4) {
  int var5;
  do {
      var5 = this.getIntVolatile(var1, var2);
  } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
  return var5;
}

// ------------------------- OpenJDK 8 -------------------------
// Unsafe.java
public final int getAndAddInt(Object o, long offset, int delta) {
   int v;
   do {
       v = getIntVolatile(o, offset);
   } while (!compareAndSwapInt(o, offset, v, v + delta));
   return v;
}
```

其中整个**比较+更新**操作封装在`compareAndSwapInt()`中，该方法是通过调用JNI方法中一个CPU指令完成的，属于原子操作

其中的CPU指令即为**cmpxchg**，在内部比较寄存器中的A和内存中的值V，如果相等，就把B值存入内存中

### CAS存在的问题

1. 循环时间开销大：如果CAS操作长时间不成功（说明写操作比较多），会导致其一直自旋，给CPU带来很大的开销
2. ABA问题，解决思路就是使用版本号，每次操作都更新版本号
   - 在JDK1.5版本之后，提供了一个`AtomicStampedReference`类来解决ABA问题