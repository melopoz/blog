---
title: Mysql
tags: 
  - todo
  - 索引
  - 事务
  - mvcc
  - buffer pool
categories: DB
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



## 索引

InnoDB 唯一主键，B+树（只有叶子节点存数据，普通索引的叶子节点不存数据，存的是主键索引，再通过主键索引(聚簇索引树)找到数据就是‘回表’）

聚簇索引 （InnoDB）

辅助索引（普通索引）

联合索引

InnoDB不支持哈希索引，只有一个自适应哈希索引

#### 最左匹配原则

如果建立三个字段的联合索引`index_a_b_c`，查询时`...where c = ".." and b = ".." and a = ".."`也是会生效的，因为mysql的优化器会对where条件的字段进行重排序。



### InnoDB自适应哈希索引（Adaptive Hash Index，AHI）

一条查询语句如果使用的是普通索引并需要回表，可能会因为无序导致多次随机IO。

InnoDB监控对表的索引页的查询，如果**满足要求**就会建立哈希索引。

这个哈希索引是通过缓存中的索引页构建的，而且也是放在buffer poll中，所以速度很快。

> 查询：`show variables like '%ap%hash_index';`      --innodb_adaptive_hash_index
>
> 打开/关闭自适应哈希索引： `set global innodb_adaptive_hash_index=off / on`

哈希索引嘛，特点都一样：只能用等值查询；不能排序；有可能冲突

要求：**对这个页的访问模式是一样的**。

> 例如对（a，b）这个联合索引可能会有两种访问模式：A:`where a = ?`、B:`where a = ? and b = ?`，
>
> 如果交替进行A、B两种查询，则不会构造AHI；
>
> 必须以下方两种方式访问才会构造AHI。
>
> > 1. 以A / B模式访问了100次
> > 2. 页通过某一种模式被访问了`rows / 16`次

自适应哈希索引不可以人为干预，



### 索引注意事项

1. 给千万级以上的数据量过大的表添加索引

   > 先把原表结构复制出`t_copy`，在`t_copy`中添加索引，然后将原表中的数据导入新表`t_copy`，然后重命名这两个表

2. 联合索引设计

   > 常见的一些区分度不大的字段，比如`status`、`sex` 等等，在数据量很大的情况下，也可以添加索引，放再联合索引的左侧，不过为了让联合索引能生效，无论是否筛选`status`、`sex`都要在where条件语句中添加`status = ？ and  sex = ？`，如果不需要筛选就用`sex in (1,2)`即可。视情况而定

3. 





## mysql缓存：buffer pool

mysql 会在内存中存放： 

- 数据页(sql 必须完全一致才会命中缓存)；

- 索引页；

- 锁信息   lock info；

- 插入缓冲 insert buffer；(B+树实现)（非唯一 & 辅助索引）

  > 对于非聚集索引的insert/update，先判断插入的非聚集索引页是否在缓冲池(缓存)中，若存在则直接插入；若不存在，则先放到insert buffer中，等待**checkpoint**机制将 insert buffer 和 辅助索引页的子节点进行merge；
  >
  > > 这通常可以将多个插入合并到一个操作中（因为在一个索引页中），提高了对非聚集索引的插入性能。

- undo log；

- 自适应哈希索引 


> 其实mysql在缓存后边写入磁盘也是交给OS，OS还会有一层OS Cache，OS什么时候把cache刷到硬盘可就不知道了。。



## 分区

mysql5.7自动就能搞定，相当于分表，而且比分表方便

> # todo



## mysql 后台线程

`Master Thread`：处理用户查询线程；~~将缓冲池中的数据刷新回磁盘，保证数据一致性~~，~~以及undo页回收(交由Page Cleaner Thread线程)~~；

`IO Thread`：负责io请求的回调；(write x4、read x4、insert buffer thread、log IO thread)

`Purge Thread`：回收已经使用并分配的undo log页；

`Page Cleaner Thread`：1.2.x版本引入，专门负责脏页刷新回磁盘，减轻master thread压力



## Checkpoint

把页从缓冲池刷新回磁盘的机制

作用： 1 缩短数据库的恢复时间；2 缓冲池不够用了就将脏页刷新到磁盘；3 重做日志不可用的时候，刷新脏页



## 脏数据 、 脏页

脏页 是已提交的事务修改的数据更新到内存中，还未`fsync`到磁盘中；

脏数据 是其他事务未提交的数据；

fsync：同步信号



## MVCC 快照机制 

> 多版本并发控制

可以省去锁的开销。

mvcc的实现就是在每行添加两列（insertTime，deleteTime）

每个事务在begin的时候会获取当前的系统版本号curTime，

- select：只查询 `insertTime <= curTime` & `deleteTime > curTime` 的记录；
- insert：添加一行记录并 `set insertTime = curTime`；
- delete：对该行 `set deleteTime = curTime`；
- udpate：新增一条记录，`set insertTime = curTime`；对之前的记录 `set deleteTime = curTime`



## InnoDB事务的实现

事务commit时，必须实现将事务的所有日志（redo + undo）写入到日志文件进行持久化。

### redo log -- 保证事务持久性D

是**物理日志**，存放在日志文件中；恢复提交事务修改的页操作；记录的是页的物理修改操作。

- 将事务的执行日志**顺序**(基本上都是顺序写)记录下来，数据库运行的时候是不需要读redo log的。

- 每次将重做日志写入到日志文件之后都要调用`fsync`以确保日志写入磁盘，`fsync`的效率取决于磁盘性能，所以这等于放大了 磁盘性能对 数据库执行修改操作时的性能 的影响。

### undo log -- 帮助事务进行回滚及MVCC功能

是**逻辑日志**存放在数据库内部的一个**undo段**（**undo segment**，位于共享表空间），回滚行记录到某个特定版本

> 可通过 py_innodb_page_info.py 工具查看到 `undo Log Page: 1234`

- 事务的回滚就是通过undo log 实现的

  > 比如 事务中执行了 insert，事务回滚的时候就会执行 delete，逻辑取消事务中执行的的修改操作。

  > 通过undo log，数据是恢复了，但是数据结构和页本身可能相差很大。（因为不能影响其他事务在这个空间的操作啊）
  >
  > > 比如执行 insert 10000条数据，表空间会增大，rollback之后，表空间并不会恢复到原来的大小。

- 实现MVCC的非锁定读取

  > 当这个事务中读取一行记录，但该记录已经被其他事务占用，则当前事务可以通过undo读取该行之前版本的信息。

- 由于undo log也需要持久性保护，所以生成undo log的时候会伴随生成redo log

  > 毕竟undo log也是在表中存的数据，是逻辑日志

### purge -- 将缓冲池中的数据刷回磁盘时进行merge

delete操作可能并不会直接删除原有数据

因为InnoDB支持MVCC，所以数据不能再事务提交之后就立即处理（其他事务可能也在引用这行数据）。

在最后purge的时候才会真正完成 delete、update操作，update也是删除之前版本的那‘行’记录



## InnoDB的锁

https://www.cnblogs.com/rjzheng/p/9950951.html

> `lock in share mode`、`for update`

`Record Lock`行锁

`Gap Lock`间隙锁（只在可重复读和序列化执行的事务隔离级别下才有）

`Next-Key Lock`       InnoDB对于行的查询都是采用这个锁定算法

> 如果筛选条件为`where id = 10`，id是主键，则不会加间隙锁，Next-Key Lock会降级为Record Lock，只锁定id=10，此时insert id = 9 是不会阻塞的

`Repeatable`隔离级别下，如果where条件是没有索引的列，会在每行加锁、每个间隙加间隙锁，所以像是锁表。

`RU`和`RC`级别下，如果where条件是没有索引的列，也只会加行锁

### InnoDB解决幻读：

使用 `Next-key Lock` (`Record Lock`+`Gap Lock`)机制

> 锁定一个范围和该记录本身

幻读就是： (幻象问题)

> 事务A 执行 `select * from user wehre age > 20 for update` ... 还未完成；
>
> 与此同时 事务B执行`insert into user  (...(age=)25...)` 并提交 已完成。
>
> 事务A 再次执行`select * from user wehre age > 20 for update` 会返回刚才事务B插入的数据。



Next-Key Lock 会把范围 `age: (20, +∞)` 加锁，以至事务B不能执行该insert。

如果where的是唯一索引 就会在索引上加锁。





# 其他

### join

mysql优化器会让小表驱动大表，因为底层也是两次遍历比较，原理相当于两层for循环，如果使用BTree，查找一条数据的时间复杂度=log(n)，所以两种for嵌套的时间复杂度比较如下：

> 大表.rows * log(小表.rows)      肯定>    小表.rows * log(大表.rows)
>
> > B+Tree 的最大深度 log(n)，n:节点数，而且是B+Tree，每次查询肯定都要找到叶子节点。所以肯定是log(n)

但要注意，join on 的条件中，两个表的连接字段必须字符集相同&类型相同。否则索引是不会生效的。

而且join后的sql只有非驱动表的索引会生效。

left join 左边的表是驱动表，但是驱动表的索引不会生效，比如

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/leftjoin%E5%B7%A6%E8%A1%A8%E9%A9%B1%E5%8A%A8%E6%97%A0%E6%95%88.png" style="zoom:50%;" />



### 自连接的应用

类似排名这种逻辑，可以用自连接来做。比如   `某学生在班级中某科目的分数排名`   ，自连接的时候用count()看比自己分高的同学的数量即可得到。要注意   **distinct** 去重。

> 应用：leetcode [185. 部门工资前三高的所有员工](https://leetcode-cn.com/problems/department-top-three-salaries/)

```mysql
select d.Name Department, e.Name Employee, e.Salary Salary
from Employee e left join Department d
on e.DepartmentId = d.Id
where e.id in (
    select e1.Id from Employee e1 left join Employee e2 
    on e1.DepartmentId = e2.DepartmentId and e1.Salary < e2.Salary
    group by e1.id
    having count(distinct e2.Salary) < 3
)
and e.DepartmentId in (select DepartmentId from Department)

-- 你测试用例来个Department表是空的可太秀了  淦
```



### 数字相关函数

`trancate(表达式，保留n位小数)`    不会四舍五入

`round(表达式，保留n位小数)`    会四舍五入

> 应用：leetcode [262. 行程和用](https://leetcode-cn.com/problems/trips-and-users/)

```mysql
select Request_at as Day, 
	round(count(if(Status != 'completed', Status, null)) / count(Status) , 2) as "Cancellation Rate"
from Trips t
	inner join Users u on t.Client_id = u.Users_id and u.Banned = "No"
	inner join Users u2 on t.Driver_id = u2.Users_id and u2.Banned = "No"
and Request_at < "2013-10-04" and Request_at >= "2013-10-01"
group by Request_at
```



### 总结MySQL查询的一般性思路：

- 能用单表优先用单表，即便是需要用group by、order by、limit等，效率一般也比多表高

- 不能用单表时优先用连接，连接是SQL中非常强大的用法，小表驱动大表+建立合适索引+合理运用连接条件，基本上连接可以解决绝大部分问题。但join级数不宜过多，毕竟是一个接近指数级增长的关联效果

- 能不用子查询、笛卡尔积尽量不用，虽然很多情况下MySQL优化器会将其优化成连接方式的执行过程，但效率仍然难以保证

- 自定义变量在复杂SQL实现中会很有用，例如LeetCode中困难级别的数据库题目很多都需要借助自定义变量实现

- 如果MySQL版本允许，某些带聚合功能的查询需求应用窗口函数是一个最优选择。除了经典的获取3种排名信息，还有聚合函数、向前向后取值、百分位等，具体可参考官方指南。以下是官方给出的几个窗口函数的介绍：

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/mysql8.0-window%20functions.png" style="zoom:67%;" />


最后的最后再补充一点，本题将查询语句封装成一个自定义函数并给出了模板，实际上是降低了对函数语法的书写要求和难度，而且提供的函数写法也较为精简。然而，自定义函数更一般化和常用的写法应该是分三步：

定义变量接收返回值
执行查询条件，并赋值给相应变量
返回结果







# 大大的 TODO

# TiDB

直接把mysql dump下来，加载到TiDB，非常简单地进行扩容

> # todo



# 分库分表

https://blog.csdn.net/J080624/article/details/86476027

> # todo



# 数据备份 监听mysql的binlog

> # todo



# infobright

> # todo

