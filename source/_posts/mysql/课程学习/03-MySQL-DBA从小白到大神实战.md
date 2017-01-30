---
title: MySQL DBA从小白到大神实战-03 深入MySQL体系结构
date: 2017-01-23 13:42
tags:
- mysql
---

# 第三课 深入MySQL体系结构

## 1. thread pool的原理是什么？
线程池的原理很简单，类似于操作系统中的缓冲区的概念，它的流程如下：先启动若干数量的线程，并让这些线程都处于睡眠状态，当客户端有一个新请求时，就会唤醒线程池中的某一个睡眠线程，让它来处理客户端的这个请求，当处理完这个请求后，线程又处于睡眠状态。

MySQL线程池只在MariaDB，Oracle MySQL企业版中提供，Oracle MySQL社区版并不提供。

在传统方式下，MySQL线程调度方式有两种：每个连接一个线程(one-thread-per-connection)和所有连接一个线程（no-threads）。在实际生产中，一般用的是前者。即每当有一个客户端连接到MySQL服务器，MySQL服务器都会为该客户端创建一个单独的线程，请求结束后，销毁线程。连接数越多，则相应的线程会越多。这种方式在高并发情况下，会导致线程的频繁创建和释放。

<!-- more -->

## 2. 为什么用double write就能解决page坏的问题？
InnoDB 的Page Size一般是16KB，其数据校验也是针对这16KB来计算的，将数据写入到磁盘是以Page为单位进行操作的，mysql的page size跟系统文件的page size是不一致的，在写数据的时候, 系统并不是把整个buffer pool page一次性写到disk上，在极端情况下（比如断电）往往并不能保证这一操作的原子性，16K的数据，写入4K时，发生了系统断电/OS crash ，只有一部分写是成功的，这种情况下就是partial page write问题。

mysql在恢复的过程中是检查page的checksum（检验和），checksum就是pgae的最后事务号，发生partial page write问题时，page已经损坏，找不到该page中的事务号，就无法恢复（ redo里面是没有保留这个损坏page完全的镜像，就无法从REDO里恢复）。

为了解决 partial page write 问题 ，当mysql将脏数据flush到data file的时候, 先使用memcopy 将脏数据复制到内存中的double write buffer ，之后通过double write buffer再分2次，每次写入1MB到共享表空间，然后马上调用fsync函数，同步到磁盘上，避免缓冲带来的问题，在这个过程中，doublewrite是顺序写，开销并不大，在完成doublewrite写入后，在将double write buffer写入各表空间文件，这时是离散写入。如果发生了极端情况（断电），InnoDB再次启动后，发现了一个Page数据已经损坏，那么此时就可以从double write buffer中进行数据恢复了。

##### double write的优点是什么?
double write解决了partial page write的问题，它能保证即使double write部分发生了partial page write但也能恢复。另外一个好处就是double write能减少redo log的量, 有了double write，redo log只记录了二进制的变化量，也就等同于binary log，而通过前段时间的测试确实发现，在double write关闭的情况下，redo log比binary logs要大。

##### double write的缺点是什么?
虽然mysql称double write是一个buffer, 但其实它是开在物理文件上的一个buffer, 其实也就是file, 所以它会导致系统有更多的fsync操作, 而我们知道硬盘的fsync性能是很慢的, 所以它会降低mysql的整体性能. 但是并不会降低到原来的50%. 这主要是因为: 
1. double write是一个连接的存储空间, 所以硬盘在写数据的时候是顺序写, 而不是随机写, 这样性能更高。 
2. 另外将数据从double write buffer写到真正的segment中的时候, 系统会自动合并连接空间刷新的方式, 每次可以刷新多个pages。另外将数据从double write buffer写到真正的segment中的时候, 系统会自动合并连接空间刷新的方式, 每次可以刷新多个pages。

##### double write在恢复的时候是如何工作的?
如果是写double write buffer本身失败，那么这些数据不会被写到磁盘，innodb此时会从磁盘载入原始的数据，然后通过innodb的事务日志来计算出正确的数据，重新写入到double write buffer。如果double write buffer写成功的话，但是写磁盘失败，innodb就不用通过事务日志来计算了，而是直接用buffer的数据再写一遍。在恢复的时候，innodb直接比较页面的checksum，如果不对的话，就从硬盘载入原始数据，再由事务日志开始推演出正确的数据，所以innodb的恢复通常需要较长的时间。

##### 查看是否开启double write
```
show variables like '%double%';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| innodb_doublewrite | ON    |
+--------------------+-------+
```

## 3. InnoDB redo log与binlog有什么区别？有了InnoDB redo log为什么还要binlog?
- binlog会记录所有与MySQL数据库有关的日志记录，包括InnoDB、MyISAM、Heap等其他存储引擎的日志；而InnoDB存储引擎的redo log只记录有关该引擎本身的事务日志。
- 无论将binlog文件记录的格式设为STATEMENT还是ROW，又或是MIXED，其记录的都是关于一个事务的具体操作内容，即该日志是逻辑日志；而InnoDB存储引擎的redo log是关于每个页（page）更改的物理情况。
- binlog文件仅在事务提交后进行写入，即只写磁盘一次，不论这时该事务多大；而在事务进行的过程中，却不断有重做日志条目（redo entry）被写入到重做日志文件中。

binlog是MySQL Server层记录的日志，所有引擎产生的日志都会通过binlog进行封装；MySQL的特点就是支持多存储引擎，为了兼容绝大部分引擎来支持类似复制这样的特性，就需要采用binlog日志来用实现。简单的说，binlog 是mysqld 记录全局数据结构变化的log，用于复制和恢复；innodb redo log 是innodb 引擎自己记录事务过程的log，用于回滚和crash 恢复。

## 课程笔记
1.MySQL是单进程多线程
2.MySQL存储引擎是可插拔的，有InnoDB，MyISAM(早期)等
3.存储引擎是用来处理数据库相关的CRUD的操作
4.CRUD是指在做计算处理时的增加(Create)、读取(Retrieve)(重新得到数据)、更新(Update)和删除(Delete)几个单词的首字母简写
5.存储引擎的对象是表
6.MySQL数据库与实例的关系是一对一的
7.MySQL的数据库是物理操作系统文件或其他形式文件类型的集合，实例是由数据库后台进程/线程以及一个共享内存区组成
8.MySQL 5.6 InnoDB架构中Buffer Pool
![mysql_innodb_architecture_00](/img/mysql_innodb_architecture_00.png)
(1)`index page`: 数据缓存放在index page里，因MySQL数据的存储结构是Btree，所以称index page，page的概念相当于oracle中的buffer，1 page默认大小16k 
(2)`data dictionary`: 数据字典的缓冲，其文件是存放在iblog目录下的ibdata中，如`/u01/my3306/log/iblog/ibdata1`
![mysql_innodb_architecture_01](/img/mysql_innodb_architecture_01.png)
(3)`lock info`: 行锁放在lock info中，当行锁达到一定值的时候，行锁就会升级为表锁
(4)`undo page`: 缓存UNDO操作，DML操作修改前镜像放到undo page中，其文件也是存放在iblog目录下的ibdata中
(5)`insert buffer page`: 缓存二级索引（非唯一索引，或称辅助索引）
(6)`adaptive hash index`: 自适应哈希索引，InnoDB存储引擎会监控对表上索引的查找，如果观察到建立哈希索引可以带来速度的提升，则建立哈希索引，所以称之为自适应（adaptive）的。自适应哈希索引通过缓冲池的B+树构造而来，因此建立的速度很快。而且不需要将整个表都建哈希索引，InnoDB存储引擎会自动根据访问的频率和模式来为某些页建立哈希索引。
(7)`Buffer Pool`的大小一般设置为物理内存的60%-80%，在MySQL中可以通过以下命令查询：
```
mysql> show variables like '%buffer_pool_size%';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
```

9.`redo log buffer`: 缓存redo log，通过redo log thead写到redo log文件中存放在iblog目录下的ibdata中，如`/u01/my3306/log/iblog/ib_logfile0`
10.查找算法：链表遍历、二分查找、Btree查找、HASH查找
11.当数据库关闭时，把热块保存(缓存)到文件，在打开时再从文件加载到内存里，参数和设置方法如下：
```
show variables like '%dump%';
+-------------------------------------+-------+
| Variable_name                       | Value |
+-------------------------------------+-------+
| innodb_buffer_pool_dump_at_shutdown | OFF   |
| innodb_buffer_pool_dump_now         | OFF   |
+-------------------------------------+-------+

set global innodb_buffer_pool_dump_at_shutdown=1;
set global innodb_buffer_pool_dump_now=1;

exit

# 关闭数据库
mysqladmin shutdown

# 查看缓存的文件
ll /u01/my3306/log/iblog/
total 4145172
-rw-rw----. 1 mysql mysql        884 Jan 24 17:38 ib_buffer_pool   //这个就是缓存的文件
-rw-rw----. 1 mysql mysql   33554432 Jan 24 17:38 ibdata1
-rw-rw----. 1 mysql mysql   16777216 Jan 24 17:38 ibdata2
-rw-rw----. 1 mysql mysql 1048576000 Jan 24 17:38 ib_logfile0
-rw-rw----. 1 mysql mysql 1048576000 Jan 17 17:37 ib_logfile1
-rw-rw----. 1 mysql mysql 1048576000 Jan 17 17:37 ib_logfile2
-rw-rw----. 1 mysql mysql 1048576000 Jan 17 17:37 ib_logfile3

# 指定参数启动数据库
mysqld_safe --defaults-file=/u01/my3306/my.cnf &
```

12.安装MySQL Utilities
(1)选择MySQL Utilities适合的版本下载：https://dev.mysql.com/downloads/utilities/
![mysql_utilities_download_00](/img/mysql_utilities_download_00.png)
(2)选择Connector/Python适合的版本下载（依赖包）：https://dev.mysql.com/downloads/connector/python/
![mysql_utilities_download_01](/img/mysql_utilities_download_01.png)
(3)上传安装包到服务器/tmp目录，并安装（root用户下）
```
ll
total 32836
-rw-r--r--. 1 root root    258776 Jan 25 11:50 mysql-connector-python-2.1.5-1.el6.x86_64.rpm
-rw-r--r--. 1 root root    892500 Jan 25 11:39 mysql-utilities-1.6.4-1.el6.noarch.rpm

rpm -ivh mysql-connector-python-2.1.5-1.el6.x86_64.rpm
rpm -ivh mysql-utilities-1.6.4-1.el6.noarch.rpm
```
(4)MySQL Utilities--mysqlfrm
```
# 以诊断模式查看表结构定义文件
mysqlfrm --diagnostic user.frm
```

13.查看错误日志所在位置
```
mysql> show variables like '%log_error%';
+---------------------+---------------------------+
| Variable_name       | Value                     |
+---------------------+---------------------------+
| binlog_error_action | IGNORE_ERROR              |
| log_error           | /u01/my3306/log/error.log |  //错误日志所在位置
+---------------------+---------------------------+
```

14.开启慢查询
```
mysql> show variables like '%slow%';
+---------------------------+--------------------------+
| Variable_name             | Value                    |
+---------------------------+--------------------------+
| log_slow_admin_statements | ON                       |
| log_slow_slave_statements | OFF                      |
| slow_launch_time          | 2                        |
| slow_query_log            | ON                       |   //开启慢查询
| slow_query_log_file       | /u01/my3306/log/slow.log |   //慢查询日志位置
+---------------------------+--------------------------+ 

mysql> show variables like '%query_time%';
+-----------------+----------+
| Variable_name   | Value    |
+-----------------+----------+
| long_query_time | 1.000000 |   //慢查询时间为1s
+-----------------+----------+

```

15.通用日志默认是不开启的（通用日志主要用在数据库审计）
```
mysql> show variables like '%gen%';
+------------------+----------------------------+
| Variable_name    | Value                      |
+------------------+----------------------------+
| general_log      | OFF                        |
| general_log_file | /u01/my3306/data/mysql.log |
+------------------+----------------------------+
```

16.最大用户连接数
```
mysql> show variables like '%max_user_connect%';
+----------------------+-------+
| Variable_name        | Value |
+----------------------+-------+
| max_user_connections | 2800  |
+----------------------+-------+
```

17.查出mysqld进程号为27507
```
ps -ef | grep 3306

root      5475  5183  0 11:44 pts/1    00:00:00 grep 3306
mysql    26656     1  0 Jan17 ?        00:00:00 /bin/sh /u01/my3306/bin/mysqld_safe --defaults-file=/u01/my3306/my.cnf --user=mysql
mysql    27507 26656  0 Jan17 ?        00:01:32 /u01/my3306/bin/mysqld --defaults-file=/u01/my3306/my.cnf --basedir=/u01/my3306 --datadir=/u01/my3306/data --plugin-dir=/u01/my3306/lib/plugin --log-error=/u01/my3306/log/error.log --open-files-limit=65535 --pid-file=/u01/my3306/run/mysqld.pid --socket=/u01/my3306/run/mysql.sock --port=3306
```

18.查看mysqld进程27507下所有线程
``` 
pstack 27507

Thread 28 (Thread 0x7f9b070c4700 (LWP 27508)):
#0  0x00000038a040b68c in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
#1  0x00000000009536bb in os_event_wait_low(os_event*, long) ()
#2  0x0000000000950aa6 in os_aio_simulated_handle(unsigned long, fil_node_t**, void**, unsigned long*) ()
#3  0x0000000000a49ed7 in fil_aio_wait(unsigned long) ()
#4  0x00000000009b9638 in io_handler_thread ()
#5  0x00000038a0407aa1 in start_thread () from /lib64/libpthread.so.0
#6  0x00000038a00e8aad in clone () from /lib64/libc.so.6
# ...... 中间略过
Thread 1 (Thread 0x7f9b18f637e0 (LWP 27507)):
#0  0x00000038a00df283 in poll () from /lib64/libc.so.6
#1  0x0000000000585284 in handle_connections_sockets() ()
#2  0x000000000058cd51 in mysqld_main(int, char**) ()
#3  0x00000038a001ed1d in __libc_start_main () from /lib64/libc.so.6
#4  0x000000000057dbc1 in _start ()
```

19.read/write thread
```
mysql> show variables like '%io_thread%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| innodb_read_io_threads  | 4     |     # 预读
| innodb_write_io_threads | 10    |
+-------------------------+-------+
```

20.purge thread: 清undo page
```
mysql> show variables like '%purge%';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| gtid_purged                |       |
| innodb_max_purge_lag       | 0     |
| innodb_max_purge_lag_delay | 0     |
| innodb_purge_batch_size    | 300   |
| innodb_purge_threads       | 1     |
| relay_log_purge            | ON    |
+----------------------------+-------+
```