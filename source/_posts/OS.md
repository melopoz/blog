---
title: OS
tags: 
  - 内存管理
  - 零拷贝
  - IO多路复用
  - select
  - poll
  - epoll
categories: OS
date: 2021/01/01 20:46:25
updated: 2021/01/01 20:46:25
---



# 操作系统四大特征：

### 并发

两个事件在**同一段时间内**发生，即宏观上同时运行；**并行**是两个事件在统一时刻发生，多(核)CPU可支持并行

### 共享（资源复用）

多个并发执行的进程共同使用同一系统资源，比如内存

> 互斥共享：同一时刻只有一个进程可以访问，需要同步机制来实现互斥。互斥的资源成为**临界资源**。e.g.:摄像头
>
> 同时访问：允许多个进程在**同一段时间内**访问同一资源，即宏观上的同时。（微观上还是交替进行）e.g.:磁盘

### 虚拟

程序需要放到内存中才能运行，但是太大的程序（比如10个G）不能都放到内存中，只把部分数据放到内存中，让4G内存也能运行10G的程序。

> **时分复用**（把时间段分片）；**空分复用**（把较宽的信道分成多个较窄的信道）
>
> 空分复用 e.g.：需要100M内存的程序只用30M的内存，因为程序总是趋向于使用最近使用过的数据和指令，所以内存中只存放一部分，用完内存中这部分或必须要用内存中没有的数据，则把其他部分换进来

> 相关：虚拟内存，内存映射...

### 异步

因为任务多，资源少的情况，所以进程通常都是走走停停的方式进行。





# CPU

> **Register** ：寄存器，是CPU对接收到的信息直接处理的地方
>
> **word**：字，MIPS指令集术语，一个32bit的二进制串，组成一个word，一个register可load一个word，一个instruction可以是一个word
>
> **Instruction**：指令，32位的cpu通常就是指这个instruction的长度为32bit

## 寻址空间

32位CPU， 理论上 寻址空间 2^32, 4GB（32bit，可以表示2^32^ = 4G 种可能(不是4GByte)）

64位CPU， 理论上 寻址空间 2^64, 16EB = 160亿GB（只是理论上支持，但是大多64位PC也只能到32GB）

但实际寻址空间还取决于**总线宽度**，而且一般没有那么大内存，64位机器常用的也就`2^42~2^47`（`4T~`）吧



# 总线 Bus

数据总线

地址总线

控制总线



# 内核态 / 用户态

内核态：操作系统管理程序运行时的状态。运行在内核态的程序可以访问计算机的任何资源，比如协调CPU，分配内存资源；

用户态：应用程序运行时的状态。用户态的程序只能访问当前CPU上执行的程序所在的地址空间。

区分内核态和用户态是为了**安全，控制权限，保护操作系统程序**。

OS在内核态会在合适的情况（比如执行完某IO命令）把CPU使用权还给应用程序，即转换回用户态。但是从用户态转换到内核态，只能通过[中断机制](#中断机制)来实现。



### Linux内核空间

Linux会给程序分配4G（2^32）的虚拟地址空间（[虚拟内存](#虚拟内存)），这4G又分为3G用户空间和1G内核空间（只能在内核态访问）（Windows为2:2），

> 使用命令`cat /proc/cpuinfo` 可以查看到 `address sizes   : 36 bits physical, 48 bits virtual` （在win10的Ubuntu子系统中）
>
> 2^32^物理地址空间，2^48^虚拟地址空间



# 进程管理

> 进程：程序的执行过程，是暂时的。 （也可以说包含正在运行的程序实体和其占据的所有系统资源, CPU,内存,网络）
>
> 进程又可以有多个线程，需要操作系统来执行上下文切换，让多个进程/线程使用同一个CPU资源

### 进程调度

多任务系统中用于管理CPU的时间片分配，OS**内核**必须有能力挂起正在 CPU 上运行的进程，并恢复以前挂起的某个进程的执行。

> 进程在不同的操作系统中有些称为process，有些称为task。操作系统中进程数据结构包含了很多元素，往往用链表连接。
>
> 进程相关的内容主要包括：虚拟地址空间，优先级，生命周期(阻塞，就绪，运行等)，占有的资源(例如信号量，文件等)。
>
> CPU在每个系统滴答(Tick)中断产生的时候检查就绪队列里面的进程(遍历链表中的进程struct)，如有符合调度算法的新进程需要切换，保存当前运行的进程的信息(包括栈信息等)后挂起当前进程，选择新的进程运行，这就是**进程调度**。

> 进程的优先级差异是CPU调度的基本依据，调度的终极目标是让高优先级的活动能够即时得到CPU的计算资源(即时响应)，低优先级的任务也能公平分配到CPU资源。因为需要保存进程运行的上下文(process context)等，**进程的切换本身是有成本的**，调度算法在进程切换频率上也需要考虑效率。

#### CFS：Completely Fair Scheduler

> 在**早期的Linux**操作系统中，主要采用的是**时间片轮转算法**(Round-Robin)，内核在就绪的进程队列中选择高优先级的进程运行，每次运行相等的时间。该算法简单直观，但仍然会导致某些低优先级的进程长时间无法得到调度。为了提高调度的公平性，在Linux 2.6.23之后，引入了称为**完全公平调度器CFS**(Completely Fair Scheduler)。
>
> CPU在任何时间点只能运行一个程序，用户在使用优酷APP看视频时，同时在微信中打字聊天，优酷和微信是两个不同的程序，为什么看起来像是在同时运行？CFS的目标就是让所有的程序看起来都是以相同的速度在多个并行的CPU上运行，即nr_running 个运行的进程，每个进程以1/nr_running的速度并发执行，例如如有2个可运行的任务，那么每个以50%的CPU物理能力并发执行。
>
> CFS引入了"虚拟运行时间"的概念，虚拟运行时间用p->se.vruntime (nanosec-unit) 表示，通过它记录和度量任务应该获得的"CPU时间"。在理想的调度情况下，任何时候所有的任务都应该有相同的p->se.vruntime值(上面提到的以相同的速度运行)。因为每个任务都是并发执行的，没有任务会超过理想状态下应该占有的CPU时间。CFS选择需要运行的任务的逻辑基于p->se.vruntime值，非常简单：它总是挑选p->se.vruntime值最小的任务运行(最少被调度到的任务)。
>
> CFS使用了基于时间排序的红黑树来为将来任务的执行排时间线。所有的任务按p->se.vruntime关键字排序。CFS从树中选择最左边的任务执行。随着系统运行，执行过的任务会被放到树的右边，逐步地地让每个任务都有机会成为最左边的任务，从而在一个可确定的时间内获得CPU资源。
>
> 总结来说，CFS首先运行一个任务，当任务切换(或者Tick中断发生的时候)时，该任务使用的CPU时间会加到p->se.vruntime里，当p->se.vruntime的值逐渐增大到别的任务变成了红黑树最左边的任务时(同时在该任务和最左边任务间增加一个小的粒度距离，防止过度切换任务，影响性能)，最左边的任务被选中执行，当前的任务被抢占。
>
> <img src="https://raw.githubusercontent.com/melopoz/pics/master/img/CFS%E5%AE%8C%E5%85%A8%E5%85%AC%E5%B9%B3%E8%B0%83%E5%BA%A6.png" style="zoom:50%;" />

参考：https://blog.csdn.net/weixin_39722375/article/details/111206454



### 进程阻塞

正在执行的进程，`请求系统资源失败`、`等待某种操作的完成`、`新数据尚未到达`或`无新任务可以执行`等，则由系统自动执行阻塞原语(`Block`)，使自己由运行状态变为阻塞状态。

是进程自身的一种主动行为，只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。`当进程进入阻塞状态，会释放CPU资源`



### 中断机制

> CPU正常运行期间，停止当前操作，执行其他特殊操作的行为就叫中断，负责跳转的指令就是`中断指令`。

- 内部中断：中断信号来源于CPU内部，如地址越界、除数为0；
- 外部中断：如IO完成、时钟中断（线程切换）、控制台中断（Ctrl+C）



#### 缺页中断

每当所要访问的内存页不在时，会产生一次`缺页中断`，此时操作系统会根据页表中的外存地址在外存中找到所缺的一页，将其调入内存。 

> 缺页中断是一种特殊的中断，它与一般的中断的区别是：
>
> 1. 在指令执行期间产生和处理中断信号
>
>    CPU通常在一条指令执行完后检查是否有中断请求，而缺页中断是在指令执行时间，发现所要访问的指令或数据不在内存时产生和处理的。
>
> 2. 一条指令在执行期间可能产生多次缺页中断
>
>    如一条读取数据的多字节指令，指令本身跨越两个页面，若指令后一部分所在页面和数据所在页面均不在内存，则该指令的执行至少产生两次缺页中断。



# 缓存IO

> 缓存 IO 又被称作标准 IO，大多数文件系统的默认 IO 操作都是缓存 IO

在 Linux 的缓存 IO 机制中，OS会将 IO 的数据缓存在文件系统的页缓存`page cache`(在内存中)中，也就是说：

> 数据会先被拷贝到`操作系统内核的缓冲区`中（disk -> memory）
>
> 然后才会从`操作系统内核的缓冲区`拷贝到`应用程序的地址空间`（用户态，虚拟地址对应的物理地址）

也就是需要 两次拷贝，这些数据拷贝操作所带来的 CPU 以及内存开销非常大，

所以很多应用程序会利用[零拷贝](#零拷贝)技术来避免这种多次拷贝操作的发生以提高性能。

> 补充：（个人理解是这样）
>
> page cache：页缓存，用来存储文件数据中的业务数据部分。重点是cache（缓存）
>
> buffer cache：针对磁盘块的缓存，存储任意一个扇区的数据。重点是buffer（缓冲），比如MySQL的插入缓冲，读写的时候 放到buffer中，然后由OS决定何时刷到磁盘。
>
> > 也就是说在没有文件系统的情况下（准确的表达是，无论有否文件系统的存在，设备驱动程序读取的数据（也即直接对磁盘进行操作的数据）都是缓存到buffer cache中的），直接对磁盘进行操作的数据会缓存到buffer cache中，例如，文件系统的元数据都会缓存到buffer cache中

# 内存管理

内存 本来存放的是正在运行的程序相关的数据，但是因为物理内存大小限制，只能把必须的、使用频繁的部分放入内存，内存满了就在内存中删除非必须的，加载必须的数据。

> RAM（**Random Access** Memory）：随机读写存储，主内存
>
> ROM（**Read Only** Memory）：只读内存，比如主板上用来存放BIOS程序。

首先，CPU是通过地址从`RAM/ROM/外设`中获取数据（程序、数据）以执行/运算；CPU能识别多大范围内的地址，取决于CPU的[寻址空间](#寻址空间)

如果执行的程序都放进内存，可能会导致内存不够用，所以使用了[虚拟内存](#虚拟内存)技术，**让程序看起来内存空间足够大并且可用地址是连续的**



## 交换空间 swap

如果物理内存不够用，则会通过swap将暂时不用的物理页换到硬盘上。

> 由物理内存和一部分磁盘空间组成
>
> 如果物理内存不够用了 ， 就通过LRU算法把最近使用频率最低的内存数据放到磁盘（交换空间）
>
> 如果这块内存需要被使用，再通过缺页异常，把磁盘中那部分数据刷回内存

Linux内核就是维护了一张表来记录每一页存储在物理内存还是交换空间(磁盘)。

### 页面置换算法

LRU、OPT、FIFO...算了 哥   `https://www.google.com/search?q=页面置换算法`



## 虚拟内存

又叫 `线性地址`、`虚拟地址空间`

OS使用`虚拟内存`抽象了物理内存的使用，上层（应用）调用的时候使用虚拟地址，让CPU转换成实际的物理地址再去访问内存。

> 最大的地址空间和实际系统有多少物理内存无关，也因此称为**虚拟**地址空间。Linux是通过**段页机制**来将虚拟地址和物理地址进行映射。
>

> **MMU**（内存管理单元）：CPU中用来管理虚拟存储器、物理存储器的控制线路，同时也**负责虚拟地址映射为物理地址**，以及提供硬件机制的内存访问授权
>
> > CPU通过**TLB**（Translation Lookaside Buffer，转换检测缓冲区）提升虚拟地址到物理地址的转换速度

绝大多数情况下，虚拟地址空间比实际系统可用的物理内存(RAM)大，内核和CPU必须考虑如何将实际可用的物理内存映射到虚拟地址空间。一个方法是通过页表(Page Table)将虚拟地址映射到物理地址。每个单元格是一个`page frame`。[内存映射mmap](#内存映射mmap)

<img alt="mmap" src="https://raw.githubusercontent.com/melopoz/pics/master/img/mmap.png" style="zoom:50%;" />

> 有可能 `虚拟内存中的某个地址并未使用` 或者 `对应的数据还没有被加载到内存`，或者`物理内存页被置换到了硬盘上`，这时候会导致[缺页中断](#缺页中断)，由中断处理程序将数据放到内存中之后再继续执行。

##### 多级页表

> 因为虚拟地址空间的绝大多数区域实际并没有使用，这些页实际并没有和page frame关联，引入多级页表(multilevel paging)能极大降低页表使用的内存，提高查询效率。https://www.zhihu.com/question/63375062



## 内存映射mmap

> Memory Mapping，底层的一个函数 `mmap()`

内存映射是将来自某个数据源的数据(也可以是某个设备的I/O端口等)转移到某个进程的虚拟内存空间。

> 任何**对内存的改动会自动转移到原数据源**，例如将某个文件的内容映射到内存中，只需要通过读该内存来获取文件的内容，通过将改动写到该内存来修改文件的内容，**内核确保任何改动都会自动体现到文件里**。
>
> 在内核中，**实现设备驱动**时，外设(外部设备)的输入和输出区域可以被映射到虚拟地址空间，读写这些空间会被系统重定向到设备，从而对设备进行操作，极大地简化了驱动的实现。

<img src="https://raw.githubusercontent.com/melopoz/pics/master/img/%E6%96%87%E4%BB%B6%E6%98%A0%E5%B0%84%E5%88%B0%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98.png" style="zoom:50%;" />

> 内核必须跟踪哪些物理页已经被分配了，哪些还是空闲的，避免两个进程使用RAM中的同一个区域。内存的分配和释放是非常频繁的任务，内核必须确保完成的速度尽量快，所以**内核只能分配整个page frame**，它将`把内存分为更小的部分的任务`交给了用户空间，用户空间的程序库可以将`从内核收到的page frame`分成更小的区域后分配给进程。

mmap() 函数：

> 在驱动程序的`mmap()`系统调用中，使用`remap_page_range()`将**该块ROM映射到用户虚拟空间**。这样内核空间和用户空间都能访问这段被映射后的虚拟地址。
>
> `remap_page_range()`函数的功能是`构造用于映射一段物理地址的新页表`，**实现了内核空间与用户空间的映射**。
>
> 在内核**驱动程序的初始化阶段**，通过`ioremap()`**将物理地址映射到内核虚拟空间**；



#### Java中的应用：

> MappedByteBuffer使用mmap的文件映射，在full gc时才会进行释放。
>
> 当close时，需要手动清除内存映射文件，可以反射调用sun.misc.Cleaner方法



## 零拷贝

> **零拷贝**（**Zero-copy**）技术是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域。这种技术通常用于**通过网络传输文件时节省CPU周期和内存带宽**。

并不是真的**零**拷贝，只是减少内核态和用户态的切换次数 和 数据拷贝次数。



### 实现方式

#### mmap+read/write

直接调用系统的`read()`和 内存映射文件并读取`mmap()`的区别：

- 直接调用`read()`，需要内核来调用，步骤：1）从硬盘拷贝到内核缓冲区(内核空间)，2）从内核空间拷贝到用户空间；（即需要用到[缓存IO](#缓存IO)）

  > 首先将文件内容从硬盘拷贝到内核空间的一个缓冲区，然后再将这些数据拷贝到用户空间，这个过程实际上发生了**两次**数据拷贝；

- 普通文件/设备通过`mmap()`被映射到进程地址空间（虚拟内存）后，进程可以像访问普通内存一样对文件进行访问，只要在分配的地址范围内进行读取或者写入即可，不必再调用 read / write。具体步骤如下：

  > 调用`mmap()`返回一个虚拟空间的指针`ptr`，指向进程逻辑空间（虚拟内存）中的一个地址，进程再操作这个数据的时候，相当于读写内存：
  >
  > > 由MMU将逻辑地址转换为物理地址，如果没能找到对应的物理地址，会引发缺页中断，CPU的中断处理程序会在swap中寻找对应的数据页，如果再找不到，就会使用mmap()建立的映射关系，将硬盘文件的数据读取到物理内存中。

`mmap()`虽然也是系统调用，但它没有直接调用`read()`进行数据拷贝，**真正的数据拷贝是在缺页中断处理时进行的**，由于mmap()将文件直接映射到用户空间，所以**中断处理程序**根据这个映射关系，**直接将文件从硬盘拷贝到用户空间**，**只进行**了**一次数据拷贝** 。因此，内存映射的效率要比read/write效率高。

> 零拷贝是有一次拷贝过程，但是该拷贝操作如果存在，也不是直接属于此进程，而是由中断处理程序处理一个缺页异常，这样看来进程其实是没有进行拷贝操作的。估计因此才叫的**零**拷贝，该拷贝操作是无法避免的，必须得读取文件到内存中。

所以mmap()对**处理大文件**提高了效率。



#### sendfile

Linux2.1的sendfile()如下，**三次拷贝**（两次DMA拷贝，一次CPU拷贝）

<img alr="Linux2.1 sendfile()" src="https://raw.githubusercontent.com/melopoz/pics/master/img/linux2.1-sendfile.png" style="zoom:50%;" />

> 图片来自https://mp.weixin.qq.com/s/dt0h2UhaoRECvjpeMZMsUA



#### Linux2.4-sendfile：DMA Scatter/Gatter

> DMA（Direct Memory Access，直接存储器访问）
>
> 引入新的硬件支持 -- Scatter/Gatter（分散/聚集）（两次DMA拷贝，没有CPU拷贝）

<img alr="Linux2.4 sendfile()" src="https://raw.githubusercontent.com/melopoz/pics/master/img/Linux-2.4-sendfile.png" style="zoom:50%;" />

### 应用

RocketMQ

> mmap+read/write

Kafka

> 持久化使用 mmap+write
>
> 发送数据使用 sendfile

java.nio.channels.FileChannel#transferFrom()/transferTo()

> 使用 sendfile
>
> ```java
> File file = new File("demo.zip");
> RandomAccessFile raf = new RandomAccessFile(file, "rw");
> FileChannel fileChannel = raf.getChannel();
> SocketChannel socketChannel = SocketChannel.open(new InetSocketAddress("", 1234));
> // 直接使用了transferTo()进行通道间的数据传输
> fileChannel.transferTo(0, fileChannel.size(), socketChannel);
> ```
>
> <img src="https://raw.githubusercontent.com/melopoz/pics/master/img/java-nio-sendfile.png" style="zoom:50%;" />





## mmap 和 sendfile总结

1、都是Linux内核提供、实现零拷贝的API；
2、sendfile 是将读到内核空间的数据，转到socket buffer，进行网络发送；
3、mmap将磁盘文件映射到内存，支持读和写，对内存的操作会反映在磁盘文件上。





# IO设备管理

操作系统向I/O设备发送命令，比如访问设备、读写数据、读写设备配置，还要捕捉中断，处理设备的错误，并提供设备功能的抽象。



# 文件管理

将硬盘的扇区组织成文件系统，实现文件读写操作

比如 win使用NTFS、FAT32、FAT16；macOS使用APFS（apple file system）；linux使用EXT2 等等

> 一个操作系统可以支持多种底层不同的文件系统（比如NTFS, FAT, ext3, ext4），为了给内核和用户进程提供统一的文件系统视图，Linux在用户进程和底层文件系统之间加入了一个抽象层，即**虚拟文件系统(Virtual File System, VFS)**
>
> 进程所有的文件操作都通过VFS，由VFS来适配各种底层不同的文件系统，完成实际的文件操作。



Linux 内核 看去吧...

https://www.kernel.org/doc/html/v4.14/index.html