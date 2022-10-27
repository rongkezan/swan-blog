---
title: Linux 性能监控
date: {{ date }}
categories:
- Linux
---

## 查看内存运行情况

```sh
free -h
              total        used        free      shared  buff/cache   available
Mem:           7.4G        444M        3.3G        936K        3.7G        6.6G
Swap:            0B          0B          0B
```

信息释义：

- Mem：物理内存
- Swap：交换分区，存放虚拟内存，当内存不够时，把一部分硬盘虚拟成内存使用
- total：内存总数
- used：已经使用的内存
- free：空闲内存
- shared：多个进程共享内存
- buff：I/O缓存，内存与硬盘的缓冲，IO设备的读写缓冲区
- cache：高速缓存，内存与CPU的缓冲
- avaliable：剩余可用内存数

## 查看CPU使用情况

```sh
top
```

```sh
top - 17:49:22 up 64 days, 30 min,  1 user,  load average: 0.00, 0.01, 0.05
Tasks: 108 total,   1 running, 107 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.8 us,  0.6 sy,  0.0 ni, 98.7 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  7732968 total,  3418912 free,   455204 used,  3858852 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6971960 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                      
29560 root      10 -10  201976  19536   8688 S   1.7  0.3   2441:26 AliYunDun                               
```

- top - 17:49:22 ：当前时间
- up 64 days，30min ：开机了多少时间
- 1 user ：当前在线用户
- load average：当前的系统负载情况，分别是1min、5min、15min