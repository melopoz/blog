---
title: Linux I/O模型
tags: 
  - IO
  - select
  - epoll
  - IO多路复用
categories: 
  - [OS]
  - [IO]
date: 2020/01/01 20:46:25
updated: 2020/11/12 16:00:25
---



I/O 简单点说就是：读取/写入 若干字节 从/到 **单个文件描述符**，用户进程需要调用内核并由内核来完成IO操作。

- 阻塞 / 非阻塞

  - 阻塞：如果文件描述符不处于就绪状态则**阻塞直到其就绪**

  - 非阻塞：如果文件描述符不处于就绪状态则**返回一个错误码**，比如 `EAGAIN`

- 同步 / 异步

  > 同步和异步 是指 函数返回之时， 能不能保证任务完成。
  >
  > 比如调用了read()直接返回了一个Future对象，你需要再调Future的函数来获取read()的结果。

  - 同步 IO 操作将导致请求的进程一直被阻塞，直到 IO 操作完成。从这个层次来，阻塞 IO、非阻塞 IO 操作、IO 多路复用都是同步 IO
  - 异步 IO 操作不会导致请求的进程被阻塞。当发出 IO 操作请求，直接返回，等待 IO 操作完成后，再通知调用进程。

阻塞非阻塞说的是内核



## Linux 五种 IO模型

> 用户态的进程调用read()其实是有[缓存IO](https://melopoz.github.io/blog/2021/01/01/OS/#%E7%BC%93%E5%AD%98IO)的，有两次拷贝过程。



### 同步阻塞-BIO

> blocking IO，用户进程 阻塞等待 kernel 返回结果，kernel 阻塞等待 fd读写完毕再返回结果。

- 在IO执行的两个阶段都被block了；
- 只需要一次系统调用（read）；
- 数据处理最及时；
- 内核实现简单；
- 一个线程（进程）只能处理一个fd的IO。



### 同步非阻塞 -NIO

>   non-blocking IO

> 用户态调用内核会直接得到结果，因为内核不等待读真正完成就返回结果（未完成就返回ERROR）。
>
> 用户态得到的结果如果并不是已完成，就不断轮询，轮询期间也可以做别的事情。
>
> > 非阻塞将`大整片时间的阻塞`分成`N 多的小片时间的阻塞`，所以进程不断地有机会拿到CPU时间片。
>
> 当用户进程发出 read 操作时，如果 kernel 中的数据还没有准备好，那么它并不会 block 用户进程，而是立刻返回一个 error。用户进程可以再次发送 read 操作。
>
> 一旦 kernel 中的数据准备好了，并且再次收到了用户进程的 system call，那么它就将数据（从内核空间）拷贝到了用户内存，然后返回。

- 需要由用户进程轮询自己关注的fd数据是否准备好了（用户进程调用 kernel 函数 read/recvfrom）；
- 只需要一次系统调用（read）；
- 一个线程（进程）只能处理一个fd的IO。



### IO多路复用 

> I/O多路复用（IO multiplexing）是由 **kernel** 监视（轮询）**多个fd**，一旦某个fd就绪（读就绪/写就绪），就通知用户进程进行相应的读写操作。
>
> > 比如Linux的[select/poll/epoll](#select/poll/epoll)，macOS的`kqueue`。

用户进程阻塞在`select/poll/epoll`函数上，这些函数只用来返回客户进程感兴趣的fd是否IO就绪；

用户进程得知fd已就绪后，需要再次调用读写操作来进行IO读写。

- 需要两次系统调用（而且[IO多路复用 第二步读取 推荐用非阻塞函数](#IO多路复用 第二步读取 推荐用非阻塞函数)）；
- 将一个用户进程要处理的多个fd的IO都阻塞在一个函数上（不用创建多个代价高昂的线程就达到了提升性能的目的），而BIO会有n次阻塞；
- 系统开销小
- 其实也是同步阻塞模型

> 在网络编程中，如果处理的连接数不是很高的话，使用`select/epoll`的不一定比使用`多线程 + blocking IO`性能更好（线程上下文切换带来的开销因为线程数量少所以可以忽略），可能延迟还更大。

IO多路复用优势并不是对于单个连接能处理得更快，而是在于能用少量线程处理大量的连接。



### 异步非阻塞IO -AIO

> Asynchronous IO，异步IO并不会顺序执行。

用户进程发起`aio_read`操作之后，kernel会立刻返回，所以不会对用户进程产生任何block。

然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，拷贝完成后，kernel会给用户进程发送一个`signal`或`执行一个基于线程的回调函数`来完成这次 IO 处理过程，以通知用户进程 `IO操作完成了`。

> 如果 kernel 发送`signal`给用户进程时，
>
> - 用户进程正在执行任务，就会打断用户进程，调用事先注册的`信号处理函数`(跟中断处理程序一样)，这个函数可以决定何时以及如何处理这个异步任务；
>
>   > 一般是把事件 “登记” 一下放进队列，然后返回该进程原来在做的事。
>
> - 如果用户进程调用内核态函数并阻塞（例如以同步阻塞方式读写磁盘），那就把这个通知挂起，等到它快要回到用户态的时候，再触发信号通知；
>
> - 如果用户进程是 被挂起 / sleep 的，就唤醒它，等他被调度拿到CPU，就会处理信号通知。

- 内核实现比较难

##### 异步阻塞模型

异步但是需要顺序调用几个IO操作的时候，就需要用异步阻塞方式，一般都是`实时系统`或者`延迟敏感的事务`。



### 信号驱动IO

> 

<img alt="IO模型时序图" src="https://raw.githubusercontent.com/melopoz/pics/master/img/IO%E6%A8%A1%E5%9E%8B%E6%97%B6%E5%BA%8F%E5%9B%BE.png" style="zoom:33%;" />

---



### select/poll/epoll



---



- 忙轮询

  无一例外从头到尾闷头轮询，如果长时间没有fd就绪，就相当于一直空轮询了

- 无差别轮询

  让一个代理（内核函数）去轮询，如果没有就绪的事件，就阻塞当前线程，直到关注的fd中有就绪事件，在linux中这个代理就是 select/poll

  ```c
  while true {
      select(streams[]) // 这里如果没有就绪的时间就阻塞
      for i in streams[] {
          if i has data
              read until unavailable//这里处理fd的时候还要注意 是不是已经处理过了 如果read()可能会一直阻塞下去，因为可能已经被其他人处理
      }
  }
  ```

  如果streams[]中有就绪的流，就会唤醒当前线程，如果100个stream中只有1个stream就绪了，我们还是要遍历这100个stream，时间复杂度永远是O(n)，处理的流越多，性能就越慢。

- 最小轮询 epoll

  可以比 无差别轮询 监听更多的fd

  ```c
  while true {
      active_stream[] = epoll_wait(epollfd)// 阻塞 直到有事件就绪；只返回已经就绪的事件；epollfd-关注的文件描述符
      for i in active_stream[] {
          read or write till unavailable
      }
  }
  ```





>https://blog.csdn.net/chewbee/article/details/78108223
>
>有更详细的 select/poll/epoll 函数原型的介绍



### 使用epoll的例子

可以监视多个fd上发生的指定的n种事件，获取到这些事件然后做相应的处理，比如epoll的使用一般都是这个框架

```c++
for( ; ; )
    {
        nfds = epoll_wait(epfd,events,20,500);//阻塞 直到有事件（就绪的io）
        for(i=0;i<nfds;++i)
        {
            if(events[i].data.fd==listenfd) //有新的连接
            {
                connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen); //accept这个连接
                ev.data.fd=connfd;
                ev.events=EPOLLIN|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //将新的fd添加到epoll的监听队列中
            }
            else if( events[i].events&EPOLLIN ) //接收到数据，读socket
            {
                n = read(sockfd, line, MAXLINE)) < 0    //读
                ev.data.ptr = md;     //md为自定义类型，添加数据
                ev.events=EPOLLOUT|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识符，等待下一个循环时发送数据，异步处理的精髓
            }
            else if(events[i].events&EPOLLOUT) //有数据待发送，写socket
            {
                struct myepoll_data* md = (myepoll_data*)events[i].data.ptr;    //取数据
                sockfd = md->fd;
                send( sockfd, md->ptr, strlen((char*)md->ptr), 0 );        //发送数据
                ev.data.fd=sockfd;
                ev.events=EPOLLIN|EPOLLET;
                epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改标识符，等待下一个循环时接收数据
            }
            else
            {
                //其他的处理
            }
        }
    }
```

> 这里也有关于epoll的参数介绍 https://www.cnblogs.com/fnlingnzb-learner/p/5835573.html



### IO多路复用 第二步读取 推荐用非阻塞函数

问题  https://www.zhihu.com/question/37271342

> 假如我调用了一个 select 函数，并且关注了几个fd， select 函数就会一直阻塞直到我关注的事件发生. 假如当有套接口可读时， select 函数就返回了，告诉我们套接口已经可读，接下来（**IO Multiplexing的第二步系统调用**）客户端进程可以用 **阻塞的read** 或者 **非阻塞的 read**：
>
> > 阻塞 read 是无数据可读就阻塞进程；
> >
> > 非阻塞 read是无数据可读就返回一个 EWOULDBLOCK 错误。
>
> 那么问题来了：既然 select 都返回可读了，那就表示一定能读了，阻塞函数read也就能读取了也就不会阻塞了，非阻塞read的话，也有数据读了，也不会返回错误了，那么这俩不都一样了？一样直接读取数据知道读完，为什么还得用非阻塞函数？还有 Reactor 模式也是用的 IO 多路复用与非阻塞 IO，这是什么道理呢？

答案

> 1. > Under Linux, select() may report a socket file descriptor as "ready for reading", while nevertheless a subsequent read blocks.  This could for example happen when data has arrived but upon examination has wrong checksum and is discarded.  There may be other circumstances in which a file descriptor is spuriously reported  as ready.  Thus it  may be safer to use O_NONBLOCK on sockets that should not block.
>    >
>    > 链接：https://www.zhihu.com/question/37271342/answer/81808421
>
>    Linux环境下, select()可能收到socket fd回复的“读就绪”, 紧接着read()会阻塞。这可能因为数据准备完成之后发生了错误被丢弃了。 其他情况下fd可能被错误地报告为“读就绪”。所以在不该阻塞的IO操作中 还是用非阻塞比较安全。
>
> 2. select()返回可读之后就直接去read()并不一定能读取到数据，select()内部调用fd是否就绪 和 调用read() 是两次系统调用，这之间是可能有其他事件发生的，
>
>    举个例子： 惊群现象，投一粒米，所有鸽子都来吃，只有一个吃到，然后所有鸽子都得回去睡觉。
>
>    这个可读的数据可能就是已经被其他进程/线程读完了。
>
>    链接：https://www.zhihu.com/question/37271342/answer/81757593





# Java中的NIO

todo！！！

java中使用的堆外内存，也是用户空间的，因为他是JVM进程使用`malloc`申请的内存，只不过在JVM管理的内存范围内。

https://sulangsss.github.io/2018/12/08/Java/Advance/ByteBuffer/

不过GC也会帮助清理堆外内存，GC清理了DirectByteBuffer对象，下次GC会调用Cleaner对象的clean()

https://kaiwu.lagou.com/course/courseInfo.htm?sid=&courseId=516#/detail/pc?id=4923

https://tech.meituan.com/2016/11/04/nio.html