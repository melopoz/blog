---
title: Redis
tags: 
  - DB
  - 缓存
  - 分布式锁
  - Cluster集群
  - redis常见问题
  - 选举策略
categories: 
  - [NoSQL]
  - [分布式]
---



配置文件示例

http://download.redis.io/redis-stable/redis.conf

参考

https://www.cnblogs.com/kismetv/p/9137897.html

---



### 本地缓存和分布式缓存

> 本地缓存，比如java中的map，guava实现，轻量、快速，但是如果有多个实例，就需要每个实例都保存一份缓存，不具有一致性。

>  分布式缓存，比如redis、memcached，缓存具有一致性。但是需要保持服务的高可用。

---



### redis VS memcached

1. redis数据类型丰富，memcached只支持string；
2. redis支持数据持久化，可以在宕机、重启之后进行数据恢复。memcached只能存储在内存；
3. redis原生支持集群cluster，memcached没有原生集群模式；
4. redis使用单线程的IO多路复用模型，memcached是**多线程**、非阻塞的IO复用的网络模型。redis6也使用了多线程IO

---



### redis 工作模式

多个cli连接server

> redis需要处理cli的命令(get,set,lpush,rpop...)，对`接收客户端链接，处理请求，返回命令结果`等任务，redis使用主进程和主线程完成。
>
> redis后台还有RDB持久化，AOF重写等任务，这些任务会启动子进程来完成。

---



### redis 线程模型

https://www.jianshu.com/p/8f2fb61097b8 这个图更清楚一点

> redis内部使用文件事件处理器（file event handler），这个handler是单线程的，所有请求都要经过这个入口才能被处理，自然就有了先后顺序，所以说是单线程模型。

> 在这个模型中，Redis 服务器用主线程执行 I/O 多路复用程序、文件事件分派器以及事件处理器。而且，尽管多个文件事件可能会并发出现，Redis 服务器是顺序处理各个文件事件的。        ——《Redis设计与实现》

> IO多路复用的 Reactor 模式

master线程监听多个socket，将产生的事件加入任务队列，事件分派器从队列中取出事件并执行，这也就保证按顺序执行。

![redis处理客户端一个命令的过程](https://raw.githubusercontent.com/melopoz/pics/master/img/redis%E5%A4%84%E7%90%86%E4%B8%80%E6%AC%A1%E5%91%BD%E4%BB%A4%E8%BF%87%E7%A8%8B.png)

---



### redis6线程模型

采用多线程处理IO(默认不开启)，事件处理器仍然是单线程。

> ```
> io-threads-do-reads no #yes开启  多线程
> io-threads 4 #配置<=cpu核数性能最高，之前单线程就是只能谁用一个cpu，现在如果线程数量过多增加了线程上下文切换的话性能可能不会提升
> #官方建议：4核的机器建议设置为2或3个线程，8核的建议设置为6个线程，线程数一定要小于机器核数。
> #官方认为线程数并不是越大越好，超过了8个基本就没什么意义了
> ```

IO线程只能是同时都在读 或者 同时都在写；

IO线程时负责读写socket解析命令，命令处理还是通过单线程的文件事件处理器处理；

> 1. 主线程把准备好了的socket放到等待队列，然后阻塞等待IO线程去处理socket（生成对应的事件push到队列）
>
>    > 感觉就相当于 用多线程IO同时对一批socket进行处理，以提高QPS
>
> 2. 主线程顺序处理事件（交由对应的处理器处理），处理结果再交给IO线程写到对应的socket
>
> 3. 等待IO线程的写操作都完成，解除IO线程和等待队列中的socket的绑定，清空等待队列（socket都处理完了嘛）
>
> 4. 再走1   这么个循环。

> todo！ 看看redis源码吧   https://github.com/redis/redis.git       git@github.com:redis/redis.git

可以看看这篇文章：https://ruby-china.org/topics/38957%EF%BC%89

---



### fork子线程时 主线程阻塞

#### 直观感受

在Redis的实践中，众多因素限制了Redis单机的内存不能过大，例如：

- 当请求暴增，需要扩容时，Redis内存过大会导致扩容时间太长；
- 当主机宕机时，切换主机后需要挂载从库，Redis内存过大导致挂载速度过慢；
- 以及持久化过程中的`fork`操作。

#### 原理

父进程通过`fork`操作可以创建子进程；

子进程创建后，与父子进程共享代码段，不共享进程的数据空间，所以子进程会获得父进程的数据空间的副本。

> 在操作系统`fork`的实际实现中，基本都采用了写时复制技术：**copy on write**
>
> > 即在父/子进程试图修改数据空间之前，父子进程实际上共享数据空间；
> >
> > 但是当父/子进程的任何一个试图修改数据空间时，操作系统会为修改的那一部分(内存的一页)制作一个副本。

> 虽然`fork`时，子进程不会复制父进程的数据空间，只是会复制内存页表（页表相当于内存的索引、目录）

父进程的数据空间越大，内存页表越大，所以fork时复制耗时也会越多。

#### 影响

在Redis中，无论是RDB持久化的`bgsave`，还是AOF重写的`bgrewriteaof`，都需要fork出子进程来进行操作。

如果Redis内存过大，会导致fork操作时复制内存页表耗时过多，而Redis主进程又是单线程的，在进行fork时，是完全阻塞的，也就意味着无法响应客户端的请求，会造成请求延迟过大。

#### 解决方案

1. 限制单机内存；
2. 适度放宽AOF重写的触发条件；
3. 砸钱（硬件搞强一点）
4. 非要用虚拟机就用使用Vmware或KVM虚拟机，不要使用Xen虚拟机

对于不同的硬件、不同的操作系统，fork操作的耗时会有所差别，一般来说，如果Redis单机内存达到了**10GB**，fork时耗时可能会达到**百毫秒级别**（如果使用Xen虚拟机，这个耗时可能达到秒级别）。

> 因此，一般来说Redis单机内存一般要限制在10GB以内；不过这个数据并不是绝对的，可以通过观察线上环境fork的耗时来进行调整。
>
> > 执行命令`info stats`，查看`latest_fork_usec`的值，单位为微秒。

---



### redis 常用命令

- **del**、**exists**、**type**

- redis **expires**

  在redis中添加元素的时候设置过期时间：set key value 存活时间

- **expire**  重新设置key的存活时间

- **persist** 去掉一个key的过期时间，使之成为持久化key

- **ttl**  以秒为单位，返回 key 的剩余生存时间

- **rename** 对一个key改名，之后存活时间继续计时

- **setnx**  不存在就插入

```shell
localhost:6379> config get auto-aof-rewrite-min-size
1)"auto-aof-rewrite-min-size"
2)"67108864"

localhost:6379> info persistence  #持久化信息
1) aof_current_size:149
2) aof_base_size:149
```





### 数据类型-操作

#### string

`set`、`mset`、`get`、`mget`、`getset`、`strlen`、`incr`、`incrby`、`decr`、`decrby`（原子增量）



#### list

`lpush`、`rpush`、`lpop`、`rpop`、`llen`  查看list中元素的个数

> ```shell
> > rpush mylist 1 2 3 4 5 "foo bar"
> (integer) 9
> ```

`lrange [key] [indexStart] [indexEnd]` 遍历get

> 两个索引参数都可以为负，-1是列表的最后一个元素，-2是列表的倒数第二个元素，依此类推。
>
> ```
> > lrange mylist 0 -1
> 1) "first"
> 2) "A"
> 3) "B"
> 4) "1"
> 5) "2"
> 6) "3"
> 7) "4"
> 8) "5"
> 9) "foo bar"
> ```

`ltrim [key] [indexStart] [indexEnd]` 修剪

> 仅保留`indexStart`到`indexEnd`的元素并返回，范围之外的全都丢弃。

`brpop [key] [timeout]`、`blpop ..` 

> b：blocked阻塞
>
> ```
> 127.0.0.1:7963> brpop list1 list2 5
> (nil)
> (5.06s)
> ```

`lpoprpush [key1] [key2]`   转移列表的数据

`linsert`  插入到指定位置 

`linsert [list1] [before/after] [v1] [v2]`  在v1前边插入一个v2，返回新len，如果a不存在返回-1

使用场景

> - 用户的最新动态（最近5、10条）
> - 队列（不安全）

注意事项

> 因为不能对集合中每项都设置TTL，但是可以对整个集合设置TTL。**所以，我们可以将每个时间段的数据放在一个集合中。然后对这个集合设置过期时间。**



#### hash

命令：`hset`，`hget`，`hmset`，`hmget`，`hgetall`，`hexists`，`hsetnx` 见名知意吧



#### set

`sadd [key] [v1] [v2] ...`

``spop [key]`

`sismember [key] [value]`  set中是否存在这个value，返回 1 / 0

`smembers [key]`   所有元素

`srandmember [key] [n]`  随机n个元素，比如随机两个  `srandmember setdemo 2`

`sunion [key1] [key2]`   并集



#### zset  

sortedset，增加了权重参数score，使集合中的元素按照score有序排列

```sh
zadd zset 1 one  #如果one已经存在，就会覆盖原来的分数
zadd zset 2 two  
zadd zset 3 three
zincrby zset 1 one #增长分数
zscore zset two #获取分数
zrange zset 0 -1 withscores #范围遍历
zrangebyscore zset 10 25 withscores #指定范围的值
zrangebyscore zset 10 25 withscores limit 1 2 #分页
Zrevrangebyscore zset 10 25 withscores  #指定范围的值
zcard zset  #元素数量
Zcount zset #获得指定分数范围内的元素个数
Zrem zset one two #删除一个或多个元素
Zremrangebyrank zset 0 1 #按照排名范围删除元素
Zremrangebyscore zset 0 1 #按照分数范围删除元素
Zrank zset 0 -1 #分数最小的元素排名为0
Zrevrank zset 0 -1 #分数最大的元素排名为0
Zinterstore #
zunionstore rank:last_week 7 rank:20150323 rank:20150324 rank:20150325  weights 1 1 1 1 1 1 1
```

---



#### hash扩容：渐进式扩容

hash字典的初始容量为4（3.2.8版本是这样）

redis的hash类型在    **元素数量与数组size相同时（且没有在进行bgsave）** 或  **元素数量是数组size的5倍（不论是否在bgsave）**的时候就会开始扩容，新数组的size为元素数量*2。

redis hash 维护一个rehashidx来表示重新hash的索引，默认值为 -1，如果 >=0 说明开始 rehash 了，则每次对这个hash进行操作的时候将 rehashidx 处的元素进行rehash，当idx进行到最后的时候再置为-1结束扩容。这样在rehash过程中也能正常提供服务。



#### hash缩容

当hash数组中的元素数量所占比例小于负载因子（0.1），不论是否正在bgsave或者bgwriteaof，都会进行缩容

---



### 删除过期key

> 定期删除+惰性删除

- 定期删除（redis默认100ms）

  redis默认每隔100ms就**随机抽取**一些设置了过期时间的key，检查并删除。

  > 随机抽取的原因：如果key数量巨大，每隔100ms遍历所有设置过期时间的key，可能严重增大CPU的负载。

- 惰性删除

  redis的定期删除可能导致一些key没有及时删除。如果一个key已经过期但还留在内存，只有查到了这个key，这个key才会被删除。

如果redis的定期删除漏掉了很多过期的key，并且没有及时查这些key，就会浪费内存。解决这个问题就需要**redis内存淘汰机制**。

---



### 内存淘汰机制

>  数据库中有2000w数据，redis中只存20w数据，如何保证redis中的数据是热点数据？

#### 8种数据淘汰策略：

当内存不足以容纳新写入数据时，

1. volatile-lru   ```从 已设置ex的数据集中 移除 最近最少使用的key```
2. volatile-random   ```从 已设置ex的数据集中 移除 随机key```
3. volatile-lfu   ```从 已设置ex的数据集中 移除 最不经常使用的 key ```
4. volatile-ttl   ```从 已设置ex的数据集中 优先移除有更早过期时间的key ```
5. allkeys-lru   ```从 键空间 移除 最近最少使用的key```
6. allkeys-random   ```从 键空间 移除 随机key```
7. allkeys-lfu   ```从 键空间 移除 最不经常使用的 key```
8. no-eviction   ```禁止淘汰，内存不足直接报错。```

>  注 7、8 为 Redis 4.0 新增。
>
>  **volatile**为前缀的策略都是从**已过期的数据集**中进行淘汰。
>
>  **allkeys**为前缀的策略都是面向**所有key**进行淘汰。
>
>  **LRU**（least recently used）最近最少用到的。
>
>  **LFU**（Least Frequently Used）最不常用的。

---



### 持久化机制

redis支持持久化，这是redis不同于memcached很重要的一点。

#### 1. 快照 snapshotting （RDB）（默认）

在redis.conf中有如下配置

```properties
save 900 1		#在900s(15min)之后，至少1个key发生变化，自动触发BGSAVE命令创建快照
save 300 10		#在300s(5min)之后，至少10个key发生变化，snapshot
save 60 10000	#在60s(1min)之后，至少1w个key发生变化，snapshot
```

#### 2. 追加文件 append-only （AOF）

> 特点：实时性，数据全

##### 开启AOF

在redis.conf中配置```appendonly yes```

> 开启AOF之后，redis会将每条会更改redis中的数据的命令写入硬盘的aof文件，aof文件位置和rdb文件相同，通过dir参数设置，默认文件名是```appendonly.aof```。

##### 三种不同的AOF方式：

在redis.conf中

```properties
appendfsync always		# 每次发生数据修改都会写入AOF，(严重降低redis的速度)
appendfsync everysec	# 每秒同步一次
appendfsync no			# 由操作系统决定何时同步
```

#### 3. 混合持久化 RDB+AOP 

> 4.0开始支持， 默认关闭。 通过配置项  ```aof-use-rdb-preamble```  开启。

混合持久化在AOF重写的时候把RDB的内容写入到aof文件的开头。

> 优点：重启之后恢复加载更快，避免丢失过多数据
>
> 缺点：aof文件中的rdb部分的压缩格式不再是aof格式，可读性差，aof文件可能过大。

在执行GBREWRITEAOF命令时，redis服务器维护一个aof重写缓冲区，并开启一个子进程**重写AOF**，在子进程工作期间，将所有命令记录到缓冲区，当子进程创建完aof文件之后，将缓冲区的内容追加到新aof文件末尾，使新aof文件和数据库状态一致，最后用新aof文件替换旧aof文件。

> 命令：```BGREWRITEAOF```  ```bgrewriteaof```
>
> 解决AOF文件体积过大的问题，用户可以使用这个命令让redis重写aof文件（手动rewrite）。

##### 重写AOF/压缩AOF：（目的是减小AOF文件体积）（手动触发、自动触发）

> aof文件会越来越大，aof重写是**从redis服务器中的数据**转化为写命令存到新的aof文件中，**不会读旧的aof文件**，所以**过期的数据不再写入aof**，**无效的命令不再写入aof**，**多条命令可能合并成一个（注）**。
>
> > **注**
> >
> > 不过为了**防止单条命令过大**造成客户端缓冲区溢出，对于list、set、hash、zset类型的key，并不一定只使用一条命令；而是以某个常量为界将命令拆分为多条。这个常量的配置为
> >
> > ```define REDIS_AOF_REWRITE_ITEMS_PER_CMD 64```

#### AOF配置

```properties
#是否开启AOF
appendonly no 
#AOF文件名
appendfilename "appendonly.aof"
#RDB文件和AOF文件所在目录
dir ./
#fsync持久化策略
appendfsync everysec
#AOF重写期间是否禁止fsync；如果开启该选项，可以减轻文件重写时CPU和硬盘的负载（尤其是硬盘），但是可能会丢失AOF重写期间的数据；需要在负载和安全性之间进行平衡
no-appendfsync-on-rewrite no
#如果AOF文件结尾损坏，Redis启动时是否仍载入AOF文件
aof-load-truncated yes

#执行AOF重写时，文件的最小体积，默认值为64MB。文件重写触发条件之一
auto-aof-rewrite-percentage 100
#执行AOF重写时，当前AOF大小(即aof_current_size)和上一次重写时AOF大小(aof_base_size)的比值。文件重写触发条件之一
auto-aof-rewrite-min-size 64mb
#只有当auto-aof-rewrite-min-size和auto-aof-rewrite-percentage两个参数同时满足时，才会自动触发AOF重写，即bgrewriteaof操作。
```

---



### 缓存穿透

穿透  `同时大量请求一个不存在的key`

> 一个**不存在**的key，缓存不会起作用，请求直接打到DB，如果流量大，DB危险

解决方案：

> 1. 严格参数验证。例如id<0直接拦截。
>
> 2. 如果DB查询key也不存在，就缓存key=null,expires=较短时间。可以防止用这个id反复请求的暴力攻击。
>
> 3. 根据业务情况，使用**布隆过滤器**，如果key根本不可能存在，直接拦截。
>
>    3+2更安全，因为布隆过滤器有一定误判率，只能说key可能存在。

### 缓存击穿

击穿  `同时大量请求一个存在但是失效的key，这个key失效`

> 一个**存在**的key，在缓存过期的一刻，同时大量请求这个key，这些请求都会打到DB

解决方案：

> 1. 设置热点数据永不过期（因为大量请求说明可能是热点数据）。
>
> 2. 加锁。
>
>    ```java
>    public V getData(K key) throws InterruptException {
>        V v = getDataFromCache(key);
>        if (v == null){ // cache未命中
>            if (reentrantLock.tryLock()){ // 获取DB锁，如果能细化到key更好
>                try {
>                    v = getDataFromDB(key);
>                    if (v != null) { // DB中有数据
>                        setDataToCache(key, v); // 同步到cache 
>                    } else {
>                        //如果key不存在,并且请求也很多,都走这个同步可能服务超时,不同步可能会缓存穿透,可以在cache设置key=null,expires=30s.
>                    }
>                } finally { // 释放锁
>                    reentrantLock.unlock();
>                }
>            } else { // 拿不到锁就过一会重新拿
>                Thread.sleep(1000);
>                v = getData(key);
>            }
>        }
>        return v;
>    }
>    ```
>
>    

### 缓存雪崩

> 缓存中的数据**同时大面积失效**，就会有大量请求打到数据库

解决方案：

> 1. 热点数据永不过期
> 2. 失效时间设置随机

具体：

事前：保证redis集群的高可用，发现宕机尽快补。

事发：本地缓存+hystrix限流&服务降级，保证DB正常运行

事后：利用持久化机制尽快恢复缓存

---



### Redis可实现功能

#### 缓存

...



#### 分布式锁

**加锁**

```java
public static boolean tryGetDistributedLock(
    String lockKey, String requestId, int expireTime) {
	String result = jedis.set(
        // key， value为请求id 解锁还需要这个用这个id， setnx 不存在才set， 失效时间
        lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
    if (LOCK_SUCCESS.equals(result)) {
        return true;
    }
    return false;
}
```

**解锁**：要注意不能释放别人的锁，所以需要CAS操作，这个操作需要利用lua脚本

> redis在执行lua脚本相当于执行cpu原语，是个原子操作。  详见 [Lua 脚本 原子执行](#Lua 脚本 原子执行)  



##### 你以为lua脚本就稳了吗？

Redis 单点的情况确实没问题。

如果是在[哨兵模式](#哨兵模式)中, A 拿到锁 `set lockKey lockValue`命令只在 master 节点执行完成，还没有同步到 slave 的时候， master 挂了，集群将重新选举 master ， 然后 B 再试图拿锁， 也会成功。  这就出错了...   



##### 解决 - RedLock

redlock算法 ， 用于多个 redis 实例的场景， https://blog.brickgao.com/2018/05/06/distributed-lock-with-redlock/

> **加锁**：向多半的节点发送 `setnx lockKey lockValue` 命令， 过半节点成功才算加锁成功

> **解锁**：向全部节点发送 `del` 命令

> 这是一种基于【大多数都同意】的一种机制      又想起了 paxos？https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95

已有的 RedLock 开源实现：python: redlock-py，  java: **Redisson**



#### 附近的人-空间搜索

https://developer.aliyun.com/article/515466

> GeoHash是一种地址编码方式。能够把**二维空间经纬度**数据编码成一个**字符串**。  **redis3.2新增**
>
> 1. 字符串越长，表示的范围越精确。编码长度为8时，精度在19米左右，而当编码长度为9时，精度在2米左右。
> 2. 字符串相似的表示距离相近，利用字符串的前缀匹配，可以查询附近的地理位置。这样就实现了快速查询某个坐标附近的地理位置。
> 3. geohash计算的字符串，可以反向解码出原来的经纬度。
>
> 当所需存储的对象数量过多时，可通过设置多key(如一个省一个key)的方式对对象集合变相做sharding，避免单集合数量过多。
>
> Redis内部使用有序集合(zset)保存位置对象，元素的score值为其经纬度对应的52位的geohash值。

#### 命令

**geoadd** key longitude latitude member [longitude latitude member] []...

添加地理位置，返回integer    [经度 维度 成员]

```shell
127.0.0.1:6379> geoadd BeiJing-area 116.2161254883 39.8865577059 ShiJingShan 116.1611938477 39.7283134103 FangShan
```

**geopos** key member [member] []...

获取地理位置的坐标，可以批量

```shell
127.0.0.1:6379> geopos Beijing-areas ShiJingShan FangShan
1)  1) "116.21612817049026489"
	2) "39.88655846536294547"
2)  1) "116.16119652986526489"
	2) "39.72831328866426048"
```

**geodist**  key member1 member2 [unit]

计算两个位置之间的距离，通过已存在的KEY下的2个位置计算距离，单位的距离有：m米 km千米 mi英里 ft英尺

```shell
127.0.0.1:6379> geopos Beijing-areas ShiJingShan FangShan km
1)  12313123232
```

**geohash** key member [member] [] ...

该命令返回11个字符的 Geohash 字符串，没有任何精度损失。缩短删除右侧的字符。它会失去精确度，但仍会指向同一区域。

```shell
127.0.0.1:6379> GEORADIUS Sicily 15 37 200 km WITHCOORD WITHDIST # Sicily[15,37] 半径200km 打印坐标 打印距离
1) ["Palermo","190.4424",["13.361389338970184","38.115556395496299"]]
```

**georadius ** key [longitude] [latitude] [radius] [m|km|ft|mi] [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]

当前位置附近的所有位置 georadius

```
127.0.0.1:6379> geopos Beijing-areas ShiJingShan FangShan km
1)  12313123232
```

**georadiusbymember ** [key] [member] [半径] [单位]

显示已添加的某个位置 为中心点的距离多少范围内的信息

> **georadiusbymember ** [key] [member] [半径] [单位] COUNT 1 DESC ---withcoord(显示经纬度) --withdist(展示距离)
>
> count 1 desc 展示最近的一个

```shell
127.0.0.1:6379> georadiusbymember beijing shijingshan 50000 m #beijing shijingshan 50km内的所有位置
1) "shijingshan"
2) "tongzhou"
3) "fangshan"
4) "daxing" 
```



#### 签到记录 / PV / UV

##### 利用 string - bitmap 结构

https://oss.redislabs.com/redisbloom/  这里有一个实现

> 位图不是实际的数据类型，而是在字符串类型上定义的一组面向位的操作，最大512M，所以最多存储`2^32`位
>
> 内存占用：(最大偏移量$offset/8/1024/1024)MB        bit->byte->kb->Mb

`setbit [key] [offset] [value]`

`getbit [key] [offset]`

`bitcount [key]`

`bitop`

> - 用户签到，日期是主维度
>
>   key:`prefix_活动名称_日期`，offset：`用户id`，value：`是否已签到`
>
>   用一个bitmap代表一天的签到情况，将用户ID作为偏移量，通过`setBit`设置该位置的值为1，bitmap的key用`prefix+活动名称+时间`表示
>
>   `getBit`查询该位置是否为1，代表用户是否签到了
>
>   `bitCount`今日签到数量
>
> - 用户签到，用户是主维度
>
>   key：用户ID，offset：日期，value：是否已签到
>
>   > 这样用户太多，key很多，value倒是没多少位



#### 布隆过滤器

还需要自己准备好点的hash算法

拓展一下：布谷鸟过滤器 CuckooFilter   https://juejin.cn/post/6844903861749055502



#### 做消息队列

利用list结构，redis本身就提供了 两端的push和pop操作 和 列表的操作 

和MQ的区别：

> api不同；
>
> redis不能分组，不想kafka可以达到负载均衡的效果
>
> MQ中 rocketMQ和rabbitMQ满足AMQP，RocketMQ支持事务消息
>
> 毕竟不是消息队列，可靠性啥的一点都没得谈。



#### 限流

直接自增自减  Semaphore

---





## 集群策略

### 主从复制

master可以读、写；

slave只提供读服务，并接受master同步过来的数据。

##### 工作机制

> slave启动之后发送sync请求到master，master在后台保存快照和保存快照期间的命令，发送给slave。

##### 主从配置

master无需配置，修改slave节点的配置：

```properties
slaveof [masterIP] [masterPort]
masterauth [masterPassword]
```

连接成功后可以使用``info replication``查看其他节点的信息

##### 缺点

如果master宕机，不能自动将slave转换成master

https://www.cnblogs.com/L-Test/p/11626124.html

### 哨兵模式

哨兵模式**sentinel**，比较特殊。哨兵是一个独立的进程，独立运行，他监控多个redis实例。

##### 工作机制

哨兵的功能：

- 监控master和slave是否正常运行；
- master出现故障就将slave转化为master；
- 多个哨兵互相监控；
- 多个哨兵同时监控一个redis

##### 哨兵和redis之间的通信

https://blog.csdn.net/q649381130/article/details/79931791

##### 故障切换过程

如果被ping的节点超时未回复，哨兵认为其**主观下线**，如果是master下线，哨兵会询问其他哨兵是否也认为该master**主观下线**，如果达到（配置文件中的）quorum个投票，哨兵会认为该master**客观下线**，并选举领头哨兵节点发起故障恢复。

###### 选举领头哨兵 raft算法

> 发现master下线的A节点像其他哨兵发送消息要求选自己为领头哨兵
>
> 如果目标节点没有选过其他人（没有接收到其他哨兵的相同要求），就选A为领头哨兵
>
> 若超过一半的哨兵同意选A为领头，则A当选
>
> 如果多个哨兵同时参与了领头，可能一轮投票无人当选，A就会等待随机事件后再次发起请求

选出新master之后，会发送消息到其他slave使其接受新master，最后更新数据。已停止的旧master会降为slave，恢复服务之后继续运行。

###### 领头哨兵挑选新master的规则

> 优先级最高（slave-priority配置）
>
> 复制偏移量最大
>
> id最小

##### 哨兵配置  sentinel.conf

```properties
# 设置主机名称 地址 端口 选举所需票数
sentinel monitor [master-name] [ip] [port] [quorum]
# demo
sentinel monitor mymaster 192.168.0.107 6379 1
```

##### 启动哨兵节点

```sh
bin/redis-server etc/sentinel.conf --sentinel &
```

##### 查看指定哨兵节点信息

```sh
#可以在任何一台服务器上查看指定哨兵节点信息：
bin/redis-cli -h 192.168.0.110 -p 26379 info Sentinel
```



### Cluster集群

> 官方推荐集群至少要3台以上master，即 3 master 3 slave。

配置 redis.conf

```properties
#开启cluster模式
cluster-enable yes
#集群模式下的集群配置文件
cluster-config-file nodes-6379.conf
#集群内节点之间最长响应时间
cluster-node-timeout 15000
```

> 启动6个redis-server，可以借助ruby，或者自己写群起脚本



#### 工作机制

对key进行【crc16算法】【% 16384】的操作，通过计算结果找到对应插槽所对应的节点，然后直接跳转到这个节点进行存取操作。

为了保证高可用，使用主从模式。master宕机，启用slave。如果一个节点的主从都宕机，则集群不可用。

> 为什么slots数量使用16384 ？
>
> > 正常的心跳包携带一个节点的全部配置信息，其中就包含slots信息，16384bit = 2048byte  2k了，如果用65536bit的话，就需要8k
> >
> > 集群是每秒ping-pong，一旦集群的节点数量多了就会占用大量带宽
> >
> > https://www.cnblogs.com/rjzheng/p/11430592.html  这个dalao的文章有redis源码 心跳包消息结构



#### 特点

- 所有redis节点彼此互联（PING-PONG机制），使用二进制协议优化传输速度和带宽

  > 

- 集群中超过半数的节点检测失效才认为节点fail

  > master选举机制？ todo 

- redic-cil与节点直连，不需要中间代理层，任意连接集群中一个可用节点即可。

  > 如果客户端请求的key不在当前节点，会返回`xxx`消息，客户端再去连接该节点执行命令。



## 集群中key的分布

如果分布不均匀怎么办

一致性hash todo

消息队列尾部热点问题；
redis https://www.cnblogs.com/rjzheng/p/11430592.html
redis & mq  https://www.zhihu.com/question/43557507
redis https://zyfcodes.blog.csdn.net/article/details/90384502
redis 一致性hash  https://www.cnblogs.com/myseries/p/10959050.html



# Lua 脚本 原子执行

Lua 脚本在 Redis 中是原子执行的，Redis 在执行`EVAL`命令的时候，一直到执行完毕并返回结果之前，会阻塞所有其他客户端的命令，so Lua脚本中要写逻辑特别复杂的脚本， 必须保证 Lua 脚本的效率。



#### SCRIPT LOAD

加载脚本到缓存，以复用

```
...:6379> SCRIPT LOAD "return 'abc'"
“1b936e3fe509bcbc9cd0664897bbe8fd0cac101b”
...:6379> EVALSHA 1b936e3fe509bcbc9cd0664897bbe8fd0cac101b 0
"abc"
```

#### SCRIPT FLUSH

清除所有缓存， 不能筛选， 只能全删

#### SCRIPT EXISTS

```
...:6379> SCRIPT EXISTS 1b936e3fe509bcbc9cd0664897bbe8fd0cac101b  1b936e3fe509bcbc9cd0664897bbe8fd0cac1012
1) (integer) 1
2) (integer) 0
```

#### SCRIPT KILL

终止正在执行的脚本，如果脚本中已经执行了一部分写命令，则 kill 命令无效

若不对数据进行持久化， 可通过 shutdown nosave 来终止脚本...



#### 注意

- Lua 脚本执行异常也不会回滚， 所以脚本逻辑要有较高的健壮性
- Lua 脚本执行是原子性的，会阻塞其他客户端的命令，所有效率要高
- 在集群中使用 Lua 脚本的话要确保脚本中用到的 key 都在相同机器(相同的插槽slot)中，可用 Redis Hash Tags 技术
- 不要在脚本中用全局变量， 局部变量效率会高



##### Lua脚本 释放锁

```lua
if redis.call('get', KEYS[1]) == ARGV[1] 
    then 
	    return redis.call('del', KEYS[1]) 
	else 
	    return 0 
end
```

##### java实现

```java
private static final Long lockReleaseOK = 1L;
// lua脚本，用来释放分布式锁
static String luaScript = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end";
 
public static boolean releaseLock(String key ,String lockValue){
	if(key == null || lockValue == null) {
		return false;
	}
	try {
		Jedis jedis = getJedisPool().getResource();
		Object res =jedis.eval(luaScript,Collections.singletonList(key),Collections.singletonList(lockValue));
		jedis.close();
		return res!=null && res.equals(lockReleaseOK);
	} catch (Exception e) {
		return false;
	}
}
```











这个链接关于redis好全面。。

https://zyfcodes.blog.csdn.net/article/details/90384502