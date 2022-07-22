---
title: Linux 命令
date: {{ date }}
categories:
- Linux
---

## 常用命令

### ls

```shell
### ls
ls -a # 列出所有，包含隐藏
ls -l # 列出详细信息
ls -h # 配合-l显示文件大小

# 通配符
* 可以代表任意多个的字符
? 代表任意一个字符
[] 表示可以匹配字符组的任一个 
例如 [abc] [a-d]
示例：ls [abc]a.txt
```

### cd

```shell
cd 		# 切换到当前用户主目录
cd ~	# 切换到当前用户主目录
cd .	# 保持在当前目录
cd ..	# 返回上一级
cd -	# 在最近两次目录之间切换
```

### mkdir

```shell
mkdir dir				# 创建目录
mkdir -p dir1/dir2		# 递归创建子目录
```

### rm cp mv

```shell
rm -r # 递归删除文件夹下的所有内容
rm -f # 强制删除，无需提示
cp -r # 复制目录文件，则递归复制子文件
mv		# 移动文件
```

### grep

```shell
# 衔接：前一个命令的输出作为后一个命令的输入
# 管道会触发创建子进程
# 常用管道命令: more grep
ls -a | grep [keyword]

# 详细参数
-n 显示匹配行及行号
-v 显示不包含匹配文本的所有行(求反)
-i 忽略大小写
^a	行首，搜寻以a开头的行
ke$	行尾，搜寻以ke结尾的行
示例：grep -nvi “hello world” 1.txt
```

### netstat

```shell
# 列出所有端口(包括监听和未监听的)
netstat -a
# 列出所有TCP端口
netstat -at
# 列出所有的UDP端口
netstat -au
# 列出所有监听状态下的端口
netstat -l
# 列出所有监听状态下的TCP端口
netstat -lt
# 列出所有监听状态下的UDP端口
netstat -lu
# 列出所有监听状态下的UNIX端口
netstat -lx
# 列出指定端口的进程
netstat -an | grep ':80'
```

### tar

```shell
独立命令(只能使用其中一个):
-c: 建立压缩档案
-x: 解压
-t: 查看内容
-r: 向压缩归档文件末尾追加文件
-u: 更新原压缩包中的文件

可选命令:
-z: 有gzip属性的
-j: 有bz2属性的
-Z: 有compress属性的
-v: 显示所有过程
-O: 将文件解开到标准输出

必需命令:
-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名。

tar -xvf file.tar 			# 解压file.tar并展示过程
tar -xvzf file.tar.gz 	# 解压file.tar.gz并展示过程
tar -xjvf file.tar.bz2	# 解压file.tar.bz2并展示过程
tar -xZvf file.tar.Z		# 解压file.tar.Z并展示过程

tar -cvf all.tar * 					# 将目录里所有文件打包成all.tar
tar -czf jpg.tar.gz *.jpg   # 将目录里所有jpg文件打包成jpg.tar后，并且将其用gzip压缩，生成一个名为jpg.tar.gz的压缩包
tar -cjf jpg.tar.bz2 *.jpg 	# 将目录里所有jpg文件打包成jpg.tar后，并且将其用bzip2压缩，生成一个名为jpg.tar.bz2的压缩包
tar -cZf jpg.tar.Z *.jpg    # 将目录里所有jpg文件打包成jpg.tar后，并且将其用compress压缩，生成一个jpg.tar.Z的压缩包

### rar,zip
unrar e file.rar
unzip file.zip

rar a jpg.rar *.jpg # rar格式的压缩
zip jpg.zip *.jpg 	# zip格式的压缩
```

### tail

**命令格式：**

```shell
tail [param] [doc]
```

**参数：**

- -f 循环读取
- -q 不显示处理信息
- -v 显示详细的处理信息
- -c<数目> 显示的字节数
- -n<行数> 显示文件的尾部 n 行内容
- --pid=PID 与-f合用,表示在进程ID,PID死掉之后结束
- -q, --quiet, --silent 从不输出给出文件名的首部
- -s, --sleep-interval=S 与-f合用,表示在每次反复的间隔休眠S秒

### echo

```shell
echo
echo 1 > 1.txt	# 将1输出到1.txt中
echo 1 >> 1.txt	# 将11追加到1.txt中
```

### cat

```shell
cat			# 查看文件内容
cat -b	# 对非空输出行号
cat -n	# 对所有行输出行号
more		# 分屏查看文件内容
```

### find

```shell
find [路径] -name ”*.py“ # 查找指定路径下扩展名是.py的文件，包括子目录
```

## 系统相关命令

```shell
# 查看系统时间
date
# 查看日历, -y 查看一年的日历
cal
# 显示磁盘剩余空间 -h:以人性化的方式显示
df -h
# 显示目录下的文件大小
du -h [目录名]
# 查看进程状态 a:显示所有进程 u:显示进程详细状态 x:显示没有控制终端的进程
ps aux
# 动态显示运行中的程序并排序
top
# 终止指定代号的进程，-9表示强行终止
kill [pid]
kill -9 [pid]
# 根据进程名查看运行进程
ps -ef | grep [进程名]
```

软链接

```shell
# 类似于windows的快捷方式，源文件要使用绝对路径，方便移动链接文件后还能使用
ln -s [source] [target]
```

硬链接

```shell
# 硬连接的源文件被删除，目标文件还是可以打开
ln [source] [target]
```

在 Linux 中，文件名和文件数据是分开存储的

在 Linux 中，只有文件的硬链接数等于0才会被删除

使用 ls -l 可以查看一个文件的硬链接数量

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210406230655759.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)



## Vi命令

1. 命令行模式

  	按「ESC」键

2. 插入模式

```
「i」：进入插入模式，从光标当前位置开始输入文件。
「a」：进入插入模式，从目前光标所在位置的下一个位置开始输入文字。
「o」：进入插入模式，插入新的一行，从行首开始输入文字。
```

3. 移动光标

```
「ctrl」+「b」：屏幕往"后"移动一页。
「ctrl」+「f」：屏幕往"前"移动一页。
「ctrl」+「u」：屏幕往"后"移动半页。
「ctrl」+「d」：屏幕往"前"移动半页。
「0」：移到文章的开头。
「G」：移动到文章的最后。
「$」：移动到光标所在行的"行尾"。
「^」：移动到光标所在行的"行首"
「w」：光标跳到下个字的开头
「e」：光标跳到下个字的字尾
「b」：光标回到上个字的开头
```

4. 删除文字

```
「x」：删除光标所在位置的"后面"1个字符。
「#x」：删除光标所在位置的"后面"#个字符。
「X」：删除光标所在位置的"前面"1个字符。
「#X」：删除光标所在位置的"前面"#个字符。
「dd」：删除光标所在行。
「#dd」：从光标所在行开始删除#行
```

5. 复制文字

```
「yw」：复制光标到行尾的字符。
「#yw」：复制#个字符。
「yy」：复制光标所在行。
「#yy」：复制包含光标所在行及后面的#行文字。
「p」：粘贴复制的文字到光标所在位置。
```

6. 替换文字

```
「r」：替换光标所在处的字符。
「R」：替换光标所到之处的字符，直到按下「ESC」键为止。
```

7. 返回上一次操作

```
按「u」键
```

8. 更改

```
「cw」：更改光标后到行尾的所有字符
「c#w」：更改光标后的#个字符
```

9. 跳至指定的行

```
「ctrl」+「g」：列出光标所在行的行号。
「#G」：移动光标至文章的第#行行首。
```

## 附录

从指定url下载资源

```shell
yum install -y wget
wget http://download.redis.io/releases/redis-5.0.5.tar.gz
```

Linux开机启动项目录

```powershell
/etc/init.d
```

Linux环境变量

```shell
# 修改环境变量
vi /etc/profile
# 重启环境变量
source /etc/profile
```

nc命令

```powershell
# 安装包
yum install -y nmap-ncat
# 监听TCP/UDP端口
nc localhost 6379
```

后台启动程序并输出日志

```shell
nohup java -jar myproject.jar >myproject.log 2>&1 &

命令详解：
nohup：不挂断地运行命令，退出帐户之后继续运行相应的进程。
>myproject.log：控制台输出到myproject.log日志文件中。
2>&1：标准错误重定向到标准输出。
最后的&：让改作业在后台运行。
```

父子进程

```
/bin/bash 开启一个子进程
父子进程间数据是隔离的，但是父进程可以使用 export 使子进程看到数据
export 的环境变量，子进程修改不会影响父进程，父进程修改也不会影响子进程
```

