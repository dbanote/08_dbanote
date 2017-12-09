---
title: MySQL DBA从小白到大神实战-13 深入分析Online DDL原理
date: 2017-04-21
tags:
- mysql
categories:
- MySQL DBA从小白到大神实战
---

## 用oak对表sbtest1做添加字段和增加索引的Online DDL
### 用sysbench工具准备测试环境
```perl
# 创建数据库
mysql> create database lyj;

# 用sysbench工具准备测试环境
sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/insert.lua \
--oltp-table-size=1000000 --mysql-table-engine=innodb --db-driver=mysql \
--mysql-user=root --mysql-password=root123 --mysql-port=3306 \
--mysql-host=10.245.231.202 --mysql-db=lyj \
--events=0 --time=60 --oltp-tables-count=2 --report-interval=10 --threads=2 prepare

sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/insert.lua \
--oltp-table-size=1000000 --mysql-table-engine=innodb --db-driver=mysql \
--mysql-user=root --mysql-password=root123 --mysql-port=3306 \
--mysql-host=10.245.231.202 --mysql-db=lyj \
--events=0 --time=60 --oltp-tables-count=2 --report-interval=10 --threads=2 run
```

<!-- more -->
### 查看测试数据及表结构
``` perl
mysql> use lyj
mysql> select count(*) from sbtest1;
+----------+
| count(*) |
+----------+
|  1000000 |
+----------+

mysql> desc sbtest1;
+-------+------------------+------+-----+---------+----------------+
| Field | Type             | Null | Key | Default | Extra          |
+-------+------------------+------+-----+---------+----------------+
| id    | int(10) unsigned | NO   | PRI | NULL    | auto_increment |
| k     | int(10) unsigned | NO   | MUL | 0       |                |
| c     | char(120)        | NO   |     |         |                |
| pad   | char(60)         | NO   |     |         |                |
+-------+------------------+------+-----+---------+----------------+

mysql> show create table sbtest1\G;
*************************** 1. row ***************************
       Table: sbtest1
Create Table: CREATE TABLE `sbtest1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `k_1` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=1176193 DEFAULT CHARSET=utf8 MAX_ROWS=1000000
```

### 下载OAK工具
OAK全称是Openark–kit，工具包其中的oak-online-alter-table小工具是用来实现Online DDL的。
官网地址：[http://code.openark.org/forge/openark-kit ](http://code.openark.org/forge/openark-kit) 
google code在中国无法下载，国内下载地址：[http://pan.baidu.com/s/1pLpuC91 ](http://pan.baidu.com/s/1pLpuC91)

下载OAK工具包后解压
```perl
tar xvpf openark-kit-196.tar.gz
cd openark-kit-196/scripts
ll
total 240
-rw-r--r-- 1 1000 1000 11014 May  6  2013 oak-apply-ri
-rw-r--r-- 1 1000 1000 10993 May  6  2013 oak-block-account
-rw-r--r-- 1 1000 1000 23920 May  6  2013 oak-chunk-print
-rw-r--r-- 1 1000 1000 29484 May  6  2013 oak-chunk-update
-rw-r--r-- 1 1000 1000  5495 May  6  2013 oak-get-slave-lag
-rw-r--r-- 1 1000 1000 18175 May  6  2013 oak-hook-general-log
-rw-r--r-- 1 1000 1000  6308 May  6  2013 oak-kill-slow-queries
-rw-r--r-- 1 1000 1000  5624 May  6  2013 oak-modify-charset
-rw-r--r-- 1 1000 1000 39186 May  6  2013 oak-online-alter-table     # 这个就是用来实现Online DDL的
-rw-r--r-- 1 1000 1000  7803 May  6  2013 oak-prepare-shutdown
-rw-r--r-- 1 1000 1000 15039 May  6  2013 oak-purge-master-logs
-rw-r--r-- 1 1000 1000  8365 May  6  2013 oak-repeat-query
-rw-r--r-- 1 1000 1000 19306 May  6  2013 oak-security-audit
-rw-r--r-- 1 1000 1000  5904 May  6  2013 oak-show-limits
-rw-r--r-- 1 1000 1000  8648 May  6  2013 oak-show-replication-status

# 使用这个工具，需要安装MySQL-python包
yum -y install MySQL-python
```

### 用oak对表sbtest1做添加字段和索引的Online DDL
#### 确认该表是否符合oak-online-alter-table的执行条件
``` perl
# 检查是否是单列唯一索引（联合索引和联合主键是不可以的）
show create table sbtest1\G;

# 检查有无触发器
SELECT TRIGGER_SCHEMA,TRIGGER_NAME,EVENT_OBJECT_SCHEMA,EVENT_OBJECT_TABLE FROM information_schema.TRIGGERS
  WHERE event_object_schema= 'lyj';

# 检查有无外键
select * from information_schema.key_column_usage 
 where table_schema='lyj' and table_name='sbtest1' and referenced_table_name is not null;
select * from information_schema.key_column_usage
  where referenced_table_schema='lyj' and referenced_table_name='sbtest1';

# 要确保无大查询
```

#### 用OAK执行Online DDL脚本
``` python
python oak-online-alter-table -u root --ask-pass -S /u01/run/3306/mysql.sock -d lyj -t sbtest1 -g new_sbtest1 -a "add last_update_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,add key last_update_time(last_update_time)" --sleep=300 --skip-delete-pass
#----------------------------------------------------------------------------------------
-- Connecting to MySQL
Password:      # 输入root用户密码
-- Table lyj.sbtest1 is of engine innodb
-- Checking for UNIQUE columns on lyj.sbtest1, by which to chunk
-- Possible UNIQUE KEY column names in lyj.sbtest1:
-- - id
-- Table lyj.new_sbtest1 has been created
-- Table lyj.new_sbtest1 has been altered
-- Checking for UNIQUE columns on lyj.new_sbtest1, by which to chunk
-- Possible UNIQUE KEY column names in lyj.new_sbtest1:
-- - id
-- Checking for UNIQUE columns on lyj.sbtest1, by which to chunk
-- - Found following possible unique keys:
-- - id (int)
-- Chosen unique key is 'id'
-- Shared columns: c, pad, k, id
-- Created AD trigger    # 创建sbtest1 -> new_sbtest1的触发器
-- Created AU trigger
-- Created AI trigger
-- Attempting to lock tables

-- Tables locked WRITE
-- id (min, max) values: ([1L], [1000000L])
-- Tables unlocked
-- - Reminder: altering lyj.sbtest1: add last_update_time timestamp...
-- Copying range (1), (1000), progress: 0%
.......
-- + Will sleep for 0.3 seconds
-- Copying range 100% complete. Number of rows: 1000000
-- Ghost table creation completed. Note that triggers on lyj.sbtest1 were not removed
#----------------------------------------------------------------------------------------
```

#### 数据一致性校验
``` perl
# OAK在Online DDL进行前后，都可以正常对原表进行DML操作，完成后可使用以下命令进行数据一致性检验
select sum(crc32(concat(ifnull(id,'NULL'),ifnull(k,'NULL')))) as sum from sbtest1
union all
select sum(crc32(concat(ifnull(id,'NULL'),ifnull(k,'NULL')))) as sum from new_sbtest1;

select count(*) from sbtest1
union all
select count(*) from new_sbtest1;

desc sbtest1;
desc new_sbtest1;
+------------------+------------------+------+-----+-------------------+-----------------------------+
| Field            | Type             | Null | Key | Default           | Extra                       |
+------------------+------------------+------+-----+-------------------+-----------------------------+
| id               | int(10) unsigned | NO   | PRI | NULL              | auto_increment              |
| k                | int(10) unsigned | NO   | MUL | 0                 |                             |
| c                | char(120)        | NO   |     |                   |                             |
| pad              | char(60)         | NO   |     |                   |                             |
| last_update_time | timestamp        | NO   | MUL | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |   # 新增加的字段
+------------------+------------------+------+-----+-------------------+-----------------------------+

mysql> show create table new_sbtest1\G;
*************************** 1. row ***************************
       Table: new_sbtest1
Create Table: CREATE TABLE `new_sbtest1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  `last_update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `k_1` (`k`), 
  KEY `last_update_time` (`last_update_time`)     # 新增加的索引
) ENGINE=InnoDB AUTO_INCREMENT=1000001 DEFAULT CHARSET=utf8 MAX_ROWS=1000000
```

#### 表切换
``` perl
use lyj;
set names utf8;
rename table sbtest1 to old_sbtest1,new_sbtest1 to sbtest1;
drop trigger sbtest1_AI_oak;
drop trigger sbtest1_AU_oak;
drop trigger sbtest1_AD_oak;
drop table old_sbtest1;
```


## 用oak做Online DDL，如果rename那一步操作错了，在从库上rename了怎么恢复？
用oak做Online DDL，若上面表切换的步骤中错在从库上rename了，并且也进行了旧表删除和trigger删除，主库上也有新的数据插入，这时两个库的状态如下：
``` perl
# 主库
insert into sbtest1 values(3000000,3000000,'aaaaaa','aaaaaa');
insert into sbtest1 values(3000001,3000001,'aaaaaa','aaaaaa');
commit;

show tables;
+---------------+
| Tables_in_lyj |
+---------------+
| new_sbtest1   |
| sbtest1       |
| sbtest2       |
+---------------+

SELECT TRIGGER_SCHEMA,TRIGGER_NAME,EVENT_OBJECT_SCHEMA,EVENT_OBJECT_TABLE FROM information_schema.TRIGGERS
  WHERE event_object_schema= 'lyj';
+----------------+----------------+---------------------+--------------------+
| TRIGGER_SCHEMA | TRIGGER_NAME   | EVENT_OBJECT_SCHEMA | EVENT_OBJECT_TABLE |
+----------------+----------------+---------------------+--------------------+
| lyj            | sbtest1_AI_oak | lyj                 | sbtest1            |
| lyj            | sbtest1_AU_oak | lyj                 | sbtest1            |
| lyj            | sbtest1_AD_oak | lyj                 | sbtest1            |
+----------------+----------------+---------------------+--------------------+

select count(*) from sbtest1;
+----------+
| count(*) |
+----------+
|  1000002 |
+----------+

# 从库
show tables;
+---------------+
| Tables_in_lyj |
+---------------+
| sbtest1       |
| sbtest2       |
+---------------+

SELECT TRIGGER_SCHEMA,TRIGGER_NAME,EVENT_OBJECT_SCHEMA,EVENT_OBJECT_TABLE FROM information_schema.TRIGGERS
   WHERE event_object_schema= 'lyj';
Empty set (0.00 sec)

select count(*) from sbtest1;
+----------+
| count(*) |
+----------+
|  1000000 |
+----------+

desc sbtest1;
+------------------+------------------+------+-----+-------------------+-----------------------------+
| Field            | Type             | Null | Key | Default           | Extra                       |
+------------------+------------------+------+-----+-------------------+-----------------------------+
| id               | int(10) unsigned | NO   | PRI | NULL              | auto_increment              |
| k                | int(10) unsigned | NO   | MUL | 0                 |                             |
| c                | char(120)        | NO   |     |                   |                             |
| pad              | char(60)         | NO   |     |                   |                             |
| last_update_time | timestamp        | NO   | MUL | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
+------------------+------------------+------+-----+-------------------+-----------------------------+

# slave状态
show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.245.231.202
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000019
          Read_Master_Log_Pos: 385847848
               Relay_Log_File: relaylog.000021
                Relay_Log_Pos: 385847426
        Relay_Master_Log_File: binlog.000019
             Slave_IO_Running: Yes
            Slave_SQL_Running: No            # 同步已经停止了
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 1146
                   Last_Error: Error executing row event: 'Table 'lyj.new_sbtest1' doesn't exist'  # 停掉同步的原因是没有lyj.new_sbtest1表
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 385847266
              Relay_Log_Space: 2764989916
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 1146
               Last_SQL_Error: Error executing row event: 'Table 'lyj.new_sbtest1' doesn't exist'
```

恢复步骤如下：
``` perl
# 在主库上查看trigger创建脚本
show create trigger sbtest1_AI_oak\G;
show create trigger sbtest1_AU_oak\G;
show create trigger sbtest1_AD_oak\G;

# 在从库用sbtest1重建new_sbtest1
create table new_sbtest1 as select * from sbtest1;

# 在从库上生成trigger(使用上面查到的创建脚本)
CREATE DEFINER=`root`@`localhost` TRIGGER lyj.sbtest1_AI_oak AFTER INSERT ON lyj.sbtest1
        FOR EACH ROW
            REPLACE INTO lyj.new_sbtest1 (`c`, `pad`, `k`, `id`) VALUES (NEW.`c`, NEW.`pad`, NEW.`k`, NEW.`id`);

DELIMITER //
CREATE TRIGGER lyj.sbtest1_AU_oak AFTER UPDATE ON lyj.sbtest1
        FOR EACH ROW
        BEGIN
            DELETE FROM lyj.new_sbtest1 WHERE (id) = (OLD.id);
            REPLACE INTO lyj.new_sbtest1 (`c`, `pad`, `k`, `id`) VALUES (NEW.`c`, NEW.`pad`, NEW.`k`, NEW.`id`);
        END;
//
DELIMITER ;
## Mysql解释器一遇到;号时就结束，回车以后就执行，所以加delimiter关键字，delimiter作用就是把;分号替换成指定的符号。

CREATE DEFINER=`root`@`localhost` TRIGGER lyj.sbtest1_AD_oak AFTER DELETE ON lyj.sbtest1
        FOR EACH ROW
            DELETE FROM lyj.new_sbtest1 WHERE (id) = (OLD.id);

# 在从库上启动slave同步
start slave;

# 查看slave状态
show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.245.231.202
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000019
          Read_Master_Log_Pos: 385848190
               Relay_Log_File: relaylog.000021
                Relay_Log_Pos: 385848350
        Relay_Master_Log_File: binlog.000019
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes      # 可以看到同步已恢复

# 可以在主库上插入数据测试同步
insert into sbtest1 values(3000002,3000002,'aaaaaa','aaaaaa');
insert into sbtest1 values(3000003,3000003,'aaaaaa','aaaaaa');
commit;

# 重新在主库上执行表切换
use lyj;
set names utf8;
rename table sbtest1 to old_sbtest1,new_sbtest1 to sbtest1;
drop trigger sbtest1_AI_oak;
drop trigger sbtest1_AU_oak;
drop trigger sbtest1_AD_oak;
drop table old_sbtest1;
```

## 笔记
### MySQL 5.6 Online DDL概述
官方文档：[https://dev.mysql.com/doc/refman/5.6/en/innodb-create-index-overview.html ](https://dev.mysql.com/doc/refman/5.6/en/innodb-create-index-overview.html)
Online DDL主要是解决DDL锁表的问题，保证了在进行表变更的时候不会阻塞线上业务的读写，数据库仍能正常对外提供访问。

### OnlineDDL SOP
1. 确认该表是否符合oak-online-alter-table 的执行条件
  - 单列唯一索引（联合索引和联合主键是不可以的，促发mysql一个bug）
  - 没有foreign key
  - 没有定义触发器（有也先删除了）
2. 已有触发器？检查触发器 -> 备份触发器 -> 删除触发器
3. 执行oak 命令，至于执行时间，如果表有1亿，大概要执行12 个小时，就需要在一周业务量少的时候执行
4. Online DDL之后要进行数据一致性校验：如果DDL改变了表字段类型，可能导致表数据变化
5. 表切换
6. 删除触发器
7. 删除表

### online DDL checklist功能
1. 检查是否有大查询
2. 检查是否有外键和触发器

### MySQL5.6 Online DDL可以做到DDL\DML\SELECT同时进行
官方文档：[https://dev.mysql.com/doc/refman/5.6/en/innodb-create-index-concurrency.html ](https://dev.mysql.com/doc/refman/5.6/en/innodb-create-index-concurrency.html)
- Locking Options for Online DDL
  - LOCK=DEFAULT
  - LOCK=NONE
  - LOCK=SHARED
  - LOCK=EXCLUSIVE

- Performance of In-Place versus Table-Copying DDL Operations
  - ALGORITHM=DEFAULT
  - ALGORITHM=INPLACE
  - ALGORITHM=COPY

### COPY与INPLACE的原理
#### copy方式
1. 新建带索引（主键索引）的临时表
2. 锁原表，禁止DML，允许查询
3. 将原表数据拷贝到临时表
4. 禁止读写,进行rename，升级字典锁
5. 完成创建索引操作

#### inplace方式
1. 创建索引(二级索引)数据字典
2. 加共享表锁，禁止DML，允许查询
3. 读取聚簇索引，构造新的索引项，排序并插入新索引
4. 等待打开当前表的所有只读事务提交
5. 创建索引结束

### Online DDL 实现细节
#### Prepare阶段
1. 创建临时frm文件
2. 持有EXCLUSIVE-MDL锁，禁止读写  （不能有长查询）
3. 根据ALTER类型，确定执行方式(copy,online-rebuild,online-norebuild)
4. 更新数据字典的内存对象
5. 分配row_log对象记录增量，大小设置参数innodb_online_alter_log_max_size
6. 生成临时ibd文件

```perl
show variables like '%log_max_size%';
+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| innodb_online_alter_log_max_size | 134217728 |    # ddl的过程中，记录DML操作日志，默认大小128M
+----------------------------------+-----------+
```

#### ddl执行阶段
1. 降级EXCLUSIVE-MDL锁，允许读写
2. 扫描原表的聚簇索引每条记录
3. 遍历新表的聚簇索引和二级索引，逐一处理
4. 根据记录构造对应的索引项
5. 将构造索引项插入sort_buffer块
6. 将sort_buffer块插入新的索引
7. 处理ddl执行过程中产生的增量(仅rebuild类型需要)

#### commit阶段
1. 升级到EXCLUSIVE-MDL锁，禁止读写
2. 应用最后row_log中产的日志
3. 更新innodb的数据字典表
4. 提交事务(刷事务的redo日志)
5. 修改统计信息
6. rename临时idb文件，frm文件
7. 变更完成


### Mysql的Online DDL测试
```perl
# 用sysbench工具准备测试环境
sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/insert.lua \
--oltp-table-size=1000000 --mysql-table-engine=innodb --db-driver=mysql \
--mysql-user=root --mysql-password=root123 --mysql-port=3306 \
--mysql-host=10.245.231.202 --mysql-db=lyj \
--events=0 --time=60 --oltp-tables-count=2 --report-interval=10 --threads=2 prepare

sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/insert.lua \
--oltp-table-size=1000000 --mysql-table-engine=innodb --db-driver=mysql \
--mysql-user=root --mysql-password=root123 --mysql-port=3306 \
--mysql-host=10.245.231.202 --mysql-db=lyj \
--events=0 --time=60 --oltp-tables-count=2 --report-interval=10 --threads=2 run

# session 1:
alter table sbtest1 add name varchar(10);
# session 2:
delete from sbtest1 where id<1000;
Query OK, 999 rows affected (0.02 sec)    # 可以并发执行DML操作，执行速度很快

# session 1:
alter table sbtest2 add name varchar(10),ALGORITHM=INPLACE,LOCK=NONE;
# session 2:
delete from sbtest2 where id<1000;
Query OK, 999 rows affected (0.93 sec)    # 可以并发执行DML操作，执行速度较上面的慢些
```

### 参考
[mysql 5.6 原生Online DDL解析](http://seanlook.com/2016/05/24/mysql-online-ddl-concept/)