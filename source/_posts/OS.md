---
title: OS
tags: 
  - 内存管理
categories:
  - [OS]
---



# 操作系统四大特征：

### 并发

两个事件在**同一段时间内**发生，即宏观上同时运行；并行是两个事件在统一时刻发生，多(核)CPU可支持并行

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

相关：虚拟内存，内存映射...

### 异步

因为任务多，资源少的情况，所以进程通常都是走走停停的方式进行。



# 进程管理

> 进程：程序的执行过程，是暂时的。 （也可以说包含正在运行的程序实体和其占据的所有系统资源, CPU,内存,网络）

多任务系统中用于管理CPU的时间片分配

进程又可以有多个线程，需要操作系统来执行上下文切换，让多个进程/线程使用同一个CPU资源



# 内存管理

内存本来是存放正在运行的程序，但是因为内存大小等原因，只能把必须的、使用频繁的部分放入内存，内存满了就在内存中删除非必须的，加载必须的。

> RAM（Random Access Memory）：随机读写存储，主内存
>
> ROM（Read Only Memory）：只读内存，比如主板上用来存放BIOS程序。

首先，CPU是通过地址从`RAM/ROM/外设`中获取数据（程序、数据）以执行/运算；CPU能识别多大范围内的地址，取决于CPU的

##### 寻址空间

32位CPU， 理论上 寻址空间 2^32, 4GB

64位CPU， 理论上 寻址空间 2^64, 16EB = 160亿GB（只是理论上支持，但是64位PC也只能到32GB）

但实际寻址空间还取决于总线宽度，而且一般没有那么大内存，64位机器常用的也就`2^42~2^47`（`4T~`）吧

如果执行的程序都放进内存，可能会导致内存不够用，

> 而且程序总是趋向于使用最近使用过的数据和指令

所以使用了**虚拟内存**技术，**让程序看起来内存空间非常大并且可用地址是连续的**

##### 虚拟内存（线性地址）（虚拟地址空间）

> 最大的地址空间和实际系统有多少物理内存无关，也因此称为虚拟地址空间，内核把物理空间再抽象出一层，供应用程序使用
>
> Linux是通过段页机制来讲虚拟地址和物理地址进行映射。

虚拟内存抽象了物理内存的使用，上层（应用）调用的时候使用虚拟地址，让CPU转换成实际的物理地址再去访问内存。

> MMU（内存管理单元）：CPU中用来管理虚拟存储器、物理存储器的控制线路，同时也**负责虚拟地址映射为物理地址**，以及提供硬件机制的内存访问授权
>
> > CPU通过TLB（Translation Lookaside Buffer，转换检测缓冲区）提升虚拟地址到物理地址的转换速度

### 补充

Linux会给程序分配4G（2^32）的虚拟地址空间，这4G又分为3G用户空间和1G内核空间（只能在内核态访问）（Windows为2:2），

> 使用命令`cat /proc/cpuinfo` 可以查看到 `address sizes   : 36 bits physical, 48 bits virtual` （在win10的Ubuntu子系统中）
>
> 2^36^物理地址空间，2^48^虚拟地址空间



# IO设备管理

对IO设备（磁盘）的高层次抽象。

操作系统向I/O设备发送命令，比如访问设备、读写数据、读写设备配置，还要捕捉中断，处理设备的错误，并提供设备功能的抽象。



# 文件管理

将硬盘的扇区组织成文件系统，实现文件读写操作

比如 win使用NTFS、FAT32、FAT16；macOS使用APFS（apple file system）；linux使用EXT2 等等



# 内核态 / 用户态

内核态：操作系统管理程序运行时的状态。运行在内核态的程序可以访问计算机的任何资源，比如协调CPU，分配内存资源；

用户态：应用程序运行时的状态。用户态的程序只能访问当前CPU上执行的程序所在的地址空间。

区分内核态和用户态是为了**安全，控制权限，保护操作系统程序**。

OS在内核态会在合适的情况（比如执行完某IO命令）把CPU使用权还给应用程序，即转换回用户态。但是从用户态转换到内核态，只能通过**中断机制**。

中断包括很多，比如 程序需要请求操作系统帮忙比如IO、IO操作完成、程序异常（需要异常处理程序工作）。。

中断可以分为

- 内中断：中断信号来源于CPU内部，如地址越界、除数为0；
- 外中断：如IO完成、时钟中断（线程切换）、控制台中断（Ctrl+C）



# 内存映射 mmap（Memory Map）

> 底层的一个函数 mmap()

用mmap映射一个设备，意味着**使用户空间的一段地址关联到设备内存上**，这使得只要程序在分配的地址范围内进行读取或者写入，实际上就是对设备的访问。这种数据传输是直接的，不需要用到内核空间作为数据转移的中间站。

remap_page_range()函数的功能是构造用于映射一段物理地址的新页表，**实现了内核空间与用户空间的映射**。在内核**驱动程序的初始化阶段**，通过**ioremap()将物理地址映射到内核虚拟空间**；在驱动程序的**mmap系统调用中**，使用remap_page_range()将该**块ROM映射到用户虚拟空间**。这样内核空间和用户空间都能访问这段被映射后的虚拟地址。

- read()是系统调用，步骤1）从硬盘拷贝到内核缓冲区(内核空间)，2）从内核空间拷贝到用户空间；
- 有了虚拟内存空间，则**mmap()**返回一个虚拟空间的指针`ptr`，指向进程逻辑空间中的一个地址，若要操作其中数据，必须由MMU将逻辑地址转换为物理地址`（如果没能找到对应的物理地址，会引发缺页中断，响应中断的函数会在swap中寻找对应的数据页，如果找不到，就会使用mmap()建立映射关系，将硬盘文件读取到物理内存中）`，



### 零拷贝 △△△△△△△△

从代码层面上看，从硬盘上将文件读入内存，都要经过文件系统进行数据拷贝，并且数据拷贝操作是由文件系统和硬件驱动实现的，理论上来说，拷贝数据的效率是一样的。但

> 是通过内存映射的方法访问硬盘上的文件，效率要比read和write系统调用高，这是为什么呢？

原因是**read()**是系统调用，其中进行了数据拷贝，它首先将文件内容从硬盘拷贝到内核空间的一个缓冲区，然后再将这些数据拷贝到用户空间，在这个过程中，实际上完成了 两次数据拷贝 ；

而**mmap()**也是系统调用，如前所述，mmap()中没有进行数据拷贝，真正的数据拷贝是在缺页中断处理时进行的，由于mmap()将文件直接映射到用户空间，所以中断处理函数根据这个映射关系，直接将文件从硬盘拷贝到用户空间，只进行了 一次数据拷贝 。因此，内存映射的效率要比read/write效率高。这也就是零拷贝了。

> 零拷贝是有一次拷贝过程，但是该拷贝操作如果存在，操作不是直接属于这次进程的范围内，而是在处理一个缺页异常，可能因此才叫零拷贝。。。该拷贝操作是正常的、必须的读取文件到内存中的操作。

所以mmap()对处理大文件提高了效率。



### 交换空间 swap

如果物理内存不够用，则会通过swap将暂时不用的物理页换到硬盘上。

> 由物理内存和一部分磁盘空间组成
>
> 如果物理内存不够用了 ， 就通过LRU算法把最近使用频率最低的内存数据放到磁盘（交换空间）
>
> 如果这块内存需要被使用，再通过缺页异常，把磁盘中那部分数据刷回内存

Linux内核就是维护了一张表来记录每一页存储在物理内存还是交换空间(磁盘)。







### 参考

https://zhuanlan.zhihu.com/p/73453720

https://xie.infoq.cn/article/19ac8a31e0f4c3869846b5530

https://zhuanlan.zhihu.com/p/67059173

https://zhuanlan.zhihu.com/p/116896185

https://blog.csdn.net/aspirinvagrant/article/details/18991263

https://www.cnblogs.com/still-smile/p/12155181.html 内存映射