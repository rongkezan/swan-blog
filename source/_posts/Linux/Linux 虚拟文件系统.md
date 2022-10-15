---
title: Linux 虚拟文件系统
date: {{ date }}
categories:
- Linux
---

## 观察文件分区

命令 `df -h`

```sh
[root@master ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        3.7G     0  3.7G   0% /dev
tmpfs           3.7G     0  3.7G   0% /dev/shm
tmpfs           3.7G  2.5M  3.7G   1% /run
tmpfs           3.7G     0  3.7G   0% /sys/fs/cgroup
/dev/vda1        40G   37G  1.1G  98% /
tmpfs           3.7G   12K  3.7G   1% /var/lib/kubelet/pods/026a6dff-d5d0-451b-a5e9-ebd16a0952ea/volumes/kubernetes.io~secret/kube-proxy-token-297hw
tmpfs           3.7G   12K  3.7G   1% /var/lib/kubelet/pods/cc5c4cb2-7677-41bd-9e59-c7ca6e6e57bd/volumes/kubernetes.io~secret/calico-node-token-5sq4k
tmpfs           756M     0  756M   0% /run/user/0
```

## 链接

### 硬链接

> 指向同一个物理文件

```sh
[root@node1 demo]# echo Hello > 1.txt
[root@node1 demo]# ln 1.txt 2.txt
```

stat 两个文件，发现这两个文件 `Inode` 一致

```sh
[root@node1 demo]# stat 1.txt 
  File: ‘1.txt’
  Size: 6         	Blocks: 8          IO Block: 4096   regular file
Device: fd01h/64769d	Inode: 1179691     Links: 2
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2022-10-14 15:12:36.007592557 +0800
Modify: 2022-10-14 15:12:31.478424680 +0800
Change: 2022-10-14 15:12:45.275936096 +0800
```

```sh
[root@node1 demo]# stat 2.txt
  File: ‘2.txt’
  Size: 6         	Blocks: 8          IO Block: 4096   regular file
Device: fd01h/64769d	Inode: 1179691     Links: 2
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2022-10-14 15:12:36.007592557 +0800
Modify: 2022-10-14 15:12:31.478424680 +0800
Change: 2022-10-14 15:12:45.275936096 +0800
```

如果将某一个文件删掉不会有影响，只是指向该文件的指针会少1个

```sh
[root@node1 demo]# rm -rf 1.txt 
[root@node1 demo]# stat 2.txt
  File: ‘2.txt’
  Size: 6         	Blocks: 8          IO Block: 4096   regular file
Device: fd01h/64769d	Inode: 1179691     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2022-10-14 15:12:36.007592557 +0800
Modify: 2022-10-14 15:12:31.478424680 +0800
Change: 2022-10-14 18:48:16.485278741 +0800

```

### 软链接

> 相当于创建快捷方式

```sh
[root@node1 demo]# ln -s 2.txt 2s.txt
[root@node1 demo]# ll
total 4
lrwxrwxrwx 1 root root 5 Oct 14 18:51 2s.txt -> 2.txt
-rw-r--r-- 1 root root 6 Oct 14 15:12 2.txt
```

如果把源文件删了，那么软链接就会丢失，会报红

```sh
[root@node1 demo]# rm -rf 2.txt 
[root@node1 demo]# ll
total 0
lrwxrwxrwx 1 root root 5 Oct 14 18:51 2s.txt -> 2.txt
```

## 文件描述符

> 文件描述符 就是 描述文件的具体信息，指针，偏移
>
> 内核为每一个进程各自维护了一套数据，数据中包含了进程的FD，即文件描述符

查看文件描述符 `FD`

$$ ：当前bash的进程ID号

```sh
[root@node1 fd]# cd /proc/$$/fd
[root@node1 fd]# lsof -op $$
COMMAND  PID USER   FD   TYPE DEVICE OFFSET     NODE NAME
bash    3896 root  cwd    DIR    0,3        52168349 /proc/3896/fd
bash    3896 root  rtd    DIR  253,1               2 /
bash    3896 root  txt    REG  253,1          657149 /usr/bin/bash
bash    3896 root  mem    REG  253,1          665045 /usr/lib/locale/locale-archive
bash    3896 root  mem    REG  253,1          657092 /usr/lib64/libnss_files-2.17.so
bash    3896 root  mem    REG  253,1          657074 /usr/lib64/libc-2.17.so
bash    3896 root  mem    REG  253,1          657080 /usr/lib64/libdl-2.17.so
bash    3896 root  mem    REG  253,1          657148 /usr/lib64/libtinfo.so.5.9
bash    3896 root  mem    REG  253,1          657067 /usr/lib64/ld-2.17.so
bash    3896 root  mem    REG  253,1          786746 /usr/lib64/gconv/gconv-modules.cache
bash    3896 root    0u   CHR  136,0    0t0        3 /dev/pts/0
bash    3896 root    1u   CHR  136,0    0t0        3 /dev/pts/0
bash    3896 root    2u   CHR  136,0    0t0        3 /dev/pts/0
bash    3896 root  255u   CHR  136,0    0t0        3 /dev/pts/0
```

