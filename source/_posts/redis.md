---
title: Redis
tags: 
  - DB
  - 缓存
categories: 
  - [NoSQL]
---



# Redis

配置文件示例

http://download.redis.io/redis-stable/redis.conf

参考

https://www.cnblogs.com/kismetv/p/9137897.html

---



### 本地缓存和分布式缓存

> 本地缓存，比如java中的map，guava实现，轻量、快速，但是如果有多个实例，就需要每个实例都保存一份缓存，不具有一致性。

>  分布式缓存，比如redis、memcached，缓存具有一致性。但是需要保持服务的高可用。

---



### redis 工作模式

多个cli连接server

> redis需要处理cli的命令(get,set,lpush,rpop...)，对`接收客户端链接，处理请求，返回命令结果`等任务，redis使用主进程和主线程完成。
>
> redis后台还有RDB持久化，AOF重写等任务，这些任务会启动子进程来完成。



### redis 线程模型

https://www.jianshu.com/p/8f2fb61097b8 这个图更清楚一点

> redis内部使用文件事件处理器（file event handler），这个handler是单线程的，所有请求都要经过这个入口才能被处理，自然就有了先后顺序，所以说是单线程模型。

> 在这个模型中，Redis 服务器用主线程执行 I/O 多路复用程序、文件事件分派器以及事件处理器。而且，尽管多个文件事件可能会并发出现，Redis 服务器是顺序处理各个文件事件的。        ——《Redis设计与实现》

> IO多路复用的 Reactor 模式

master线程监听多个socket，将产生的事件加入任务队列，事件分派器从队列中取出事件并执行，这也就保证按顺序执行。

![](redis%E5%A4%84%E7%90%86%E4%B8%80%E6%AC%A1%E5%91%BD%E4%BB%A4%E8%BF%87%E7%A8%8B.png)

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





### hash扩容：渐进式扩容

hash字典的初始容量为4（3.2.8版本是这样）

redis的hash类型在    **元素数量与数组size相同时（且没有在进行bgsave）** 或  **元素数量是数组size的5倍（不论是否在bgsave）**的时候就会开始扩容，新数组的size为元素数量*2。

redis hash 维护一个rehashidx来表示重新hash的索引，默认值为 -1，如果 >=0 说明开始 rehash 了，则每次对这个hash进行操作的时候将 rehashidx 处的元素进行rehash，当idx进行到最后的时候再置为-1结束扩容。这样在rehash过程中也能正常提供服务。



### hash缩容

当hash数组中的元素数量所占比例小于负载因子（0.1），不论是否正在bgsave或者bgwriteaof，都会进行缩容



---

### redis VS memcached

1. redis数据类型丰富，memcached只支持string；
2. redis支持数据持久化，可以在宕机、重启之后进行数据恢复。memcached只能存储在内存；
3. redis原生支持集群cluster，memcached没有原生集群模式；
4. redis使用单线程的IO多路复用模型，memcached是**多线程**、非阻塞的IO复用的网络模型。redis6也使用了多线程IO



---

### redis删除过期key：定期删除+惰性删除

- 定期删除（redis默认100ms）

  redis默认每隔100ms就**随机抽取**一些设置了过期时间的key，检查并删除。

  > 随机抽取的原因：如果key数量巨大，每隔100ms遍历所有设置过期时间的key，可能严重增大CPU的负载。

  

- 惰性删除

  redis的定期删除可能导致一些key没有及时删除。如果一个key已经过期但还留在内存，只有查到了这个key，这个key才会被删除。

如果redis的定期删除漏掉了很多过期的key，并且没有及时查这些key，就会浪费内存。解决这个问题就需要**redis内存淘汰机制**。

---



## redis内存淘汰机制

>  数据库中有2000w数据，redis中只存20w数据，如何保证redis中的数据是热点数据？

### 8种数据淘汰策略：

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
> **volatile**为前缀的策略都是从**已过期的数据集**中进行淘汰。
>
> **allkeys**为前缀的策略都是面向**所有key**进行淘汰。
>
> **LRU**（least recently used）最近最少用到的。
>
> **LFU**（Least Frequently Used）最不常用的。

---



## 持久化机制 保证重启后数据可恢复

redis支持持久化，这是redis不同于memcached很重要的一点。

### 1. 快照 snapshotting （RDB）（默认）

在redis.conf中有如下配置

```properties
save 900 1		#在900s(15min)之后，至少1个key发生变化，自动触发BGSAVE命令创建快照
save 300 10		#在300s(5min)之后，至少10个key发生变化，snapshot
save 60 10000	#在60s(1min)之后，至少1w个key发生变化，snapshot
```

### 2. 追加文件 append-only （AOF）

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

### 3. 混合持久化 RDB+AOP 

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

---

### AOF配置

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



## 缓存穿透、缓存击穿、缓存雪崩

穿透  ```同时大量请求一个不存在的key```

击穿  ```同时大量请求一个存在但是失效的key，这个key失效```

### 缓存穿透

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



## 数据类型-命令操作-实际用例

### 常用通用命令

- **del**、**exists**、**type**

- redis **expires**

  在redis中添加元素的时候设置过期时间：set key value 存活时间

- **expire**  重新设置key的存活时间

- **persist** 去掉一个key的过期时间，使之成为持久化key

- **ttl**  以秒为单位，返回 key 的剩余生存时间

- **rename** 对一个key改名，之后存活时间继续计时

- **setnx**  不存在就插入





### 数据类型

#### string

​	set、mset、get、mget、getset、strlen

​	incr、incrby、decr、decrby  （原子增量）

​	setbit、getbit、bitcount、bitop  （位图）

位图不是实际的数据类型，而是在字符串类型上定义的一组面向位的操作，最大512M，所以最多存储2^32^位

#### list

1. lpush、rpush、lpop、rpop

   > 用例：BlockingQueue，如果list为空，pop会直接返回null，不会完成其他任何工作。

2. llen  查看list中元素的个数

3. lrange --遍历get

   需要两个索引，要返回的范围的第一个和最后一个元素。两个索引都可以为负，-1是列表的最后一个元素，-2是列表的倒数第二个元素，依此类推。

   ```
   > rpush mylist 1 2 3 4 5 "foo bar"
   (integer) 9
   > lrange mylist 0 -1
   1) "first"
   2) "A"
   3) "B"
   4) "1"
   5) "2"
   6) "3"
   7) "4"
   8) "5"
   9) "foo bar"
   ```

4. ltrim --修剪

   仅从索引0到2列出元素，其他的都丢弃。

   > 用例：lpush并ltrim 0 99，添加新元素并丢弃超出范围的元素。

5. brpop、blpop

   按照参数中的list的顺序pop完list1的元素再pop list2中的元素，如果都为空，就阻塞等待n秒，n秒内如果list中有了元素就返回，否则就返回nil

   ```
   blpop list1 list2 list3... n秒
   ```

   ```
   127.0.0.1:7963> brpop list1 list2 5
   (nil)
   (5.06s)
   ```

6. lpoprpush   转移列表的数据

   ```
   rpoplpush list1 list2
   ```

7. linsert       插入到指定位置 

   ```linsert list1 before/after a v```  在a前边插入一个v，返回新len，如果a不存在返回-1

8. 常见用例 及 问题+解决方案

   - 用户的最新动态（最近5、10条）

   - 队列（不安全）

9. **注意**

   > 因为不能对集合中每项都设置TTL，但是可以对整个集合设置TTL。**所以，我们可以将每个时间段的数据放在一个集合中。然后对这个集合设置过期时间。**

#### hash

字典 field-value

命令：hset hget hmset hmget hgetall hexists hsetnx 

```
> hmset user:1000 username antirez birthyear 1977 verified 1
OK
> hget user:1000 username
"antirez"
> hget user:1000 birthyear
"1977"
> hgetall user:1000
1) "username"
2) "antirez"
3) "birthyear"
4) "1977"
5) "verified"
6) "1"
> hmget user:1000 username birthyear no-such-field
1) "antirez"
2) "1977"
3) (nil)
> hincrby user:1000 birthyear 10
(integer) 1987
> hincrby user:1000 birthyear 10
(integer) 1997
```

#### set

- sadd、spop
- sismember  是否存在  返回1/0
- smembers   所有元素
- srandmember  随机n个元素    ```srandmember key 2```
- sunion   并集

```
> sadd myset 1 2 3
(integer) 3
> smembers myset
1) 3
2) 1
3) 2
> sismember myset 3
(integer) 1
> sismember myset 30
(integer) 0
```

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







#### PFADD

#### 位图 setbit

#### HyperLogLog    hll



## Redis可实现功能

### 1.分布式锁

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

### 2.附近的人-空间搜索

https://www.cnblogs.com/ningbj/p/11711875.html

> GeoHash是一种地址编码方式。能够把二维空间经纬度数据编码成一个字符串。

> ```sh
> #geoadd key 经度 纬度 名字 [经度 纬度 名字] ...  返回integer
> GEOADD key longitude latitude member [longitude latitude member ...]
> #当所需存储的对象数量过多时，可通过设置多key(如一个省一个key)的方式对对象集合变相做sharding，避免单集合数量过多。
> #Redis内部使用有序集合(zset)保存位置对象，元素的score值为其经纬度对应的52位的geohash值。
> ```
>
> ```sh
> #geopos
> GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [ASC|DESC] [COUNT count] [STORE key] [STORedisT key]
> #
> ```
>
> ```sh
> #geodist
> ```
>
> ```sh
> #geohash
> ```
>
> ```sh
> #georadius
> ```
>
> ```sh
> #georadiusbymember
> ```



### 3.位图-签到记录、PV、UV

### 4.布隆过滤器 setbit

### 5.队列 list push pop

### 6.限流





## 集群策略

### 1. 主从复制

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

### 2. <span id = "redis-sentinel">哨兵模式</span>

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



### 3. 集群 cluster

官方推荐集群至少要3台以上master，最好3master 3slave。

##### 配置 redis.conf

```properties
#开启cluster模式
cluster-enable yes
#集群模式下的集群配置文件
cluster-config-file nodes-6379.conf
#集群内节点之间最长响应时间
cluster-node-timeout 15000
```

启动6个redis-server，可以借助ruby，或者自己写群起脚本

##### 工作机制

对key进行【crc16算法】【%16384】的操作，通过计算结果找到对应插槽所对应的节点，然后直接跳转到这个基点进行存取操作。

为了保证高可用，使用主从模式。master宕机，启用slave。如果一个节点的主从都宕机，则集群不可用。

##### 特点

- 所有redis节点彼此互联（PING-PONG机制），使用二进制协议优化传输速度和带宽
- 集群中超过半数的节点检测失效才认为节点fail
- redic-cil与节点直连，不需要中间代理层，任意连接集群中一个可用节点即可。





## 整合SpringCache

配置，在dao代码加注解实现



## 其他命令

```
localhost:6379> config get auto-aof-rewrite-min-size
1)"auto-aof-rewrite-min-size"
2)"67108864"
```

```
localhost:6379> info persistence
1) aof_current_size:149
2) aof_base_size:149
```













# 分布式锁

https://www.cnblogs.com/demingblog/p/9542124.html

问题 不能释放别人的锁

场景：

A 拿到锁并执行业务逻辑，但是太慢了 锁超时自动释放并由B拿到，然后A执行完毕， 想要释放锁。

> 如果锁不加区分，A会释放B拿到的锁，  可能会有其他线程 加锁成功， 与B冲突！
>
> 所以需要为锁添加持有人属性。   可以get(lockKey) ， if = 当前执行人(如 threadId),  释放锁， 否则 说明自己执行超时了， 考虑回滚











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









# 还可能有问题！

上述做法看似完美实现了分布式锁功能。但只限 Redis 单点的情况！

如果是在[redis sentinel 集群](#redis-sentinel)中, A 拿到锁 `set lockKey lockValue`命令只在 master 节点执行完成，还没有同步到 slave 的时候， master 挂了，集群将重新选举 master ， 然后 B 再试图拿锁， 也会成功。  这就出错了...   

这种情况发生概率极小， 但总有不能容忍这个瑕疵的系统。



### 解决 - redlock

redlock算法 ， 用于多个 redis 实例的场景， 

加锁的时候 向多半的节点发送 `setnx lockKey lockValue` 命令， 过半节点成功才算加锁成功；

释放锁的时候 向全部节点发送 `del` 命令。

> 这是一种基于【大多数都同意】的一种机制



已有的开源实现：

python: redlock-py

java: redission



# Redisson

实现分布式锁



开源的 看源码去吧



