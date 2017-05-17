---
title: MySQL-DBA从小白到大神实战-14 运维MySQL过程中线上故障分析与排查
date: 2017-04-25
tags:
- mysql
---

## MySQL常见故障
1. 数据库连接数爆满
2. 大事务导致数据库挂死
3. MySQL 主库crash
4. 大规模行锁竞争
5. 线上执行update报错
6. 主从库不一致,复制中止
<!-- more -->

## 重现故障5，并处理

### 构建测试环境数据
``` perl
sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/insert.lua \
--oltp-table-size=10000000 --mysql-table-engine=innodb --db-driver=mysql \
--mysql-user=root --mysql-password=root123 --mysql-port=3306 \
--mysql-host=10.245.231.202 --mysql-db=lyj \
--events=0 --time=60 --oltp-tables-count=1 --report-interval=10 --threads=1 prepare

# 主从库查看表数据
mysql> select count(*) from sbtest1;
+----------+
| count(*) |
+----------+
| 10000000 |
+----------+
```

### 修改主从库innodb_buffer_pool_size的值
``` perl
show variables like '%innodb_buffer_pool_size%';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |  # 原先是128M
+-------------------------+-----------+

# 关闭主从数据库修改参数文件中的值
vi /u01/conf/my3306.cnf
innodb_buffer_pool_size= 1M

# 启动主从数据库后查看参数值
show variables like '%innodb_buffer_pool_size%';
+-------------------------+---------+
| Variable_name           | Value   |
+-------------------------+---------+
| innodb_buffer_pool_size | 5242880 |   # 虽在my3306.cnf参数中设置成1M，但设置成小于5M时，仍会变为5M
+-------------------------+---------+
```

### 在主库执行批量更新语句报错
``` perl
mysql> update sbtest1 set pad='408658831-01630576472-82254536572-63286619302-57396238088' where id>5000 and id<10000000;
mysql> ERROR 1206 (HY000): The total number of locks exceeds the lock table size    # 出现报错信息
```

### 解决故障
1.关闭从库，修改innodb_buffer_pool_size后重启
``` perl
vi /u01/conf/my3306.cnf
innodb_buffer_pool_size= 2G

# 重启查看参数修改已生效
show variables like '%innodb_buffer_pool_size%';
+-------------------------+------------+
| Variable_name           | Value      |
+-------------------------+------------+
| innodb_buffer_pool_size | 2147483648 |
+-------------------------+------------+
```

2.登陆mha server在线切换主从库
``` perl
# 查看mha状态
/usr/local/bin/masterha_check_status --conf=/u01/mha/etc/app.cnf
app (pid:7760) is running(0:PING_OK), master:mydb1

# 在主库mydb1上
set global event_scheduler=off;

# 在mha server上关闭管理进程并切换
/usr/local/bin/masterha_stop --conf=/u01/mha/etc/app.cnf
/usr/local/bin/masterha_master_switch --master_state=alive --conf=/u01/mha/etc/app.cnf
#------------------------------------------------------------------------------------
Wed Apr 26 17:41:44 2017 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='mydb2 or 10.245.231.203', MASTER_PORT=3306, MASTER_LOG_FILE='binlog.000019', MASTER_LOG_POS=120, MASTER_USER='rep', MASTER_PASSWORD='xxx';

Wed Apr 26 17:41:47 2017 - [info] Switching master to mydb2(10.245.231.203:3306) completed successfully
...
#------------------------------------------------------------------------------------

# 查看新主库状态
mysql> show master status\G;
*************************** 1. row ***************************
             File: binlog.000019
         Position: 120
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

# 在新从库上开启同步
CHANGE MASTER TO
  MASTER_HOST='10.245.231.203',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='binlog.000019',
  MASTER_LOG_POS=120,
  MASTER_USER='rep',
  MASTER_PASSWORD='rep123';

# 启动同步
start slave;

# 显示从库状态
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.245.231.203
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000019
          Read_Master_Log_Pos: 120
               Relay_Log_File: relaylog.000002
                Relay_Log_Pos: 280
        Relay_Master_Log_File: binlog.000019
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
.....
```

3.关闭新从库（原主库），修改innodb_buffer_pool_size后重启
``` perl
vi /u01/conf/my3306.cnf
innodb_buffer_pool_size= 2G

# 重启查看参数修改已生效
show variables like '%innodb_buffer_pool_size%';
+-------------------------+------------+
| Variable_name           | Value      |
+-------------------------+------------+
| innodb_buffer_pool_size | 2147483648 |
+-------------------------+------------+
```

4.重复步骤2
5.执行批量update
``` perl
mysql> use lyj
mysql> update sbtest1 set pad='408658831-01630576472-82254536572-63286619302-57396238088' where id>5000 and id<10000000;
ERROR 1197 (HY000): Multi-statement transaction required more than 'max_binlog_cache_size' bytes of storage; increase this mysqld variable and try again   # 又出现报错

# 按上面步骤修改max_binlog_cache_size的大小
show variables like '%max_binlog_cache_size%';
+-----------------------+------------+
| Variable_name         | Value      |
+-----------------------+------------+
| max_binlog_cache_size | 4294967296 |
+-----------------------+------------+

# 再次执行更新
mysql> update sbtest1 set pad='408658831-01630576472-82254536572-63286619302-57396238088' where id>5000 and id<10000000;
Query OK, 9994999 rows affected (2 min 24.62 sec)
Rows matched: 9994999  Changed: 9994999  Warnings: 0
```

## 请给出MySQL数据丢失的最佳解决方案
### 误删除导致丢失数据的解决方案
利用备份和mysqlbinlog工具进行恢复
### 宕机导致InnoDB事务数据丢失数据的解决方案
1. 设置`innodb_flush_log_at_trx_commit = 1`，在每次事务提交的时候，都把log buffer刷到文件系统中去，并且调用文件系统的`flush`操作将缓存刷新到磁盘上去，最大限度保证事务的redo日志刷到磁盘。
2. 设置`sync_binlog = 1`，在每次事务提交之后，MySQL将进行一次fsync之类的磁盘同步指令来将binlog_cache中的数据强制写入磁盘。即使系统Crash，也最多丢失binlog_cache中未完成的一个事务。

### 数据库复制导致数据丢失的解决方案
确保binlog全部传到从库
1. 使用semi sync（半同步）方式，事务提交后，必须要传到slave，事务才能算结束。对性能影响很大，依赖网络适合小tps系统。
2. 双写binlog，通过DBDR OS层的文件系统复制到备机，或者使用共享盘保存binlog日志。
3. 在数据层做文章，比如保证数据库写成功后，再异步队列的方式写一份，部分业务可以借助设计和数据流解决