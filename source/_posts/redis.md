---
title: Redis
tags:
  - 位图
  - 缓存
  - 分布式锁
  - 一致性hash+虚拟槽slot
  - 选举策略
categories: 
  - [NoSQL]
  - [分布式]
---



配置文件示例

http://download.redis.io/redis-stable/redis.conf

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

master可以读、写；   slave只提供读服务，并接受master同步过来的数据。

> slave启动之后发送sync请求到master，master在后台保存快照和保存快照期间的命令，发送给slave。

master无需配置，只需修改slave节点的配置：

```properties
slaveof [masterIP] [masterPort]
masterauth [masterPassword]
```

连接成功后可以使用``info replication``查看其他节点的信息

如果master宕机，不能自动将slave转换成master，还得手动修改。客户端配置的redis地址可不一定方便修改。



### 哨兵模式

哨兵进程`sentinel`是一个独立的进程，负责监控多个redis实例

> - sentinel监控master和slave是否正常运行
> - master出现故障就将slave转化为master
> - 多个sentinel互相监控
> - 只有一个master
> - 客户端需要连接sentinel集群获取master实例

哨兵配置  sentinel.conf

```properties
# 设置主机名称 地址 端口 选举所需票数
sentinel monitor [master-name] [ip] [port] [quorum]
# demo
sentinel monitor mymaster 192.168.0.107 6379 1
```

启动哨兵节点

```sh
bin/redis-server etc/sentinel.conf --sentinel &
```

查看指定哨兵节点信息

```sh
#可以在任何一台服务器上查看指定哨兵节点信息：
bin/redis-cli -h 192.168.0.110 -p 26379 info Sentinel
```

故障切换过程

> 如果被ping的节点超时未回复，哨兵则认为其**主观下线**，如果是master下线，哨兵会询问其他哨兵是否也认为该master**主观下线**，如果达到（配置文件中的）`quorum`个投票，哨兵会认为该master**客观下线**，并选举出领头哨兵节点，领头哨兵发起故障恢复。

选举领头哨兵 raft算法

> 发现master下线的A节点向其他哨兵发送消息要求选自己为领头哨兵
>
> 如果目标节点没有选过其他人（没有接收到其他哨兵的相同要求），就选A为领头哨兵
>
> 若**超过一半的哨兵同意**选A为领头，则A当选
>
> 如果多个哨兵同时参与了领头，可能一轮投票无人当选，A就会等待随机事件后再次发起请求
>
> > 选出新master之后，会发送消息到其他slave使其接受新master，最后更新数据。
> >
> > 已停止的旧master会降为slave，恢复服务之后继续运行。

领头哨兵挑选新master的规则

> 优先级最高（slave-priority配置）；复制偏移量最大；id最小
>



### Cluster集群 ☆

为了保证高可用，使用主从模式。master宕机就启用slave。如果一个节点的主从都宕机，则集群不可用。

> 多个master(最少3个)，  多个slaver

配置 redis.conf

```properties
#开启cluster模式
cluster-enable yes
#集群模式下的集群配置文件
cluster-config-file nodes-6379.conf
#集群内节点之间最长响应时间  默认15s 关系到集群节点通信占用的带宽
cluster-node-timeout 15000
# 还会在加一个端口号用来集群之间节点的通讯  6379+10000  16379
```

启动集群 (需要Ruby版本 > 2.2)

> `redis-trib.rb create --replicas 1 [ip]:6380 [...] [ip]:6381 [ip]:6385 `

查看集群中的节点

```sh
[root@buke107 src]\# redis-cli -c -h 192.168.0.107 -p 6381 # 连接任一节点
192.168.0.107:6381> cluster nodes # 查看集群节点
# nodeId | ip:port | nodeType | masterId | 0？ | ？ | 连接数 | 节点对应的槽位slots(master节点才有)
868456121fa4e6c8e7abe235a88b51d354a944b5 192.168.0.107:6382 master - 0 1523609792598 3 connected 10923-16383
d6d01fd8f1e5b9f8fc0c748e08248a358da3638d 192.168.0.107:6385 slave 868456121fa4e6c8e7abe235a88b51d354a944b5 0 1523609795616 6 connected
5cd3ed3a84ead41a765abd3781b98950d452c958 192.168.0.107:6380 master - 0 1523609794610 1 connected 0-5460
b8e047aeacb9398c3f58f96d0602efbbea2078e2 192.168.0.107:6383 slave 5cd3ed3a84ead41a765abd3781b98950d452c958 0 1523609797629 1 connected
68cf66359318b26df16ebf95ba0c00d9f6b2c63e 192.168.0.107:6384 slave 90b4b326d579f9b5e181e3df95578bceba29b204 0 1523609796622 5 connected
90b4b326d579f9b5e181e3df95578bceba29b204 192.168.0.107:6381 myself,master - 0 0 2 connected 5461-10922
# 节点类型还会展示的信息  myself:当前节点   fail:节点下线
```

关于cluster的其他操作:   https://www.cnblogs.com/kevingrace/p/7910692.html

> `添加主/从节点`、`重新分配slot`、`查看集群状态信息`、`改变slave的master`、`删除主/从节点`
>
> `复制迁移`(登录到slave并更换其follow的master)、`cluster集群节点升级`(升级redis版本)
>
> `缓存清理`(在集群中删除指定的key需要链接到对应的节点...)，节点太多就用脚本吧
>
> > 查看cluster信息找到所有master
> >
> > 遍历master（连接该master，执行del key）直到del成功，结束遍历
> >
> > `再搜吧...`



#### 工作机制 原理

先了解一下分布式数据分布方案。

> 分布式 数据分布方案(寻址算法)：
>
> - 节点取余
>
>   根据数据(或数据的某一特征)计算hash并取余节点数量得到数据所在位置，比如Java的HashMap(强行举例...)。
>
>   > 缺点：当节点数量变化(扩容/缩容)时，数据节点映射关系需要重新计算，会导致大量数据的迁移(HashMap优化：翻倍扩容，可以使数据迁移从80%降到50%)
>
> - 一致性哈希
>
>   指定范围内（比如0-2^32^）所有数字首尾相连形成一个哈希环，并为每个节点分配一个该范围内的值，寻址时计算key的hash值，然后在环上找到第一个`token >= hash(key)`的节点。
>
>   > 优点：节点数量变化时，只影响（相邻的）一个节点。
>   >
>   > 缺点：很可能会分布不均匀（负载不均衡）
>
> - 虚拟槽分区（**cluster采用**）
>
>   用hash函数把数据分散到**固定范围**内的整数集合中，每个整数定义为槽`slot`。
>
>   slot数量远大于节点数，每个节点负责一部分slot，这样更方便数据的拆分和集群扩容（增加节点）
>
>   （相当于也融合了虚拟节点的概念吧）
>
>   > 与一致性哈希的不同：
>   >
>   > > 一致性哈希直接根据数据的hash值找对应的节点；
>   > >
>   > > 虚拟槽分区是先根据数据的hash找到节点，再去该节点进行操作。
>
>   > 优点：
>   >
>   > - 节点负载均衡
>   > - 解耦数据和节点的关系，方便数据的拆分和集群扩容；
>   > - 节点自身维护节点和slot的映射关系，无需客户端运算；
>
>   带图的描述：https://www.cnblogs.com/myseries/p/10959050.html
>
> 数据分布存储的集群模式**都**会有的缺点：
>
> - key批量操作受限。mset、mget等操作，key可能存在于多个节点上，所以不可用(报错)；([集群小技巧](#集群小技巧)可以解决)
> - key事务操作受限。伪事务，本来也不用...；
> - 不支持多数据库。（本来也鸡肋）单机redis可支持16个数据库，集群模式下只能使用一个数据库空间，即db0。
>
> 还有一些不算缺点，算是不足之处的
>
> - key作为数据的最小粒度，可能value还是比较大，集群也无法将一个大的键值对象（如hash、list等）映射到不同的节点。

redis cluster 的数据分布方案采用的是`虚拟槽分区(16384个slot)`方案（+虚拟节点）

所以 redis cluster 的节点处理请求会先计算key对应的slot

- 如果对应的节点是自身，直接执行命令并返回结果；
- 否则返回`MOVED重定向错误`通知客户端访问正确的节点（`MOVED`带有key所在slot信息 和 对应节点的地址信息）。

客户端收到`MOVED`再去对应的节点请求执行。

> 连接集群的客户端需要处理`MOVED`命令，所以和连接redis单机的客户端还是不同的。



##### 集群内节点通信

> Redis Cluster 采用P2P的`Gossip`(流言/八卦)协议，原理就是：节点彼此不断交换信息，一段时间后所有节点都会知道集群所有节点的信息。

集群内节点的通信的消息头都是使用的`clusterMsg`这个结构（下边有源码），包含id、myslots、消息类型、节点标识等等。



###### Gossip消息类型

- meet：`通知新节点加入`（“加入我们吧”），meet通信完成后，接收方会周期进行ping-pong；
- ping：每秒发送给多个其他节点，用于`检测节点是否在线并进行信息交换`，封装了`自身和部分其他节点的状态数据`；
- pong：回应`meet`和`ping`，封装了`自己的状态数据`。也可用来向集群广播自己的状态信息来`通知整个集群更新"我"的状态`。
- fail：（"xx下线了"）判断集群内另一个节点下线后，就广播fail消息，接收方把该节点的状态置为下线。

> Gossip特点：
>
> 扩展性：节点可任意增减，状态信息最终都会和其他节点同步；
>
> 最终一致性：只保证最终一致性
>
> 容错性：每个节点都有数据，宕机一部分也没事
>
> 健壮性：去中心化，每个节点都有持有数据



###### Redis Cluster 消息 数据结构

消息头：clusterMsgData + [redis为什么是16384个槽](#redis为什么是16384个槽) 里边的源码

消息体：meet、ping、pong等类型的消息采用MsgDataGossip数组作为消息体。



###### 发送消息的规则

- 每秒随机选取5个节点，找出最久没有通信的节点发送ping消息  ——1个
- 每100毫秒(1秒10次)都会扫描本地节点列表，如果发现节点最近一次接受pong消息的时间大于`cluster-node-timeout / 2`则立刻发送ping消息

所以每个节点每秒需要发送ping消息的数量 = `1 + 10 * num (node.pong_received > cluster_node_timeout / 2)`

> `node`：本地节点列表的每个节点
>
> `node.pong_received`：上次收到`该node`的`pong`消息已经过去多久
>
> `cluster_node_timeout`：配置节点超时时间，默认 `15s` 
>
> > 当我们的带宽资源紧张时，可以适当调大这个参数，如从默认15秒改为30秒来降低集群通信的带宽占用率。
> >
> > 调得太大了也不行，会影响消息交换的频率，从而影响`故障转移`、`槽信息更新`、`新节点发现的速度`。
> >
> > 需要根据业务容忍度和资源消耗进行平衡，节点数越多集群的总消息量就越大。[redis为什么是16384个槽](#redis为什么是16384个槽)

ps: 集群中的每个节点都会单独开辟一个TCP通道用于节点之间彼此通信，通信端口号 = 基础端口 + 10000，比如16379



#### 集群扩容

rehash重新分片，每增加一个节点只会影响到一个老节点，就是从老节点上分一些slot到新节点。

`redis-trib.rb`，这个脚本 就129行代码，可以看看。。https://github.com/redis/redis/blob/unstable/src/redis-trib.rb todo



#### 集群高可用 & 故障转移 & 新master选举

Cluster保证集群高可用和Sentinel类似。

某节点认为`master1`宕机，就会广播`fail`消息，此时`master1`是主观宕机状态，如果集群内`超过半数节点`都认为`master1`主观宕机，就会标记`master1`为客观宕机。

然后就要开始进行故障转移：

集群中`正常的master`进行投票，从`master1`的`slave`中选出一个，当某个slave获得了`半数以上的选票`，升为master。

新master会`停止复制master1`节点，并将`master1负责的所有slot`分配给自己，然后广播`pong`消息。



#### 集群小技巧 & 优化点

- hash_tag：一举两得  （注意不要过度使用，[数据倾斜](#数据倾斜)）

  - 可以利用{}给不同的key命名，以达到不同的key对应相同的槽的效果，这样还可以在集群中会用mget，mset等命令，（否则报错）
  - Redis IO 优化：Pipeline

  > redis计算槽的时候如果key的内容有`{...}`，会使用`{}`里边的内容，这里`...`称为`hash_tag`，可以让不同的key映射到相同的slot。
  >
  > ```c
  > def key_hash_slot(key):
  >     int keylen = key.length();
  >     for (s = 0; s < keylen; s++):
  >         if (key[s] == '{'):
  >             break;
  >         if (s == keylen) return crc16(key,keylen) & 16383;
  >         for (e = s+1; e < keylen; e++):
  >             if (key[e] == '}') break;
  >             if (e == keylen || e == s+1) return crc16(key,keylen) & 16383;
  >     /* 使用{和}之间的有效部分计算槽 */
  >     return crc16(key+s+1,e-s-1) & 16383;
  > ```
  >
  > 由于Pipeline只能向一个节点批量发送执行命令，相同的hash_tag对应相同的slot，而相同slot必然会对应到唯一的节点
  >
  > 这样一来....不就可以用pipeline了嘛



#### 常见问题

##### 数据倾斜

- 节点和槽的分配不均【不常见】 ，可以使用redis-trib.rb rebalance命令进行平衡；
- 不同槽对应键数量差异过大。注意不能过度使用hashtag；
- 可能包含big key，一般不能有太大的key ，需要找到大集合后根据业务场景进行拆分



##### 请求倾斜

> 集群内特定节点请求量 / 流量过大 导致节点之间负载不均

对于热点key: 避免bigkey ,不要使用hash_tag，本地缓存+MQ



##### redis为什么是16384个槽

> 1. 正常的心跳包携带一个节点的全部配置信息，其中就包含`myslots`信息，16384bit = 2048byte  2k了，如果用65536bit的话，就需要8k
>
> 2. 集群是每秒ping-pong，一旦集群的节点数量多了就会占用大量带宽
>
>    > 集群节点越多，心跳包的消息体内携带的数据越多。如果节点过1000个，也会导致网络拥堵。
>    >
>    > 因此，redis作者不建议redis cluster节点数量超过1000个。而对于节点数在1000以内的redis cluster集群，16384个槽位够用了。
>
> 3. 槽位越小，节点少的情况下，压缩比高
>
>    > Redis Cluster master的配置信息中，它所负责的哈希槽是通过一张bitmap的形式来保存的。
>    >
>    > 在传输过程中，会对bitmap进行压缩，如果bitmap的填充率slots / N很高的话(N表示集群节点总数)，bitmap的压缩率就会变低。
>    >
>    > 如果节点数很少，而哈希槽数量很多的话，bitmap的压缩率就很低。
>
> 再`考虑到负载均衡和扩展，槽位也不能太少`，折中考虑，作者决定取16384个槽
>
> 消息头的源码：
>
> ```c
> // https://github.com/redis/redis/blob/unstable/src/cluster.h
> #define CLUSTER_SLOTS 16384  // 这里声明常量
> typedef struct {
>     char sig[4];        /* Signature "RCmb" (Redis Cluster message bus). 消息总线*/
>     uint32_t totlen;    /* Total length of this message 消息总长度*/
>     uint16_t ver;       /* Protocol version, currently set to 1. 协议版本号*/
>     uint16_t port;      /* TCP base port number. */
>     uint16_t type;      /* Message type 消息类型：meet、ping、pong*/
>     uint16_t count;     /* Only used for some kind of messages. 某些信息会用...*/
>     uint64_t currentEpoch;  /* The epoch accordingly to the sending node. */
>     uint64_t configEpoch;   /* The config epoch if it's a master, or the last
>                                epoch advertised by its master if it is a
>                                slave. */
>     uint64_t offset;    /* Master replication offset if node is a master or 主节点的已复制偏移量 或
>                            processed replication offset if node is a slave. 从节点已处理的复制偏移量 */
>     char sender[CLUSTER_NAMELEN]; /* Name of the sender node 发送者name*/
>     // 这里这里这里！！！
>     unsigned char myslots[CLUSTER_SLOTS/8]; // 发送节点负责的slot，这个数组大小是16384/8=2048 byte，2kb
>     char slaveof[CLUSTER_NAMELEN];
>     char myip[NET_IP_STR_LEN];    /* Sender IP, if not all zeroed. 不全为0就是发送方的ip*/
>     char notused1[34];  /* 34 bytes reserved for future usage. 保留字节*/
>     uint16_t cport;      /* Sender TCP cluster bus port 发送方tcp总线端口*/
>     uint16_t flags;      /* Sender node flags 发送方的flags 这里这里这里！！！*/
>     unsigned char state; /* Cluster state from the POV of the sender */
>     unsigned char mflags[3]; /* Message flags: CLUSTERMSG_FLAG[012]_... */
>     union clusterMsgData data;
> } clusterMsg;
> ```
>
> 消息体的源码：
>
> ```
> 。。算了哥  下次一定          todo
> ```
>
> ```c
> // https://github.com/redis/redis/blob/unstable/src/cluster.h 
> // 特么太多了
> clusterMsgDataGossip
> clusterMsgData 消息体
> 
> ```



# redis源码

| src文件名                                                    | 干啥的                                                       |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ae.c 、 ae.h 、 ae_epoll.c 、 ae_evport.c 、 ae_kqueue.c 、 ae_select.c | 事件处理器，以及各个具体实现。                               |
| redis.h                                                      | Redis 的主要头文件，记录了 Redis 中的大部分数据结构， 包括服务器状态和客户端状态。 |
| zmalloc.c 、 zmalloc.h                                       | 内存管理程序。                                               |
| sds.c 、 sds.h                                               | SDS 数据结构的实现，SDS 为 Redis 的默认字符串表示。          |
| adlist.c 、 adlist.h                                         | 双端链表数据结构的实现。                                     |
| dict.c 、 dict.h                                             | 字典数据结构的实现                                           |
| bio.c 、 bio.h                                               | Redis 的后台 I/O 程序，用于将 I/O 操作放到子线程里面执行， 减少 I/O 操作对主线程的阻塞。 |
| rdb.c 、 rdb.h                                               | RDB 持久化功能的实现。                                       |
| aof.c                                                        | AOF 功能的实现。                                             |
| cluster.c 、 cluster.h                                       | Redis Cluster 的集群实现。                                   |
| sentinel.c                                                   | Redis Sentinel 的实现。                                      |
| crc16.c 、 crc64.c 、 crc64.h                                | 计算 CRC 校验和。 crc16算法                                  |
| redis-trib.rb                                                | Redis 集群的管理程序。Ruby脚本                               |



# Lua 脚本 原子执行

Lua 脚本在 Redis 中是原子执行的，Redis 在执行`EVAL`命令的时候，一直到执行完毕并返回结果之前，会阻塞所有其他客户端的命令，所以Lua脚本中要写逻辑特别复杂的脚本， 必须保证 Lua 脚本的效率。



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



#### 注意事项

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



# 一些小坑

```java
// 错误用法 .. 第三个参数不是timeout...
stringRedisTemplate.opsForValue().set("hashkey:key1","value",60*60*1000);

// api源码
@Override
public void set(K key, V value, long offset) {
    byte[] rawKey = rawKey(key);
    byte[] rawValue = rawValue(value);
    execute(connection -> {
        connection.setRange(rawKey, rawValue, offset);// 用指定的字符串覆盖给定key所储存的字符串值，覆盖的位置从偏移量offset开始
        return null;
    }, true);
}

// 正确用法：（key, value, offset, timeout）
stringRedisTemplate.opsForValue().set("hashkey:key1","value",60*60*1000,TimeUnit.MILLISECONDS);
```

<img alt="redisTemplate.opsForValue().set(k, v, offset)" src="https://raw.githubusercontent.com/melopoz/pics/master/img/20210330005146.png" style="zoom:50%;" />

