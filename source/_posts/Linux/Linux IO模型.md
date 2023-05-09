---
title: Linux IO模型
date: {{ date }}
categories:
- Linux
---

## 名词解释

### 用户空间和内核空间

操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核，保证内核的安全，操心系统将虚拟空间划分为两部分：内核空间，用户空间。

### 进程切换

为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。进程切换很消耗资源。

### 进程阻塞

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用CPU资源的。

### 文件描述符

是一个用于表述指向文件的引用的抽象化概念。它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。一个 Socket 连接实际上就是一个文件描述符。

查看一个进程的文件描述符

```shell
cd /proc/{pid}/fd
```

### 缓存IO

缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

**说明**：客户端连接先到达内核，read命令可以读文件描述符

### 零拷贝

零拷贝是通过系统调用 `sendfile` 实现的

如果需要拷贝 file.txt 文件，用户空间线程调用内核的 read ，再去调用 write 返回到网卡。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202230912874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

 如果通过零拷贝，就可以直接调用内核的 sendfile，可以不跟用户空间产生IO

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202230508156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 阻塞IO（Blocking I/O）

> 我去饭馆吃饭，点完菜，只能等着厨师做好，服务员端上来，我才能愉快干饭。这段时间，我就只能坐在座位上干等。

当发起一个IO操作时，比如读取数据，系统会调用read()函数。如果请求的数据没有准备好，此时进程会被挂起（blocked），进入等待状态。直到数据准备好，而且复制到应用进程的缓冲区，这时候才会返回。

**流程**

1. 客户端连接进入时阻塞，等待用户线程响应
2. 用户线程响应后返回给客户端

**缺点**：从调用到返回，整个时间段都是阻塞的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PMZOEonJxWeAYtnuB7ibZeUF12WG7DsvhicuM5hqswJdvYY5Sz6FTYJiaiaxu1OcNQodIribgQlj6xgW6r5oDp84H5A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 非阻塞IO（Non-Blocking I/O）

> 就像是我奢侈一把，想吃个西餐，于是就去了肯德基，点完餐，我就可以坐着刷刷手机。当然，我还需要时不时地看看我的餐是不是已经备好，餐备好了，就去取一下。

所谓非阻塞IO，是在调用IO操作时，如果缓冲区没有数据的话，直接返回一个错误码。应用进程需要不断轮询，来检查数据是否准备好。数据准备好了，就返回数据。

**流程**

1. 客户端连接进入时不阻塞，而是把每个文件描述符放入List
2. 对 List 里的文件描述符进行遍历，如果有数据则直接返回

**缺点**：整个IO请求的过程中，虽然应用每次发起IO请求后可以立即返回，但是为了等到数据，仍需要不断地轮询、重复请求，消耗了大量的CPU的资源。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PMZOEonJxWeAYtnuB7ibZeUF12WG7Dsvh7q2urnumVCHJL7OgiaAHs9bv99h1n3NMr0kNxhiaC6kGOtmPbZLdDYsw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

## 多路复用IO（I/O Multiplexing）

> 就像是我想吃顿好的，于是选择去吃自助餐，自助餐有很多餐区，我先看看哪个餐区有我想吃的菜，然后端着盘子去取就行了，一个人就可以取多个菜，肉、蔬菜、水果，什么都能吃一点，而且不用怎么等。

多路指的是多个数据通道，复用指的是一个进程可以同时监控多个文件描述符（比如socket），当某个文件描述符状态发生变化（比如变得可读或可写），多路复用的函数将返回变化的文件描述符。

这样，在数据传输过程中，同一个进程中不同的任务都能被处理。特点是在数据传输过程中，进程能够同时处理多个任务，提高了程序的效率。

select、poll、epoll 等都是 I/O 多路复用的具体实现。

**流程**

1. 用户空间线程调用 select
2. 内核轮询所有文件描述符并标记 ready 的文件描述符，之后将所有文件描述符返回给用户线程
3. 用户线程遍历所有文件描述符挑出 ready 的去调用 read

**缺陷**：用户态和内核态传递数据的成本较高

![图片](https://mmbiz.qpic.cn/mmbiz_png/PMZOEonJxWeAYtnuB7ibZeUF12WG7DsvhsbaF9RHMsCKQxKX6DedO7aSaWYBtKG90DlY17uv2VmdKoYy4n2KpoA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### epoll

> 用户空间和内核空间之间多出了一块虚拟的共享空间，共享空间是通过内核的系统调用 mmap 实现的。
>
> 共享空间的增删改操作由内核空间完成，但查询是用户空间和内核空间都可以查。

**流程**

1. 每次客户端连接进来用户空间线程就将其文件描述符放入共享空间的红黑树中
2. 此时用户空间调用 wait 等待事件
3. 当内核准备好了数据，就将红黑树中已经 ready 的文件描述符放入链表中
4. 此时用户空间 wait 释放，取链表中的文件描述符去调用 read

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202232431428.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 信号驱动式IO （Signal-Driven I/O）

> 就像是我去吃饭，外带，跟服务员打声招呼，餐好了通知我，这时候我就可以去干其它事情，餐备好之后，服务员通知我，我取餐就行了。

信号驱动式IO利用信号机制来进行数据传输。

进程首先告诉内核，当数据准备好时，请发送一个SIGIO信号。进程继续执行其他任务，等到收到信号后，再开始进行数据传输。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PMZOEonJxWeAYtnuB7ibZeUF12WG7DsvhcRYyNYxlWpVON0e75F26t12vibLaGOHKjbXSSKpFjaUHjUpERvKMGibg/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)

## 异步IO

> 就像是准备吃饭了，我自己懒得动，直接在某团上点个餐，点完之后爱干啥干啥，等着快递小哥给我送到就行了。

异步IO是指当发起一个IO操作后，系统会立即返回。异步IO操作在后台进行数据传输，数据传输完成后，系统将通知进程。这样，在整个数据传输的过程中，进程都可以执行其他任务，不需要等待。

![图片](https://mmbiz.qpic.cn/mmbiz_png/PMZOEonJxWeAYtnuB7ibZeUF12WG7Dsvh5QwFxXr7tYoCtcibgA6NX0ic8kzD6MvOPAEXZf42zr8ib388n4e4V5LcA/640?wx_fmt=png&tp=wxpic&wxfrom=5&wx_lazy=1&wx_co=1)