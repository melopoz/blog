---
title: Java中的IO
tags: 
categories:
  - [Java]
---



## IO

1.4之前， 传统io，阻塞 所以又叫BIO（Blocking IO）

数据的读取写入必须阻塞在一个线程内等待其完成。

比如用socket，想要多个client同时访问就要开同样多个线程来处理client的链接，线程资源太宝贵了，这不OK。



## NIO

New IO / Non blocking IO

### 核心组件：

- ##### channels

  通道是双向的，流的读写是单向的

- ##### buffers

  比如ByteBuffer，其他每种java基本类型(除了boolean)都对应一种缓冲区

- ##### selectors

  用于单个线程处理多个通道。

### NIO的特点也就是和IO的不同之处：

- NIO面向缓冲区，IO面向流

- NIO有选择器，IO没有
- NIO非阻塞，IO阻塞

### NIO读写数据的方式：

- 从通道（channels）读取：创建一个缓冲区，请求通道读取数据。
- 从通道（channels）写入：创建一个环城区，填充数据，并要求通道写入数据。



## AIO

**Asynchronous** IO

**异步IO**是基于**事件**和**回调机制**实现的，应用操作之后会直接返回。当后台处理完成，操作系统通知相应线程进行后续的操作。