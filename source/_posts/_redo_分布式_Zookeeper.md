---
title: zookeeper
tags: 
  - redo
  - 分布式协调服务
categories: 
  - 分布式
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



# zookeeper

分布式协调服务，为其他分布式应用程序提供基本的同步和组服务



为了可靠性，不能一个宕机就完蛋，也要搭建集群，搭建集群就分布式，分布式就要保证一致性，一致性的解决方案。。又是Paxos



选举一个leader，发送提案，多半同意则发送提交，

> 这里会有个问题 https://www.zhihu.com/question/324291664
>
> https://www.zhihu.com/question/324291664/answer/909822937 这个答案很清楚。 （就是依靠 最少半数节点+读时检测 保证的一致性）



因为要保证一致性，所以节点越多性能就越低，一般3、5个节点即可

> 奇数就是因为 保证节点一致性需要超过半数同意提案嘛， 5个节点和6个节点都需要3个节点同意才行，所以六个节点多出来那个没必要，还会增加节点通信的开销。



## ZAB zookeeper原子广播协议

Paxos的简单实现吧



## 下载安装

镜像下载链接：

https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.14/

zookeeper读取/conf/zoo.cfg文件作为配置文件，可复制zoo_simple.cfg在修改

```
bin/zkServer.sh start 		#启动zk服务
bin/zkServer.sh status 		#查看zk服务的状态
bin/zkServer.sh restart 	#重启
bin/zkServer.sh stop 		#停止
bin/zkCli.sh 				#连接zk服务  -server host:port
```

```
ls / 					#查看根目录下的内容
ls2 / 					#查看根目录下的内容和更新次数等具体信息
create /test "info1" 	#创建一个新的znode节点，和关联的字符串
get /test
set /test "info-update"
delete /test 			#删除节点
quit 					#退出客户端
```

```
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





