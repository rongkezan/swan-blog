---
title: IO模型
date: {{ date }}
categories:
- Java
---

## 基本概念说明

**用户空间和内核空间**

操作系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核，保证内核的安全，操心系统将虚拟空间划分为两部分：内核空间，用户空间。

**进程切换**

为了控制进程的执行，内核必须有能力挂起正在CPU上运行的进程，并恢复以前挂起的某个进程的执行。这种行为被称为进程切换。因此可以说，任何进程都是在操作系统内核的支持下运行的，是与内核紧密相关的。进程切换很消耗资源。

**进程阻塞**

正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。可见，进程的阻塞是进程自身的一种主动行为，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用CPU资源的。

**文件描述符**

是一个用于表述指向文件的引用的抽象化概念。它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。一个 Socket 连接实际上就是一个文件描述符。

查看一个进程的文件描述符

```shell
cd /proc/{pid}/fd
```

**缓存IO**

缓存 I/O 又被称作标准 I/O，大多数文件系统的默认 I/O 操作都是缓存 I/O。在 Linux 的缓存 I/O 机制中，操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。

**说明**

客户端连接先到达内核，read命令可以读文件描述符

## BIO

> 阻塞

同步阻塞，等待 read 命令时，线程一直处于阻塞状态。所以每次连接都要抛出一个新的线程。

**流程**

1. 客户端连接进入时阻塞，等待用户线程响应
2. 用户线程响应后返回给客户端

**弊端**

用户需要等待read将socket中的数据读取到buffer后，才继续处理接收的数据。整个IO请求的过程中，用户线程是被阻塞的，这导致用户在发起IO请求时，不能做任何事情，对CPU的资源利用率不够。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202222236861.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## NIO

> 轮询

同步非阻塞，轮询文件描述符调用 read，一个线程可以对应多个客户端连接。

**流程**

1. 客户端连接进入时不阻塞，而是把每个文件描述符放入List
2. 对 List 里的文件描述符进行遍历，如果有数据则直接返回

**弊端**

用户需要不断地调用read，尝试读取socket中的数据，直到读取成功后，才继续处理接收的数据。整个IO请求的过程中，虽然用户线程每次发起IO请求后可以立即返回，但是为了等到数据，仍需要不断地轮询、重复请求，消耗了大量的CPU的资源。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021020222244755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## NIO 多路复用

> 内核增加了 select 系统调用

**流程**

1. 用户空间线程调用 select
2. 内核轮询所有文件描述符并标记 ready 的文件描述符，之后将所有文件描述符返回给用户线程
3. 用户线程遍历所有文件描述符挑出 ready 的去调用 read

**缺陷**：用户态和内核态传递数据的成本较高

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202224819803.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## epoll

> 用户空间和内核空间之间多出了一块虚拟的共享空间，共享空间是通过内核的系统调用 mmap 实现的。
>
> 共享空间的增删改操作由内核空间完成，但查询是用户空间和内核空间都可以查。

**流程**

1. 每次客户端连接进来用户空间线程就将其文件描述符放入共享空间的红黑树中
2. 此时用户空间调用 wait 等待事件
3. 当内核准备好了数据，就将红黑树中已经 ready 的文件描述符放入链表中
4. 此时用户空间 wait 释放，取链表中的文件描述符去调用 read

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202232431428.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

## 零拷贝

> 零拷贝是通过系统调用 sendfile 实现的

如果需要拷贝 file.txt 文件，用户空间线程调用内核的 read ，再去调用 write 返回到网卡。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202230912874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

 如果通过零拷贝，就可以直接调用内核的 sendfile，可以不跟用户空间产生IO

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210202230508156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)