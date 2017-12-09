---
title: MySQL DBA从小白到大神实战-05 MySQL DBA日常操作
date: 2017-02-24
tags:
- mysql
categories:
- MySQL DBA从小白到大神实战
---

## 使用mysqld_mutil start命令启动多实例3306、3307，使用mysqld_multi report命令显示结果。

### 规划多实例目录
``` perl
mkdir -p /u01/mysql             # 程序目录
mkdir -p /u01/conf              # 配置文件
mkdir -p /u01/data/3306         
mkdir -p /u01/data/3307
mkdir -p /u01/log/3306/iblog
mkdir -p /u01/log/3307/iblog
mkdir -p /u01/log/3306/binlog
mkdir -p /u01/log/3307/binlog
mkdir -p /u01/run/3306
mkdir -p /u01/run/3307
mkdir -p /u01/tmp/3306
mkdir -p /u01/tmp/3307

chown -R mysql:mysql /u01
chmod -R 755 /u01
```

<!--more-->

### 重新编译安装mysql
``` perl
cd /u01/mysql-5.6.35
rm -rf CMakeCache.txt

cmake \
-DCMAKE_INSTALL_PREFIX=/u01/mysql \
-DINSTALL_DATADIR=/u01/data/3306  \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DEXTRA_CHARSETS=all \
-DWITH_SSL=yes \
-DWITH_EMBEDDED_SERVER=1 \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_MYISAM_STORAGE_ENGINE=1 \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_ARCHIVE_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_PARTITION_STORAGE_ENGINE=1 \
-DMYSQL_UNIX_ADDR=/u01/run/3306/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DENABLED_LOCAL_INFILE=1 \
-DSYSCONFDIR=/etc \
-DWITH_READLINE=on

make && make install
```

### 配置多实例独自参数文件
``` perl
cd /u01/conf
vi my3306.cnf
----------------------------------------------------------
[client]
port=3306
socket=/u01/run/3306/mysql.sock

[mysql]
pid_file=/u01/run/3306/mysqld.pid

[mysqld]
autocommit=1
general_log=off
explicit_defaults_for_timestamp=true

# system
basedir=/u01/mysql
datadir=/u01/data/3306
max_allowed_packet=1g
max_connections=3000
max_user_connections=2800
open_files_limit=65535
pid_file=/u01/run/3306/mysqld.pid
port=3306
server_id=3306
skip_name_resolve=ON
socket=/u01/run/3306/mysql.sock
tmpdir=/u01/tmp/3306

#binlog
log_bin=/u01/log/3306/binlog/binlog
binlog_cache_size=32768
binlog_format=row
expire_logs_days=7
log_slave_updates=ON
max_binlog_cache_size=2147483648
max_binlog_size=524288000
sync_binlog=100

#logging
log_error=/u01/log/3306/error.log
slow_query_log_file=/u01/log/3306/slow.log
log_queries_not_using_indexes=0
slow_query_log=1
log_slave_updates=ON
log_slow_admin_statements=1
long_query_time=1

#relay
relay_log=/u01/log/3306/relaylog
relay_log_index=/u01/log/3306/relay.index
relay_log_info_file=/u01/log/3306/relay-log.info

#slave
slave_load_tmpdir=/u01/tmp/3306
slave_skip_errors=OFF

#innodb
innodb_data_home_dir=/u01/log/3306/iblog
innodb_log_group_home_dir=/u01/log/3306/iblog
innodb_adaptive_flushing=ON
innodb_adaptive_hash_index=ON
innodb_autoinc_lock_mode=1
innodb_buffer_pool_instances=8

#default
innodb_change_buffering=inserts
innodb_checksums=ON
innodb_buffer_pool_size= 128M
innodb_data_file_path=ibdata1:32M;ibdata2:16M:autoextend
innodb_doublewrite=ON
innodb_file_format=Barracuda
innodb_file_per_table=ON
innodb_flush_log_at_trx_commit=1
innodb_flush_method=O_DIRECT
innodb_io_capacity=1000
innodb_lock_wait_timeout=10
innodb_log_buffer_size=67108864
innodb_log_file_size=1048576000
innodb_log_files_in_group=4
innodb_max_dirty_pages_pct=60
innodb_open_files=60000
innodb_purge_threads=1
innodb_read_io_threads=4
innodb_stats_on_metadata=OFF
innodb_support_xa=ON
innodb_use_native_aio=OFF
innodb_write_io_threads=10

[mysqld_safe]
datadir=/u01/data/3306
----------------------------------------------------------

# 将上面内容中3306替换成3307，server编辑保存my3307.cnf
vi my3307.cnf
```

### 初始化多实例数据库
``` perl
su - mysql

# 修改环境变量
vi /home/mysql/.bash_profile
----------------------------------------------------------
PATH=$PATH:$HOME/bin:/u01/mysql/bin
----------------------------------------------------------

# 初始化数据库实例
cd /u01/mysql/scripts
./mysql_install_db --defaults-file=/u01/conf/my3306.cnf \
--datadir=/u01/data/3306 --basedir=/u01/mysql --user=mysql

./mysql_install_db --defaults-file=/u01/conf/my3307.cnf \
--datadir=/u01/data/3307 --basedir=/u01/mysql --user=mysql

# 使用独立参数启动实例
mysqld_safe --defaults-file=/u01/conf/my3306.cnf &
mysqld_safe --defaults-file=/u01/conf/my3307.cnf &

# 登陆3306实例
mysql --socket=/u01/run/3306/mysql.sock

show variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3306  |
+---------------+-------+

# 登陆3307实例
mysql --socket=/u01/run/3307/mysql.sock

show variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3307  |
+---------------+-------+

# 关闭实例
mysqladmin --socket=/u01/run/3306/mysql.sock shutdown &
mysqladmin --socket=/u01/run/3307/mysql.sock shutdown &
```

### 修改/etc/my.cnf参数
``` perl
# root用户下操作
vi /etc/my.cnf
----------------------------------------------------------
[mysqld_multi]
    mysqld=/u01/mysql/bin/mysqld_safe
    mysqladmin=/u01/mysql/bin/mysqladmin
    user=root
    log=/u01/log/multi.log

[mysqld3306]
    port=3306
    basedir=/u01/mysql
    datadir=/u01/data/3306
    socket=/u01/run/3306/mysql.sock
    server-id=3306
    log_bin=/u01/log/3306/binlog/binlog
    slow_query_log_file=/u01/log/3306/slow.log
    innodb_data_home_dir=/u01/log/3306/iblog
    innodb_log_group_home_dir=/u01/log/3306/iblog
    log_error=/u01/log/3306/error.log
    pid_file=/u01/run/3306/mysqld.pid
    tmpdir=/u01/tmp/3306
    relay_log=/u01/log/3306/relaylog
    relay_log_index=/u01/log/3306/relay.index
    relay_log_info_file=/u01/log/3306/relay-log.info
    slave_load_tmpdir=/u01/tmp/3306

[mysqld3307]
    port=3307
    basedir=/u01/mysql
    datadir=/u01/data/3307
    socket=/u01/run/3307/mysql.sock
    server-id=3307
    log_bin=/u01/log/3307/binlog/binlog
    slow_query_log_file=/u01/log/3307/slow.log
    innodb_data_home_dir=/u01/log/3307/iblog
    innodb_log_group_home_dir=/u01/log/3307/iblog
    log_error=/u01/log/3307/error.log
    pid_file=/u01/run/3307/mysqld.pid
    tmpdir=/u01/tmp/3307
    relay_log=/u01/log/3307/relaylog
    relay_log_index=/u01/log/3307/relay.index
    relay_log_info_file=/u01/log/3307/relay-log.info
    slave_load_tmpdir=/u01/tmp/3307

[client]
    port=3306
    socket =/u01/run/3306/mysql.sock

[mysqld]
    autocommit=1
    general_log=off
    explicit_defaults_for_timestamp=true
    max_allowed_packet=1g
    max_connections=3000
    max_user_connections=2800
    open_files_limit=65535
    skip_name_resolve=ON

    #binlog
    binlog_cache_size=32768
    binlog_format=row
    expire_logs_days=7
    log_slave_updates=ON
    max_binlog_cache_size=2147483648
    max_binlog_size=524288000
    sync_binlog=100

    log_queries_not_using_indexes=0
    slow_query_log=1
    log_slave_updates=ON
    log_slow_admin_statements=1
    long_query_time=1

    slave_skip_errors=OFF

    innodb_adaptive_flushing=ON
    innodb_adaptive_hash_index=ON
    innodb_autoinc_lock_mode=1
    innodb_buffer_pool_instances=8

    #default
    innodb_change_buffering=inserts
    innodb_checksums=ON
    innodb_buffer_pool_size= 128M
    innodb_data_file_path=ibdata1:32M;ibdata2:16M:autoextend
    innodb_doublewrite=ON
    innodb_file_format=Barracuda
    innodb_file_per_table=ON
    innodb_flush_log_at_trx_commit=1
    innodb_flush_method=O_DIRECT
    innodb_io_capacity=1000
    innodb_lock_wait_timeout=10
    innodb_log_buffer_size=67108864
    innodb_log_file_size=1048576000
    innodb_log_files_in_group=4
    innodb_max_dirty_pages_pct=60
    innodb_open_files=60000
    innodb_purge_threads=1
    innodb_read_io_threads=4
    innodb_stats_on_metadata=OFF
    innodb_support_xa=ON
    innodb_use_native_aio=OFF
    innodb_write_io_threads=10
----------------------------------------------------------
```

### mysqld_multi启动/关闭多实例，查看多实例运行状态
``` perl
# 启动多实例数据库
mysqld_multi start             # 启动全部配置的多实例
mysqld_multi start 3306        # 启动指定server-id的实例
mysqld_multi start 3306-3307   # 启动指定server-id连续的实例

# 查看多实例运行状态
mysqld_multi report
Reporting MySQL servers
MySQL server from group: mysqld3306 is running
MySQL server from group: mysqld3307 is running

# 关闭多实例数据库
mysqld_multi stop
mysqld_multi stop 3306
mysqld_multi stop 3306-3307
```

## 在线迁移MySQL 3306实例上的数据库jfedu到MySQL 3307上。

### 登陆3306实例，创建jfedu数据库
``` perl
# 登陆3306实例
mysql -S /u01/run/3306/mysql.sock

# 查看port和server_id
show variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3306  |
+---------------+-------+

show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3306  |
+---------------+-------+

# 创建jfedu数据库，创建表t1并插入数据
create database jfedu;
use jfedu;
create table t1(id int,name varchar(10));
insert into t1 values(1,'AAAAA');
commit;
```

### 在3306实例上创建一个复制帐号
``` perl
grant replication slave,replication client on *.* to 'repl'@'%' identified by 'repl4Slave';
```

### 备份jfedu数据库
``` perl
mysqldump --single-transaction --master-data=2 -uroot jfedu > /tmp/jfedu.sql
# --single-transaction 不锁表
# --master-data=2 在备份文件中记录master_log_file和master_log_pos的值

# 设置主从复制时会用到master_log_file和master_log_pos的值
cat /tmp/jfedu.sql | grep MASTER_LOG_FILE
-- CHANGE MASTER TO MASTER_LOG_FILE='binlog.000022', MASTER_LOG_POS=750;
```

### 在3307实例上恢复jfedu数据库
``` perl
# 登陆3306实例
mysql -S /u01/run/3307/mysql.sock

# 查看port和server_id
show variables like 'port';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| port          | 3307  |
+---------------+-------+

show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 3307  |
+---------------+-------+

# 在3307上创建jfedu数据库
create database jfedu;

# 恢复数据库
use jfedu;
source /tmp/jfedu.sql;
```

### 3306实例在恢复时有数据插入
``` perl
use jfedu
insert into t1 values(2,'BBBBB');
commit;
```

### 在3307实例上执行change master设置主从复制，并启动复制
``` perl
# 设置主从复制
change master to
master_host='127.0.0.1',
master_port=3306,
master_user='repl',
master_password='repl4Slave',
master_log_file='binlog.000022',
master_log_pos=750;

# 启动复制
start slave;

# 查看复制状态
show slave status\G;

*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 127.0.0.1
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000022
          Read_Master_Log_Pos: 949
               Relay_Log_File: relaylog.000002
                Relay_Log_Pos: 479
        Relay_Master_Log_File: binlog.000022
             Slave_IO_Running: Yes               # Yes表示复制正常
            Slave_SQL_Running: Yes               # Yes表示复制正常
......

# 观察3307上数据是否复制过来
select * from jfedu.t1;
+------+-------+
| id   | name  |
+------+-------+
|    1 | AAAAA |
|    2 | BBBBB |
+------+-------+
```

### 3306实例设置成read only
``` perl
show variables like 'read_only';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| read_only     | OFF   |
+---------------+-------+

set global read_only=1;
```

### 数据全部一致后，停止复制
``` perl
stop slave;
```

## MySQL5.6升级到MySQL5.7，正常登录到MySQL5.7，详细步骤。

### 下载MySQL5.7

下载地址：[https://dev.mysql.com/downloads/mysql ](https://dev.mysql.com/downloads/mysql)
![mysql5.7下载](http://oligvdnzp.bkt.clouddn.com/0224_mysql57_01.png)

### 上传并解压软件包
``` perl
[mysql@mysql ~]$ cd /u01
#选择下载的软件包上传
[mysql@mysql u01]$ rz
[mysql@mysql u01]$ ll
total 698668
drwxr-xr-x   2 mysql mysql      4096 Feb 24 11:16 conf
drwxr-xr-x   4 mysql mysql      4096 Feb 24 09:45 data
drwxr-xr-x   4 mysql mysql      4096 Feb 24 11:00 log
drwxr-xr-x  13 mysql mysql      4096 Feb 24 10:16 mysql
drwxr-xr-x  35 mysql mysql      4096 Feb 24 10:05 mysql-5.6.35
-rwxr-xr-x.  1 mysql mysql  32167628 Jan 17 11:16 mysql-5.6.35.tar.gz
-rw-r--r--   1 mysql mysql 683233280 Feb 24 16:36 mysql-5.7.17-linux-glibc2.5-x86_64.tar   # 这个就是二进制mysql5.7软件
drwxr-xr-x   4 mysql mysql      4096 Feb 24 09:50 run
drwxr-xr-x   4 mysql mysql      4096 Feb 24 09:45 tmp

tar -xpvf mysql-5.7.17-linux-glibc2.5-x86_64.tar
tar -xpvf mysql-5.7.17-linux-glibc2.5-x86_64.tar.gz

# 因为是二进制包，解压后就可以用了，文件夹名改短些
mv mysql-5.7.17-linux-glibc2.5-x86_64 mysql5.7

# 修改环境变量
vi /home/mysql/.bash_profile
PATH=$PATH:$HOME/bin:/u01/mysql5.7/bin
```

### 查看mysql版本信息
``` perl
mysql -V
--------------
mysql  Ver 14.14 Distrib 5.7.17, for linux-glibc2.5 (x86_64) using  EditLine wrapper
--------------

mysqld_multi start 3306
mysql -S /u01/run/3306/mysql.sock

status;
--------------
mysql  Ver 14.14 Distrib 5.7.17, for linux-glibc2.5 (x86_64) using  EditLine wrapper

Connection id:          7
Current database:
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         5.6.35-log Source distribution
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    utf8
Db     characterset:    utf8
Client characterset:    utf8
Conn.  characterset:    utf8
UNIX socket:            /u01/run/3306/mysql.sock
Uptime:                 3 min 57 sec

Threads: 2  Questions: 16  Slow queries: 0  Opens: 70  Flush tables: 1  Open tables: 63  Queries per second avg: 0.067
--------------
```

### 升级数据库字典
``` perl
mysql_upgrade -S /u01/run/3306/mysql.sock
--------------
Checking if update is needed.
Checking server version.
Error: Server version (5.6.35-log) does not match with the version of
the server (5.7.17) with which this program was built/distributed. You can
use --skip-version-check to skip this check.
--------------

mysql_upgrade -S /u01/run/3306/mysql.sock --skip-version-check
--------------
Checking if update is needed.
Running queries to upgrade MySQL server.
mysql_upgrade: [ERROR] 1726: Storage engine 'InnoDB' does not support system tables. [mysql.plugin]
--------------

# 重启下数据库
mysqld_multi stop
mysqld --defaults-file=/u01/conf/my3306.cnf &
mysql_upgrade -S /u01/run/3306/mysql.sock --skip-version-check
--------------
Checking if update is needed.
Running queries to upgrade MySQL server.
Checking system database.
mysql.columns_priv                                 OK
mysql.db                                           OK
mysql.engine_cost                                  OK
mysql.event                                        OK
mysql.func                                         OK
mysql.general_log                                  OK
mysql.gtid_executed                                OK
mysql.help_category                                OK
mysql.help_keyword                                 OK
mysql.help_relation                                OK
mysql.help_topic                                   OK
mysql.innodb_index_stats                           OK
mysql.innodb_table_stats                           OK
mysql.ndb_binlog_index                             OK
mysql.plugin                                       OK
......
Repairing tables
mysql_old.innodb_index_stats
Error    : Unknown error 1146
status   : Operation failed
mysql_old.innodb_table_stats
Error    : Unknown error 1146
status   : Operation failed
mysql_old.slave_master_info
Error    : Unknown error 1146
status   : Operation failed
mysql_old.slave_relay_log_info
Error    : Unknown error 1146
status   : Operation failed
mysql_old.slave_worker_info
Error    : Unknown error 1146
status   : Operation failed
Upgrade process completed successfully.
Checking if update is needed.
--------------
```

### 再查看mysql版本信息，升级完成
``` perl
mysql -S /u01/run/3306/mysql.sock

mysql> status;
--------------
mysql  Ver 14.14 Distrib 5.7.17, for linux-glibc2.5 (x86_64) using  EditLine wrapper

Connection id:          4
Current database:
Current user:           root@localhost
SSL:                    Not in use
Current pager:          stdout
Using outfile:          ''
Using delimiter:        ;
Server version:         5.7.17-log MySQL Community Server (GPL)
Protocol version:       10
Connection:             Localhost via UNIX socket
Server characterset:    latin1
Db     characterset:    latin1
Client characterset:    utf8
Conn.  characterset:    utf8
UNIX socket:            /u01/run/3306/mysql.sock
Uptime:                 6 min 3 sec

Threads: 1  Questions: 3174  Slow queries: 0  Opens: 368  Flush tables: 1  Open tables: 25  Queries per second avg: 8.743
--------------

mysql> select version();
+------------+
| version()  |
+------------+
| 5.7.17-log |
+------------+
```

## 课堂笔记整理

### MySQL启动
``` perl
# 推荐启动方式
mysqld_safe --defaults-file=/u01/conf/my3306.cnf &

# 编译安装可以使用的启动方式（很少用）
cd /u01/mysql/support-files/
./mysql.server start

# rpm包安装可以使用的启动方式
/etc/init.d/mysqld start
service mysqld start

# mysqld的启动方式
mysqld --defaults-file=/u01/conf/my3306.cnf &

# 多实例启动方式，需要先修改my.cnf参数，并放到/etc目录下（建议单个实例维护）
mysqld_mutil start

# 查看mysql启动时参数my.cnf查找顺序
mysqld --verbose --help | grep my.cnf

# mysqld_safe方式启动mysql后相关进程，mysqld_safe进程有监控mysqld进程的作用，mysqld异常终止时，mysqld_safe会重启mysqld
ps -ef | grep mysql | grep -v bash | grep -v grep | grep -v su | grep -v ps
mysql     5503 18586  0 16:33 pts/1    00:00:00 /bin/sh /u01/mysql/bin/mysqld_safe --defaults-file=/u01/conf/my3306.cnf
mysql     6342  5503  0 16:33 pts/1    00:00:00 /u01/mysql/bin/mysqld --defaults-file=/u01/conf/my3306.cnf --basedir=/u01/mysql --datadir=/u01/data/3306 --plugin-dir=/u01/mysql/lib/plugin --log-error=/u01/log/3306/error.log --open-files-limit=65535 --pid-file=/u01/run/3306/mysqld.pid --socket=/u01/run/3306/mysql.sock --port=3306

# mysqld方式启动mysql后相关进程
ps -ef | grep mysql | grep -v bash | grep -v grep | grep -v su | grep -v ps
mysql     6401 18586  2 16:36 pts/1    00:00:00 mysqld --defaults-file=/u01/conf/my3306.cnf
```

### MySQL关闭
``` perl
# 使用mysqld_safe和mysqld启动的关闭方式（推荐的方式）
mysqladmin shutdown
mysqladmin --socket=/u01/run/3306/mysql.sock shutdown &

# 编译安装可以使用的关闭方式
cd /u01/mysql/support-files/
./mysql.server stop

# rpm安装可以使用的关闭方式
/etc/init.d/mysqld stop
service mysqld stop

# 多实例关闭方式
mysqld_mutil stop

# kill进程
kill -9 pid
```

### MySQL登陆
``` perl
# 本地登陆
mysql
mysql -u$username -p$password

# 远程登陆
mysql -u$username -p$password -h$ip

# 多实例
mysql -u$username -p$password -P$port
mysql --socket=/u01/run/3306/mysql.sock --port=3306
```

### 帐户权限设置-创建/删除用户
``` perl
select user,host,password from mysql.user;
+------+-----------+----------+
| user | host      | password |
+------+-----------+----------+
| root | localhost |          |     # 密码都为空，不安全
| root | mysql     |          |
| root | 127.0.0.1 |          |
| root | ::1       |          |
|      | localhost |          |
|      | mysql     |          |
+------+-----------+----------+

# insert方式创建用户(用insert方式创建的用户，需要刷新缓存)
insert into mysql.user(user,host,password) values('mytt','127.0.0.1',password(123456));
flush privileges;

select user,host,password from mysql.user;
+------+-----------+-------------------------------------------+
| user | host      | password                                  |
+------+-----------+-------------------------------------------+
| root | localhost |                                           |
| root | mysql     |                                           |
| root | 127.0.0.1 |                                           |
| root | ::1       |                                           |
|      | localhost |                                           |
|      | mysql     |                                           |
| mytt | 127.0.0.1 | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
+------+-----------+-------------------------------------------+

# mytt登陆mysql命令如下
mysql -umytt -p123456 -h127.0.0.1

select user();
+----------------+
| user()         |
+----------------+
| mytt@127.0.0.1 |
+----------------+

# create方式创建用户
create user lyj@'%' identified by '123456';  # %代表可以从任意位置登陆
mysql -ulyj -p123456 -h127.0.0.1

# 删除用户
drop user lyj;
```

### 帐户权限设置-用户授权
``` perl
# 创建新用户并授权
grant all privileges on *.* to lyj@'%' identified by '123456';

# 查看用户权限
show grants for lyj@'%';
+-------------------------------------------------------------------------------------------------------------+
| Grants for lyj@%                                                                                            |
+-------------------------------------------------------------------------------------------------------------+
| GRANT ALL PRIVILEGES ON *.* TO 'lyj'@'%' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
+-------------------------------------------------------------------------------------------------------------+

# 给已有用户授权
grant select on *.* to mytt@127.0.0.1;
show grants for mytt@127.0.0.1;
+--------------------------------------------------------------------------------------------------------------+
| Grants for mytt@127.0.0.1                                                                                    |
+--------------------------------------------------------------------------------------------------------------+
| GRANT SELECT ON *.* TO 'mytt'@'127.0.0.1' IDENTIFIED BY PASSWORD '*6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9' |
+--------------------------------------------------------------------------------------------------------------+
```

### 帐户权限设置-权限等级
1. 核心开发权限 select/insert/delete/update
2. 管理权限--表级 create table/drop table/lock table
3. 管理权限--server级别 create database/create user等

### MySQL数据库安全配置
1.禁用/删除多余的管理员帐号，设置用户密码
``` perl
# 删除多余的帐号
delete from mysql.user where user !='root' and password='';
flush privileges;

# 使用set方式设置root用户密码
set password for root@localhost = password('123456');
set password for root@127.0.0.1 = password('123456');

# 使用mysqladmin工具设置密码
## 格式: mysqladmin -u用户名 -p旧密码 password 新密码
## 密码为空时，直接输入回车确认
mysqladmin -uroot -p password 654321
Enter password: 
Warning: Using a password on the command line interface can be insecure.

## 修改已有密码
mysqladmin -uroot -p654321 password 123456
## 执行后会出现如下警告提示，可以忽略：
## Warning: Using a password on the command line interface can be insecure.
## 翻译过来的意思：在命令行界面上使用密码是不安全的。

# 登陆mysql
mysql -uroot -p123456

# 删除所有密码为空的帐号
delete from mysql.user where password='';
flush privileges;

# 确保所有用户都有密码
select user,host,password from mysql.user;
+------+-----------+-------------------------------------------+
| user | host      | password                                  |
+------+-----------+-------------------------------------------+
| root | localhost | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
| root | 127.0.0.1 | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
| mytt | 127.0.0.1 | *6BB4837EB74329105EE4568DDA7DC67ED2CA2AD9 |
+------+-----------+-------------------------------------------+
```
2.删除掉db表数据(test权限)
``` perl
use mysql;
truncate table mysql.db;
```
3.删除test库
``` perl
drop database test;
```
4.修改管理员帐户名
5.密码复杂度要求
6.权限最小化

### 表操作--线上可以直接删除表吗?
生产环境上，不可以直接删除表，要删除表步骤如下：
``` perl
# 构建环境
drop database lyj;
create database lyj;
use lyj;
create table t1 (id int, name varchar(10));
insert into t1 values (1,'lyj');

# 1.查看表
show tables;

# 2.检查表是否被访问
show processlist;

# 3.重命名临时表
rename table t1 to t1_bak;

# 4.备份临时表
mysqldump -uroot -p123456 lyj t1_bak > /tmp/lyj_t1_bak_20170223.sql

# 5.一段时间后删除临时表
drop table t1_bak;
show tables from lyj like '%t1%';
```

### 常用命令
``` perl
show databases;
use mysql;
show tables;
select user,host,password from mysql.user;
grant all privilege on *.* to test_1@'%';
mysql -h127.0.0.1 -utest_1
select user();
create database jianfeng;
create table user(id int,name varchar(10));
grant select on jianfeng.user to test_1@'%';
flush privileges;
show master status\G;
change master to xxx;
show engine innodb status\G
show tables from information_schema like 'INNODB%';
mysqld_safe --defaults-file=/u01/my3306/my.cnf &
mysqladmin -S /u01/my3306/run/mysql.sock shutdown &

/u01/mysql/bin/mysql_secure_installation
```
