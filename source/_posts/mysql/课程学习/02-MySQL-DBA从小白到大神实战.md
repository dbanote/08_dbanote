---
title: MySQL DBA从小白到大神实战-02 MySQL标准化、自动化部署
tags:
- mysql
---

## 为什么数据目录和日志目录需要分开?
这里的分开，我理解是将MySQL的数据目录和日志目录分别放到不同类型的磁盘中。  
假设生产环境中服务器上有两种类型的磁盘，分别是SSD和SAS，SSD比SAS的响应时间要快（SSD响应时间约0.1毫秒，SAS的响应时间约10毫秒），为了更好的利用磁盘，一般会把活跃的数据放到SSD上，冷数据放到SAS磁盘上。  
数据目录下的数据一般是随机读写的热数据，放到SSD盘中会有较高的响应速度；  
日志目录下的日志是顺序读写的冷数据，放到SAS盘中满足写日志高吞吐量的需求。
<!--more-->

## 如何标准化配置多实例?（例如：一台物理主机上部署3306与3307两个实例）
标准化配置多实例，主要是标准化每个实例的目录和内存设置，这样每个实例的参数设置也很容易达到标准化。标准化配置的MYSQL实例，方便实施监控及运维管理。  
因一个MySQL实例最多占用64G的物理内存，所以在物理内存较高的服务器上，一般会安装多个MySQL实例，MySQL不同实例是通过端口来区别的。  
例如：一台64G的物理主机上部署3306与3307两个MYSQL实例  

### 标准化目录
```
实例1：
/data/my3306
/log/my3306

实例2：
/data/my3307
/log/my3307

# /data 目录挂载到SSD盘上
# /log  目录挂载到SAS盘上
```

### 标准化内存
```
实例1：
15G: InnoDB buffer cache
5G : mysql server层

实例2：
15G: InnoDB buffer cache
5G : mysql server层

OS:
24G
```

### 标准化参数
结合标准化的目录及内存设置，设置标准化的参数

## 详细描述MySQL编译安装的过程（截图安装步骤）

### 关闭防火墙和SELINUX
``` perl
service iptables status
#------------------------------------------------
Table: filter
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED 
2    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
3    ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
4    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22 
5    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination         
1    REJECT     all  --  0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited 

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination         
#------------------------------------------------

service iptables stop
#------------------------------------------------
iptables: Setting chains to policy ACCEPT: filter          [  OK  ]
iptables: Flushing firewall rules:                         [  OK  ]
iptables: Unloading modules:                               [  OK  ]
#------------------------------------------------

service iptables status
#------------------------------------------------
iptables: Firewall is not running.
#------------------------------------------------

chkconfig iptables off

vi /etc/selinux/config
#------------------------------------------------
SELINUX=disabled
#------------------------------------------------
```

### 配置sysctl.conf
``` perl
# 查看服务器内存
free
#------------------------------------------------
             total       used       free     shared    buffers     cached
Mem:       8174352     616628    7557724        172     151904     253892
-/+ buffers/cache:     210832    7963520
Swap:     16531452          0   16531452
#------------------------------------------------

vi /etc/sysctl.conf
#------------------------------------------------
# 修改
kernel.shmmax = 4398046511104

# 添加
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
#------------------------------------------------
# kernel.shmmax算法：修改为物理内容的50%、60%
# 8G:kernel.shmmax = (8G*1024*1024*1024*1024)*50% = 4398046511104

# 使配置立即生效
sysctl -p
```

### 检查是否已安装MySQL
``` perl
rpm -qa | grep mysql
#------------------------------------------------
mysql-libs-5.1.73-7.el6.x86_64
#------------------------------------------------

# 删除mysql-libs-5.1.73-7.el6.x86_64包
rpm -e --nodeps mysql-libs-5.1.73-7.el6.x86_64
```

### 下载MySQL源码
``` perl
Download MySQL Community Server  https://dev.mysql.com/downloads/mysql/5.6.html#downloads
Select Version: 5.6.35 -> Select Platform: Source Code
-> 选择【Generic Linux (Architecture Independent), Compressed TAR Archive】下载 

# 配置yum源，安装lrzsz(代替ftp上传和下载的工具)
mkdir /media/disk
mkdir /media/cdrom
mount /dev/cdrom /media/cdrom

cp -rf /media/cdrom/* /media/disk
umount /media/cdrom

cp /etc/yum.repos.d/public-yum-ol6.repo /etc/yum.repos.d/public-yum-ol6.repo.bak
vi /etc/yum.repos.d/public-yum-ol6.repo
#------------------------------------------------
name=Oracle Linux $releasever Latest ($basearch)
baseurl=file:///media/disk/Server
gpgcheck=0
enabled=1
#------------------------------------------------

yum -y install lrzsz

# 上传到服务器的/u01目录下
mkdir /u01
cd /u01
rz

# 选择mysql源码包，上传
rz waiting to receive.
Starting zmodem transfer.  Press Ctrl+C to cancel.
Transferring mysql-5.6.35.tar.gz...
  100%   31413 KB    15706 KB/sec    00:00:02       0 Errors 

ll /u01/mysql*
-rw-r--r--. 1 root root 32167628 Jan 17 11:16 /u01/mysql-5.6.35.tar.gz
```

### 添加MySQL用户和组
``` perl
groupadd -g 501 mysql
useradd -u 501 mysql -g mysql
echo "mysql123" | passwd --stdin mysql

id mysql
uid=501(mysql) gid=501(mysql) groups=501(mysql)
```

### 配MySQL环境变量
``` perl
vi /home/mysql/.bash_profile
#------------------------------------------------
PATH=$PATH:$HOME/bin:/u01/my3306/bin
#------------------------------------------------
```

### 创建目录及授权
``` perl
mkdir -p /u01/my3306/data
mkdir -p /u01/my3306/log/iblog
mkdir -p /u01/my3306/log/binlog
mkdir -p /u01/my3306/run
mkdir -p /u01/my3306/tmp

chown -R mysql:mysql /u01/my3306
chmod -R 755 /u01/my3306
```

### 解压mysql5.6
```
cd /u01
tar xvpf mysql-5.6.35.tar.gz
```

### 安装cmake及相关依赖包
```
yum install -y  cmake gcc gcc-c++ ncurses-devel bison zlib libxml openssl 
```

### 编译并安装
``` perl
cd /u01/mysql-5.6.35

cmake \
-DCMAKE_INSTALL_PREFIX=/u01/my3306 \
-DINSTALL_DATADIR=/u01/my3306/data  \
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
-DMYSQL_UNIX_ADDR=/u01/my3306/run/mysql.sock \
-DMYSQL_TCP_PORT=3306 \
-DENABLED_LOCAL_INFILE=1 \
-DSYSCONFDIR=/etc \
-DWITH_READLINE=on

# 第一次CMAKE出现错误提示
#---------------------------------------------------------------------------------------------
CMake Error: The following variables are used in this project, but they are set to NOTFOUND.
Please set them or make sure they are set and tested correctly in the CMake files:
OPENSSL_INCLUDE_DIR
   used as include directory in directory /u01/mysql-5.6.35/CMakeFiles/CMakeTmp
#---------------------------------------------------------------------------------------------

#安装openssl-devel包
yum -y install openssl-devel

#重新cmake需要删除当前目录下CMakeCache.txt，然后再重新执行
rm -rf CMakeCache.txt

#编译并安装
make
make install
```

### MySQL参数配置
``` perl
cd /u01/my3306
vi my.cnf
#----------------------------------------------------------
[client]
port=3306
socket=/u01/my3306/mysql.sock

[mysql]
pid_file=/u01/my3306/run/mysqld.pid

[mysqld]
autocommit=1
general_log=off
explicit_defaults_for_timestamp=true

# system
basedir=/u01/my3306
datadir=/u01/my3306/data
max_allowed_packet=1g
max_connections=3000
max_user_connections=2800
open_files_limit=65535
pid_file=/u01/my3306/run/mysqld.pid
port=3306
server_id=101
skip_name_resolve=ON
socket=/u01/my3306/run/mysql.sock
tmpdir=/u01/my3306/tmp

#binlog
log_bin=/u01/my3306/log/binlog/binlog
binlog_cache_size=32768
binlog_format=row
expire_logs_days=7
log_slave_updates=ON
max_binlog_cache_size=2147483648
max_binlog_size=524288000
sync_binlog=100

#logging
log_error=/u01/my3306/log/error.log
slow_query_log_file=/u01/my3306/log/slow.log
log_queries_not_using_indexes=0
slow_query_log=1
log_slave_updates=ON
log_slow_admin_statements=1
long_query_time=1

#relay
relay_log=/u01/my3306/log/relaylog
relay_log_index=/u01/my3306/log/relay.index
relay_log_info_file=/u01/my3306/log/relay-log.info

#slave
slave_load_tmpdir=/u01/my3306/tmp
slave_skip_errors=OFF


#innodb
innodb_data_home_dir=/u01/my3306/log/iblog
innodb_log_group_home_dir=/u01/my3306/log/iblog
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
datadir=/u01/my3306/data
#----------------------------------------------------------

# 编译后重新修改目录权限
chown -R mysql:mysql /u01/my3306
```

### 初始化MySQL脚本
```
su - mysql
cd /u01/my3306/scripts
./mysql_install_db --defaults-file=/u01/my3306/my.cnf \
--datadir=/u01/my3306/data --basedir=/u01/my3306 --user=mysql
```

### 启动MySQL
```
/u01/my3306/bin/mysqld_safe --defaults-file=/u01/my3306/my.cnf --user=mysql &
```

### 登录MySQL
``` perl
mysql
# 或者
mysql -h127.0.0.1 -uroot

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.6.35-log Source distribution

Copyright (c) 2000, 2016, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+
4 rows in set (0.01 sec)
```