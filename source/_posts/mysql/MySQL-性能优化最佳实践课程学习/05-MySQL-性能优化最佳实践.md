---
title: MySQL性能优化最佳实践 - 05 MySQL核心参数优化
date: 2017-09-06
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

## back_log参数的作用
指定MySQL可能的TCP/IP的连接数量（一个TCP/IP连接占256k），默认是50。
当MySQL主线程在很短的时间内得到非常多的连接请求，该参数就起作用，之后主线程花些时间（尽管很短）检查连接并且启动一个新线程。
back_log参数的值指出在MySQL暂时停止响应新请求之前的短时间内多少个请求可以被存在堆栈中。如果系统在一个短时间内有很多连接，则需要增大该参数的值，该参数值指定到来的TCP/IP连接的侦听accept队列的大小。
不同的操作系统在这个accept队列大小上有它自己的限制，设定back_log高于你的操作系统的限制将是无效的。

参考值：
``` perl
back_log=300
# 或
back_log=500
```

<!-- more -->
## thread_cache_size参数的作用
thread_cache_size线程池，线程缓存。这个值表示可以重新利用保存在缓存中线程的数量，当断开连接时如果缓存中还有空间,那么客户端的线程将被放到缓存中,如果线程重新被请求，那么请求将从缓存中读取，如果缓存中是空的或者是新的请求，那么这个线程将被重新创建，如果有很多新的线程，增加这个值可以改善系统性能。
每建立一个连接，都需要一个线程来与之匹配，此参数用来缓存空闲的线程，以至不被销毁，如果线程缓存中有空闲线程，这时候如果建立新连接，MYSQL就会很快的响应连接请求。
可根据物理内存设置规则如下：
1G  ―> 8
2G  ―> 16
3G  ―> 32
大于3G  ―> 64

参考值：
``` perl
thread_cache_size = 64 

mysql> show variables like 'thread_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| thread_cache_size | 10    |
+-------------------+-------+

mysql> show global status like 'Thread%';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 9     |
| Threads_connected | 2     |
| Threads_created   | 12    |
| Threads_running   | 2     |
+-------------------+-------+

set global thread_cache_size=64
```

## table_open_cache参数的作用
table_open_cache，表高速缓存的大小。

当某一连接访问一个表时，MySQL会检查当前已缓存表的数量。如果该表已经在缓存中打开，则会直接访问缓存中的表已加快查询速度；如果该表未被缓存，则会将当前的表添加进缓存并进行查询。

在执行缓存操作之前，table_cache用于限制缓存表的最大数目：如果当前已经缓存的表未达到table_cache，则会将新表添加进来；若已经达到此值，MySQL将根据缓存表的最后查询时间、查询率等规则释放之前的缓存。

参考值：
``` perl
table_open_cache=1024
```

## 学习笔记
###CPU的优化
``` perl
# innodb_thread_concurrency设置的值约等CPU数
innodb_thread_concurrency=32

# 连接的优化
```

### 内存的优化
``` perl
# 关闭查询缓存，缓存到应用层(如redis)，不建议在MYSQL层面开始，5.6有BUG
query_cache_type=0
query_cache_size=0
```

### IO的优化
``` perl
# 单实例时物理内存的60-70%
innodb_buffer_pool_size=50G

# 每秒后台进程处理IO数据的上限，一般设置总IO的75%左右。 SSD设置成20000
innodb_io_capacity=20000

# innodb redo日志组数量(iblog)
innodb_log_files_in_group=4

# innodb redo日志组文件大小
innodb_log_file_size=1000M

# RAID使用直接写，提高性能
innodb_flush_method=O_DIRECT

# 脏页达到50%就写磁盘
innodb_max_dirty_pages_pct=50

# 设置一个表对应一个文件，保持默认
innodb_file_per_table=on

# 设置PAGE大小(默认16k)
innodb_page_size=4k

# SSD盘设置为0
innodb_flush_neighbors=0
```

### 连接的优化 
``` perl
# 指定MySQL可能的TCP/IP的连接数量，现在一般都是长连接，这个值不建议设置过大
back_log=300

# 最大的连接数
max_connections=3000

# 最大的用户连接数(和上面差20，是留给管理用的)
max_user_connections=2980

# 表描述符缓存大小，可减少文件打开/关闭次数
table_open_cache=1024

# 线程池，线程缓存
thread_cache_size=512

# 连接超时断开
wait_timeout=120

# 交互超时断开
interactive_timeout=120
```

### 数据库一致性优化
``` perl
innodb_flush_log_at_trx_commit=1
sync_binlog=1
```
