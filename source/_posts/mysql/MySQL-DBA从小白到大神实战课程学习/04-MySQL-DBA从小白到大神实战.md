---
title: MySQL DBA从小白到大神实战-04 揭密MySQL databock and binlog的格式
date: 2017-03-17
toc_list_number: false
tags:
- mysql
---

## 1. 为什么创建一个InnoDB表只分配了96K而不是1M？
Mysql中`extent`(区)中是64个连续的`page`(页)，是分配空间的最小单位，标准大小是1M。 但在用户启用了参数`innodb_file_per_table=on`后，创建一个Innodb表时，初始分配的大小却是为96K。这是因为在每个段开始时，先用32个页大小的碎片页(fragment page)来存放数据，在使用完这些页之后才是64个连续页的申请。这样做的目的是，对于一些小表，或者是undo这类的段，可以在开始时申请较少的空间，节省磁盘容量的开销。 下面通过示例来显示InnoDB存储引擎对于区的申请方式。

<!--more-->

### 创建一个测试用的InnoDB表

创建数据库
``` perl
mysql> create database lyj;
```

查看数据库创建信息，并切换到lyj数据库
``` perl
mysql> show create database lyj;
+----------+--------------------------------------------------------------+
| Database | Create Database                                              |
+----------+--------------------------------------------------------------+
| lyj      | CREATE DATABASE `lyj` /*!40100 DEFAULT CHARACTER SET utf8 */ |
+----------+--------------------------------------------------------------+

mysql> use lyj;

mysql> select database();
+------------+
| database() |
+------------+
| lyj        |
+------------+
```

查看是否启用独享表空间存储方式（innodb_file_per_table=ON），启用后每一个表会单独的生成一个`table_name.ibd`的文件
``` perl
mysql> show variables like '%per_table%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |
+-----------------------+-------+
```

在lyj数据库中创建一张表
``` perl
create table lyj_t1 
  (col1 int not null auto_increment, 
   col2 varchar(7000),
   primary key (col1)
  ) engine=innodb;
# 将col2字段设置为varchar(7000)，这样能保证一个页最多可以存放2条记录
```

可通过以下命令查看表的创建信息
``` perl
mysql> show create table lyj_t1\G;
*************************** 1. row ***************************
       Table: lyj_t1
Create Table: CREATE TABLE `lyj_t1` (
  `col1` int(11) NOT NULL AUTO_INCREMENT,
  `col2` varchar(7000) DEFAULT NULL,
  PRIMARY KEY (`col1`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

### 查验创建表时分配的初始大小为96K
``` perl
[mysql@mysql lyj]$ pwd
/u01/my3306/data/lyj
[mysql@mysql lyj]$ ll
total 112
-rw-rw----. 1 mysql mysql    61 Feb 15 14:20 db.opt
-rw-rw----. 1 mysql mysql 29070 Feb 15 17:07 lyj_t1.frm   # frm 表定义
-rw-rw----. 1 mysql mysql 98304 Feb 15 17:07 lyj_t1.ibd   # idb 数据和索引 创建初始分配大小是 98304/1024=96K
```

### 插入数据观察表空间大小变化
插入2条记录
``` perl
insert into lyj_t1 select null, repeat('a',7000);
insert into lyj_t1 select null, repeat('a',7000);

system ls -lh /u01/my3306/data/lyj/lyj_t1.ibd
-rw-rw----. 1 mysql mysql 96K Feb 15 17:16 /u01/my3306/data/lyj/lyj_t1.ibd

# 这时可以通过py_innodb_page_info工具来查看表空间
# 根据之前对表的定义，插入的这两条记录应该位于同一个页中
# 所有记录都在一个页中，因此这时还没有非叶节点
$ python py_innodb_page_info.py -v /u01/my3306/data/lyj/lyj_t1.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0000>  # 这个就是数据页，page level表示所在层，0表示页子节点
page offset 00000000, page type <Freshly Allocated Page>
page offset 00000000, page type <Freshly Allocated Page>
Total number of page: 6:
Freshly Allocated Page: 2
Insert Buffer Bitmap: 1
File Space Header: 1
B-tree Node: 1
File Segment inode: 1
```

> `py_innodb_page_info`工具下载地址：[py_innodb_page_type.zip](http://olg2mr2vf.bkt.clouddn.com/py_innodb_page_type.zip)

再插入1条记录
``` perl
insert into lyj_t1 select null, repeat('a',7000);

$ python py_innodb_page_info.py -v /u01/my3306/data/lyj/lyj_t1.ibd
page offset 00000000, page type <File Space Header>
page offset 00000001, page type <Insert Buffer Bitmap>
page offset 00000002, page type <File Segment inode>
page offset 00000003, page type <B-tree Node>, page level <0001>
page offset 00000004, page type <B-tree Node>, page level <0000>
page offset 00000005, page type <B-tree Node>, page level <0000>
Total number of page: 6:
Insert Buffer Bitmap: 1
File Space Header: 1
B-tree Node: 3
File Segment inode: 1
```

创建批量插入数据的存储过程
``` sql
drop procedure if exists load_t1;
delimiter //
create procedure load_t1(count int unsigned)
begin
  declare s int unsigned default 1;
  declare c varchar(7000) default repeat('a', 7000);
    while s<= count do
    insert into lyj_t1 select null,c;
    set s=s+1;
  end while;
end;
//
delimiter;
```

调用存储过程，插入60条记录后观察表空间大小
``` perl
mysql> call load_t1(60);

mysql> select count(*) from lyj_t1;
+----------+
| count(*) |
+----------+
|       63 |
+----------+

mysql> system ls -lh /u01/my3306/data/lyj/lyj_t1.ibd
-rw-rw----. 1 mysql mysql 576K Feb 15 18:10 /u01/my3306/data/lyj/lyj_t1.ibd
```

再插入1条记录后观察表空间大小
``` perl
mysql> call load_t1(1);

mysql> select count(*) from lyj_t1;
+----------+
| count(*) |
+----------+
|       64 |
+----------+

mysql> system ls -lh /u01/my3306/data/lyj/lyj_t1.ibd
-rw-rw----. 1 mysql mysql 2.0M Feb 15 18:15 /u01/my3306/data/lyj/lyj_t1.ibd
# 因为已经用完了32个碎片页，新的页就会采用区的方式进行空间的申请
```

## 2. Innodb数据块解析，解析第2行记录格式

### 创建测试表并插入数据
创建测试用数据库
``` perl
drop database lyj;
create database lyj DEFAULT CHARACTER SET gbk COLLATE gbk_chinese_ci;

# 查看当前数据库中存在那些数据库，以及数据库创建信息
show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| lyj                |
| mysql              |
| performance_schema |
| test               |
+--------------------+

# 显示数据库创建信息，若加\G参数是格式化输出信息
show create database lyj;
+----------+-------------------------------------------------------------+
| Database | Create Database                                             |
+----------+-------------------------------------------------------------+
| lyj      | CREATE DATABASE `lyj` /*!40100 DEFAULT CHARACTER SET gbk */ |
+----------+-------------------------------------------------------------+

show create database lyj\G
*************************** 1. row ***************************
       Database: lyj
Create Database: CREATE DATABASE `lyj` /*!40100 DEFAULT CHARACTER SET gbk */
```

使用lyj数据库
``` perl
use lyj

# 查看当前选择的数据库
select database();
+------------+
| database() |
+------------+
| lyj        |
+------------+

# 若要删除数据库可使用以下命令
drop database lyj;
```

关闭自动提交
``` perl
show variables like '%autocommit%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+

set autocommit = 0;

show variables like '%autocommit%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
```

在lyj数据库中创建表并插入测试数据
``` perl
create table lyj_t1 
  (id int,name1 varchar(10),name2 varchar(10),name3 varchar(10),name4 varchar(10),name5 varchar(10));

show tables;
+---------------+
| Tables_in_lyj |
+---------------+
| lyj_t1        |
+---------------+

insert into lyj_t1 values(1,'A','BB','CCC','DDDD',null);
insert into lyj_t1 values(2,'aaaaaaaaaa','bbbbbbbbbb','ccccc','',null);
insert into lyj_t1 values(3,'aaaaaaaaaa','bbbbbbbbbb','ccccc','dddddd','e');
commit;

insert into lyj_t1 values(4,'aaaaaaaaaa','bbbbbbbbbb',null,'dddddd','e');
commit;

select * from lyj_t1;
+------+------------+------------+-------+--------+-------+
| id   | name1      | name2      | name3 | name4  | name5 |
+------+------------+------------+-------+--------+-------+
|    1 | A          | BB         | CCC   | DDDD   | NULL  |
|    2 | aaaaaaaaaa | bbbbbbbbbb | ccccc |        | NULL  |
|    3 | aaaaaaaaaa | bbbbbbbbbb | ccccc | dddddd | e     |
|    4 | aaaaaaaaaa | bbbbbbbbbb | NULL  | dddddd | e     |
+------+------------+------------+-------+--------+-------+
```

### 解析数据块

#### 使用hexdump工具导出表空间文件前4个块
``` perl
# 一个page 16k, 导出的16进制文件一行是16byte ==> 1个page在16制文件中有1024 row，即一个块是1024行

# 导出第1个块 File Space Header
hexdump -C -v /u01/my3306/data/lyj/lyj_t1.ibd | head -n 1024 > /tmp/1.txt

# 导出第2个块 Insert Buffer Bitmap
hexdump -C -v /u01/my3306/data/lyj/lyj_t1.ibd | head -n 2048 | tail -n 1024 > /tmp/2.txt

# 导出第3个块 File Segment inode
hexdump -C -v /u01/my3306/data/lyj/lyj_t1.ibd | head -n 3072 | tail -n 1024 > /tmp/3.txt

# 导出第4个块 Used Page，也就是行记录开始处
hexdump -C -v /u01/my3306/data/lyj/lyj_t1.ibd | head -n 4096 | tail -n 1024 > /tmp/4.txt
```

使用sz工具下载到windows本地查看，需要先secureCRT会话设置里先设置本地下载路径
![secureCRT中本地地址路径设置](http://oligvdnzp.bkt.clouddn.com/securecrt_01.png)

下载命令
``` perl
sz /tmp/1.txt
sz /tmp/2.txt
sz /tmp/3.txt
sz /tmp/4.txt
```

#### 查看导出块类型标识
![第1个块 File Space Header    标识(偏移量19开始)：0x0008](http://oligvdnzp.bkt.clouddn.com/innodb_block_01.png) 第1个块类型标识：0x0008
![第2个块 Insert Buffer Bitmap 标识(偏移量19开始)：0x0005](http://oligvdnzp.bkt.clouddn.com/innodb_block_02.png) 第2个块类型标识：0x0005
![第3个块 File Segment inode   标识(偏移量19开始)：0x0003](http://oligvdnzp.bkt.clouddn.com/innodb_block_03.png) 第3个块类型标识：0x0003
![第4个块 Used Page            标识(偏移量19开始)：0x45BF](http://oligvdnzp.bkt.clouddn.com/innodb_block_04.png) 第4个块类型标识：0x45BF

#### 查看行记录格式
``` perl
mysql> show table status like '%lyj_t1%'\G;
*************************** 1. row ***************************
           Name: lyj_t1
         Engine: InnoDB
        Version: 10
     Row_format: Compact       # 行的格式是Compact（简洁的）
           Rows: 4
 Avg_row_length: 4096
    Data_length: 16384
Max_data_length: 0
   Index_length: 0
      Data_free: 0
 Auto_increment: NULL
    Create_time: 2017-02-16 10:20:15
    Update_time: NULL
     Check_time: NULL
      Collation: gbk_chinese_ci
       Checksum: NULL
 Create_options: 
        Comment: 
```

#### Compact行记录结构
![Compact行记录结构](http://oligvdnzp.bkt.clouddn.com/compact.png)

#### 解析块头
记录是放在第4个块中，下面详细解析`/tmp/4.txt`文件
```
1    0000c000  51 07 91 31 00 00 00 03  ff ff ff ff ff ff ff ff  |Q..1............|
2    0000c010  00 00 00 00 00 2b d4 0d  45 bf 00 00 00 00 00 00  |.....+..E.......|
3    0000c020  00 00 00 00 00 13 00 02  01 5b 80 06 00 00 00 00  |.........[......|
4    0000c030  01 29 00 02 00 03 00 04  00 00 00 00 00 00 00 00  |.)..............|
5    0000c040  00 00 00 00 00 00 00 00  00 24 00 00 00 13 00 00  |.........$......|
6    0000c050  00 02 00 f2 00 00 00 13  00 00 00 02 00 32 01 00  |.............2..|
7    0000c060  02 00 1f 69 6e 66 69 6d  75 6d 00 05 00 0b 00 00  |...infimum......|
8    0000c070  73 75 70 72 65 6d 75 6d  04 03 02 01 20 00 00 10  |supremum.... ...|
9    0000c080  00 2b 00 00 00 00 02 0a  00 00 00 00 0e 56 95 00  |.+...........V..|
10   0000c090  00 01 42 01 10 80 00 00  01 41 42 42 43 43 43 44  |..B......ABBCCCD|
11   0000c0a0  44 44 44 00 05 0a 0a 20  00 00 18 00 3b 00 00 00  |DDD.... ....;...|
12   0000c0b0  00 02 0b 00 00 00 00 0e  56 95 00 00 01 42 01 1e  |........V....B..|
13   0000c0c0  80 00 00 02 61 61 61 61  61 61 61 61 61 61 62 62  |....aaaaaaaaaabb|
14   0000c0d0  62 62 62 62 62 62 62 62  63 63 63 63 63 01 06 05  |bbbbbbbbccccc...|
15   0000c0e0  0a 0a 00 00 00 20 00 41  00 00 00 00 02 0c 00 00  |..... .A........|
16   0000c0f0  00 00 0e 56 95 00 00 01  42 01 2c 80 00 00 03 61  |...V....B.,....a|
17   0000c100  61 61 61 61 61 61 61 61  61 62 62 62 62 62 62 62  |aaaaaaaaabbbbbbb|
18   0000c110  62 62 62 63 63 63 63 63  64 64 64 64 64 64 65 01  |bbbcccccdddddde.|
19   0000c120  06 0a 0a 08 00 00 28 ff  47 00 00 00 00 02 0d 00  |......(.G.......|
20   0000c130  00 00 00 0e 5e 98 00 00  01 45 01 10 80 00 00 04  |....^....E......|
21   0000c140  61 61 61 61 61 61 61 61  61 61 62 62 62 62 62 62  |aaaaaaaaaabbbbbb|
22   0000c150  62 62 62 62 64 64 64 64  64 64 65 00 00 00 00 00  |bbbbdddddde.....|
     ......
1024 0000fff0  00 00 00 00 00 70 00 63  f2 c3 5d 27 00 2b d4 0d  |.....p.c..]'.+..|
```

- 第1行中的`51 07 91 31`和最后一行的`f2 c3 5d 27`是checksum，调用mysql中的一个校验函数，若检验不一致判断该块有问题
- 第1行中的`00 00 00 03`，代表这是第几个page(0开始)，这里代表是第4个page
- 第1行中的`ff ff ff ff ff ff ff ff`中前4个字节代表记录上一页指针地址，后4个字节代表记录下一页指针地址，全是`f`代表上一页和下一页无记录
- 第2行中的`00 00 00 00 00 2b d4 0d`是日志地址，LSN(Log Segment Number)，其中后4个字节和块最后一行后4个字节`00 2b d4 0d`是一致的
- 第2行中的`45 bf`代表页（块）的类型
- 第3行中的`00 00 00 13`代表tablespace的编号
- 第3行中的`00 02`代表有两个slot，对应块最后一行尾部的`00 70 00 63`，每2个字节代表一个slot
- 第3行中的`01 5b`代表该页下一行的空闲开始位置，也就是`0000c15b`
- 第3行中的`80 06`能算出块中有几行记录，compact中该值初始是8002，8006-8002=4，代表该页有4行记录
- 第3行中的`00 00 00 00`代表可重用字节，这里没有删除过记录，值都是0
- 第4行中的`01 29`代表最后一行记录的开始位置，也就是该块中第4行记录的开始位置
- 第4行中的`00 02 00 03`其中`00 02`为往右插，`00 03`为连续插入3次
- 第4行中的`00 04`代表有4行记录
- 第4行中的`00 00 00 00 00 00 00 00`代表页上最大事务数
- 第5行中的`00 00`代表page level <0000>，也就是叶子节点
- 第5行中的`00 00 00 00 00 00 00 24`代表索引的ID
- 第5-6行中的`00 00 00 13 00 00 00 02 00 f2` 代表段头管理非叶节点的
- 第6行中的`00 00 00 13  00 00 00 02 00 32` 代表段头管理叶节点的
- 第6-7行中的`01 00 02 00 1f`代表最小虚拟值的行头，固定的5个节点
- 第7行中的`69 6e 66 69 6d 75 6d 00`代表的就是infimum，8个字节，后面的`00`占位
- 第7行中的`05 00 0b 00 00`代表最大虚拟值的行头，固定的5个节点
- 第8行中的`73 75 70 72 65 6d 75 6d`代表的就是supremum，8个字节

#### 解析第1行记录
```
8    0000c070  73 75 70 72 65 6d 75 6d  04 03 02 01 20 00 00 10  |supremum.... ...|
9    0000c080  00 2b 00 00 00 00 02 0a  00 00 00 00 0e 56 95 00  |.+...........V..|
10   0000c090  00 01 42 01 10 80 00 00  01 41 42 42 43 43 43 44  |..B......ABBCCCD|
11   0000c0a0  44 44 44 00 05 0a 0a 20  00 00 18 00 3b 00 00 00  |DDD.... ....;...|
```

- 第8行中的`04 03 02 01`代表变长字段长度列表，逆序，表中5个字段类型为varchar，有NULL数据
- 第8行中的`20`代表null标志位，第1行记录有null值，十六进制`20`转成二进制是100000，逆序，表中一共6列，1表示第6列为null，0表示其他列有值
- 第8-9行中的`00 00 10 00 2b` 是记录头信息，固定5个字节  c078(本行记录开位置)+2b(偏移量)=c0a3(下一行记录开始位置)
- 第9行中的`00 00 00 00 02 0a` 是RowID，固定6个字节，表没有主键
- 第9行中的`00 00 00 00 0e 56` 是事务ID，固定6个字节
- 第9-10行中的`95 00 00 01 42 01 10` 回滚指针Roll Pointer，固定7个字节
- 第10行中的`80 00 00 01` 是ID列int 占4字节 值是1
- 第10-11行的`41 42 42 43 43 43 44 44 44 44`是列的数据，结合变长字段长度列表，可以算出varchar每列的值
  - 第2列 41
  - 第3列 42 42 
  - 第4列 43 43 43 
  - 第5列 44 44 44 44
  - 第6列 null

#### 解析第2行记录
```
11   0000c0a0  44 44 44 00 05 0a 0a 20  00 00 18 00 3b 00 00 00  |DDD.... ....;...|
12   0000c0b0  00 02 0b 00 00 00 00 0e  56 95 00 00 01 42 01 1e  |........V....B..|
13   0000c0c0  80 00 00 02 61 61 61 61  61 61 61 61 61 61 62 62  |....aaaaaaaaaabb|
14   0000c0d0  62 62 62 62 62 62 62 62  63 63 63 63 63 01 06 05  |bbbbbbbbccccc...|
```

从第1行记录解析中，已可算出，第二行开始位置为c0a3
- 第11行中的`00 05 0a 0a`代表变长字段长度列表，逆序，表中5个字段类型为varchar，有NULL数据
- 第11行中的`20`代表null标志位，第2行记录有null值，十六进制`20`转成二进制是100000，逆序，表中一共6列，1表示第6列为null，0表示其他列有值
- 第11行中的`00 00 18 00 3b` 是记录头信息，固定5个字节   c0a3(本行记录开位置)+3b(偏移量)=c0de(下一行记录开始位置)
- 第11-12行中的`00 00 00 00 02 0b` 是RowID，固定6个字节，表没有主键
- 第12行中的`00 00 00 00 0e 56` 是事务ID，固定6个字节
- 第12中的`95 00 00 01 42 01 1e` 回滚指针Roll Pointer，固定7个字节
- 第13行中的`80 00 00 02` 是ID列int 占4字节 值是2
- 第13-14行的`61 61 61 61 61 61 61 61 61 61 62 62 62 62 62 62 62 62 62 62 63 63 63 63 63`是列的数据，结合变长字段长度列表，可以算出varchar每列的值
  - 第2列 61 61 61 61 61 61 61 61 61 61
  - 第3列 62 62 62 62 62 62 62 62 62 62 
  - 第4列 63 63 63 63 63
  - 第5列 空值
  - 第6列 null

#### 解析第3行记录
```
14   0000c0d0  62 62 62 62 62 62 62 62  63 63 63 63 63 01 06 05  |bbbbbbbbccccc...|
15   0000c0e0  0a 0a 00 00 00 20 00 41  00 00 00 00 02 0c 00 00  |..... .A........|
16   0000c0f0  00 00 0e 56 95 00 00 01  42 01 2c 80 00 00 03 61  |...V....B.,....a|
17   0000c100  61 61 61 61 61 61 61 61  61 62 62 62 62 62 62 62  |aaaaaaaaabbbbbbb|
18   0000c110  62 62 62 63 63 63 63 63  64 64 64 64 64 64 65 01  |bbbcccccdddddde.|
```

- 第14-15行中的`01 06 05 0a 0a`代表变长字段长度列表，逆序，表中5个字段类型为varchar，没有NULL数据
- 第15行中的`00`代表null标志位，00表示没有null值
- 第15行中的`00 00 20 00 41` 是记录头信息，固定5个字节
- 第15行中的`00 00 00 00 02 0c` 是RowID，固定6个字节，表没有主键
- 第15-16行中的`00 00 00 00 0e 56` 是事务ID，固定6个字节
- 第16中的`95 00 00 01 42 01 2c` 回滚指针Roll Pointer，固定7个字节
- 第16行中的`80 00 00 03` 是ID列int 占4字节 值是3
- 第16-18行的`61 61 61 61 61 61 61 61 61 61 62 62 62 62 62 62 62 62 62 62 63 63 63 63 63 64 64 64 64 64 64 65`是列的数据，结合变长字段长度列表，可以算出varchar每列的值
  - 第2列 61 61 61 61 61 61 61 61 61 61
  - 第3列 62 62 62 62 62 62 62 62 62 62 
  - 第4列 63 63 63 63 63
  - 第5列 64 64 64 64 64 64
  - 第6列 65

#### 解析第4行记录
```
18   0000c110  62 62 62 63 63 63 63 63  64 64 64 64 64 64 65 01  |bbbcccccdddddde.|
19   0000c120  06 0a 0a 08 00 00 28 ff  47 00 00 00 00 02 0d 00  |......(.G.......|
20   0000c130  00 00 00 0e 5e 98 00 00  01 45 01 10 80 00 00 04  |....^....E......|
21   0000c140  61 61 61 61 61 61 61 61  61 61 62 62 62 62 62 62  |aaaaaaaaaabbbbbb|
22   0000c150  62 62 62 62 64 64 64 64  64 64 65 00 00 00 00 00  |bbbbdddddde.....|
```

- 第18-19行中的`01 06 0a 0a`代表变长字段长度列表，逆序，表中5个字段类型为varchar，有NULL数据
- 第19行中的`08`代表null标志位，十六进制`08`转成二进制是001000，逆序，表中一共6列，1表示第4列为null，0表示其他列有值
- 第19行中的`00 00 28 ff 47` 是记录头信息，固定5个字节
- 第19行中的`00 00 00 00 02 0d` 是RowID，固定6个字节，表没有主键
- 第19-20行中的`00 00 00 00 0e 5e` 是事务ID，固定6个字节，这里事务ID已经变化
- 第20中的`98 00 00  01 45 01 10` 回滚指针Roll Pointer，固定7个字节
- 第20行中的`80 00 00 04` 是ID列int 占4字节 值是4
- 第16-18行的`61 61 61 61 61 61 61 61  61 61 62 62 62 62 62 62 62 62 62 62 64 64 64 64  64 64 65`是列的数据，结合变长字段长度列表，可以算出varchar每列的值
  - 第2列 61 61 61 61 61 61 61 61 61 61
  - 第3列 62 62 62 62 62 62 62 62 62 62 
  - 第4列 NULL
  - 第5列 64 64 64 64 64 64
  - 第5列 65

## 3.详细描述commit命令发出后，binlog日志从内存写到磁盘的过程？

### 重启数据库
重启数据库生成新的binlog，便于观察测试
``` perl
mysqladmin shutdown
mysqld_safe --defaults-file=/u01/my3306/my.cnf &

[mysql@mysql binlog]$ pwd
/u01/my3306/log/binlog
[mysql@mysql binlog]$ ll
total 928
-rw-rw----. 1 mysql mysql 933537 Feb 17 16:38 binlog.000007
-rw-rw----. 1 mysql mysql    391 Feb 17 16:54 binlog.000008
-rw-rw----. 1 mysql mysql    120 Feb 17 16:54 binlog.000009
-rw-rw----. 1 mysql mysql    111 Feb 17 16:54 binlog.index
```

### 用一个update事务做测试
``` perl
use lyj;
begin;
update lyj_t1 set name1='aaaaaaa' where id=1;
```

### 查看binlog
``` perl
# 未commit前
+---------------+-----+-------------+-----------+-------------+---------------------------------------+
| Log_name      | Pos | Event_type  | Server_id | End_log_pos | Info                                  |
+---------------+-----+-------------+-----------+-------------+---------------------------------------+
| binlog.000009 |   4 | Format_desc |       101 |         120 | Server ver: 5.6.35-log, Binlog ver: 4 |
+---------------+-----+-------------+-----------+-------------+---------------------------------------+

# commit后
commit;

show binlog events in 'binlog.000009';
+---------------+-----+-------------+-----------+-------------+---------------------------------------+
| Log_name      | Pos | Event_type  | Server_id | End_log_pos | Info                                  |
+---------------+-----+-------------+-----------+-------------+---------------------------------------+
| binlog.000009 |   4 | Format_desc |       101 |         120 | Server ver: 5.6.35-log, Binlog ver: 4 |
| binlog.000009 | 120 | Query       |       101 |         191 | BEGIN                                 |
| binlog.000009 | 191 | Table_map   |       101 |         254 | table_id: 70 (lyj.lyj_t1)             |
| binlog.000009 | 254 | Update_rows |       101 |         339 | table_id: 70 flags: STMT_END_F        |
| binlog.000009 | 339 | Xid         |       101 |         370 | COMMIT /* xid=9 */                    |
+---------------+-----+-------------+-----------+-------------+---------------------------------------+

# 也可用以下命令查看
mysqlbinlog -v -v binlog.000009
```


### Binlog日志生成的流程
![Binlog日志生成的流程](http://oligvdnzp.bkt.clouddn.com/0217_binlog_01.png)

1. 事务commit后，日志会被write到标准I/O cache里面，每个线程是单独缓存的，线程间的日志彼此独立，不能看到不同线程的日志，这时若发生crash，日志会丢失
2. write后，每个线程的日志会flush到OS file cache里，这时日志是全局可见的，每个线程都能看到这个日志，若此时crash，日志也会丢失
3. sync操作会从日志内存里把日志写到disk中，达到持久化。

### 相关的参数
``` perl
mysql> show variables like 'sync_binlog';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| sync_binlog   | 100   |    # 代表做100次commit再写binlog，一般设置成1，保证数据的一致性
+---------------+-------+
1 row in set (0.00 sec)

mysql> show variables like 'innodb_flush_log_at_trx_commit';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_trx_commit | 1     |   # 一般设置成1，保证数据的一致性
+--------------------------------+-------+
1 row in set (0.01 sec)
```


## 补充

### 查看innodb默认块的大小
``` perl
mysql> show variables like '%innodb_page_size%';
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| innodb_page_size | 16384 |
+------------------+-------+
```

### 查看文件格式
``` perl
mysql> show variables like '%file_format%';
+--------------------------+-----------+
| Variable_name            | Value     |
+--------------------------+-----------+
| innodb_file_format       | Barracuda |    # Barracuda[,bærə''kudə] 梭鱼类
| innodb_file_format_check | ON        |
| innodb_file_format_max   | Antelope  |    # Antelope['æntɪlop] 羚羊 
+--------------------------+-----------+
```

从下图可以看出`Antelope`只包括`Redundant`（已废弃）和`Compact`，故建议将`innodb_file_format_max`也设置为`Barracuda`
![innodb文件格式](http://oligvdnzp.bkt.clouddn.com/0216_file_format.png)

### 查看tablespace编号
``` perl
mysql> select * from information_schema.innodb_sys_tables where name like '%lyj_t1%';
+----------+----------------------------+------+--------+-------+-------------+------------+---------------+
| TABLE_ID | NAME                       | FLAG | N_COLS | SPACE | FILE_FORMAT | ROW_FORMAT | ZIP_PAGE_SIZE |
+----------+----------------------------+------+--------+-------+-------------+------------+---------------+
|       14 | SYS_DATAFILES              |    0 |      5 |     0 | Antelope    | Redundant  |             0 |
|       11 | SYS_FOREIGN                |    0 |      7 |     0 | Antelope    | Redundant  |             0 |
|       12 | SYS_FOREIGN_COLS           |    0 |      7 |     0 | Antelope    | Redundant  |             0 |
|       13 | SYS_TABLESPACES            |    0 |      6 |     0 | Antelope    | Redundant  |             0 |
|       27 | lyj/lyj_t1                 |    1 |      5 |    13 | Antelope    | Compact    |             0 |
|       16 | mysql/innodb_index_stats   |    1 |     11 |     2 | Antelope    | Compact    |             0 |
|       15 | mysql/innodb_table_stats   |    1 |      9 |     1 | Antelope    | Compact    |             0 |
|       18 | mysql/slave_master_info    |    1 |     26 |     4 | Antelope    | Compact    |             0 |
|       17 | mysql/slave_relay_log_info |    1 |     11 |     3 | Antelope    | Compact    |             0 |
|       19 | mysql/slave_worker_info    |    1 |     15 |     5 | Antelope    | Compact    |             0 |
+----------+----------------------------+------+--------+-------+-------------+------------+---------------+

mysql> select * from information_schema.innodb_sys_tables where name like '%lyj_t1%';
+----------+------------+------+--------+-------+-------------+------------+---------------+
| TABLE_ID | NAME       | FLAG | N_COLS | SPACE | FILE_FORMAT | ROW_FORMAT | ZIP_PAGE_SIZE |
+----------+------------+------+--------+-------+-------------+------------+---------------+
|       30 | lyj/lyj_t1 |    1 |      5 |    16 | Antelope    | Compact    |             0 |
+----------+------------+------+--------+-------+-------------+------------+---------------+
```

### 查看未提交事务ID
``` perl
set autocommit = 0;

insert into lyj_t1 values(5,'aaaaaaaaaa','bbbbbbbbbb','ccccc','dddddd','e');

select * from information_schema.innodb_sys_idx\G;
*************************** 1. row ***************************
                    trx_id: 3694
                 trx_state: RUNNING
               trx_started: 2017-02-17 15:42:41
     trx_requested_lock_id: NULL
          trx_wait_started: NULL
                trx_weight: 2
       trx_mysql_thread_id: 23
                 trx_query: select * from information_schema.innodb_trx
       trx_operation_state: NULL
         trx_tables_in_use: 0
         trx_tables_locked: 0
          trx_lock_structs: 1
     trx_lock_memory_bytes: 360
           trx_rows_locked: 0
         trx_rows_modified: 1
   trx_concurrency_tickets: 0
       trx_isolation_level: REPEATABLE READ
         trx_unique_checks: 1
    trx_foreign_key_checks: 1
trx_last_foreign_key_error: NULL
 trx_adaptive_hash_latched: 0
 trx_adaptive_hash_timeout: 10000
          trx_is_read_only: 0
trx_autocommit_non_locking: 0

rollback;
```

### 转换成16进制函数
``` perl
select hex('infimum');
+----------------+
| hex('infimum') |
+----------------+
| 696E66696D756D |
+----------------+
```

### 文件结构图
![文件结构](http://oligvdnzp.bkt.clouddn.com/innodb_file_s.png)

### Page结构图
![Page结构](http://oligvdnzp.bkt.clouddn.com/innodb_page_s.png)

