---
title: 死锁
tags: 
  - 并发
categories: basics
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



# 四个必要条件

- 互斥
- 请求和保持（占有并等待）
- 不可剥夺（不能抢）
- 环路等待（循环等待）



# 预防死锁

1. 资源一次性分配（破坏请求条件）
2. 一次性获取所有资源，不能全部获取就释放所有（破坏保持条件）
3. 资源有序分配（按照一定算法对资源进行排序，按顺序拿资源）
4. 资源可抢夺（需要设置优先级）

> 场景：两个人同时向对方转账。
>
> 解决方案：破坏 2 / 3 / 4
>
> ##### 破坏 2 不可抢占：
>
> synchronized不能实现。需要JUC的Lock。
>
> ##### 破坏 3 占有且等待：
>
> 拿到所需全部资源再进行操作，不能全拿到就全放弃，`资源利用率低`。
>
> ##### 破坏 4 循环等待：
>
> 规定拿资源的顺序（给资源设定序号）（都按相同顺序加锁）。
>
> > 比如用相同的算法（eg：hashcode()）对资源进行标号，按照标号进行排序，拿锁的时候都按顺序拿，每个线程就会都先拿a的锁再拿b的锁。



# 死锁检测

Jstack工具

java自带的jvisualvm

Jconsule工具

阿里Arthas  artha-boot.jar
