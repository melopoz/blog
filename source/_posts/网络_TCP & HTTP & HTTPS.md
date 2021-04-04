---
title: TCP & HTTP & HTTPS
tags: 
  - http
  - https
  - 对称/非对称加密
categories: 网络
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---

# OSI

七层模型，各种协议。。

https://m.yisu.com/zixun/46320.html



# TCP

<img alt="TCP建立连接" src="https://raw.githubusercontent.com/melopoz/pics/master/img/TCP34.png" style="zoom:50%;" />

## 三次握手

> seq number (x和y) 真正数值均为随机数, 抓包看到的一般为相对的数字, 即 0 和 1

1. client 发送 SYN(seq=**x**) 到 server;   

   > client进入**SYN_SEND**状态

   > **如果失败**（server没有收到syn）：server就没收到请求，client也没收到ack，都不申请资源。

2. server 收到并回应 SYN(seq=**y**), ACK(seq=**x+1**); 

   > server进入**SYN_RECV**状态

   > **如果失败**（client没收到syn+ack）：server已经申请资源，等待client的ack；client不会申请资源；

3. client 收到 server 的SYN报文, 回应一个 ACK(seq=**y+1**)

   > client发送完进入**ESTABLISHED**状态, server收到也进入**ESTABLISHED**状态

   > **如果失败**（server没收到ack）：client变成ESTABLISHED状态，准备发送数据；server还是**SYN_RECV**状态，根据 **TCP的超时重传机制**，会等待3秒、6秒、12秒后**重新发送SYN+ACK包**，如果最终还是超时，就关闭这个连接，释放资源；
   >
   > > 如果server没收到client的第三次握手就收到了client的数据包，会回复一个**RST**，client收到**RST**会重新开始握手
   >
   > > server重发syn+ack的次数可以通过`/proc/sys/net/ipv4/tcp_synack_retries`修改，默认为 5
   >



### 为什么三次握手

> 全双工可靠连接，必须保证双方都**能接收 能发送**，建立链接时必须证明**自己能发送**且**发送的信息被接受(收到ack)**了。

两次可以保证双方都向对方发送了syn，但不能保证都收到对方的回应(ack)（**server并不能确定自己发送的syn被client收到了，必须再收到客户端的ack才行**）。

##### 呃...另外一个多余的原因：

并且三次握手可以**防止已失效的连接请求报文段突然又传送到了服务端**，具体场景如下：

> （如果仅需两次握手即可建立连接）：
>
> 1. client发送syn（syn1），等待server的ack，server没收到；
>
> 2. client等不到server的ack，又发送syn（syn2），server收到并返回了ack，双方准备连接的资源；
>
> 3. **过了一会，server收到了syn1回复一个ack并准备资源**；
>
>    > client莫名其妙收到一个ack。。不理睬，但是server已经开资源了，就会导致server资源浪费。
>    >
>    > 虽然server可以通过超时机制释放该资源，但是很多服务器还是不能容忍这种情况的。



## 四次挥手

> x、y ： 上次通信的x和y

1. client 发送 FIN(seq=x) 到 server；

   > client进入**FIN_WAIT_1**状态；client只是不再发送数据，但还可以接受数据。

2. server 回应 ACK(seq=x+1)；

   > server进入**CLOSE_WAIT**，client收到后进入**FIN_WAIT_2**状态；server可能还需要发送数据，所以不能一并发送FIN。

3. server 没有要发送的数据之后，向 client 发送 FIN(seq=y)；

   > server进入**LAST_ACK**状态

4. client 回应 ACK(seq=y+1)

   > client发送完成进图**established**状态，server收到后进入**CLOSED**状态；
   >
   > client在等待2MSL(两个最大生命周期)后，如果没有收到server重发的**FIN**包，则进入**CLOSED**状态。
   >
   > > MSL：报文段最大生存时间（任何报文段被丢弃前网络内的最长时间）。
   >
   > > **如果失败**（server没有收到client发送的ACK(y+1)）：
   > >
   > > server就会超时重传FIN(seq=y)，如果 client 再次收到 FIN 就会再发送 ACK(y+1)，server 收到 ACK，断开连接。
   > >
   > > 也因此等待时间至少是 server 的 timeout＋FIN 的传输时间，保守起见，使用2MSL。



### 为什么四次挥手

大致与握手相同，不过三次握手是因为server把ack和syn放到一起发送了，这样成了三次。

但是断开连接的时候server不能把要发送的ACK和FIN放到一起，因为主动发送FIN的意思是“我不在发送数据了”，而server其实可能还又要发送的数据。





# HTTP

超文本传输协议，基于TCP

在 HTTP/1.0 中，一个服务器在完成HTTP响应之后会断开TCP链接，这样每次请求都会先建立 TCP 连接，最后再断开， 性能太差。

在 HTTP/1.1 中设置请求头 `Connection: keep-alive`， 默认开启长链接， 若要关闭则需要设置请求头： `Connection: close`

> 所以有时候刷新网页也不需要再建立SSL/TLS连接，非对称加密只用来对 秘钥进行加密，ssl连接建立之后就使用对称加密了。



一个 TCP 连接在同一时刻只能处理一个请求， 即 两个请求的声明周期不能重叠。

浏览器对同一个 host 建立TCP连接有数量限制： Chrome 最多允许6个。

比如要请求上千张图片， 只用一个TCP连接肯定是太慢，



### HTTP/1.1 Pipelining   -队头阻塞

HTTP/1.1 是有 Pipelining 技术的， 但是浏览器默认关闭这个功能。

> 把多个HTTP请求放到同一个TCP连接中逐个发送， 不过不等待服务器的响应， 但是要按照发送请求的顺序来接收响应。
>
> 这样如果前一个请求的响应非常慢，后续的所有请求都要受到影响，即 **队头阻塞**。





### HTTP/2

1. 在应用层（HTTP）和传输层（TCP）之间添加了 **二进制分帧层**；
2. 提供了 Multiplexing 多路传输特性， HTTP/2 的 TCP 连接建立之后， 请求以 stream 的方式发送，每个 stream 的基本组成单位是 frame（二进制帧）。 在传送这些消息时 乱序发送，再在接收端重新组合；
3. Header 压缩， 压缩了 HTTP/1.1 每次都要重复发送且带有大量信息的header； `HPACK`
4. 服务端推送， server 会主动将一些资源推送给 client 并缓存起来。



HTTP/2 还是有 队头阻塞 问题，HTTP/2.0 只是解决了 HTTP 的队头阻塞问题， 并没有解决 TCP 的队头阻塞问题。

1. 如何解决了 HTTP 的队头阻塞？

   > http 阻塞的原因是要按顺序接收response， 所以http2.0 直接乱序接收，最后再按顺序拼起来。

2. 但是 http 是基于 TCP 的， TCP 还是有队头阻塞问题

3. TCP 不能升级？  **协议僵化**， 太底层的协议了， 若要升级，需要更新很多基础硬件设施。。。



### HTTP/3

TCP 队头阻塞的问题解决不了， 那就不用 TCP 了。。

所以 **HTTP/3 是基于 UDP 的**。不过核心是围绕底层的 QUIC 协议实现的。

QUIC 是在用户层构建的， 不需要每次协议升级的时候进行内核修改。   `???????`

新的HTTP头压缩机制： QPACK， 是 HPACK 的增强版。



嗯  所以呢？？？   不求甚解啦



# 确保安全通信的三个原则

1. 数据内容加密

2. 通信双方身份校验

   > 确保你是在给 （给你发消息的人）发消息

3. 数据内容完整

   > 确保对方能够完整地读完我的消息



http是明文, 不安全 所以需要加密来保证数据安全



### 对称加密 和 非对称加密

对称加密:

> 使用同一个密钥进行加密和解密, 速度快, 但是必须在通信时告诉对方使用什么密钥, 一旦密钥被截获, 相当于仍然是明文.

非对称加密:

> 公钥随便发，通信双方各自保管私钥。使用私钥加密则必须用公钥解密，使用公钥加密则必须使用私钥解密。



# HTTPS设计思路

1. server为客户端生成一个公钥并发送给client；
2. client选择一个加密算法，用公钥把这个算法加密后发送给server；
3. server收到后用私钥解密得到加密算法，以后都用这个算法进行数据包的decode。



### 缺陷

中间人可以对client伪装成server， 对server伪装成client， 仍然可以截获真数据包。



### 解决方案

既然在通信中发送公钥危险，那就别直接发送公钥了呗。把这个公钥交给可信任的第三方机构。

1. 由CA向 server 颁发证书，来证明服务器是安全的；

2. server 将证书（包含 server 的公钥）发送给 client；

3. client 收到加密后的公钥之后，用CA的公钥把server发来的公钥解密得到 通信中使用的公钥。

   > 操作系统或者浏览器本身就带有 CA 的公钥，server 发送给 client 的证书是由 CA 用私钥加密过的。

这样一来，中间人也就只能作为client请求server，但不能伪装成server回应client。



### SSL/TLS

也就是上述结局方案了。具体步骤为：

1. client 发送请求到 server， 生成并携带随机数 random1 和 支持的加密算法； `Client Hello`

2. server 取出random1和client支持的加密算法，从中取出一个算法，生成 random2；`Server Hello`

   server 将自己的数字证书（含有server的公钥）发送给 client，让 client 对自己进行身份校验； `Certificate`

   ..”server 的工作完成了“   `Server Hello Done`

3. client 校验证书是否合法，生成 random3，用服务器的公钥加密 random3 得到 premaster，并发送给 server；`Client Key Exchange` `Change Cipher Spec` `Encrypted Handshake Message ` 

4. server 收到加密的 random3（premaster）之后，用私钥解密，得到 random3 ， 用这三个随机数生成一个秘钥，这个秘钥将作为接下来 对通信数据进行对称加密 的秘钥；

   `New Session Ticket`  `Change Cipher Spec`  `Encryted Handshake Message` 接下来要使用对称加密了；

   > session ticket ： 一个状态保存机制， 下次请求https的时候，携带这个ticket， 节省握手时间、资源消耗

5. 发送 加密后的 `Applation Data`   通信数据



> 清楚明了，无敌：http://www.ruanyifeng.com/blog/2014/02/ssl_tls.html



## 总结

HTTPS = TLS 五次握手 + HTTP 三次握手

> 举个例子。。。        TLS 110-120ms       TCP 50-60ms 

1. 因为TLS很耗性能耗时间， 所以使用非对称加密 把   对称加密中使用的秘钥a   加密，然后都用这个秘钥a进行对称加密。
2. 服务器的公钥私钥只用来加密和解密“对话秘钥（sessionkey）”（对 ApplicationData 进行对称加密的秘钥），无其他作用。
3. 一般还会使用 session ticket 或 session id 技术来提高性能。





---

> CA证书  只是证明 服务器的合法性
>
> 客户端安装了抓包工具的根证书，就会信任（伪装成server的）抓包工具， 这时抓包工具就可以伪装成服务器与客户端进行通信了。
>
> 而抓包工具与真服务器进行通信是不需要证明的， 因为在这个通信中（抓包工具）就是客户端。





# 自签名， 搭建HTTPS服务



#### 需要做的事：

> - 一个提供 https 服务的服务器需要有经过 CA 认证的数字证书和自己的公钥私钥（公钥放在证书里）
> - CA 认证是需要收费的，所以我们需要自建一个 CA 机构
> - 自建的 CA 机构需要有标识自己身份的根证书（包含公钥），以及用来对服务器证书进行签名(加密)的私钥
> - 一个 https 站点，收到客户端访问时下发自己的数字证书
> - 客户端需要持有 CA 的根证书，所以我们需要手动导入自建 CA 的证书





---

自己通过 keytool 生成一个证书，

1. 生成服务端私钥 generate private key（.key)

   ```
   # Key considerations for algorithm "RSA" ≥ 2048-bit
   openssl genrsa -out server.key 2048
   
   # Key considerations for algorithm "ECDSA" ≥ secp384r1
   # List ECDSA the supported curves (openssl ecparam -list_curves)
   openssl ecparam -genkey -name secp384r1 -out server.key
   ```

2. 生成服务端自签名证书

   `openssl req -new -x509 -sha256 -key server.key -out -server.crt -days 3560`

3. 填写信息：

   ```
   Country Name (2 letter code) [AU]:CN
   State or Province Name (full name) [Some-State]:Guangdong
   Locality Name (eg, city) []:FoShan
   Organization Name (eg, company) [Internet Widgits Pty Ltd]:TestCA
   Organizational Unit Name (eg, section) []:
   Common Name (e.g. server FQDN or YOUR name) []:xxx.com #域名
   Email Address []:
   ```









# 参考

https://www.cnblogs.com/Hui4401/p/14128112.html

https://blog.csdn.net/u010983881/article/details/83588326

https://juejin.cn/post/6844903522333376525    RSA非对称加密算法

