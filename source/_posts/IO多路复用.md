简单说 IO：读取/写入 若干字节 从/到 **单个文件描述符**



- 阻塞 / 非阻塞

  - 阻塞：如果文件描述符不处于就绪状态则**阻塞直到其就绪**

  - 非阻塞：如果文件描述符不处于就绪状态则**返回一个错误码**，比如 `EAGAIN`

- 同步 / 异步

  > 同步和异步 是指 函数返回之时， 能不能保证任务完成。
  >
  > 比如调用了read()直接返回了一个Future对象，你需要再调Future的函数来获取read()的结果。

  - 同步 IO 操作将导致请求的进程一直被阻塞，直到 IO 操作完成。从这个层次来，阻塞 IO、非阻塞 IO 操作、IO 多路复用都是同步 IO
  - 异步 IO 操作不会导致请求的进程被阻塞。当发出 IO 操作请求，直接返回，等待 IO 操作完成后，再通知调用进程。

- I/O多路复用

  比如Linux的select/poll/epoll，macOS的kqueue

  I/O多路复用是通过一种机制，可以**监视多个fd**，一旦某个fd就绪（读就绪/写就绪），能够通知程序进行相应的读写操作。

  > 通知 也就是线程间的通信了，像future那种也就是通过共享变量喽



#### 几种IO多路复用

阻塞IO 处理一个IO就阻塞，所以一个线程只能处理一个fd，如果想要同时处理多个就得开多个线程， 效率不高消耗还大。

非阻塞IO 就得轮询所有你关注的fd，查看他们是否准备好了

- 忙轮询

  无一例外从头到尾闷头轮询，如果长时间没有fd就绪，就相当于一直空轮询了

- 无差别轮询

  让一个代理去轮询，如果没有就绪的事件，就阻塞当前线程，直到关注的fd中有就绪事件，在linux中这个代理就是 select/poll

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



#### I/O多路复用的优势

并不是对于单个连接能处理的更快，而是在于可以在单个线程/进程中处理更多的连接。

> 异步处理的是比较快，但是整体性能来看，上下文切换的损耗不得不考虑一下



#### 本质还是同步IO

IO多路复用(select/poll/epoll) 本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的



>https://blog.csdn.net/chewbee/article/details/78108223
>
>有更详细的 select/poll/epoll 函数原型的介绍



#### 使用epoll的例子

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



#### 关于linux select()的一个问题

问题  https://www.zhihu.com/question/37271342

> 假如我调用了一个 select 函数，并且关注了几个fd， select 函数就会一直阻塞直到我关注的事件发生. 假如当有套接口可读时， select 函数就返回了，告诉我们套接口已经可读，然后我们去读这个套接口，可以用阻塞的read或者非阻塞的 read，阻塞 read 是无数据可读就阻塞进程，非阻塞 read是无数据可读就返回一个 EWOULDBLOCK 错误。那么问题来了：既然 select 都返回可读了，那就表示一定能读了，阻塞函数read也就能读取了也就不会阻塞了，非阻塞read的话，也有数据读了，也不会返回错误了，那么这俩不都一样了？一样直接读取数据知道读完，为什么还得用非阻塞函数？还有 Reactor 模式也是用的 IO 多路复用与非阻塞 IO，这是什么道理呢？

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



