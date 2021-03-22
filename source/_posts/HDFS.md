目录树结构

角色namenode datanode 角色即进程

### 角色

#### NameNode

基于内存存储文件元数据、目录结构、文件block的映射

需要持久化方案提供数据可靠性

提供副本放置策略

#### DataNode

基于本地磁盘存储block（块）（以文件形式）

保存block的校验和数据 保证block的可靠性

和NameNode保持心跳，汇报block的列表状态



### 数据恢复策略：

- 日志文件(记录所有操作，恢复时重新执行一遍)

  > 恢复慢 数据全

- 镜像 快照 dump db 序列化

  > 恢复快 数据不全(因为只能间隔一段时间备份一下)

#### HDFS的数据恢复策略：

- EditsLog  日志
- FsImage  镜像、快照

> EditsLog和FsImage存储在本地磁盘

HDFS采用最近时间点的FsImage + 增量的EditsLog

> 任何修改文件系统元数据的操作，NameNode都会使用EditsLog记录到事务日志
>
> 使用FsImage存储内存中所有元数据状态



### Block的副本放置策略

默认为每个数据块放3个副本，按照部署在NameNode上的默认机架感知策略存放数据副本。

- 第一个block副本放在上传文件的datanode，如果是集群外上传文件到hdfs，则随机挑选一个磁盘剩余空间大，cpu比较轻松的datanode；

- 第二个block副本放在和第一个副本 不同机架 中的另一个datanode（随机选择）；

- 第三个block副本放在和第二个副本 相同机架 中的节点；

  > 这样就只通过一个交换机(当前机架的交换机)，如果放在其他机架，就要通过更多交换机传输，效率会变低

- 第n个block副本放在 随机节点。

![hdfs-副本位置](images\hdfs-副本位置.pngx)



### HDFS写流程

![hdfs-写流程](images\hdfs-写流程.pngx)

> block 块  packet小包(64kb)  chunk大块(512kb)
>
> 写入数据的时候DN以chunk为单位进行数据校验。

想NN请求上传文件，NN返回是否可以上传，client对文件进行切分(比如一个block64M，200M的文件就会分成64 64 64 8)，client向NN请求第一个block应该存储到的DN服务器，NN返回第一个DN服务器，client请求第一个DN(RPC调用，建立pipeline)，client发送第一个packet到DN1，传输完成像DN1发送第二个packet，同时DN1向DN2发送第一个packet... ...  以此类推。

>  这样就比client发送整个大包到DN1，传输完成DN1发送打包到DN2... 效率要高很多

第一个block传输完成，client再次请求NN上传第二个block



### HDFS读流程

1. 跟NN通信查询元数据(block所在的DN的节点)，找到文件块所在的DN的服务器。
2. 挑选一台DN（就近原则，然后随机）服务器，请求建立socket流。
3. DN开始发送数据（从磁盘里读取数据放入流，以packet为单位做校验）
4. 客户端以packet为单位接收，先在本地缓存，然后写入目标文件中，后面的block块就相当于append到前面的block块，最后合成最终需要的文件

#### 读取某个指定的block

下载一个文件：（获取文件的所有block元数据）

- client和NN交互文件元数据获取fileBlockLocation
- NN按距离策略排序返回
- client尝试下载block并校验数据完整性

”下载一个文件“的子集：获取某些block

>  HDFS支持client给出我文件的offset，自定义连接某个block所在的DN，自定义获取数据.
>
> 这是支持计算层分治、并行计算的核心

