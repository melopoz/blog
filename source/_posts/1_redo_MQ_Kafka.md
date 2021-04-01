---
title: Kafka
tags: redo
categories: MQ
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



混乱。。。

# kafka

百度网盘,git_note有pdf

https://blog.csdn.net/qq_37502106/article/details/80271800

https://blog.csdn.net/chenwiehuang/article/details/100172204

javademo

https://blog.csdn.net/Dongguabai/article/details/86520617



<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/kakfa%E9%9B%86%E7%BE%A4.png" style="zoom:80%;" />

## Topic 

kafka是发布/订阅模式的消息队列 (另一种是点对点消息队列，即一条消息被一个消费者消费一次就没了)

Kafka中消息是以topic进行分类的，生产 消费消息，都是面向topic的。topic是逻辑上的概念，



## Partition 分区

>  topic下的一个partition只能被同一个consumer group下的一个consumer线程来消费。

> consumer消费消息的时候如何确定topic的哪个partition呢？
>
> kafka的两种策略: (由参数parttion.assignment.strategy控制)(使用这两个策略有不同的要求)
>
> - range 随机  （默认）
> - roundrobin 循环
>
> 当有下列情况发生的时候，kafka会进行一次partition分配：
>
> 1. consumer group新增consumer
> 2. consumer 离开当前所在 group ， 包括shut down和crashes(崩溃)
> 3. topic新增partition
>
> 将分区的所有权从一个consumer转移到另一个consumer成为**rebalance**

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/kafka-%E5%88%86%E5%8C%BA%E6%95%B0%E6%B6%88%E8%B4%B9%E8%80%85%E6%95%B0.png" style="zoom:80%;" />

> producer会用多个线程并发地向不同分区所在的broker发起socket连接发送消；
>
> 同一个consumer group的consumer线程都被指定topic的某一个partition进行消费。

> kafka通过消息的key来进行partition的分配，采用hash(key)%numPartitions算法（key为null就随机取）。

而partition是物理上的概念，每个partition对应于一个log文件夹(因为log文件会被分成多个segment)，每个partition中的消息也不一样，该log文件中存储的就是producer生产的数据。Producer生产的数据会被不断追加到该log文件末端，且每条数据都有自己的offset。消费者组中的每个消费者，都会实时记录自己消费到了哪个offset，以便出错恢复时，从上次的位置继续消费。

由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位
效率低下，Kafka 采取了分片和索引机制，将每个 partition 分为多个 

### segment

。每个 segment个文件——“.index”文件和“.log”文件。这些文件位于一个文件夹下，该文件夹的命名
规则为：topic 名称+分区序号。例如，first 这个 topic 有三个分区，则其对应的文件夹为 first-
0,first-1,first-2

index 和 log 文件以当前 segment 的第一条消息的 offset 命名:

> 在kafka的server.properties配置的log.dirs为数据的目录
>
> 在该目录下有文件夹‘topic名-分区号’   其中包含如下文件
>
> > 00000000000000000000.index
> > 00000000000000000000.log
> > 00000000000000170410.index
> > 00000000000000170410.log
> > 00000000000000239430.index
> > 00000000000000239430.log
> >
> > 00000000000000000000.timeindex
> > leader-epoch-checkpoint

“.index”文件存储大量的索引信息，“.log”文件存储大量的数据，索引文件中的元数据指向对应数据文件中 message 的物理偏移地址。



## 存储

kafka创建topic



## 关键词

- LSO

  Last Stable Offset, 日志开始偏移量, 标志日志文件起始处

- LEO

  log end offset, 日志结束偏移量，表示当前日志文件中下一条待写入的消息的offset

- HW

  High Watermark. 高水位, 表示特定的消息的offset, 消费者只能消费这个offset之前的消息

- LW

  Low Watermark. 低水位, AR集合中最小的LSO值



## Replica 副本

每个partition都会有n个replica(副本)（replication-factor为n且n>1）。

replica的个数<=broker的个数，每个partition在每个broker上最多只能由一个replica，也因此可以由Broker id指定partition的具体replica。

replica默认均匀分布到所以broker上。

#### replica又分为leader和follower。 

只有leader对外提供服务（生产、消费），producer和consumer都是和leader交互。

同步的时候数据是由follower周期性或尝试去pull过来，

副本的同步机制是ISR。不完全同步，也不完全异步。



## 副本数据同步策略ISR

## AR ISR OSR

https://blog.csdn.net/weixin_43975220/article/details/93190906

> AR：Assigned Replicas 所有副本
>
> ISR：In-Sync Replicas 与leader保持一定程度同步的副本（包括leader副本）
>
> OSR：Out-Sync Replicas 与leader副本同步之后过多的副本（不包括leader副本）

AR = ISR + OSR

> 正常情况下，所有follower副本都应该与leader副本保持一定程度的同步，即AR=ISR，OSR为空。

Leader副本负责跟踪ISR中follower副本的之后状态，掉队就剔除。

只有ISR集合中的副本才有资格被选举为leader，OSR中的副本如果跟上了leader副本，还在OSR中，也没有资格。（相关配置参数：？？？）

replica.lag.time.max.ms：follower能落后leader的最长时间间隔，默认值10s。

follower因不能及时跟上leader的同步而被踢出ISR之后还会继续同步，之后如果跟上了leader，还会被加入ISR。

ISR的管理在zookeeper中的节点位置：/brokers/topics/[topic]/partitions/[partition]/state

上述节点有controller和leader进行维护：

## Controller

> kafka集群中的一个broker会被选举为controller，负责partition的管理和replica状态管理，也会执行partition重分配的任务。

Leader

> leader通过单独的线程定期检测ISR中follower





## 发送端 消息确认模式

- 0

  > 消息发送给leader后，不需要确认。性能高，风险最大，如果broker宕机，宕机之后发送的消息就会丢失

- 1

  > 只要集群中的leader确认即可返回

- -1 / all

  > 需要ISR中的所有Relica(副本)（集群中的所有broker）确认。还是有可能出现数据丢失，因为ISR可能缩小到只有一个Replica。

### commit策略

- server配置：

  ```properties
  #如果leader发现follower有10s没有发送fetch请求了，那么leader就把它从ISR移除。
  replica.lag.time.max.ms=10000
  ```

- topic配置

  ```properties
  #至少保证ISR中有几个replica
  min.insync.replicas=1
  ```

- producer配置

  ```properties
  #上述 0 1 -1
  request.required.asks=0
  ```



## consumer

consumer消费消息的时候可以指定分区



## Consumer group 组

一个topic的消息只能被同一个group中的一个consumer消费。

组内的消费者是竞争关系。

##### 作用

可以提高kafka的消费能力，当group中的consumer个数等于分区数的时候效率最高。

再多了也消费不到消息。白白浪费资源。



## 高效文件存储

拆分到segment，可以通过index快速定位message的位置和大小，并且可以通过index可以将数据映射到内存，避免io操作。

segment文件大小默认1G。



## 日志压缩

kafka根据策略定期清除过期消息，两种策略：

> 根据消息的保存时间
>
> 根据topic存储数据的大小

消息的key和value对应关系不断变化，消费者只关心key对应的最新的value。

如果开启日志压缩，kafka在后台启动一个线程，定期将相同的key对应的value合并，只保留最新的value值。



## 如何保证消息不丢失

- 生产端

  > 同步模式下：消息确认模式为1的时候(保证写入leader成功)，如果刚好leader partition挂了，会丢失数据。
  >
  > 异步模式下：缓存区满了，消息确认模式为0的时候(消息发出不需要确认)，会丢失数据。

- 消费端

  > consumer从topic中拉取到消息，还没有消费成功就提交了offset。



## 如何保证不重复消费

重复消费的原因：

1. 强行kill线程、消费系统宕机、重启，消费数据后，offset没有提交。
2. 设置offset为自动提交，如果在close之前，调用了consumer.unsubscribe()就有可能部分offset还没提交，下次就会重复消费。
3. 需要消费的消息中包含的数据处理非常耗时，超过了kafka的session timeout（0.10.0默认30s），如果offset还没有提交就re-balance了，就会在重平衡后重复消费。
4. 需要消费的消息中包含的数据处理非常耗时，在一个session周期内未完成，导致心跳机制检测报告出问题。
5. consumer重新分配partition的时候，可能从头开始消费。

解决方案：

>  kafka的offset就是为了解决重复消费问题的。

​	kafka解决不了的重复消费，就要使用以下方案：

​	结合业务逻辑对消息消费做**幂等处理**：

> 比如在redis或数据库记录已消费的消息，producer生产消息的时候入库消息未消费，consumer消费完在数据库update为已消费。



## 如何处理replica的全部宕机

​     如果leader挂掉了，会从replica中选取一个做为新的leader，并且从ISR中移除。如果所有的副本都宕机后，有两种策略：

1. 等待ISR中的任意一个replica恢复，并选举为leader；如果ISR中一个也不能恢复，该Partition永久不可用。相对较少丢失数据。
2. 选取第一个恢复的replica作为leader，不论是不是在ISR中（可能是OSR）。较高的可用性。



## leader重选举过程

https://blog.csdn.net/chenwiehuang/article/details/100172204



## 配置

kafka下的config/server.properties是配置文件

```properties
#这个目录下是可以看到kafka中的topic-分区号的 也就是说kafka的数据存储在硬盘
log.dirs=/tmp/kafka-logs
```

```properties
#开启可能会造成数据丢失
unclean.leader.election.enable=false
```



待补充待补充待补充待补充待补充待补充待补充待补充



## 命令

```sh
bin/kafka-server-start.sh config/server.properties #启动
bin/kafka-server-start.sh -daemon config/server.properties #后台启动
```

命令均在kafka的bin目录下

```sh
#创建topic
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
#查看topic列表
bin/kafka-topics.sh --list --zookeeper localhost:2181
#产生消息，创建消息生产者
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
#消费消息，创建消息消费者
bin/kafka-console-consumer.sh --broker-list localhost:9092 --topic test
#查看topic的消息
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
#删除topic
bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic test
#修改分区数为6
bin/kafka-topics.sh --alter --partitions 6 --zookeeper localhost:2181 --topic test
```

