---
title: HTTPS
tags: 
  - 安全
  - 加密
  - 非对称加密
categories:
  - [计算机网络]
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



# HTTP



### TCP三次握手

1. client 发送 SYN(seq=x) 到 server;   `client进入SYN_SEND状态`
2. server 收到并回应 SYN(seq=y), ACK(seq=x+1);   `server进入SYN_RECV状态`
3. client 收到 server 的SYN报文, 回应一个 ACK(seq=y+1) `client发送完进入ESTABLISHED状态, server收到也进入ESTABLISHED状态`

> seq number (x, y) 均为随机数, 抓包看到的一般为相对的数字, 即 0 和 1

### TCP四次挥手

1. client 发送 FIN(seq=x) 到 server;   `client只是不再发送数据, 但还可以接受数据`   `client进入FIN_WAIT_1状态`
2. server 回应 ACK(seq=x+1);   `server可能还需要发送数据到client, 所以不能一起发送FIN`   `server进入CLOSE_WAIT,client进入FIN_WAIT_2状态`
3. server 没有要发送的数据之后, 向 client 发送 FIN(seq=y);   `server进入LAST_ACK状态`
4. client 回应 ACK(seq=y+1)   `client发送完成进图established状态, server收到后进入CLOSED状态`   `client在等待固定时间(两个最大生命周期)后, 若没有接受到服务的ACK包, 则进入CLOSED状态`



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

