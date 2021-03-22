---
title: Spring Cloud Alibaba
tags: 
  - 分布式
categories:
  - [分布式]
---



# Nacos



# Sentinel



# Seata

全局唯一事务ID + 三个组件（TC、TM、RM）

> TC：transaction coordinator 事务协调器
>
> TM：transaction Manager  事务管理器
>
> RM：Resource Managere 资源管理器

http://seata.io/zh-cn/docs/overview/what-is-seata.html

## 处理过程

1. TM向TC申请开启一个全局事务，TC返回一个XID；
2. RM向TC注册分支事务，归XID所在的全局事务管辖；
3. TM向TC发起对XID的全局提交 / 回滚；
4. TC调度XID下的所有分支事务完成提交 / 回滚。

## 写隔离

- 一阶段本地事务提交前，需要确保先拿到 **全局锁** 。
- 拿不到 **全局锁** ，不能提交本地事务。
- 拿 **全局锁** 的尝试被限制在一定范围内，超出范围将放弃，并回滚本地事务，释放本地锁。

> 两个全局事务tx1和tx2， tx1先开始。
>
> tx1开启本地事务，拿到本地锁，更新操作。
>
> 本地事务提交之前先拿到全局锁，本地提交，释放本地锁。
>
> ​	此时tx2开始执行，开启本地事务，拿到本地锁，更新操作。
>
> ​	本地事务提交之前先拿到全局锁定，这时候需要等tx1提交全局事务并释放全局锁。
>
> > 如果这时候tx1需要回滚，就得获取本地锁，可是tx2在等待全局锁。(死锁)
> >
> > tx1的回滚会一直重试，直到 tx2 获取全局锁超时，放弃获取全局锁，回滚本地事务并释放本地锁。
> >
> > tx1才会拿到本地锁完成回滚。
>
> 这样来保证不会脏写。

## 读隔离

数据库本地隔离级别 读已提交 Read Committed 以及以上，Seata默认读未提交。

> 如果特定场景需要全局 读已提交，则Seata会使用 select for update语句的代理的方式。
>
> > for update 的语句会申请全局锁，如果申请不到就回滚释放本地锁，并重试。
>
> 这样来达到读已提交的隔离级别。