---
title: Mysql调优
date: {{ date }}
categories:
- Mysql
---

导入mysql官方案例数据库 https://dev.mysql.com/doc/index-other.html 

```shell
mysql -u root -p

mysql>source /sakila-db/sakila-schema.sql
mysql>source /sakila-db/sakila-data.sql
```

## 常用指令

### slow query log

> 慢查询日志

```sql
-- 查看是否开启慢查询日志
show variables like '%slow_query_log%'
-- 开启和关闭 (1和0)，数据库关闭失效
set global slow_query_log=1;
-- 查看多少时间算慢查询，默认10秒
show variables like '%long_query_time%';
-- 设置慢查询的阈值
set global long_query_time=3;
```

### show profile

> 是mysql提供可以用来分析当前会话中的语句执行的资源消耗情况，可以用于sql调优的测量

```sql
-- 查看profile是否开启，默认关闭
show variables like 'profiling';
-- 打开profile
set profiling = on;
-- 查询profiles
show profiles;
-- 查询详情
show profile cpu, block io for query [Query_ID];
```

### 执行计划 explain

```sql
--- Explain查询结果字段说明 ---
id: id越大，优先级越高；id相同，执行顺序由上至下
select_type: 查询类型，主要用于区别普通查询、联合查询、子查询等
table: 查询的表
partitions: 分区
type: 访问类型
possible_keys: 查询涉及到的字段若存在索引，将被列出，但不一定被实际使用
key: 实际使用的索引，如为NULL，则没使用索引。查询中若使用了覆盖索引，则该索引仅出现在key列表中
key_len: 索引的长度
ref: 显示索引的哪一列被使用了
rows: 根据表统计信息索引选用的情况，大致估算出找出所需的记录需要读取的行数
extra: 额外信息

--- type (至少要达到range) ---
system: 表中只有一行记录，可忽略不计
const: 一次就检索到，主键或唯一索引比较常量
eq_ref: 唯一性索引扫描，表中只有一条记录与之匹配，常见于主键或唯一索引扫描
ref: 非唯一性索引扫描，返回匹配某个单独值的所有行
range: 只检索给定范围的行，where子句中出现<、>、between、in等查询
index: 全索引扫描
all: 全表扫描

--- extra ---
Using filesort: 无法利用索引进行排序，只能利用排序算法，会消耗额外的位置
Using temporary: 建立临时表来保存中间结果，查询完成后把临时表删除
Using Index: 当前查询有覆盖索引的，直接从索引中取数据，不访问数据表
Using where: 使用了where查询
Using join buffer: 使用了连接缓存
impossible where: where子句的结果是false
select tables optimized away: 在没有group by子句的情况下，基于索引优化MIN/MAX操作
distinct: 优化distinct操作，在找到第一匹配元组后即停止找同样值的动作
```

## 数据类型优化

### 数据类型设计原则

1. **使用可以正确存储数据的最小数据类型**

   更小数据类型更快，因为它们占用更少的磁盘、内存和CPU缓存，并且处理时需要的CPU周期更少

2. 简单数据类型的操作通常需要更少的CPU周期
   1. 整型比字符操作代价更低，因为字符集和校对规则是字符比较比整型比较更复杂
   2. 使用mysql自建类型而不是字符串来存储日期和时间
   3. 用整型存储IP地址

3. 查询条件中避免包含可为NULL的列

   mysql难以优化查询中包含可为 NULL 的列，因为可为 NULL 的列使得索引、索引统计和值比较变得更加复杂

4. 推荐使用整型的自增主键

   **必须有主键原因**：B+树必须有个主键，没有主键这个树无法组织，如果没有定义主键，Mysql会通过隐藏列`rowId`来生成一个主键。

   **推荐整型原因**：因为B+树需要一直比较主键的大小，整型的比较速度比字符串快，因为比较字符串需要先转换为ASCII码。而且整型比字符串省空间。

   **推荐自增原因**：因为B+树的存储是从左到右递增的。如果使用随机数存储，那么有可能将索引插入到中间的一个已经满的子节点中，那么就会导致树需要页分裂再平衡来维护索引。如果是自增的话，每一次插入的索引都是在最后的，所以性能比较高。

### 数据类型

1. 整型：尽量使用满足需求的最小数据类型

   TINYINT 8位，SMALLINT 16位，MEDIUMINT 24位，INT 32位，BIGINT 64位

2. 字符和字符串类型

   1. varchar：根据实际内容长度保存数据

      varchar(n) n小于等于255使用额外一个字节保存长度，n>255使用额外两个字节保存长度

      varchar(5)与varchar(255)保存同样的内容，硬盘存储空间相同，但内存空间占用不同，是指定的大小 

   2. char固定长度的字符串

      最大长度为255，会自动删除末尾的空格。检索和写效率都比varchar高，以空间换时间​

3. BLOB 和 TEXT 类型

   MySQL 把每个 BLOB 和 TEXT 值当作一个独立的对象处理

   两者都是为了存储很大数据而设计的字符串类型，分别采用二进制和字符方式存储

4. 日期类型：不要使用字符串存储日期类型，占用空间大，损失日期类型函数的便捷性

   1. datetime：8字节，与时区无关，数据库底层时区配置对datetime无效，可保存到毫秒，可保存时间范围大
   2. timestamp：4字节，范围：1970-01-01到2038-01-19，采用整形存储，依赖数据库设置的时区自动更新timestamp列的值
   3. date：3字节，范围 1000-01-01~9999-12-31，可以利用日期时间函数进行日期之间的计算

### 枚举代替字符串类型

有时可以使用枚举类代替常用的字符串类型，mysql存储枚举类型会非常紧凑，会根据列表值的数据压缩到一个或两个字节中，mysql在内部会将每个值在列表中的位置保存为整数，并且在表的.frm文件中保存“数字-字符串”映射关系的查找表

```mysql
create table enum_test(e enum('fish','apple','dog') not null);
insert into enum_test(e) values('fish'),('dog'),('apple');
select e+0 from enum_test;
```

### 特殊类型数据

人们经常使用varchar(15)来存储ip地址，然而，它的本质是32位无符号整数不是字符串，可以使用INET_ATON()和INET_NTOA函数在这两种表示方法之间转换

```mysql
select inet_aton('1.1.1.1')
select inet_ntoa(16843009)
```

## 查询优化

### 优化细节

1. 当使用索引列进行查询的时候尽量不要使用表达式

```sql
select actor_id from actor where actor_id=4;
select actor_id from actor where actor_id+1=5;
```

2. 尽量使用主键查询，因为主键查询不会触发回表查询

3. 使用前缀索引：取一个字符串的前几个字节作为索引，节约存储空间

   对于 BLOB, TEXT, VARCHAR 类型的列必须要使用前缀索引，因为 mysql 不允许索引这些列的完整长度

   mysql 无法使用前缀索引做 order by 和 group by

```sql
-- 索引的选择性越高则查询效率越高
-- 索引的选择性 = 不重复的索引值 / 数据表记录总数
select count(distinct left(city,3))/count(*) as sel3,
count(distinct left(city,4))/count(*) as sel4,
count(distinct left(city,5))/count(*) as sel5,
count(distinct left(city,6))/count(*) as sel6,
count(distinct left(city,7))/count(*) as sel7,
count(distinct left(city,8))/count(*) as sel8 
from citydemo;
-- 根据前面测试创建前缀索引
alter table citydemo add key(city(7));
```

2. 索引排序：explain 出来的 type 列的值为 index ，说明 mysql 使用了索引扫描做排序

   在使用排序的时候，如果 where 和 order by 中的列能组成一个最左前缀匹配，就会使用索引排序

3. union all,in,or都能够使用索引，但是推荐使用in

6. 范围列可以用到索引，但是范围列后面的列无法用到索引，索引最多用于一个范围列

7. 强制类型转换会全表扫描

```sql
-- 不会触发索引
explain select * from user where phone=13800001234;
-- 触发索引
explain select * from user where phone='13800001234';
```

6. 当需要进行表连接的时候，最好不要超过三张表，因为需要 join 的字段，数据类型必须一致

7. 能使用limit的时候尽量使用limit

8. 单表索引建议控制在5个以内，单索引字段数不允许超过5个（组合索引）

### 查询性能低下原因

> 网络、CPU、IO、上下文切换、系统调用、生成统计信息、锁等待时间

查询性能低下的主要原因：某些查询需要筛选大量的数据，可以通过减少访问数据量进行优化

1. 查询不需要的记录：查询时要在后面添加 `limit`

2. 多表关联时返回全部列

3. 总是取出全部列 `select *`

4. 重复查询相同的数据：需重复执行返回相同数据的查询，可以使用缓存

### 执行过程的优化

#### 查询优化处理

1. 语法解析器和预处理

   mysql通过关键字将SQL语句进行解析，并生成一颗解析树，mysql解析器将使用mysql语法规则验证和解析查询，例如验证使用使用了错误的关键字或者顺序是否正确等等，预处理器会进一步检查解析树是否合法，例如表名和列名是否存在，是否有歧义，还会验证权限等等

2. 查询优化器

```sql
select count(*) from film_actor;
-- 可以看到上面这条查询语句需要经过多少个数据页才能找到对应的数据（基于成本）
show status like 'last_query_cost';
```

**mysql可能会选择错误的执行计划 原因如下**

1. 统计信息不准确：InnoDB因为其mvcc的架构，并不能维护一个数据表的行数的精确统计信息

2. 执行计划的成本估算不等同于实际执行的成本：有时候某个执行计划虽然需要读取更多的页面，但是他的成本却更小，因为如果这些页面都是顺序读或者这些页面都已经在内存中的话，那么它的访问成本将很小，mysql层面并不知道哪些页面在内存中，哪些在磁盘，所以查询之际执行过程中到底需要多少次IO是无法得知的

3. mysql的优化是基于成本模型的优化，但是有可能不是最快的优化

4. mysql不考虑其他并发执行的查询

5. mysql不会考虑不受其控制的操作成本

6. 执行存储过程或者用户自定义函数的成本

#### 优化器的优化策略

1. 静态优化：直接对解析树进行分析，并完成优化

2. 动态优化：动态优化与查询的上下文有关，也可能跟取值、索引对应的行数有关


mysql对查询的静态优化只需要一次，但对动态优化在每次执行时都需要重新评估

#### 优化器的优化类型

1. 重新定义关联表的顺序：数据表的关联并不总是按照在查询中指定的顺序进行。将外连接转化成内连接，内连接的效率要高于外连接，使用等价变换规则，mysql可以使用一些等价变化来简化并规划表达式

2. min max：使用min max的时候，使用 group by 条件，这样可以使用索引

3. 索引覆盖扫描：当索引中的列包含所有查询中需要使用的列的时候，可以使用覆盖索引

4. 等值传播： 如果两个列的值通过等式关联，那么mysql能够把其中一个列的where条件传递到另一个上

```sql
-- 这两个SQL执行效率是一样的
explain select film.film_id from film inner join film_actor 
using(film_id) where film.film_id > 500;

explain select film.film_id from film inner join film_actor 
using(film_id) where film.film_id > 500 and film_actor.film_id > 500;
```

#### 优化子查询

子查询尽可能使用关联查询代替，因为子查询产生的临时表数据集多，产生额外的IO 

#### 优化排序

排序的算法：

1. 两次传输排序：第一次数据读取是将需要排序的字段读取出来，然后进行排序，第二次是将排好序的结果按照需要去读取数据行

   缺点：需要进行两次IO，效率较低

2. 单次传输排序：先读取查询所需要的所有列，然后再根据给定列进行排序，最后直接返回排序结

   缺点：在查询的列特别多的时候，会占用大量的存储空间，无法存储大量的数据

**优化策略**

当需要排序的列的总大小超过 max_length_for_sort_data 定义的字节，mysql 会选择双次排序，反之使用单次排序，当然，用户可以设置此参数的值来选择排序的方式

#### 优化count查询

总有人认为myisam的count函数比较快，这是有前提条件的，只有没有任何where条件的count才是比较快的

**使用近似值**

在某些应用场景中，不需要完全精确的值，可以参考使用近似值来代替，比如可以使用explain来获取近似的值

其实在很多OLAP的应用中，需要计算某一个列值的基数，有一个计算近似值的算法叫hyperloglog。

**更复杂的优化**

一般情况下，count()需要扫描大量的行才能获取精确的数据，其实很难优化，在实际操作的时候可以考虑使用索引覆盖扫描，或者增加汇总表，或者增加外部缓存系统。

#### 优化关联查询

join的实现方式原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122101434106.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

1. Join Buffer会缓存所有参与查询的列而不是只有Join的列。
2. 可以通过调整join_buffer_size缓存大小，默认值是256K
3. 使用Block Nested-Loop Join算法需要开启优化器管理配置的optimizer_switch的设置block_nested_loop为on，默认为开启。

```sql
show variables like '%optimizer_switch%'
```

**join on and 解释**

两张表匹配的时候，如果使用 and ，那么最终结果集中未被 and 匹配上的记录会展示为 NULL

举例1：假如是左连接，那么下面第一行返回的结果集的基础上，再将 e.age <= 20的 d 表的记录置为NULL

```sql
select * from emp e left join dep d on e.id = d.id 
and e.age > 20;
```

举例2：左表记录都存在，右表记录都是NULL

```sql
select * from emp e left join dep d on 1 != 1;
```

**优化策略**

1. 确保 on 或者 using 子句中的列上有索引

2. 一般情况下来说，只需要在关联顺序中的第二个表的相应列上创建索引

3. 确保任何的 group by 和 order by 中的表达式只涉及到一个表中的列，这样mysql才有可能使用索引来优化这个过程

4. 优化子查询：子查询的优化最重要的优化建议是尽可能使用关联查询代替

5. 尽可能减少Join语句中NestedLoop的循环总次数，用小结果集驱动大结果集。

6. 确保Join语句中被驱动表上的Join条件字段已经被索引。

#### 优化limit分页

>在很多应用场景中我们需要将数据进行分页，一般会使用limit加上偏移量的方法实现，同时加上合适的order by 的子句，如果这种方式有索引的帮助，效率通常不错，否则的化需要进行大量的文件排序操作，还有一种情况，当偏移量非常大的时候，前面的大部分数据都会被抛弃，这样的代价太高。

要优化这种查询的话，要么是在页面中限制分页的数量，要么优化大偏移量的性能

优化此类查询的最简单的办法就是尽可能地使用覆盖索引，而不是查询所有的列

```sql
-- 查看执行计划查看扫描的行数
-- 1000行
explain select film_id,description from film order by title limit 50,5
-- 55+1+55 行
explain select film.film_id,film.description from film inner join 
(select film_id from film order by title limit 50,5) as lim using(film_id);
```

#### 优化union查询

> mysql总是通过创建并填充临时表的方式来执行union查询，因此很多优化策略在union查询中都没法很好的使用。经常需要手工的将where、limit、order by等子句下推到各个子查询中，以便优化器可以充分利用这些条件进行优化

除非确实需要服务器消除重复的行，否则一定要使用union all，因此没有all关键字，mysql会在查询的时候给临时表加上distinct的关键字，这个操作的代价很高

#### 推荐使用用户自定义变量

**自定义变量的使用**

```sql
set @one :=1
set @min_actor :=(select min(actor_id) from actor)
set @last_week :=current_date-interval 1 week;
```

**自定义变量的限制**

1. 无法使用查询缓存
2. 不能在使用常量或者标识符的地方使用自定义变量，例如表名、列名或者limit子句
3. 用户自定义变量的生命周期是在一个连接中有效，所以不能用它们来做连接间的通信
4. 不能显式地声明自定义变量地类型
5. mysql优化器在某些场景下可能会将这些变量优化掉，这可能导致代码不按预想地方式运行
6. 赋值符号：=的优先级非常低，所以在使用赋值表达式的时候应该明确的使用括号
7. 使用未定义变量不会产生任何语法错误

**自定义变量的使用案例**

优化排名语句

```sql
-- 在给一个变量赋值的同时使用这个变量
select actor_id,@rownum:=@rownum+1 as rownum from actor limit 10;
-- 查询获取演过最多电影的前10名演员，然后根据出演电影次数做一个排名
select actor_id,count(*) as cnt from film_actor group by actor_id order by cnt desc limit 10;
-- 避免重新查询刚刚更新的数据
-- 当需要高效的更新一条记录的时间戳，同时希望查询当前记录中存放的时间戳是什么
update t1 set  lastUpdated=now() where id =1;
select lastUpdated from t1 where id =1;
update t1 set lastupdated = now() where id = 1 and @now:=now();
select @now;
-- 确定取值的顺序
-- 在赋值和读取变量的时候可能是在查询的不同阶段
set @rownum:=0;
select actor_id,@rownum:=@rownum+1 as cnt from actor where @rownum<=1;
-- 因为where和select在查询的不同阶段执行，所以看到查询到两条记录，这不符合预期
set @rownum:=0;
select actor_id,@rownum:=@rownum+1 as cnt from actor where @rownum<=1 order by first_name
-- 当引入了order by之后，发现打印出了全部结果，这是因为order by引入了文件排序，而where条件是在文件排序操作之前取值的  
-- 解决这个问题的关键在于让变量的赋值和取值发生在执行查询的同一阶段：
set @rownum:=0;
select actor_id,@rownum as cnt from actor where (@rownum:=@rownum+1)<=1;
```
