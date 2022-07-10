---
title: Mysql索引
date: {{ date }}
categories:
- 数据源
- Mysql
---

## 索引概念

索引是帮助Mysql高效获取数据的排好序的数据结构

索引会影响到 where 后面的查找和 order by 后面的排序

索引的优点

1. 大大减少了服务器需要扫描的数据量
2. 助服务器避免排序和临时表
3. 将随机 IO 变成顺序 IO	

索引的用处

1. 快速查找匹配 WHERE 子句的行
2. 从consideration中消除行,如果可以在多个索引之间进行选择，mysql通常会使用找到最少行的索引
3. 如果表具有多列索引，则优化器可以使用索引的任何最左前缀来查找行
4. 当有表连接的时候，从其他表检索行数据
5. 查找特定索引列的min或max值
6. 如果排序或分组时在可用索引的最左前缀上完成的，则对表进行排序和分组
7. 在某些情况下，可以优化查询以检索值而无需查询数据行

索引和页的关系

一般来说，索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储的磁盘上。这样的话，索引查找过程中就要产生磁盘IO消耗，相对于内存IO存取，磁盘IO存取的消耗要高几个数量级，所以评价一个数据结构作为索引的优劣最重要的指标就是在查找过程中磁盘IO操作次数的渐进复杂度。换句话说，索引的结构组织要尽量减少查找过程中磁盘IO的存取次数。

为了达到这个目的，磁盘按需读取，要求每次都会预读的长度一般为页（4K）的整数倍。而且数据库系统将一个节点的大小设为等于一个页，这样每个节点只需要一次IO就可以完全载入

## 索引分类

1. 普通索引：是最基本的索引，它没有任何限制
2. 主键索引：是一种特殊的唯一索引，一个表只能有一个主键，且不能为NULL
3. 唯一索引：索引列的值必须唯一，但允许为NULL
4. 组合索引：一个索引包含多个列
5. 全文索引：主要用来查找文本中的关键字，而不是直接与索引中的值相比较，只有char，varchar，text 列上可以创建全文索引

| 类型        | 描述                                     |
| ----------- | :--------------------------------------- |
| PRIMARY KEY | 主键，索引值必须唯一，且不能为NULL       |
| UNIQUE      | 索引值必须唯一，可以为NULL，并可出现多次 |
| INDEX       | 普通索引，索引值可能出现多次             |
| FULLTEXT    | 全文索引                                 |

**回表，覆盖索引，最左匹配，索引下推**

回表：普通列创建索引，普通列索引的叶子节点存放的是主键，所以按普通列查找的时候先根据普通列的 B+ 树找到主键再根据主键的 B+ 树找到整行数据

覆盖索引：查询的列是索引列，不用回表查询

```sql
-- name建了索引，id是主键索引

-- 产生回表
select * from emp where name = ?;
-- 覆盖索引
select id from emp where name = ?;
```

最左匹配：会按照组合索引的顺序匹配索引

```sql
-- 有组合索引 name, age

-- 用到了索引，因为 mysql 会对 where 条件后的顺序进行优化
select * from emp where name = ? and age = ?;
select * from emp where age = ? and name = ?;
select * from emp where name = ?;
-- 没用到索引
select * from emp where age = ?;

-- 解决方案：为age单独建立一个索引
```

索引下推：方式2的实现

执行以下 SQL 会有两种实现方式

1. 取出符合 name 条件数据集到内存，在内存中过滤掉不符合 age 条件的数据
2. 在存储引擎中直接取出符合 name，age 的数据集

```sql
-- 有组合索引 name, age
select * from emp where name = ? and age = ?;
```

## 索引语法和匹配方式

### 基本语法
```sql
-- 创建索引
CREATE INDEX idx_name ON tb_name(column_name);
ALTER TABLE tb_name ADD INDEX idx_name(column_name);
-- 删除索引
DROP INDEX idx_name ON tb_name;
-- 查看索引
SHOW INDEX FROM tb_name;
```
### 索引匹配方式

当包含多个列作为索引，需要注意的是正确的顺序依赖于该索引的查询，同时需要考虑如何更好的满足排序和分组的需要

案例，建立组合索引a,b,c，不同SQL语句使用索引情况

| 语句                                    | 索引是否发挥作用                 |
| --------------------------------------- | -------------------------------- |
| where a = 3                             | 是，使用了 a                     |
| where a= 3 and b = 4                    | 是，使用了 a, b                  |
| where a = 3 and b = 4 and c =5          | 是，使用了 a, b, c               |
| where b = 3 or c = 4                    | 否，因为跳过了a                  |
| where a = 3 and c = 4                   | 是，使用了 a                     |
| where a = 3 and b > 10 and c = 7        | 是，使用了 a, b，因为b是范围查找 |
| where a = 3 and b like '%xx%' and c = 7 | 是，使用了 a                     |

### 索引匹配方式 案例

```sql
create table staffs(
    id int primary key auto_increment,
    name varchar(24) not null default '' comment '姓名',
    age int not null default 0 comment '年龄',
    pos varchar(20) not null default '' comment '职位',
    add_time timestamp not null default current_timestamp comment '入职时间'
  ) charset utf8 comment '员工记录表';
alter table staffs add index idx_nap(name, age, pos);
```

全值匹配：全值匹配指的是和索引中的所有列进行匹配

```mysql
explain select * from staffs where name = 'July' and age = '23' and pos = 'dev';
```

匹配最左前缀：只匹配前面的几列

```mysql
explain select * from staffs where name = 'July' and age = '23';
explain select * from staffs where name = 'July';
```

匹配列前缀：可以匹配某一列的值的开头部分，**不要让%放到前面，会让索引失效**

```mysql
explain select * from staffs where name like 'J%';
explain select * from staffs where name like '%y';
```

匹配范围值：可以查找某一个范围的数据

```mysql
explain select * from staffs where name > 'Mary';
```

精确匹配某一列并范围匹配另外一列：可以查询第一列的全部和第二列的部分

```mysql
explain select * from staffs where name = 'July' and age > 25;
```

只访问索引的查询：查询的时候只需要访问索引，不需要访问数据行，本质上就是覆盖索引

```mysql
explain select name,age,pos from staffs where name = 'July' and age = 25 and pos = 'dev';
```

### 最左匹配原理

假如创建一个（a,b）的联合索引，那么它的索引树是这样的

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200910005504881.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzI2ODkzMw==,size_16,color_FFFFFF,t_70#pic_center)
可以看到a的值是有顺序的：1，1，2，2，3，3；b的值是没有顺序的1，2，1，4，1，2

所以 b = 2 这种查询条件没有办法利用索引，因为联合索引首先是按 a 排序的，b是无序的。

## 创建索引情景

### 需要建立索引
1. 主键自动建立唯一索引
2. 频繁作为查询条件的字段应该创建索引
3. 查询中与其它表关联的字段，外键关系建立索引
4. 在高并发下倾向创建复合索引
5. 排序字段若通过索引去访问将大大提高排序速度
6. 查询中统计或分组字段需要建索引
### 不需创建索引
1. 表记录太少
2. 经常增删改的字段
3. 数据重复且平均分布的表字段

原因

1. 更新会变更 B+ 树，更新频繁的字段建立索引会大大降低数据库性能

2. 区分不大的属性，建立索引是没有意义的，不能有效的过滤数据

3. 一般区分度在80%以上的时候就可以建立索引，计算区分度 `count(distinct(列名))/count(*) `

4. 创建索引的列，不允许为null，可能会得到不符合预期的结果

## 索引的数据结构
> 数据库索引所使用的数据结构是 B+Tree 和 Hash表

1. 二叉树：极端情况变成链表
    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200209094343621.png)
2. AVL树：最长子树和最短子树的高度差不能超过1，插入数据会进行1-N次旋转，即插入效率低，查询效率高

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210123100737778.png)

3. 红黑树：最长子树不超过最短子树的两倍，损失部分查询性能提升插入性能，可能导致树的高度过高

   左旋：逆时针旋转，父节点被右孩子取代，而自己成功自己的左孩子

   右旋：顺时针旋转，父节点被左孩子取代，而自己成为自己的右孩子

   ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200209094426919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

4. B-Tree：横向存储更多数据

> 由于MySQL索引一般都存储在内存中，如果使用B-Tree作为索引的话，索引和数据存储在一块，分布在各个节点中；而内存资源往往比较宝贵，一定内存的情况下可以存储的索引数量相对有限，毕竟每条数据的大小一般远大于索引列的大小，导致内存使用率不高。

- 所有键值分布在整棵树中
- 搜索有可能在非叶子结点结束，在关键字全集内做一次查找,性能逼近二分查找
- 每个节点最多拥有m个子树
- 根节点最少有两个子树
- 分支节点至少拥有m/2颗子树（除根节点和叶子节点外都是分支节点）
- 所有叶子节点都在同一层、每个节点最多可以有m-1个key，并且以升序排列

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210123104042810.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

5. B+ Tree（B-Tree变种）

> 索引和数据分开存储，让更多的索引存储在内存中

- 非叶子节点不存储data，值存储索引（冗余），可以放更多的索引。
- 叶子节点包含所有索引字段。
- 叶子节点用指针连接，提高访问的性能。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021020710474436.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

6. Hash表 （memory 存储引擎）

- 利用 Hash 存储的话需要将所有文件添加到内存，比较消耗内存空间
- Hash 表对于等值查询很快，对于范围查询很慢

## Mysql存储引擎

### 存储引擎

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210122232753310.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjEwMzAyNg==,size_16,color_FFFFFF,t_70)

1. MyISAM：一张表有3个文件 -- frm, MYD, MYI

  执行一条查询SQL流程：首先判断条件是否带有索引，如果带有索引，那么先去MYI文件中找到这个索引所代表的数据的地址，再根据这个地址去MYD中查询相应的数据。

2. InnoDB：一张表有2个文件 -- frm, ibd

  执行一条查询SQL流程：首先判断条件是否带有索引，如果带有索引，可以直接到ibd文件中查询到相应数据。

3. memory：基于内存的存储引擎

| 文件类型 | 描述           |
| -------- | -------------- |
| frm      | 存储表结构     |
| MYD      | 存储数据       |
| MYI      | 存储索引       |
| ibd      | 存储索引和数据 |

### 聚簇索引和非聚簇索引
**聚簇索引**：索引和数据存储在一个文件，如InnoDB的主键索引。

优点

1. 数据访问更快，因为索引和数据保存在同一个树中

2. 使用覆盖索引扫描的查询可以直接使用页节点中的主键值

缺点

1. 聚簇数据最大限度地提高了IO密集型应用的性能，如果数据全部在内存，那么聚簇索引就没有什么优势
2. 插入速度严重依赖于插入顺序，按照主键的顺序插入是最快的方式
3. 更新聚簇索引列的代价很高，因为会强制将每个被更新的行移动到新的位置
4. 基于聚簇索引的表在插入新行，或者主键被更新导致需要移动行的时候，可能面临页分裂的问题
   1. 页分裂：新数据要插入索引块，如果索引块空间不足，就会把原来的块等分为两份。
   2. 页合并：删除数据维护索引时，再把两个索引块合并成一个。
5. 聚簇索引可能导致全表扫描变慢，尤其是行比较稀疏，或者由于页分裂导致数据存储不连续的时候

**非聚簇索引**：索引和数据存储在不同文件，如MyISAM的主键索引。

### 哈希索引

> 在mysql中，只有memory的存储引擎显式支持哈希索引

基于哈希表的实现，只有精确匹配索引所有列的查询才有效

哈希索引自身只需存储对应的hash值，所以索引的结构十分紧凑，这让哈希索引查找的速度非常快

**哈希索引的限制**

1. 哈希索引只包含哈希值和行指针，而不存储字段值，索引不能使用索引中的值来避免读取行

2. 哈希索引数据并不是按照索引值顺序存储的，所以无法进行排序

3. 哈希索引不支持部分列匹配查找，哈希索引是使用索引列的全部内容来计算哈希值

4. 哈希索引支持等值比较查询，也不支持任何范围查询

5. 访问哈希索引的数据非常快，除非有很多哈希冲突，当出现哈希冲突的时候，存储引擎必须遍历链表中的所有行指针，逐行进行比较，直到找到所有符合条件的行

6. 哈希冲突比较多的话，维护的代价也会很高

**案例**

```sql
-- 当需要存储大量的URL，并且根据URL进行搜索查找，如果使用B+树，存储的内容就会很大
select id from url where url=""
-- 也可以利用将url使用CRC32做哈希，可以使用以下查询方式：
select id fom url where url="" and url_crc=CRC32("")
-- 此查询性能较高原因是使用体积很小的索引来完成查找
```

### 覆盖索引

> explain 结果中 extra 为 Using Index 就是使用了覆盖索引

基本介绍

1. 如果一个索引包含所有需要查询的字段的值，我们称之为覆盖索引

2. 不是所有类型的索引都可以称为覆盖索引，覆盖索引必须要存储索引列的值

3. 不同的存储实现覆盖索引的方式不同，不是所有的引擎都支持覆盖索引，如memory不支持覆盖索引

优势

1. 索引条目通常远小于数据行大小，如果只需要读取索引，那么mysql就会极大的较少数据访问量

2. 因为索引是按照列值顺序存储的，所以对于IO密集型的范围查询会比随机从磁盘读取每一行数据的IO要少的多

3. 一些存储引擎如MYISAM在内存中只缓存索引，数据则依赖于操作系统来缓存，因此要访问数据需要一次系统调用，这可能会导致严重的

4. 由于innodb是聚簇索引，覆盖索引对innodb表特别有用

### InnoDB表必须有主键

> 推荐使用整型的自增主键

**必须有主键原因**：B+树必须有个主键，没有主键这个树无法组织。

**推荐整型原因**：因为B+树需要一直比较主键的大小，整型的比较速度比字符串快，因为比较字符串需要先转换为ASCII码。而且整型比字符串省空间。

**推荐自增原因**：因为B+树的存储是从左到右递增的。如果使用随机数存储，那么有可能将索引插入到中间的一个已经满的子节点中，那么就会导致树需要分裂再平衡来维护索引。如果是自增的话，每一次插入的索引都是在最后的，所以性能比较高。

## 索引监控

```sql
show status like 'Handler_read%';
```

参数解释

```
Handler_read_first：读取索引第一个条目的次数
Handler_read_key：通过index获取数据的次数
Handler_read_last：读取索引最后一个条目的次数
Handler_read_next：通过索引读取下一条数据的次数
Handler_read_prev：通过索引读取上一条数据的次数
Handler_read_rnd：从固定位置读取数据的次数
Handler_read_rnd_next：从数据节点读取下一条数据的次数
```

## 索引失效

1. 最佳左前缀法则：如果索引了多列，查询从索引的最左前列开始并不跳过索引中的列。
2. 不在索引上做任何操作（计算、函数、类型转换），会导致索引失效转向全表扫描
3. 存储引擎不能使用索引中范围条件右边的列
4. 尽量使用覆盖索引（只访问索引的查询），不使用select *
5. 使用 != < > 的时候使用索引会导致全表扫描
6. is null，is not null无法使用索引
7. like以通配符开头会导致索引失效，所以%要写最右边
8. 强制类型转换会导致索引失效
9. 使用OR连接不会导致索引失效（mysql 5.0以后新增 索引合并）

