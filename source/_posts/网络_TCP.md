---
title: TCP连接
tags: 
  - 3握手4挥手
categorie:
  - [计算机网络,TCP]
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



图。。 todo todo todo

https://blog.csdn.net/Shuffle_Ts/article/details/93778635



## 为什么三次握手

简单点说，TCP是面向连接，也就是要双方都在通信中能发送能接受，并在建立连接时证明自己能发送且发送的信息被接受了。

一次请求可能携带 syn+ack，但是第一次请求只能是ack请求建立连接嘛，没收到过消息怎么能发送ack呢；

所以一次显然不够，必须双方都收到对方的syn；

两次可以保证双方都向对方发送了syn，

但是两次通信肯定做不到双方都能回应一个ack（server并不能确定自己发送的syn被client收到了）；

> 第三次握手的目的：
>
> 防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误
>
> 具体场景就是：
>
> > 1. client发送syn（syn-1），等待server的ack，server没收到；
> > 2. client等不到server的ack，又发送syn（syn-2），server收到并返回了ack，双方准备连接的资源；
> > 3. 过了一会，server收到了syn-1回复一个ack并准备资源；
> > 4. client莫名其妙收到一个ack。。不会处理，不理睬
> >
> > 上述情况导致server资源浪费。虽然可以通过server长时间没接收到数据就关闭连接来释放，但还是浪费了，很多服务器还是不能容忍这种情况的。

两次 = 四次 = 六次 （偶数次的都可能会有上述问题）

三次 = 无磁 = 七次 。（就为了解决上边的问题）多几次也没用了。三次即可。



#### 如果第一次握手失败（server没收到syn）

server就没收到请求，client也没收到ack，都不申请资源。

#### 如果第二次握手失败（client没收到syn+ack)

client没收到server的ack+syn,不会申请资源；

server已经申请资源，等待client的ack；

#### 如果第三次握手失败(server没收到ack)

> Client收到server端发来的SYN + ACK ，Client变成ESTABLISHED状态。然后发送ACK到server端，server接收后变成ESTABLISHED状态。
>
> 所以client认为连接已经建立开始发送数据

Server：该TCP连接的状态为SYN_RECV，收不到cilent的ack，根据 TCP的超时重传机制，会等待3秒、6秒、12秒后重新发送SYN+ACK包，以便Client重新发送ACK包，如果最终还是超时，就关闭这个连接，就会释放资源；

> server重发syn+ack的次数可以通过`/proc/sys/net/ipv4/tcp_synack_retries`修改，默认为 5

client：因为established状态，所以会发送数据包，

server：收到client的数据包，但是因为之间没有收到client的第三次握手，所以回复一个RST；

cilent：收到RST，重新开始握手以建立连接。





## 为什么四次挥手

todo 图

https://blog.csdn.net/Shuffle_Ts/article/details/93909003



全双工

我要关闭了  -   你关吧  ， 我要关闭了  - 你关吧

相对于三次握手来说，只是  被动关闭方的 “你关吧“和”我要关闭了“不能放到一起。

因为主动发发送完数据就可以申请关闭连接，而被动方收到关闭请求的时候可能还有任务要执行，比如还有数据要返回给对方，所以要等到自己主动提出关闭连接并收到对方的确认才行。

所以“我要关闭了”只能是表示我没有 要发送的数据了。我还能接受数据。