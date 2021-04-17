---
title: zookeeper
tags: 
  - 分布式协调服务
categories: 分布式
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---

官方文档：

> https://zookeeper.apache.org/doc/r3.5.8/zookeeperProgrammers.html

常用命令：

> https://blog.csdn.net/feixiang2039/article/details/79810102



**分布式协调服务**，可以用来做

1. **数据/服务发布/订阅**（by Watch）：做配置中心，修改某节点的数据，client通过zk.watch来更新配置。
2. **分布式协调/通知**（by Watch）
3. **负载均衡**：需要client实现负载均衡算法；
4. **命名服务**：记录服务名称，如kafka；利用顺序节点生成全局唯一的顺序ID（如业务上的项目编号）；
5. **集群管理**：记录某集群每个节点的信息（状态、地址），如kafka；
6. **分布式锁**：利用zk的强一致性、临时节点
7. **Master 选举**：利用zk的强一致性，多个broker去leader位置设置为自己，只有一个broker能成功，其他broker只能get该master的信息并记录下来；
8. **分布式队列**：by 顺序节点，FIFO；或者每个task完成之后在/job下创建taskN节点，/job下节点数量达到N，说明job完成。



## 数据模型

就像Unix的文件**树**。基本单位是 **ZNode** 。ZNode有四种：

- **持久**节点（persistent）：客户端与 Zookeeper 断开会话后，该节点依旧存在，直到执行删除操作才会清除节点
- 顺序**顺序**节点（persistent_sequential）：给该节点名称加上一个数字后缀，进行顺序编号
- **临时**节点（ephemeral）：和客户端的会话绑定，会话断开后，该节点被自动删除
- 临时**顺序**节点（ephemeral_sequential）：加数字后缀，进行顺序编号

ZNode结构中包含一下数据：

- **data** : Znode存储的数据；
- **ACL**：记录 Znode 的访问权限，即哪些人或哪些 IP 可以访问本节点；
- **stat**：包含Znode的各种源数据，包括**ZXID**、版本号、时间戳、数据长度等；
- **child**：子节点引用；

---

zk的数据有**内存数据**（树）和**磁盘数据**（快照、事务日志）。zk启动时会把磁盘中的数据加载到内存（持久化）。

内存数据主要分为DataTree、DataNode。

<img alt="zk-DataTree-DataNode" src="https://raw.githubusercontent.com/melopoz/pics/master/img/zk-DataTree-DataNode.png" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/zk-DataTree-DataNode2.png" style="zoom: 40%;" />

### DataTree

就全部数据了呗。

### DataNode

java实现嘛 肯定就是map了，再考虑多线程，必是ConcurrentHashMap了。其中`nodes`的key是节点的唯一标识：`path`；`ephemerals`临时节点需要和会话绑定，所以key是`sessionId`。



### Packet

zk的client和server传递信息都会封装成**Packet**进行传递。



## Watch 监听

- **One-time trigger**：一次性的。

- **Send to the client**：可能在 `发送修改请求的client1收到ok`之前，`watches事件`就被发送到了`设置watches的client2`（client2可能就是client1）。但是每个client收到的所有watches时间顺序都是一样的。

  > 每个client收到的所有监听事件是有序的，且监听事件肯定是在数据操作成功之后产生的。

- **The data for which the watch was set**：事件有类型之分。可以理解为两个watches list。**data watches**和**child watches**

  > `getData()` 和 `exists()` 设置 **data watches**，返回 根据数据类型决定的结果。
  >
  > `getChildren()` 设置 **child watches**，返回 子节点列表。



### 设置watch 和 触发watch

通过三种调用设置watches：**exists**、**getData**、**getChildren**。事务事件都会触发

- **Created event:** Enabled with a call to **exists**.        （调用 `exists()` 来监听 `created event`）
- **Deleted event:** Enabled with a call to **exists**, **getData**, and **getChildren**.
- **Changed event:** Enabled with a call to **exists** and **getData**.
- **Child event:** Enabled with a call to **getChildren**.

<img alt="zookeeper-watch" src="https://raw.githubusercontent.com/melopoz/pics/master/img/zookeeper-watch.png" style="zoom:67%;" />

注意：

- getData在ZNode创建时不会触发！因为获取数据必须要节点已经存在。（蓝色 row2 col1）

- **Znode的子节点创建或删除**，或是这个**Znode自身被删除**时，都会触发**getChildren**操作上的watch！

  > 可以通过查看watch事件类型来区分是Znode，还是他的子节点被删除：NodeDelete表示Znode被删除，NodeDeletedChanged表示子节点被删除。



### watch实现

watch是由server在本地维护的(`map<path, watches<paths>>`)。客户端断开再连接之后还可以收到之前注册的事件。

先看[线程模型](#线程模型)，再看这一章节。

> 可以看这篇源码笔记：https://www.cnblogs.com/wuzhenzhao/p/9994450.html

#### 注册watch

1. client注册watch

   > 将请求包装成Packet（`set watch=true`）后发送到server
   >
   > （这里发送完exists、getData之后会不断的轮询Selector，以获取操作执行结果，接第三步）。

2. 服务端处理watch

   > 服务器使用责任链模式，根据节点身份调用不同的处理函数。`setupRequestProcessors()`。
   >
   > 1. **AccpectThread**的`processRequest`只是将request添加到阻塞队列 `submittedRequests`。
   >
   > 2. **SelectThread**从 `submittedRequests` 中取出并在责任链中传播，在责任链最后会将watch保存在本地（map）。
   >
   >    `FinalRequestProcessor#processRequest(request)` 调用 `statNode(path, watcher)`，statNode 调用 `addWatch(path, watcher)`。
   >
   >    > ```java
   >    > //保存 watch 事件
   >    > public synchronized void addWatch(String path, Watcher watcher) {
   >    >     HashSet<Watcher> list = watchTable.get(path);
   >    >     if (list == null) {
   >    >         list = new HashSet<Watcher>(4);
   >    >         watchTable.put(path, list);
   >    >     }
   >    >     list.add(watcher);
   >    > 
   >    >     HashSet<String> paths = watch2Paths.get(watcher);
   >    >     if (paths == null) {
   >    >         // cnxns typically have many watches, so use default cap here
   >    >         paths = new HashSet<String>();
   >    >         watch2Paths.put(watcher, paths);
   >    >     }
   >    >     paths.add(path);
   >    > }
   >    > ```
   >    >
   >    > > 观察者模式吧，在map中存储Map(`key:path`  `value:Set<Watch>`)来保存watch信息。(每个watch还存储监听的多个path哦)
   >    > >
   >    > > 对path进行操作之后调用Set中的观察者的方法。
   >
   > 3. 然后等待时间触发，就调用path对应的watch
   >
   >    > **watch 事件触发**：processRequest会调用`getZKDatabase().processTxn(hdr, txn)`，其中`procesTxn(hdr, txn)`会调用`dataWatches.triggerWatch(path, EventType.NodeDataChanged);`。
   >    >
   >    > **triggerWatch** 方法名就证明他是做什么的。。
   >    >
   >    > > triggerWatch会取出所有watch，**从watch.paths移除当前path**，并调用`w.process(e);`
   >    >
   >    > `process()` 方法会调用 `ServerCnxn#process()` 方法（abstract方法，有两个实现类：NIOServerCnxn、NettyServerCnxn）
   >    >
   >    > 其中`NIOServerCnxn#process()`是调用了`sendResponse(h, e, "notification");`。就是向client发送通知了。

3. client不断轮询Selector，读写就绪都会进入`ClientCnxnSocketNIO#doIO`，这里看收到server的watch事件通知，所以应该是 处理`读就绪`：

   > `readResponse`读取并解析header
   >
   > > xid=**-2**：是ping的response，return
   > >
   > > xid=**-4**：AuthPacket的response，return
   > >
   > > xid=**-1**：是**watch通知**`notifacation`，应该读取并构造成**Event**，然后发送到`EventThread.queueEvent`。
   > >
   > > **else**：从pendingQueue拿出Packet，调用finishPacket注册本地事件（取出watch注册到ZKWatchManager）

#### watch触发

上述步骤的**2.3**之后，client还是会进入 `doIO()` ，然后 `readResponse()` 且 `xid=-1`，解析出Event，调用`queueEvent(WatchEvent event)`

> ```java
> public void queueEvent(WatchedEvent event) {
>     if (event.getType() == EventType.None && sessionState == event.getState()) {
>         return;
>     }
>     sessionState = event.getState();
>     // materialize the watchers based on the event
>     // 封装 WatcherSetEventPair 对象，添加到 waitngEvents 队列中
>     WatcherSetEventPair pair = new WatcherSetEventPair(
>         watcher.materialize(event.getState(), event.getType(),
>                             event.getPath()), event);
>     waitingEvents.add(pair); // 添加到队列 等待事件被处理
> }
> ```
>
> `waitingEvents` 就是 `EventThread` 的阻塞队列， 可以看EventThread的run()方法，循环调用`processEvent(event)`（`watcher.process(pair.event);`）。这样就调用到了自定义Watch的回调方法。



总体流程就是：

**SendThread**负责处理连接（发送 和 解析响应）；

**EventThread**负责处理事件队列中的事件，调用watch的回调。主要是这个线程。

> 事件监听回调，线程池、阻塞队列，还是这一套。就像是线程合作中 await 和 signal 的合作方式，只不过两个线程不是在同一个机器。
>
> 处理链接的也是由线程池来负责解析和处理，多步骤解耦，提高响应速度。

<img alt="zk-EventThread" src="https://raw.githubusercontent.com/melopoz/pics/master/img/zk-EventThread%E5%A4%84%E7%90%86WatchEvent.png" style="zoom:40%;" />



## 线程模型

### Client线程模型

#### 三个队列

- **OutgoingQueue**：存储 `待发送` 的消息
- **PendingQueue**：存储 `已发送,等待响应` 的消息
- **EventQueue**：存储 `watch监听到的事件`

#### 两个线程

- **SendThread**：消息发送线程，负责与server进行通信和超时重连。（如果server认定session过期，则这个线程会退出）

  > 将 QutgoingQueue队头 发送，并加入到 PendingQueue队尾。
  >
  > 同步发送：等待Event处理结束返回结果；
  >
  > 异步发送：直接返回结果，由处理Watch的线程后台进行回调Watch。

- **EventThread**：事件处理线程`org.apache.zookeeper.ClientCnxn.EventThread.processEvent(Objectevent)`

  > 以为Watch是一次性的，如果想要继续注册Watch，就需要在回调函数中再次注册自己。



client初始化时（`new Zookeeper（args) {new ClientCnxn(args).start();}`）就会启动这两个线程：**sendThread** 和 **eventThead**。

```java
public ClientCnxn(String chrootPath, HostProvider hostProvider, int sessionTimeout, ZooKeeper zooKeeper,
        ClientWatchManager watcher, ClientCnxnSocket clientCnxnSocket,
        long sessionId, byte[] sessionPasswd, boolean canBeReadOnly) {
    this.zooKeeper = zooKeeper;
    this.watcher = watcher;
    this.sessionId = sessionId;
    this.sessionPasswd = sessionPasswd;
    this.sessionTimeout = sessionTimeout;//会话超时
    this.hostProvider = hostProvider;
    this.chrootPath = chrootPath;
    connectTimeout = sessionTimeout / hostProvider.size();// 连接超时
    readTimeout = sessionTimeout * 2 / 3; //超时
    readOnly = canBeReadOnly;
    sendThread = new SendThread(clientCnxnSocket);// 新建了一个发送线程
    eventThread = new EventThread();// 处理watcher回调event的线程
}
//启动两个线程
public void start() {
    sendThread.start();
    eventThread.start();
}
```



### Server 线程模型

就是个Reactor单线程模型。

1. 主线程（**AcceptThread**）（通过`select()`）处理连接请求，将新连接放到 **SelectorThread** 的阻塞队列 `acceptedQueue`;
2. **SelectorThread** 线程从`acceptedQueue`中取出新建立的连接 **Channel** 注册到 **SelectorThread** 的 **Selector** 上，并监听**读**事件；
3. 当 **Channel** 可读时(说明请求数据已经写入到了buffer)，就会调用 **SelectorThread**.`handleIO` 方法，在 `handleIO` 方法中会将该连接派发到工作线程中进行处理(调用workerPool.schedule方法)。

将请求扔到队列（），负责处理请求的线程（）将队列（）中的请求取出来解析并扔到事件队列，负责处理



> 源码中用到很多 BlockingQueue、synchronized





## Session管理

zkClient要和一个zkServer不断交互，所以TCP长连接很合适。

只要在**SessionTimeout**之内zkClient和zkServer能够重新连接，原来session就还有效。

- **sessionID**：客户端创建会话的时候，zkServer会通过`initializeNextSession(long id)`生成一个全局唯一的sessionID

- **TimeOut**：会话超时时间，如果客户端与服务器之间因为网络闪断导致断开连接，并在TimeOut时间内未连上其他server，则此次会话失效，此次会话创建的临时节点将被清理

- **ExpirationTime**：下次会话超时时间点。其值接近于 `当前时间+TimeOut`

  > 超时检测采用**分段策略**，提高会话的超时检查与清理效率。



### 会话超时

Leader 会定期进行会话超时检查，默认时间间隔 **expirationInterval** 为 `2000ms`。

对于session的超时处理，采用一个分段策略，具体实现可以看`ExpiryQueue`：

> <img alt="图片来自https://blog.csdn.net/mystory110/article/details/77533376" src="https://raw.githubusercontent.com/melopoz/pics/master/img/zk-ExpiryQueue%E4%BC%9A%E8%AF%9D%E8%B6%85%E6%97%B6%E7%AE%A1%E7%90%86.png" style="zoom:50%;" />
>
> 将 **ExpirationTime** 处于同一区间的 session 放到一个 `Set`中，每次更新 ExpirationTime 时将该 session 加入到新位置的Set集合。
>
> <img alt="zk-获取session过期时间" src="https://raw.githubusercontent.com/melopoz/pics/master/img/zk-%E8%8E%B7%E5%8F%96session%E8%BF%87%E6%9C%9F%E6%97%B6%E9%97%B4.png" style="zoom:50%;" />
>
> 每次只检查下次超时时间内的session。提高效率。

如果会话过期，集群中所有节点就会删除该session对应的临时节点。



### 会话激活

**SessionTrackerImpl#touchSession**

- 每次client发送请求都会激活一次session；
- 如果client在`sessionTimeOut / 3`的时间之内没有发送请求，就会主动发起一次PING请求，触发激活。





## CP模型

> Zookeeper虽然是CP模型，但也是**最终一致性**，因为读请求打到Observer节点的时候可能数据还没更新过来。
>
> zookeeper既然不是强一致性的，那我们如何能保证两个客户端读到的数据是一致性的呢，那就是sync方法，zookeeper原生客户端API和Curator客户端都提供了该sync()方法，调用sync()方法之后，zookeeper集群会保证集群所有节点数据都是一致性的，此时客户端再去任意节点读取数据，都能读取最新的数据。

- 用户请求zookeeper有可能得不到响应

  > **任何时刻对ZooKeeper的访问请求能得到一致的数据结果**（C），同时系统对网络分割具备容错性；但是它不能保证每次服务请求的可用性。
  >
  > 可以通过client调用sync()来让集群数据一致，达到强一致性。
  >
  > （注：也就是在极端环境下，ZooKeeper可能会丢弃一些请求，消费者程序需要重新请求才能获得结果）。所以说，ZooKeeper不能保证服务可用性。

- 选举leader节点的时候，集群不可用

  > 选举leader的时间太长，30 ~ 120s，**选举期间注册服务瘫痪**，服务只是能够**最终恢复**。





# 集群架构

<img alt="zookeeper架构图" src="https://raw.githubusercontent.com/melopoz/pics/master/img/zookeeper%E6%9E%B6%E6%9E%84%E5%9B%BE.png" style="zoom:50%;" />

**Leader**：处理客户端**读写**请求；

**Follower**：处理**读**请求，**写请求转发给leader**，参与leader选举；

**Observer**：处理**读**请求，**写请求转发给leader**，**不参与leader选举**。（所以

节点的四种状态：

**LOOKING**：寻找Leader状态。当服务器处于该状态时，它会认为当前集群中没有Leader，因此需要进入Leader选举状态。

**FOLLOWING**：表明当前服务器角色是Follower。

**LEADING**：表明当前服务器角色是Leader。

**OBSERVING**：表明当前服务器角色是Observer。





## ☆ 集群下的数据一致性

https://www.zhihu.com/question/324291664/answer/909822937  （就是依靠 **最少半数节点**+**读时检测** 保证的一致性）

**读时检测**：client发送请求时会携带zxid，server如果发现zxid比当前节点的最大的zxid还大，说明server自己的数据并不是最新的，就会拒绝client的请求，client就会请求下一个server节点。





## ZAB：Zookeeper原子广播协议

写入的时候：过半写入成功，则认为成功。

选举leader的时候：某节点获得过半的投票，则当选leader。

> Paxos的简单实现吧





## Leader选举

选举一个leader，发送提案，多半同意则发送提交，节点的最大**zxid是 集群中最大的**节点才会当选。

投票通信时的数据结构：

> **id**：被推举的Leader的SID。
>
> **zxid**：被推举的Leader事务ID。(自己看到的节点中最大的)
>
> **electionEpoch**：逻辑时钟，用来判断多个投票是否在同一轮选举周期中，该值在服务端是一个自增序列，每次进入新一轮的投票 进行加1操作。
>
> **peerEpoch**：被推举的Leader的epoch。
>
> **state**：当前服务器的状态。

http://ningg.top/zookeeper-lesson-2-leader-election/

https://www.cnblogs.com/leesf456/p/6107600.html



## 跨机房部署

因为要保证一致性，所以节点越多性能就越低，一般3、5个节点即可

Observer就是一个读的横向扩展，可以多一点。

Leader和Follower应该在相同机房，网络环境比较稳定， 集群内通信效率高。除非担心整个机房一起挂。



# 常见问题

- [☆ 集群下的数据一致性](#☆ 集群下的数据一致性)

- 事务是什么时候刷盘的？

  > 事务每次刷盘都是一次IO操作，所以为了减少刷盘的次数，从而提高响应性能，zookeeper会将每次事务的请求写入都是先写到一个缓冲流中。
  >
  > **requestProcessor线程** 会从 **queuedRequests队列** 取出事务执行，并写入到事务日志文件的缓冲流中，当requestProcessor统计写入缓冲流的**事务超过1000** 或者 **队列已经没有事务**了，就会开始将缓冲流中的数据刷到磁盘块中。
  >
  > 至于刷盘的方式是可选择的，通过配置控制它是异步还是同步刷到磁盘中。

- 如果zookeeper集群全部宕机， 服务还可用吗？

  > 服务是可用，但是注册中心都没了。去哪发现服务获取服务地址呢？
  >
  > 除非调用服务的通信协议中完全不需要zk，但可能也只是再一小段时间内能调用provider的服务。最终肯定就不可用了。

- 怎么防止脑裂？

  > 选举时票数过半

- https://segmentfault.com/a/1190000014479433





# 下载安装

镜像下载链接：

https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/

zookeeper读取/conf/zoo.cfg文件作为配置文件，可复制zoo_simple.cfg在修改

```sh
bin/zkServer.sh start/restart/stop/status  # 启动/重启/停止/查看状态
bin/zkCli.sh 				#连接zk服务  -server host:port

ls / 					#查看根目录下的内容
ls2 / 					#查看根目录下的内容和更新次数等具体信息
create /test "info1" 	#创建一个新的znode节点，和关联的字符串
get /test
set /test "info-update"
delete /test 			#删除节点
quit 					#退出客户端

#nc命令用来设置路由器
echo stat | nc 127.0.0.1 2181 #来查看哪个节点被选择作为follower或者leader
echo ruok | nc 127.0.0.1 2181 #测试是否启动了该Server，若回复imok表示已经启动。
echo dump | nc 127.0.0.1 2181 #列出未经处理的会话和临时节点。
echo kill | nc 127.0.0.1 2181 #关掉server
echo conf | nc 127.0.0.1 2181 #输出相关服务配置的详细信息。
echo cons | nc 127.0.0.1 2181 #列出所有连接到服务器的客户端的完全的连接 / 会话的详细信息。
echo envi | nc 127.0.0.1 2181 #输出关于服务环境的详细信息（区别于 conf 命令）。
echo reqs | nc 127.0.0.1 2181 #列出未经处理的请求。
echo wchs | nc 127.0.0.1 2181 #列出服务器 watch 的详细信息。
echo wchc | nc 127.0.0.1 2181 #通过 session 列出服务器 watch 的详细信息，它的输出是一个与 watch 相关的会话的列表。
echo wchp | nc 127.0.0.1 2181 #通过路径列出服务器 watch 的详细信息。它输出一个与 session 相关的路径。
```


