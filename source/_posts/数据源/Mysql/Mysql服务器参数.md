---
title: Mysql服务器参数
date: {{ date }}
categories:
- 数据源
- Mysql
---

## general

```shell
# 数据文件存放的目录
datadir=/var/lib/mysql
# mysql.socket表示server和client在同一台服务器，并且使用localhost进行连接，就会使用socket进行连接
socket=/var/lib/mysql/mysql.sock
# 存储mysql的pid
pid_file=/var/lib/mysql/mysql.pid
# mysql服务的端口号
port=3306
# mysql存储引擎
default_storage_engine=InnoDB
# 当忘记mysql的用户名密码的时候，可以在mysql配置文件中配置该参数，跳过权限表验证，不需要密码即可登录mysql
skip-grant-tables
```

## character

```shell
# 客户端数据的字符集
character_set_client
# mysql处理客户端发来的信息时，会把这些数据转换成连接的字符集格式
character_set_connection
# mysql发送给客户端的结果集所用的字符集
character_set_results
# 数据库默认的字符集
character_set_database
# mysql server的默认字符集
character_set_server
```

## connection

```shell
# mysql的最大连接数，如果数据库的并发连接请求比较大，应该调高该值
max_connections
# 限制每个用户的连接个数
max_user_connections
# mysql能够暂存的连接数量，当mysql的线程在一个很短时间内得到非常多的连接请求时，就会起作用，如果mysql的连接数量达到
# max_connections时，新的请求会被存储在堆栈中，以等待某一个连接释放资源，如果等待连接的数量超过back_log,则不再接受连接资源
back_log
# mysql在关闭一个非交互的连接之前需要等待的时长
wait_timeout
# 关闭一个交互连接之前需要等待的秒数
interactive_timeout
```

## log

```shell
# 指定错误日志文件名称，用于记录当mysqld启动和停止时，以及服务器在运行中发生任何严重错误时的相关信息
log_error
# 指定二进制日志文件名称，用于记录对数据造成更改的所有查询语句
log_bin
# 指定将更新记录到二进制日志的数据库，其他所有没有显式指定的数据库更新将忽略，不记录在日志中
binlog_do_db
# 指定不将更新记录到二进制日志的数据库
binlog_ignore_db
# 指定多少次写日志后同步磁盘
sync_binlog
# 是否开启查询日志记录
general_log
# 指定查询日志文件名，用于记录所有的查询语句
general_log_file
# 是否开启慢查询日志记录
slow_query_log
# 指定慢查询日志文件名称，用于记录耗时比较长的查询语句
slow_query_log_file
# 设置慢查询的时间，超过这个时间的查询语句才会记录日志
long_query_time
# 是否将管理语句写入慢查询日志
log_slow_admin_statements
```

## cache

```shell
# 索引缓存区的大小（只对myisam表起作用）
show variables like '%key_buffer_size%';

# 查询缓存的大小（未来版本被删除）
show variables like '%query_cache_size%';

# 查看缓存的相关属性
show status like '%Qcache%';
Qcache_free_blocks：# 缓存中相邻内存块的个数，如果值比较大，那么查询缓存中碎片比较多
Qcache_free_memory：# 查询缓存中剩余的内存大小
Qcache_hits：# 表示有多少此命中缓存
Qcache_inserts：# 表示多少次未命中而插入
Qcache_lowmen_prunes：# 多少条query因为内存不足而被移除cache
Qcache_queries_in_cache：# 当前cache中缓存的query数量
Qcache_total_blocks：# 当前cache中block的数量

# 超出此大小的查询将不被缓存
show variables like 'query_cache_limit';
# 缓存块最小大小    
show variables like 'query_cache_min_res_unit';
# 缓存类型，决定缓存什么样的查询
# 0 表示禁用
# 1 表示将缓存所有结果，除非sql语句中使用sql_no_cache禁用查询缓存
# 2 表示只缓存select语句中通过sql_cache指定需要缓存的查询
show variables like 'query_cache_type';
# 每个需要排序的线程分派该大小的缓冲区
show variables like 'sort_buffer_size';
# 限制server接受的数据包大小
show variables like 'max_allowed_packet';
# 表示关联缓存的大小
show variables like 'join_buffer_size';

# 线程池大小
show variables like 'thread_cache_size';
# 查看当前线程池的状态
show global status like 'Thread%';
Threads_cached		# 代表当前此时此刻线程缓存中有多少空闲线程
Threads_connected	# 代表当前已建立连接的数量
Threads_created		# 代表最近一次服务启动，已创建现成的数量，如果该值比较大，那么服务器会一直再创建线程
Threads_running		# 代表当前激活的线程数
```

## innodb

```shell
# 该参数指定大小的内存来缓冲数据和索引，最大可以设置为物理内存的80%
show variables like 'innodb_buffer_pool_size'
# 主要控制innodb将log buffer中的数据写入日志文件并flush磁盘的时间点，值分别为0，1，2
show variables like 'innodb_flush_log_at_trx_commit'
# 设置innodb线程的并发数，默认为0表示不受限制，	如果要设置建议跟服务器的cpu核心数一致或者是cpu核心数的两倍
show variables like 'innodb_thread_concurrency'
# 此参数确定日志文件所用的内存大小，以M为单位
show variables like 'innodb_log_buffer_size'
# 此参数确定数据日志文件的大小，以M为单位
show variables like 'innodb_log_file_size'
# 以循环方式将日志文件写到多个文件中
show variables like 'innodb_log_files_in_group'
# mysql读入缓冲区大小，对表进行顺序扫描的请求将分配到一个读入缓冲区
show variables like 'read_buffer_size'
# mysql随机读的缓冲区大小
show variables like 'read_rnd_buffer_size'
# 此参数确定为每张表分配一个新的文件
show variables like 'innodb_file_per_table'
```

