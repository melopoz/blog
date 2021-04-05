---
title: Kafka
tags: redo
categories: MQ
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



# 整体架构

<img alt="Kafka架构图" src="https://raw.githubusercontent.com/melopoz/pics/master/img/kafka%E6%9E%B6%E6%9E%84%E5%9B%BE.png" style="zoom:80%;" />

通过 zookeeper 保存元数据，管理集群配置、选举Leader、Consumer Group发生变化时进行rebalance。

<img alt="Kafka元数据在zookeeper中的数据结构" src="https://raw.githubusercontent.com/melopoz/pics/master/img/kafka%E5%85%83%E6%95%B0%E6%8D%AE%E5%9C%A8zookeeper%E4%B8%AD%E7%9A%84%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.png" style="zoom:60%;" />

## Topic

Kafka是发布/订阅模式的消息队列，生产、消费消息，都是面向Topic的。

> 另一种是**点对点**消息队列，即一条消息被一个消费者消费一次就没了



## Consumer Group

一条消息可以发送到不同的Consumer Group，但一个Consumer Group中只能有一个Consumer能消费这条消息。

> 即 Topic下的一个[Partition](#Partition)只能被同一个Consumer Group下的一个Consumer线程来消费。

Topic有几个分区，消费这个Topic的Consumer Group就有几个Consumer即可，效率也最高，Consumer多了就会有Consumer拿不到消息。

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/Kafka-%E5%88%86%E5%8C%BA%E6%95%B0%E6%B6%88%E8%B4%B9%E8%80%85%E6%95%B0.png" />



## Partition

物理概念，每个 Topic 包含一个或多个 Partition，一个 Partition 对应一个文件夹（命名规则：`Topic 名称+分区序号`），存储 Partition 的数据和索引文件（因为每个Partition又分为了多个[Segment](#Segment)），每个 Partition 内部是有序的，且每个 Partition 中的消息不一样。

> 由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位效率低，Kafka 采取了**分片**和**索引**机制，将每个 Partition 分为多个 [Segment](#Segment)



<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/kafka-%E5%88%86%E5%8C%BA%E6%95%B0%E6%B6%88%E8%B4%B9%E8%80%85%E6%95%B0.png" style="zoom:80%;" />

### 分区分配策略

> 通过 `Partition.assignment.strategy` 配置

- range：**随机**（默认）
- roundrobin：**循环**
- StickyAssignor：粘性分配

> 当有下列情况发生的时候，Kafka会进行一次Partition分配（**rebalance**）：
>
> 1. Topic 新增 Partition
>2. Consumer Group 新增 consumer
> 3. consumer 离开所在 group （shut down / crashes(崩溃)）



## Segment

每个 Segment 文件包括 `.index` 文件 和 `.log` 文件，这些文件位于同一个 Partition 文件夹下。Segment文件大小默认1G。

> 例如，Topic "first" 有三个分区，则其对应的文件夹为 `first-0`，`first-1`，`first-2`

`.index` 和 `.log` 文件均以当前 Segment 的第一条消息的 offset 命名:

> 在 Kafka 的 `server.properties` 配置的 `log.dirs` 为数据的目录，该目录下有文件夹 `Topic名-分区号`，每个分区中包含如下文件
>
> ```
> 00000000000000000000.index
> 00000000000000000000.log
> 00000000000000170410.index
> 00000000000000170410.log
> 00000000000000239430.index
> 00000000000000239430.log
> 
> 00000000000000000000.timeindex
> Leader-epoch-checkpoint
> ```
>
> `.index` 文件存储索引信息，`.log` 文件存储数据，索引文件中的元数据指向对应数据文件中 message 的物理偏移地址 `offset`。



## Producer

### 2种消息发送机制

- Sync Producer：低延迟，低吞吐率，无数据丢失

- Async Producer：高延迟，高吞吐率，可能会有数据丢失

  ```java
  Future<RecordMetadata> send(ProducerRecord<K, V> record);
  Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);
  ```

#### 异步发送

> 在 Kafka 1.0 以后，客户端默认发送都是异步发送，首先追加到生产者的内存缓存中，其内存存储结构如下：
>
> `ConcurrentMap<TopicPartition，Deque<ProducerBatch>> batches`
>
> Producer 会为每一个 Topic 的每一个 Partition 单独维护一个队列，即 `ArrayDeque`，内部存放的元素就是一批消息 `ProducerBatch`，即Kafka消息发送是**批量发送**的。

若需要同步发送，只要拿到 `send(record)` 方法返回的 `future`，调用get方法，此时如果消息未发送到Broker上，该方法就会被阻塞，等到 broker 返回消息发送结果后该方法会被唤醒并得到消息发送结果；

若需要异步发送，则建议使用 `send(ProducerRecord< K, V > record, Callback callback)`，但是不要调用get方法，Callback 会在收到 broker 的响应结果后被调用，并且支持**拦截器**。

> 拦截器可以在 `消息发送前` 和 `回调函数被调用前` 触发。



### 发送确认机制

通过`request.required.asks`配置

- **0**：消息发送给Leader后，不需要确认。性能高，风险最大，如果broker宕机，宕机之后发送的消息就会丢失
- **1**：只要集群中的Leader确认即可返回
- **-1**：需要ISR中的所有Relica(副本)（集群中的所有broker）确认。还是有可能出现数据丢失，因为ISR可能缩小到只有一个Replica。





## Consumer

- 消费者是主动 `pull` 消息

  > 因为 Broker 中的消息是无状态的，Broker 不知道哪个消息是可以消费的，这样还会降低Broker的压力。

- Consumer 消费消息之后必须要去通知 ZooKeeper 记录下消费的 offset（所以可能[重复消费](#重复消费)）



### 提交方式

- 自动提交
- 手动提交（灵活控制offset）



### poll模型

通过 `poll` 方法，先调用 `fetcher` 中的 `fetchedRecords` 方法

> 如果 `fetchedRecords ` 获取不到数据，就会发起一个新的 `sendFetches` 请求。

而在消费数据的时候，每个批次从Kafka Broker Server中拉取数据是有最大数据量限制，由属性 `max.poll.records` 控制，默认是500条。

> 提示：`max.poll.records` 返回的是一个poll请求的数据总和，**与多少个分区无关**；因此，每次Consumer所有分区中拉取Topic的消息数据，总条数不会超过max.poll.records所设置的值。

而在Fetcher的类中，在sendFetches方法中有限制拉取数据容量的限制，由属性（max.partition.fetch.bytes），默认1MB。

> https://matt33.com/2017/11/11/consumer-pollonce/#pollOnce-%E6%96%B9%E6%B3%95





# 消息持久化

- **LSO**：Last Stable Offset, 日志开始偏移量, 标志日志文件起始处

- **LEO**：log end offset, 日志结束偏移量，表示当前日志文件中下一条待写入的消息的offset

- **HW**：High Watermark. 高水位, 表示特定的消息的offset, 消费者只能消费这个offset之前的消息

- **LW**：Low Watermark. 低水位, AR集合中最小的LSO值



### 删除策略

Kafka日志管理器中会有一个专门的日志删除任务来**周期性**检测和删除不符合保留条件的日志分段文件，通过broker端参数 `log.retention.check.interval.ms` 来配置，默认值为 `300000` (5分钟)。

- 基于时间
- 基于大小
- 基于起始偏移量



### 顺序写

以为顺序写，所以读操作不会阻塞写操作



# 零拷贝

总的来说Kafka快的原因：

- Partition顺序读写，充分利用磁盘特性，这是基础；
- Producer生产的数据持久化到broker，采用mmap文件映射，实现顺序的快速写入；（**mmap+write**)
- Customer从broker读取数据时，Kafka使用sendfile，将磁盘文件读到OS内核缓冲区后，直接转到socket buffer进行网络发送。(**sendfile**)



# 事务特性

主要就是将多条消息的发送作为一个事务，要么都发送成功要么都失败。









# 集群

<img alt="Producer连接Kafka集群" src="https://raw.githubusercontent.com/melopoz/pics/master/img/kakfa%E9%9B%86%E7%BE%A4.png" style="zoom:80%;" />

#### Controller

> Kafka集群中的一个broker会被选举为controller，负责Partition的管理和replica状态管理，也会执行Partition重分配的任务。

Leader

> Leader通过单独的线程定期检测ISR中Follower



#### Replica 副本

> https://blog.51cto.com/zero01/2501495#h3

- 副本集是指将日志复制多份（Kafka的数据是存储在日志文件中的，这就相当于数据的备份、冗余）
- 可以为每个Topic设置副本集，所以副本集是相对于Topic来说的
- 可以配置默认的副本集数量

为了提高数据的可靠性

> 一个Topic的副本集可以分布在多个Broker中，当一个Broker挂掉了，其他的Broker上还有数据。
>
> 每个Partition在每个broker上最多只能有一个replica（也因此可以由Broker id指定Partition的具体replica；replica**默认均匀分布**到所以broker上）

我们都知道在Kafka中的Topic只是个逻辑概念，实际存储数据的是物理分区Partition，所以真正被复制的也是Partition。如下图：

<img alt="kafka副本" src="https://raw.githubusercontent.com/melopoz/pics/master/img/kafka%20replica.png" style="zoom:50%;" />



##### 副本因子

副本因子其实决定了一个Partition的副本数量是多少

> 例如副本因子为1，则代表将Topic中的所有Partition按照Broker的数量复制一份，并分布到各个Broker上（每个broker上都有备份）



##### 副本分配算法

将所有 `N` 个 Broker 和待分配的 `i` 个 Partition 排序

将第 `i` 个Partition 分配到第 `(i mod n)` 个 Broker 上

将第 `i` 个Partition 的第 `j` 个副本分配到第 `((i + j) mod n)` 个 Broker 上



##### Replica 分为 Leader 和 Follower

- **Leader**：只有Leader对外提供服务（生产、消费），producer和consumer都是和Leader交互。
- **Follower**：同步的时候数据是由Follower周期性尝试将数据pull过来，副本的同步机制是 **ISR**。不完全同步，也不完全异步。



##### 副本数据同步策略 ISR (OSR / AR)

> AR：Assigned Replicas 所有副本
>
> ISR：In-Sync Replicas 与Leader保持一定程度同步的副本（包括Leader副本）
>
> OSR：Out-Sync Replicas 与Leader副本同步落后过多的副本（不包括Leader副本）

AR = ISR + OSR

> 正常情况下，所有Follower副本都应该与Leader副本保持一定程度的同步，即AR=ISR，OSR为空。

Leader负责跟踪**ISR**中Follower副本的之后状态，掉队就移除。

> 只有ISR集合中的副本才有资格被选举为Leader

`replica.lag.time.max.ms`：Follower能落后Leader的最长时间间隔，默认 10s。

> Follower因不能及时跟上Leader的同步而被踢出ISR之后还会继续同步，之后如果跟上了Leader，还会被加入ISR。

ISR的管理在zookeeper中的节点位置：`/brokers/Topics/[Topic]/Partitions/[Partition]/state`



##### 备份恢复机制

如果Leader挂掉了，会从replica中选取一个做为新的Leader，并且从ISR中移除原Leader。

如果所有的副本都宕机后，有两种策略：

1. 等待ISR中的任意一个replica恢复，并选举为Leader；如果ISR中一个也不能恢复，该Partition永久不可用。相对较少丢失数据。
2. 选取第一个恢复的replica作为Leader，不论是不是在ISR中（可能是OSR）。较高的可用性但是可能会丢失数据。



##### Leader选举

直接从ISR中选一个最快的设为Leader





# 通信协议

基于TCP

> http://09itblog.site/?p=867





# 常见问题☆

消息丢失

https://blog.csdn.net/weixin_39983554/article/details/110361423

数据大小限制？

https://blog.csdn.net/qq_37502106/article/details/80271800

问题集锦

https://blog.csdn.net/chenwiehuang/article/details/100172204



#### 消息丢失

- 生产端

  > 同步模式下：消息确认模式为1的时候(保证写入Leader成功)，如果刚好Leader Partition挂了，会丢失数据。
  >
  > 异步模式下：缓存区满了，消息确认模式为0的时候(消息发出不需要确认)，会丢失数据。

- 消费端

  > consumer从Topic中拉取到消息，还没有消费成功就提交了offset。



#### 重复消费

1. 强行kill线程、消费系统宕机、重启等各种原因导致消费数据后，**offset没有成功提交**；

2. 设置offset为自动提交，如果在close之前（提交offset），调用了consumer.unsubscribe()就有可能部分offset还没提交，下次就会重复消费；

3. Consumer消费消息需要太长时间导致会话超时（或心跳机制检测出问题）

   > ```
   > session.timeout.ms 默认 10s
   > group.min.session.timeout.ms 默认 6s
   > group.max.session.timeout.ms 默认 30min
   > ```
   >
   > 当Consumer处理消费的业务逻辑的时候，如果在`10s`内没有处理完，那么消费者客户端就会与Kafka Broker Server断开，即使Consumer设置了**自动提交**属性，也会报错（提交失败），broker会认为这个Consumer出问题了所以把这个Partition的消息rebalance到其他的Consumer，导致重新消费。
   >
   > 所以每次拉取的消息数 和 timeout 要合理。



##### 如何保证不重复消费

只好结合业务场景对消息消费做**幂等处理**。

> 比如在redis或数据库记录已消费的消息，producer生产消息的时候入库消息未消费，consumer消费完在数据库update为已消费。
>
> 或者换RocketMQ...



#### 如何处理replica的全部宕机

[备份恢复机制](#备份恢复机制)







# 配置

配置文件`config/server.properties`

```properties
#日志所在目录
log.dirs=/tmp/Kafka-logs
#开启可能会造成数据丢失
unclean.Leader.election.enable=false
#如果Leader发现Follower有 多久(10s) 没有发送fetch请求了，那么Leader就把它从ISR移除。
replica.lag.time.max.ms=10000
#至少保证ISR中有几个replica
min.insync.replicas=1
#发送确认机制：0/1/-1
request.required.asks=0
```





# 常用命令

```sh
bin/Kafka-server-start.sh config/server.properties #启动
bin/Kafka-server-start.sh -daemon config/server.properties #后台启动
```

命令均在Kafka的bin目录下

```sh
#创建Topic
bin/Kafka-Topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --Partitions 1 --Topic test
#查看Topic列表
bin/Kafka-Topics.sh --list --zookeeper localhost:2181
#产生消息，创建消息生产者
bin/Kafka-console-producer.sh --broker-list localhost:9092 --Topic test
#消费消息，创建消息消费者
bin/Kafka-console-consumer.sh --broker-list localhost:9092 --Topic test
#查看Topic的消息
bin/Kafka-Topics.sh --describe --zookeeper localhost:2181 --Topic test
#删除Topic
bin/Kafka-Topics.sh --delete --zookeeper localhost:2181 --Topic test
#修改分区数为6
bin/Kafka-Topics.sh --alter --Partitions 6 --zookeeper localhost:2181 --Topic test
```



