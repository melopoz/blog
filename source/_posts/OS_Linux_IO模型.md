---
title: I/O base Linux 
tags: 
  - select
  - poll
  - epoll
  - IO多路复用
categories: 
  - OS
  - IO
date: 2020/01/01 20:46:25
updated: 2020/11/12 16:00:25
---



I/O 简单点说就是：读取/写入 若干字节 从/到 **单个文件描述符**，用户进程需要调用内核并由内核来完成IO操作。

用户态的进程调用read()其实是有[缓存IO](https://melopoz.github.io/blog/2021/01/01/OS/#%E7%BC%93%E5%AD%98IO)的，会有两次拷贝过程（内核将数据从磁盘/设备拷贝到内核空间，再从内核空间拷贝到用户空间）。



- 阻塞 / 非阻塞

  - 阻塞：如果文件描述符不处于就绪状态则**阻塞直到其就绪**

  - 非阻塞：如果文件描述符不处于就绪状态则**返回一个错误码**，比如 `EAGAIN`

- 同步 / 异步

  > 同步和异步 是指 函数返回之时， 能不能保证任务完成。
  >
  > 比如调用了read()直接返回了一个Future对象，你需要再调Future的函数来获取read()的结果。

  - 同步 IO 操作将导致请求的进程一直被阻塞，直到 IO 操作完成。从这个层次来，阻塞 IO、非阻塞 IO 操作、IO 多路复用都是同步 IO
  - 异步 IO 操作不会导致请求的进程被阻塞。当发出 IO 操作请求，直接返回，等待 IO 操作完成后，再通知调用进程。



# Linux 五种 IO模型

<img alt="IO模型时序图" src="https://raw.githubusercontent.com/melopoz/pics/master/img/IO%E6%A8%A1%E5%9E%8B%E6%97%B6%E5%BA%8F%E5%9B%BE.png" style="zoom:33%;" />

### 同步阻塞-BIO

> blocking IO，用户进程 阻塞等待 kernel 返回结果，kernel 阻塞等待 fd读写完毕再返回结果。

- 在IO执行的两个阶段都被block了；
- 只需要一次系统调用（read）；
- 一个线程（进程）只能处理一个fd的IO；
- 数据处理最及时；
- 内核实现简单。



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



### 异步IO -AIO

> Asynchronous IO，要注意都是 异步IO并不会顺序执行。

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

> signal-driven I/O

1. 用户进程调用系统函数，系统函数直接返回；
2. 内核在数据拷贝到用户空间之后，向用户进程发送信号；
3. 用户进程收到信号之后调用`recvfrom`（阻塞）（由内核）将数据从内核空间复制到用户空间。

> 和IO多路复用也很类似，不过IO多路复用可以处理多个fd，而且IO多路复用的第一步也是阻塞的（看最上方的图）。

异步IO和信号驱动IO的区别

> 异步 I/O 的信号是**通知应用进程 I/O 完成**，而信号驱动 I/O 的信号是**通知应用进程可以开始 I/O**

---

# select/poll/epoll

Linux中IO多路复用的实现



#### select

`select`是`POSIX`规定的，一般操作系统都有实现。

```c
#include <winsock.h>
#define __FD_SETSIZE    1024
int select(
    int nfds, // 本参数忽略，仅起到兼容作用
    fd_set* readfds, // （可选）指针，指向一组等待可读性检查的套接口fd
    fd_set* writefds, // （可选）指针，指向一组等待可写性检查的套接口fd
    fd_set* exceptfds, // （可选）指针，指向一组等待错误检查的套接口fd
    const struct timeval* timeout // timeout：select()最多等待时间，对阻塞操作则应为NULL。
);
```

select是通过维护一个用来存放大量fd的数据结构，这样会使得用户空间和内核空间在传递这些fd时复制开销也随着fd的数量线性增大。也因此单个进程所打开的fd是有一定限制的，它由`FD_SETSIZE`设置，默认值是`1024`。

> 一般来说这个数目和系统内存关系很大，具体数目可以`cat /proc/sys/fs/file-max`查看。32位机默认是1024个；64位机默认是2048。

而且select对fd进行的是线性扫描，即轮询，效率较低。

> 当套接字比较多的时候，每次select()都要遍历`fd_setsize`个fd来完成调度，不管其中fd是否活跃，都遍历一遍。
>
> 这会浪费很多CPU时间。`如果能给套接字注册某个回调函数，当他们活跃时，自动完成相关操作，那就避免了轮询`，这正是epoll与kqueue做的优化。

##### 使用select

```c
while true {// 循环
    fd_set* readfds;
    fd_set* writefds;
    fd_set* exceptfds;
    select(.. readfds, writefds, exceptfds, ...) // 这里如果没有就绪的时间就阻塞
    for ready_fd in readfds {
        if read_fd has data
            // 这里处理fd的时候还要注意 是不是已经处理过了 如果read()可能会一直阻塞下去，因为可能已经被其他人处理
            read(read_fd, timeout) until unavailable
    }
}
```



#### poll

和select不同的是存储fd_set使用的链表，理论上没有最大连接数限制，fd_set被整体复制，同样的浪费空间；

poll是`水平触发`如果本次报告了fd可用，如果用户进程没有处理，下次还会报告这个fd。



#### select 和 poll 的弱点

都要传入fd数组，内核遍历数组把所有就绪的fd标记为已就绪，然后用户进程再遍历fd数组找到已就绪的fd进行处理。一般同一时间内要处理的fd只有少部分处于就绪状态。如果监控的fd越来越多，性能会线性下降。



#### **epoll**

epoll 是Linux2.6新增的优化版本，epoll 使用`一个fd`管理多个fd，将`用户进程关注的fd的事件`存放到内核的一个`事件表`中。



epoll的接口非常简单，一共就三个函数：

###### epoll_create(int size)

创建一个epoll文件描述符 来监听多个fd

> ```
> epoll_create() creates a new epoll(7) instance.  Since Linux
> 2.6.8, the size argument is ignored, but must be greater than
> zero; see NOTES.
> ```
>
> 参数size在2.6.8之后已经没什么用了，这个fd的size是内核动态调整的。不过参数size必须>0。
>
> https://man7.org/linux/man-pages/man2/epoll_create.2.html
>
> ```c
> int epoll_create(int size);
> 
> SYSCALL_DEFINE1(epoll_create, int, size)
> {
>     if (size <= 0)
>         return -EINVAL;
>     return sys_epoll_create1(0);
> }
> 
> /*
>  * Open an eventpoll file descriptor.
>  * 打开一个时间循环文件描述符
>  */
> SYSCALL_DEFINE1(epoll_create1, int, flags)
> {
>     int error, fd;
>     struct eventpoll *ep = NULL;
>     struct file *file;
> 
>     /* Check the EPOLL_* constant for consistency.  */
>     BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);
> 
>     if (flags & ~EPOLL_CLOEXEC)
>         return -EINVAL;
>     /*
>      * Create the internal data structure ("struct eventpoll"). 内部数据结构
>      */
>     error = ep_alloc(&ep);
>     if (error < 0)
>         return error;
>     /*
>      * Creates all the items needed to setup an eventpoll file. That is,
>      * a file structure and a free file descriptor.
>      * 创建描述（监听的fd的时间轮询）的文件描述符
>      */
>     fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
>     if (fd < 0) {
>         error = fd;
>         goto out_free_ep;
>     }
>     file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
>                  O_RDWR | (flags & O_CLOEXEC));
>     if (IS_ERR(file)) {
>         error = PTR_ERR(file);
>         goto out_free_fd;
>     }
>     ep->file = file;
>     fd_install(fd, file);
>     return fd;
> 
> out_free_fd:
>     put_unused_fd(fd);
> out_free_ep:
>     ep_free(ep);
>     return error;
> }
> ```



###### epoll_ctl(...)

epoll_ctl()用于向epoll注册事件，而且明确监听的事件类型

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// epfd: 返回值（epoll文件描述符）
// op: 动作：有三个宏（类似enum）可选：EPOLL_CTL_ADD,EPOLL_CTL_MOD,EPOLL_CTL_DEL
// fd: 要监听的fd
// *event: 要监听的事件
```

事件的数据结构

```c
typedef union epoll_data {
    void *ptr;
    int fd;// 事件对应的文件描述符
    __uint32_t u32;
    __uint64_t u64;
} epoll_data_t;

struct epoll_event {
    __uint32_t events; /* Epoll events */
    epoll_data_t data; /* User data variable */
};
```



> 1. 参数 **op** 可选的三个宏：增（EPOLL\_CTL\_**ADD**），改（EPOLL\_CTL\_**MOD**），删（EPOLL\_CTL\_**DEL**）
>
> 2. 参数 ***event** 有如下选择
>
>    | event        | 描述                                                         |
>    | ------------ | ------------------------------------------------------------ |
>    | EPOLLIN      | in,对应的文件描述符**可读**（包括对端 SOCKET 正常关闭）      |
>    | EPOLLOUT     | out,对应的文件描述符**可写**                                 |
>    | EPOLLPRI     | pri,对应的文件描述符**有紧急的数据可读**（这里应该表示有带外数据到来） |
>    | EPOLLERR     | err,对应的文件描述符**发生错误**                             |
>    | EPOLLHUP     | hup,对应的文件描述符**被挂断**                               |
>    | EPOLLET      | et,将epoll从 水平触发 **LT**(Level Triggered) **设为** 边缘触发 **ET**(Edge Triggered) 模式 |
>    | EPOLLONESHOT | oneshot,只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到epoll队列里 |



###### epoll_wait(...timeout...)

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
// epfd： 方法返回值
// *events： 调用者自己开辟一块事件数组，用于存储就绪的事件
// maxevents： *events size的最大值
// timeout： 等待超时时间（0 表示立即返回;-1 表示永久阻塞,直到有就绪事件）
```



##### 使用epoll

- 调用 create 创建 epoll 描述符（即下文代码中的 epfd 变量）
- 调用 ctl： 添加关注的事件
- 循环调用wait

```c
// --------------------------- 大致代码框架：
while true {
    active_fd[] = epoll_wait(epollfd) // 阻塞 直到有事件就绪(epoll_wait只返回已经就绪的事件)
    for i in active_fd[] { // 遍历所有已就绪的fd
        read or write till unavailable // 处理对应的事件
    }
}
// --------------------------- 具体代码：网上找的一个socket使用epoll的例子
epfd = epoll_create(1);
epoll_ctl(epfd, EPPOLL_CTL_ADD, ...);
for( ; ; )
{
    nfds = epoll_wait(epfd,events,20,500);// 调用epoll_wait,获取有事件发生的fd
    for(i = 0; i < nfds; ++i)
    {
        if(events[i].data.fd==listenfd) //如果是主socket的事件，则表示有新的连接
        {
            connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen); //accept这个连接
            ev.data.fd=connfd;
            ev.events=EPOLLIN|EPOLLET;
            epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //将新的fd添加到epoll的监听队列中
        }
        else if( events[i].events&EPOLLIN ) //接收到数据，读socket
        {
            if ( (sockfd = events[i].data.fd) < 0) continue;
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
            //其他情况的处理
        }
    }
}
```



##### epoll 实现原理

> 在linux，一切皆文件．
>
> 当调用 **epoll_create** 时，内核给这个 epoll 分配一个特殊的只服务于 epoll 的 file（epoll文件描述符-**epfd**），**epfd** 用来存储用户进程关注的fd(epoll_ctl的参数 int fd，比如**socket**）；
>
> 所以当内核初始化 epoll 时，会开辟一块**内核高速cache区**，用于存放 **epfd**，用户进程关注的fd会以**红黑树**的形式保存在内核的cache(**epfd**)里，同时建立一个**list链表，用于存储准备就绪的事件**。
>
> 执行**epoll_ctl**时，把**socket**放到**epfd**的红黑树上，并**给内核中断处理程序注册一个回调函数**；
>
> 当**socket**上有数据到了，就会触发中断，内核在把网卡上的数据copy到内核中后就会调用该回调函数，把socket插入到**准备就绪链表**里。
>
> 所以调用**epoll_wait**时，只要在timeout时间内，这个list链表有数据则拷贝至用户空间的events数组(参数***event**)中。



#### epoll的两种模式

水平触发(**LT** level trigger) 和 边缘触发 (**ET** edge trigger)。

> LT（**默认**）：用户进程**可以不立即处理** epoll\_wait 返回的事件，下次调用epoll_wait时，**会再次**响应应用程序并**通知此事件**。
>
> > 同时支持 `block socket` 和 `no-block socket`
>
> ET：用户进程**必须立即处理** epoll\_wait 返回的事件，如果不处理，下次调用 epoll_wait 时，epoll **不会再次**响应应用程序并**通知此事件**。
>
> > 只支持非阻塞套接口 `no-block socket`
>
> 一个常见问题：谁效率高。。
>
> > ET模式减少了fd事件被重复触发的次数，所以只针对这一次epoll调用来说，ET模式**效率更高**。
> >
> > 但是在ET模式下，为了保证数据都会处理，用户态必须用一个循环，将所有的数据一次性处理结束。所以用户态的逻辑复杂了，write/read 调用增多了，可能性能并不会更高。



#### epoll 和 select/poll 的区别/优化

1. epoll 没有fd数量限制，上限是最大可以打开文件的数目，一般远大于2048，1G内存的机器上是大约10万左右；
2. epoll 只需在用户空间和内核空间copy一个fd——**epfd**，而且 epfd 占用空间小，使用了CPU高级缓存；
3. epoll 只返回已就绪的fd，select / poll 都是返回用户进程关注的全部fd；





# 常见问题 & 拓展

Redis 和 Memcache 都是使用了基于IO多路复用的高性能网络库。



## Linux IO多路复用 使用了mmap？

select/poll/epoll 中均并没发现用mmap的痕迹啊，而且我觉得...本身就是在内核空间直接操作文件了，还用啥mmap啊...



## IO多路复用 第二步读取 推荐用非阻塞函数

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





## [Java中的NIO](#https://melopoz.github.io/blog/2021/01/01/Java_IO/#NIO)
