---
title: MySQL DBA从小白到大神实战-06 深入浅出MySQL备份与恢复
tags:
- mysql
---

## 使用mydumper工具全库备份
>mydumper是针对mysql数据库备份的一个轻量级第三方的开源工具，备份方式采用逻辑备份。
mydumper支持多线程，备份速度远高于原生态的mysqldump。

### 下载mydumper
下载地址：[https://launchpad.net/mydumper ](https://launchpad.net/mydumper)
<!-- more -->

### 编译安装
``` perl
# 安装编译所需的依赖包(参照：https://answers.launchpad.net/mydumper/+faq/349)
yum install -y glib2-devel mysql-devel zlib-devel pcre-devel

# 将下载的mydumper-0.9.1.tar.gz上传到服务器/tmp目录下，root用户执行以下命令
cd /tmp
tar -xvpf mydumper-0.9.1.tar.gz
cd mydumper-0.9.1
cmake .
make && make install
# ------------------------------------------------------------------------------------------
Scanning dependencies of target mydumper
[ 25%] Building C object CMakeFiles/mydumper.dir/mydumper.c.o
[ 50%] Building C object CMakeFiles/mydumper.dir/server_detect.c.o
[ 75%] Building C object CMakeFiles/mydumper.dir/g_unix_signal.c.o
Linking C executable mydumper
[ 75%] Built target mydumper
Scanning dependencies of target myloader
[100%] Building C object CMakeFiles/myloader.dir/myloader.c.o
Linking C executable myloader
[100%] Built target myloader
[ 75%] Built target mydumper
[100%] Built target myloader
Install the project...
-- Install configuration: ""
-- Installing: /usr/local/bin/mydumper
-- Removed runtime path from "/usr/local/bin/mydumper"
-- Installing: /usr/local/bin/myloader
-- Removed runtime path from "/usr/local/bin/myloader"
# ------------------------------------------------------------------------------------------
```

### 使用mydumper备份全库
``` perl
mkdir -p /u01/backup

mydumper \
    --user=root \
    --password='' \
    --socket=/u01/run/3306/mysql.sock \
    --regex '^(?!(mysql))' \
    --outputdir=/u01/backup/ \
    --compress \
    --verbose=3 \
    --logfile=/u01/backup/mydumper.log
# --regex '^(?!(mysql))' 这个正则表达式的意思是除了mysql数据库，其他数据库都备份
```

查看备份结果
``` perl
cd /u01/backup/
ll
total 20
-rw-rw-r-- 1 mysql mysql  84 Feb 28 16:43 jfedu-schema-create.sql.gz
-rw-rw-r-- 1 mysql mysql 174 Feb 28 16:43 jfedu.t1-schema.sql.gz
-rw-rw-r-- 1 mysql mysql 153 Feb 28 16:43 jfedu.t1.sql.gz
-rw-rw-r-- 1 mysql mysql 134 Feb 28 16:43 metadata
-rw-rw-r-- 1 mysql mysql 969 Feb 28 16:43 mydumper.log

# 查看备份日志，可以看出备份开启了多线程
vi mydumper.log 
2017-02-28 16:43:47 [INFO] - Connected to a MySQL server
2017-02-28 16:43:47 [INFO] - Started dump at: 2017-02-28 16:43:47

2017-02-28 16:43:47 [INFO] - Written master status
2017-02-28 16:43:47 [INFO] - Thread 1 connected using MySQL connection ID 20
2017-02-28 16:43:47 [INFO] - Thread 2 connected using MySQL connection ID 21
2017-02-28 16:43:47 [INFO] - Thread 3 connected using MySQL connection ID 22
2017-02-28 16:43:47 [INFO] - Thread 4 connected using MySQL connection ID 23
2017-02-28 16:43:47 [INFO] - Non-InnoDB dump complete, unlocking tables
2017-02-28 16:43:47 [INFO] - Thread 1 dumping data for `jfedu`.`t1`
2017-02-28 16:43:47 [INFO] - Thread 2 dumping schema for `jfedu`.`t1`
2017-02-28 16:43:47 [INFO] - Thread 3 shutting down
2017-02-28 16:43:47 [INFO] - Thread 4 shutting down
2017-02-28 16:43:47 [INFO] - Thread 1 shutting down
2017-02-28 16:43:47 [INFO] - Thread 2 shutting down
2017-02-28 16:43:47 [INFO] - Finished dump at: 2017-02-28 16:43:47
```


## 误操作truncate table gyj_t1;利用mysqldump的备份和binlog日志对表gyj_t1做完全恢复。

### 测试场景构建
``` perl
use jfedu;
create table gyj_t1(id int,name varchar(10));
insert into gyj_t1 values(1,'AAAAA');
commit;
```

### 使用mysqldump全库备份 
``` perl
mysqldump -h127.0.0.1 --single-transaction --master-data=2 -P3306 -A > /tmp/all_database_20170302.sql
```

### 备份后DML操作再truncate
``` perl
insert into gyj_t1 values(2,'BBBBBB');
commit;
truncate table gyj_t1;
```

### 完全恢复表并验证数据
``` perl
# 从备份文件中找出需要恢复表的建表语句：
cat /tmp/all_database_20170302.sql | sed -e '/./{H;$!d;}' -e 'x;/CREATE TABLE `gyj_t1`/!d;q' 

# 从备份文件中找出需要恢复表的数据
cat /tmp/all_database_20170302.sql | grep --ignore-case  'insert into `gyj_t1`'

# 恢复之前需要确认是否设置自动提交，若不是恢复前先执行以下命令修改
set autocommit=1;

# 因为是truncate表，表结构不需要恢复，只需要恢复数据即可
cat /tmp/all_database_20170302.sql | grep --ignore-case  'insert into `gyj_t1`' | mysql -uroot -p jfedu

# 查看恢复的数据
select * from gyj_t1;
+------+-------+
| id   | name  |
+------+-------+
|    1 | AAAAA |
+------+-------+

# 查看备份时binlog位置
grep MASTER /tmp/all_database_20170302.sql
-- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000030', MASTER_LOG_POS=1957;

# 找出误操作语句的位置
mysql --socket=/u01/run/3306/mysql.sock -e "show binlog events in 'binlog.000030'" |grep -i truncate
binlog.000030   2161    Query   101     2250    use `jfedu`; truncate table gyj_t1

# 用mysqlbinlog命令在binlog中找出相关记录
mysqlbinlog -v --base64-output=decode-rows /u01/log/3306/binlog/binlog.000030 > /tmp/30.sql
vi /tmp/30.sql
# ------------------------------------------------------------------------------------------
......
#170302 13:46:20 server id 101  end_log_pos 1957 CRC32 0xfb6ea7a5       Xid = 512
COMMIT/*!*/;
# at 1957
#170302 13:49:53 server id 101  end_log_pos 2030 CRC32 0x27b2b661       Query   thread_id=4     exec_time=0     error_code=0
SET TIMESTAMP=1488433793/*!*/;
SET @@session.sql_mode=1073741824/*!*/;
BEGIN
/*!*/;
# at 2030
#170302 13:49:53 server id 101  end_log_pos 2083 CRC32 0x5e3da140       Table_map: `jfedu`.`gyj_t1` mapped to number 109
# at 2083
#170302 13:49:53 server id 101  end_log_pos 2130 CRC32 0xbb45eb36       Write_rows: table id 109 flags: STMT_END_F
### INSERT INTO `jfedu`.`gyj_t1`
### SET
###   @1=2
###   @2='BBBBBB'
# at 2130
#170302 13:49:53 server id 101  end_log_pos 2161 CRC32 0x54d510bc       Xid = 950
COMMIT/*!*/;    # 这个就是truancate前最后操作的位置
# at 2161
#170302 13:50:23 server id 101  end_log_pos 2250 CRC32 0xc37cd27b       Query   thread_id=4     exec_time=0     error_code=0
SET TIMESTAMP=1488433823/*!*/;
truncate table gyj_t1
/*!*/;
# at 2250
......
# ------------------------------------------------------------------------------------------

# 使用mysqlbinlog恢复从备份到
mysqlbinlog --start-position=1957 --stop-position=2161 /u01/log/3306/binlog/binlog.000030 | mysql -uroot -p

select * from gyj_t1;
+------+--------+
| id   | name   |
+------+--------+
|    1 | AAAAA  |
|    2 | BBBBBB |
+------+--------+
```

## 利用Innobackupex的备份和binlog日志对MySQL数据库做完全恢复
>`Xtrabackup`是一个对InnoDB做数据备份的工具，支持在线热备份（备份时不影响数据读写），是商业备份工具InnoDB Hotbackup的一个很好的替代品。Xtrabackup有两个主要的工具：`xtrabackup`、`innobackupex`。
（1）xtrabackup只能备份InnoDB和XtraDB两种数据表，而不能备份MyISAM数据表
（2）innobackupex是用perl脚本封装了xtrabackup，能同时备份处理innodb和myisam，但在处理myisam时需要加一个读锁
（3）相关帮助文档：[https://www.percona.com/docs/wiki/index.html ](https://www.percona.com/docs/wiki/index.html)

### 下载安装Xtrabackup（二进制）
[https://www.percona.com/downloads/XtraBackup/LATEST/ ](https://www.percona.com/downloads/XtraBackup/LATEST/)
![下载Xtrabackup](http://oligvdnzp.bkt.clouddn.com/0301_xtrabackup_01.png)

上传并解压缩安装软件包
``` perl
cd /tmp
tar -xvpf percona-xtrabackup-2.4.6-Linux-x86_64.tar.gz
cd /tmp/percona-xtrabackup-2.4.6-Linux-x86_64/bin
cp * /usr/local/bin/
cd /tmp/percona-xtrabackup-2.4.6-Linux-x86_64/man/man1
cp * /usr/share/man/man1

# 可以使用以下命令查看帮助
xtrabackup --help
innobackupex --help
man xtrabackup
man innobackupex
```

### 测试场景构建
``` perl
create database lyj;
use lyj
create table t1(id int,name varchar(10));
insert into t1 values(1,'AAAAA');
insert into t1 select * from t1;
......
commit;
select count(*) from t1;
+----------+
| count(*) |
+----------+
|       64 |
+----------+
```

### 使用Innobackupex备份全库
``` perl
mkdir -p /u01/backup

innobackupex \
  --defaults-file=/u01/conf/my3306.cnf \
  --user=root \
  --password='' \
  --socket=/u01/run/3306/mysql.sock \
  --no-timestamp \
  /u01/backup/xtrabackup_20170302
```

### 应用备份期间日志
``` perl
innobackupex \
  --defaults-file=/u01/backup/xtrabackup_20170302/backup-my.cnf \
  --apply-log \
  --user=root \
  --password='' \
  /u01/backup/xtrabackup_20170302
# --defaults-file 配置文件参数必须放在第一位，否则会报错
# --apply-log 应用备份期间日志 
```

### 查看innobackupex备份结果
``` perl
cd /u01/backup/xtrabackup_20170302
ll
total 4165684
-rw-r----- 1 mysql mysql        434 Mar  2 10:23 backup-my.cnf
-rw-r----- 1 mysql mysql   33554432 Mar  2 10:32 ibdata1
-rw-r----- 1 mysql mysql   16777216 Mar  2 10:23 ibdata2
-rw-r----- 1 mysql mysql 1048576000 Mar  2 10:32 ib_logfile0
-rw-r----- 1 mysql mysql 1048576000 Mar  2 10:31 ib_logfile1
-rw-r----- 1 mysql mysql 1048576000 Mar  2 10:32 ib_logfile2
-rw-r----- 1 mysql mysql 1048576000 Mar  2 10:32 ib_logfile3
-rw-r----- 1 mysql mysql   12582912 Mar  2 10:32 ibtmp1
drwxr-x--- 2 mysql mysql       4096 Mar  2 10:23 jfedu
drwxr-x--- 2 mysql mysql       4096 Mar  2 10:23 lyj
drwxr-x--- 2 mysql mysql       4096 Mar  2 10:23 mysql
drwxr-x--- 2 mysql mysql       4096 Mar  2 10:23 performance_schema
-rw-r----- 1 mysql mysql         20 Mar  2 10:23 xtrabackup_binlog_info
-rw-rw-r-- 1 mysql mysql         20 Mar  2 10:31 xtrabackup_binlog_pos_innodb
-rw-r----- 1 mysql mysql        113 Mar  2 10:31 xtrabackup_checkpoints        # 检查点
-rw-r----- 1 mysql mysql        572 Mar  2 10:23 xtrabackup_info
-rw-r----- 1 mysql mysql    8388608 Mar  2 10:31 xtrabackup_logfile            # log日志
```

### 备份后DML操作
``` perl
insert into t1 select * from t1;
......
commit;
select count(*) from t1;
+----------+
| count(*) |
+----------+
|     4096 |
+----------+
```

### 摸拟rm误操作
``` perl
rm -rf /u01/data/3306/*
# 这时关闭数据库已不能正常启动了
```

### 拷备份数据到数据库目录
``` perl
cp -rf /u01/backup/xtrabackup_20170302/* /u01/data/3306/
# 注意文件权限，如果不是mysql，使用以下语句修改，xtrabackup_* 可以不拷贝
chown -R mysql:mysql /u01/data/3306/
```

### 登陆数据库验证备份恢复
``` perl
# 启动数据库
use lyj
select count(*) from t1; 
+----------+
| count(*) |
+----------+
|       64 |
+----------+
# 恢复了备份时（包括备份期间）的64条记录
```

### 使用mysqlbinlog完全恢复
``` perl
cat /u01/backup/xtrabackup_20170302/xtrabackup_binlog_info
binlog.000027   28905477

mysqlbinlog --start-position=28905477 /u01/log/3306/binlog/binlog.000027 | mysql -uroot -p

# 登陆数据库
select count(*) from t1;
+----------+
| count(*) |
+----------+
|     4096 |
+----------+
```

## 随堂笔记

### mysqldump常用参数及使用示例
``` perl
# 查看帮助
mysqldump --help

# 一些常用参数
--single-transaction  # 备份执行期间不阻塞DML，在生产环境备份时一定要加此参数
-A, --all-databases Dump all the databases. This will be same as --databases # 备份所有的数据库
--master-data[=#]   # --master-data=2 或 --master-data=1，一般做主从的时侯需要加此参数
--add-drop-database Add a DROP DATABASE before each create. # 备份中不生成创建数据库命令
--add-drop-table    Add a DROP TABLE before each create.    # 备份中不生成创建表命令

# 示例
mysqldump -h127.0.0.1 --single-transaction -P3306 -A > /tmp/all_database.sql
mysqldump -h127.0.0.1 -P3306 -uroot -p --single-transaction --master-data=2 jfedu > /tmp/jfedu.sql
```

### 分析mysqldump的执行流程
``` perl
# 打开general.log（通常是关闭的）
show variables like '%gen%';
+------------------+--------------------------+
| Variable_name    | Value                    |
+------------------+--------------------------+
| general_log      | OFF                      |
| general_log_file | /u01/data/3306/mysql.log |
+------------------+--------------------------+

set global general_log=1;

# 在新窗口追踪日志
tail -f /u01/data/3306/mysql.log

# 使用mysqldump备份jfedu数据库
mysqldump -h127.0.0.1 -P3306 -uroot -p --single-transaction --master-data=2 jfedu > /tmp/jfedu.sql

# 分析general.log，下面这段就是追踪到的mysqldump执行流程
# ------------------------------------------------------------------------------------------
170228 15:39:29     9 Connect   root@127.0.0.1 on 
                    9 Query     /*!40100 SET @@SQL_MODE='' */
                    9 Query     /*!40103 SET TIME_ZONE='+00:00' */
                    9 Query     FLUSH /*!40101 LOCAL */ TABLES
                    9 Query     FLUSH TABLES WITH READ LOCK      # 数据库只读锁定命令
                    9 Query     SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
                    9 Query     START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
                    9 Query     SHOW VARIABLES LIKE 'gtid\_mode'
                    9 Query     SHOW MASTER STATUS
                    9 Query     UNLOCK TABLES
                    9 Query     SELECT LOGFILE_GROUP_NAME, FILE_NAME, TOTAL_EXTENTS, INITIAL_SIZE, ENGINE, EXTRA FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'UNDO LOG' AND FILE_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IN (SELECT DISTINCT LOGFILE_GROUP_NAME FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('jfedu'))) GROUP BY LOGFILE_GROUP_NAME, FILE_NAME, ENGINE ORDER BY LOGFILE_GROUP_NAME
                    9 Query     SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('jfedu')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME
                    9 Query     SHOW VARIABLES LIKE 'ndbinfo\_version'
                    9 Init DB   jfedu
                    9 Query     SAVEPOINT sp
                    9 Query     show tables
                    9 Query     show table status like 't1'
                    9 Query     SET SQL_QUOTE_SHOW_CREATE=1
                    9 Query     SET SESSION character_set_results = 'binary'
                    9 Query     show create table `t1`
                    9 Query     SET SESSION character_set_results = 'utf8'
                    9 Query     show fields from `t1`
                    9 Query     SELECT /*!40001 SQL_NO_CACHE */ * FROM `t1`
                    9 Query     SET SESSION character_set_results = 'binary'
                    9 Query     use `jfedu`
                    9 Query     select @@collation_database
                    9 Query     SHOW TRIGGERS LIKE 't1'
                    9 Query     SET SESSION character_set_results = 'utf8'
                    9 Query     ROLLBACK TO SAVEPOINT sp
                    9 Query     RELEASE SAVEPOINT sp
                    9 Quit
# ------------------------------------------------------------------------------------------

# 关闭general.log
set global general_log=0;
```

### 修改隔离级别
``` perl
show variables like '%iso%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+

set global tx_isolation='read-committed';   # 重启mysql失效
set session tx_isolation='read-committed';  # 只在当前会话中有效

# 修改my.cnf永久有效
vi /etc/my.cnf
#-------------------------------------
transaction_isolation=read-committed
#-------------------------------------

show variables like '%iso%';
+---------------+----------------+
| Variable_name | Value          |
+---------------+----------------+
| tx_isolation  | READ-COMMITTED |
+---------------+----------------+
```

### 查看mydumper帮助
``` 
su - mysql
mydumper --help

Usage:
  mydumper [OPTION...] multi-threaded MySQL dumping

Help Options:
  -?, --help                  Show help options

Application Options:
  -B, --database              Database to dump
  -T, --tables-list           Comma delimited table list to dump (does not exclude regex option)
  -o, --outputdir             Directory to output files to
  -s, --statement-size        Attempted size of INSERT statement in bytes, default 1000000
  -r, --rows                  Try to split tables into chunks of this many rows. This option turns off --chunk-filesize
  -F, --chunk-filesize        Split tables into chunks of this output file size. This value is in MB
  -c, --compress              Compress output files
  -e, --build-empty-files     Build dump files even if no data available from table
  -x, --regex                 Regular expression for 'db.table' matching
  -i, --ignore-engines        Comma delimited list of storage engines to ignore
  -m, --no-schemas            Do not dump table schemas with the data
  -d, --no-data               Do not dump table data
  -G, --triggers              Dump triggers
  -E, --events                Dump events
  -R, --routines              Dump stored procedures and functions
  -k, --no-locks              Do not execute the temporary shared read lock.  WARNING: This will cause inconsistent backups
  --less-locking              Minimize locking time on InnoDB tables.
  -l, --long-query-guard      Set long query timer in seconds, default 60
  -K, --kill-long-queries     Kill long running queries (instead of aborting)
  -D, --daemon                Enable daemon mode
  -I, --snapshot-interval     Interval between each dump snapshot (in minutes), requires --daemon, default 60
  -L, --logfile               Log file name to use, by default stdout is used
  --tz-utc                    SET TIME_ZONE='+00:00' at top of dump to allow dumping of TIMESTAMP data when a server has data in different time zones or data is being moved between servers with different time zones, defaults to on use --skip-tz-utc to disable.
  --skip-tz-utc               
  --use-savepoints            Use savepoints to reduce metadata locking issues, needs SUPER privilege
  --success-on-1146           Not increment error count and Warning instead of Critical in case of table doesn't exist
  --lock-all-tables           Use LOCK TABLE for all, instead of FTWRL
  -U, --updated-since         Use Update_time to dump only tables updated in the last U days
  --trx-consistency-only      Transactional consistency only
  -h, --host                  The host to connect to
  -u, --user                  Username with privileges to run the dump
  -p, --password              User password
  -P, --port                  TCP/IP port to connect to
  -S, --socket                UNIX domain socket file to use for connection
  -t, --threads               Number of threads to use, default 4
  -C, --compress-protocol     Use compression on the MySQL connection
  -V, --version               Show the program version and exit
  -v, --verbose               Verbosity of output, 0 = silent, 1 = errors, 2 = warnings, 3 = info, default 2
```
