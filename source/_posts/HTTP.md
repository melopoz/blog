---
title: HTTP
tags: 
categories:
  - [计算机网络]
---



## HTTP 基于 TCP 



在 HTTP/1.0 中，一个服务器在完成HTTP响应之后会断开TCP链接，这样每次请求都会先建立 TCP 连接，最后再断开， 性能太差。



所以在 HTTP/1.1 中设置请求头 `Connection: keep-alive`， 默认开启长链接， 若要关闭则需要设置请求头： `Connection: close`

所以有时候刷新网页也不需要再建立SSL/TLS连接。



一个 TCP 连接在同一时刻只能处理一个请求， 即 两个请求的声明周期不能重叠。



浏览器对同一个 host 建立TCP连接有数量限制： Chrome 最多允许6个。

比如要请求上千张图片， 只用一个TCP连接肯定是太慢，



## HTTP/1.1 Pipelining   -队头阻塞

HTTP/1.1 是有 Pipelining 技术的， 但是浏览器默认关闭这个功能。

> 把多个HTTP请求放到同一个TCP连接中逐个发送， 不过不等待服务器的响应， 但是要按照发送请求的顺序来接收响应。
>
> 这样如果前一个请求的响应非常慢，后续的所有请求都要受到影响，即 **队头阻塞**。





## HTTP/2

1. 在应用层（HTTP）和传输层（TCP）之间添加了 **二进制分帧层**；
2. 提供了 Multiplexing 多路传输特性， HTTP/2 的 TCP 连接建立之后， 请求以 stream 的方式发送，每个 stream 的基本组成单位是 frame（二进制帧）。 在传送这些消息时 乱序发送，再在接收端重新组合；
3. Header 压缩， 压缩了 HTTP/1.1 每次都要重复发送且带有大量信息的header； `HPACK`
4. 服务端推送， server 会主动将一些资源推送给 client 并缓存起来。





HTTP/2 还是有 队头阻塞 问题，HTTP/2.0 只是解决了 HTTP 的队头阻塞问题， 并没有解决 TCP 的队头阻塞问题。

1. 如何解决了 HTTP 的队头阻塞？

   > http 阻塞的原因是要按顺序接收response， 所以http2.0 直接乱序接收，最后再按顺序拼起来。

2. 但是 http 是基于 TCP 的， TCP 还是有队头阻塞问题

3. TCP 不能升级？  **协议僵化**， 太底层的协议了， 若要升级，需要更新很多基础硬件设施。。。



## HTTP/3

TCP 队头阻塞的问题解决不了， 那就不用 TCP 了。。

所以 HTTP/3 是基于 UDP 的。不过核心是围绕底层的 QUIC 协议实现的。

QUIC 是在用户层构建的， 不需要每次协议升级的时候进行内核修改。   `???????`

新的HTTP头压缩机制： QPACK， 是 HPACK 的增强版。



嗯  所以呢？？？   不求甚解啦





# 参考

https://blog.csdn.net/hollis_chuang/article/details/111150623

https://www.hollischuang.com/archives/2066