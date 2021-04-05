---
title: synchronized
tags: 
  - todo
  - synchronized
categories: Java
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---

# todo todo  大写的todo！！！！



https://blog.csdn.net/lengxiao1993/article/details/81568130



无锁

偏向锁

轻量级锁

重量级锁



### 无锁->偏向锁 

> 也就是无锁加锁过程

- 如果为可偏向状态（可以用jvm参数设置为不启用偏向锁    -XX:-UseBiasedLocking）
  - 如果 CAS 操作成功， 则认为已经获取到该对象的偏向锁， 执行同步块代码 
  - 如果 CAS 操作失败， 则说明有另外一个线程B 抢先获取了偏向锁。 这种状态说明该对象的竞争比较激烈， 此时需要**撤销** B 获得的偏向锁，将 Thread B 持有的锁升级为轻量级锁
- 如果为已偏向状态
  - MarkWord.threadID == threadID，当前线程获取到偏向锁，继续执行同步代码块；
  - MarkWord.threadID != threadID，该对象偏向其他线程，需要撤销偏向锁。



### 偏向锁->轻量级锁

当超过一个线程竞争某一个对象时，会发生偏向锁的撤销操作。撤销之后对象可能处于两种情况：

1. 不可偏向的无锁状态

   > 原来获取了偏向锁的线程已经执行完同步代码块，对象处于闲置状态

2. 不可偏向的轻量级锁(已锁)状态

   > 原来获取了偏向锁的线程还未执行完同步代码块，偏向锁依旧有效，有竞争，转为轻量级锁加锁状态。



### 轻量级锁->重量级锁

轻量级锁加锁过程：

- 根据标志位（tag bits）判断对象处于不可偏向的无锁状态
- 在当前线程栈针中创建锁记录空间
- CAS操作将对象头中的MarkWord替换为所记录的指针
  - 成功：当前线程获得锁
  - 失败：该对象已经被加锁了，先自旋CAS，再失败，就升级重量级锁



### 重量级锁

重量级锁依赖于操作系统的互斥量（**mutex**） 实现。 

该操作会导致进程从**用户态与内核态之间的切换**， 是一个开销较大的操作。





## java 对象头

![java-对象头](images\java-对象头.png)