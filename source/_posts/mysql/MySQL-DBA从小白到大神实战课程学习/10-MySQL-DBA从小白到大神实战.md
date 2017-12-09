---
title: MySQL DBA从小白到大神实战-10 深入理解MySQL主从复制
date: 2017-3-31
tags:
- mysql
categories:
- MySQL DBA从小白到大神实战
---

## 如何解决主从复制延迟的问题？
1. 加大主从库之间网络带宽
2. 使用SSD盘
3. 使用缓存服务器Redis/memcached

<!-- more -->

## 如何判断主从复制是否同步？
```perl
# 在从库上查看同步状态
show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.245.231.201
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000037     # I/O线程 接收的binlog文件
          Read_Master_Log_Pos: 1653              # I/O线程 binlog文件偏移量
               Relay_Log_File: relaylog.000002   
                Relay_Log_Pos: 604
        Relay_Master_Log_File: binlog.000037     # SQL线程 已应用的binlog文件
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
            ......
          Exec_Master_Log_Pos: 1653              # SQL线程 已应用的binlog文件偏移量
         ......
        Seconds_Behind_Master: 0                 # 从库和主库延迟时间

# 如果主库实例正常，同时查看主库状态
*************************** 1. row ***************************
             File: binlog.000037
         Position: 1653
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)          
```

查看主库`File`和从库I/O线程`Master_Log_File`和SQL线程`Relay_Master_Log_File`是不是相同，如果不同（如`Relay_Master_Log_File`小），说明有延迟；如果相等，再比较主库的`Position`和从库的`Read_Master_Log_Pos`和`Exec_Master_Log_Pos`是不是相同，如果偏移量也相同，说明主从复制是同步的。

## [sql_slave_skip_counter]参数设为1怎么理解？
>set global sql_slave_skip_counter = N
><font color='red'>This statement skips the next N events from the master.</font>
><font color='blue'>即是跳过N个events，这里最重要的是理解event的含义!在mysql中,对于sql的 binary log 实际上是由一连串的event组成的一个组,即事务组。</font>
在备库上设置`global sql_slave_skip_counter =N`会跳过当前时间来自于master的之后N个事件，这对于恢复由某条SQL语句引起的从库复制有效。
此语句只在当`slave threads`是停止时才有效，每忽略一个事务，N减一，直到N减为0！

```perl
# 当slave threads停止时，跳过1个事务
set global sql_slave_skip_counter=1;
```

## 线上快速搭建Mysql主从复制流程
1. 初始化mysql主从库
2. 主创建复制帐号，授权
3. 修改主和从的`server_id`(全局中要唯一)
4. 主库做全备(初始化没数据的库可以省略此步)
5. 从库做恢复(初始化没数据的库可以省略此步)
6. 备库执行`change master`设置复制
7. 备库启动复制(I/O线程、SQL线程)
8. 查看复制状态

![](http://oligvdnzp.bkt.clouddn.com/0331_mysql_master_slave_01.png)

### 主库配置
```perl
prompt my3306-> 
grant replication slave,replication client on *.* to rep@'%' identified by 'rep123';

show variables like '%server_id%';
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| server_id      | 101   |   # 保证全局唯一
| server_id_bits | 32    |
+----------------+-------+

show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| jfedu              |
| mysql              |
| performance_schema |
+--------------------+

# 使用mysqldump工具备份jfedu数据库
mysqldump -h127.0.0.1 -P3306 -uroot -pmysqlroot --single-transaction --master-data=2  > /tmp/all.sql

# 测试备份后DML操作
use jfedu
show tables;
+-----------------+
| Tables_in_jfedu |
+-----------------+
| gyj_t1          |
| gyj_t2          |
| t1              |
+-----------------+

create table gyj_t3 (id int,name varchar(10));
insert into gyj_t3 values (1,'AAAAA');
commit;

show tables;
+-----------------+
| Tables_in_jfedu |
+-----------------+
| gyj_t1          |
| gyj_t2          |
| gyj_t3          |
| t1              |
+-----------------+

# 查看从库配置复制时需用到的CHANGE MASTER内容
cat /tmp/jfedu.sql | grep 'CHANGE MASTER'
-- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000037', MASTER_LOG_POS=1329;
```

### 从库配置
```perl
vi /u01/conf/my3307.cnf
------------------------
# 修改从库server_id，确保全局唯一
server_id=102
------------------------

prompt my3307-> 
set global server_id=102;

show variables like '%server_id%';
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| server_id      | 102   |
| server_id_bits | 32    |
+----------------+-------+

show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+

create database jfedu default character set utf8;
use jfedu
source /tmp/jfedu.sql

show tables;
+-----------------+
| Tables_in_jfedu |
+-----------------+
| gyj_t1          |
| gyj_t2          |
| t1              |
+-----------------+

# 设置复制
CHANGE MASTER TO 
    MASTER_HOST='10.245.231.201',
    MASTER_PORT=3306,
    MASTER_USER='rep',
    MASTER_PASSWORD='rep123',
    MASTER_LOG_FILE='binlog.000037', 
    MASTER_LOG_POS=1329;

# 开启slave
start slave;
## 或者分两条
start slave io_thread;
start slave sql_thread;

# 主库备份后的表已经传到从库
show tables;
+-----------------+
| Tables_in_jfedu |
+-----------------+
| gyj_t1          |
| gyj_t2          |
| gyj_t3          |
| t1              |
+-----------------+

# 查看slave状态
show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.245.231.201
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000037
          Read_Master_Log_Pos: 1653
               Relay_Log_File: relaylog.000002
                Relay_Log_Pos: 604
        Relay_Master_Log_File: binlog.000037
             Slave_IO_Running: Yes       # 状态需为YES
            Slave_SQL_Running: Yes       # 状态需为YES
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1653
              Relay_Log_Space: 770
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 101
                  Master_UUID: 3049e83f-fef5-11e6-9165-005056a02d56
             Master_Info_File: /u01/data/3307/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
1 row in set (0.00 sec)

ERROR: 
No query specified

# 停用slave
stop slave;
```

## 主从复制的详细过程分析
![](http://oligvdnzp.bkt.clouddn.com/0331_mysql_master_slave_02.png)
```perl
my3306-> show processlist\G;
*************************** 1. row ***************************
     Id: 258167
   User: root
   Host: localhost
     db: jfedu
Command: Query
   Time: 0
  State: init
   Info: show processlist
*************************** 2. row ***************************
     Id: 260791
   User: rep
   Host: 10.245.231.201:33130
     db: NULL
Command: Binlog Dump     # Dump线程会发送所有binlog日志到从库
   Time: 2101
  State: Master has sent all binlog to slave; waiting for binlog to be updated
   Info: NULL
2 rows in set (0.00 sec)

my3306-> show master status\G;
*************************** 1. row ***************************
             File: binlog.000037   # binlog位置
         Position: 1653
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 

my3306-> show binlog events in 'binlog.000037';

my3307-> show processlist;
+----+-------------+-----------+-------+---------+------+-----------------------------------------------------------------------------+------------------+
| Id | User        | Host      | db    | Command | Time | State                                                                       | Info             |
+----+-------------+-----------+-------+---------+------+-----------------------------------------------------------------------------+------------------+
|  5 | root        | localhost | jfedu | Query   |    0 | init                                                                        | show processlist |
|  6 | system user |           | NULL  | Connect | 2307 | Waiting for master to send event                                            | NULL             |
|  7 | system user |           | NULL  | Connect | 4181 | Slave has read all relay log; waiting for the slave I/O thread to update it | NULL             |
+----+-------------+-----------+-------+---------+------+-----------------------------------------------------------------------------+------------------+

mysqlbinlog -vv /u01/log/3306/binlog/binlog.000037
mysqlbinlog -vv /u01/log/3307/binlog/binlog.000017

# 从库生成binlog以下参数需要开启
show variables like 'sql_log_bin';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| sql_log_bin   | ON    |
+---------------+-------+

cat /u01/data/3307/master.info
cat /u01/log/3307/relay-log.info
```

## 主从复制相关参数
### Master
```perl
server_id          # 全局唯一
read_only=OFF      # 主库设置成OFF关闭
sql_log_bin=ON     # 默认是开启
binlog_format=ROW
binlog_cache_size  # 有大事务时设置大些
max_binlog_size
expire_logs_days   # binlog日志保留时间
binlog-do-db
binlog-ignore-db
```

### Slave
```perl
server_id          # 全局唯一
read_only=ON       # 非级联时设置ON
sql_log_bin=ON     # 默认是开启
log_slave_updates
replicate-do-db
replicate-ignore-db
replicate-do-table
replicate-ignore-table
```

## Binlog日志格式
![](http://oligvdnzp.bkt.clouddn.com/0331_mysql_master_slave_03.png)

## Semi-sync半同步复制
- Semi-sync最早是由Google实现的一个补丁
- Semi-sync性能比较差（尤其是在网络慢且大并发时），在主数据库宕机后，Semi-sync能减少数据丢失的可能性，但不能绝对保证数据不丢失
- Semi-sync就是保证主库将日志先传输到备库，然后再返回给应用事务提交成功，流程如下:
![](http://oligvdnzp.bkt.clouddn.com/0331_mysql_master_slave_04.png)

开启步骤
```perl
ll /u01/mysql/lib/plugin/semi*
-rwxr-xr-x 1 mysql mysql 424735 Feb 24 10:00 /u01/mysql/lib/plugin/semisync_master.so
-rwxr-xr-x 1 mysql mysql 247911 Feb 24 10:00 /u01/mysql/lib/plugin/semisync_slave.so

# 主库上执行
install plugin rpl_semi_sync_master soname 'semisync_master.so';

show variables like '%semi%';
+------------------------------------+-------+
| Variable_name                      | Value |
+------------------------------------+-------+
| rpl_semi_sync_master_enabled       | OFF   | # 需要设置成ON
| rpl_semi_sync_master_timeout       | 10000 |
| rpl_semi_sync_master_trace_level   | 32    |
| rpl_semi_sync_master_wait_no_slave | ON    |
+------------------------------------+-------+
4 rows in set (0.00 sec)

set global rpl_semi_sync_master_enabled=on;

# 从库上执行
install plugin rpl_semi_sync_slave soname 'semisync_slave.so';

show variables like '%semi%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| rpl_semi_sync_slave_enabled     | OFF   |  # 需要设置成ON
| rpl_semi_sync_slave_trace_level | 32    |
+---------------------------------+-------+

set global rpl_semi_sync_slave_enabled=on;
```

## 杂记
1. Mysql主从复制是异步复制  问题：数据会丢失  课题：怎么解决数据丢失？
2. Mysql支持一主多从复制
3. Mysql支持级联复制
4. Mysql支持双向复制
5. Mysql 5.7开始可以多主一从，可以做多元复制，5.7以前只能有一个master
6. 主服务器 写数据  binlog  -->  从服务器  I/O线程（接收binlog，写到中继日志）  SQL线程（读relay log，event 应用日志写到从服务器数据库）