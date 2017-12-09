---
title: MySQL DBA从小白到大神实战-11 构建高可用MySQL系统
date: 2017-04-06
tags:
- mysql
categories:
- MySQL DBA从小白到大神实战
---

## MHA部署

### 主机名/IP规划
主机名 | IP地址 | 用途
:-: | :-: | :-:
mymha | 10.245.231.201 | Master
mydb1 | 10.245.231.202 | Slave
mydb2 | 10.245.231.203 | MHA manager
  | 10.245.231.204 | VIP虚拟IP

<!-- more -->

### 系统基本配置
``` perl
cat /etc/issue
racle Linux Server release 6.8
Kernel \r on an \m

service iptables stop
chkconfig iptables off
chkconfig ip6tables off
chkconfig bluetooth off
chkconfig cups off

sed -i "s/id:5/id:3/g" /etc/inittab
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

echo "10.245.231.201  mymha" >> /etc/hosts
echo "10.245.231.202  mydb1" >> /etc/hosts
echo "10.245.231.203  mydb2" >> /etc/hosts

vi /etc/sysctl.conf
#-------------------------------------------------
kernel.shmmax = 4398046511104      # 修改原值
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
#-------------------------------------------------

sysctl -p

mkdir /media/disk
mount /dev/cdrom /media/disk

cp /etc/yum.repos.d/public-yum-ol6.repo /etc/yum.repos.d/public-yum-ol6.repo.bak
vi /etc/yum.repos.d/public-yum-ol6.repo
#------------------------------------------------
[public_ol6_latest]
name=Oracle Linux $releasever Latest ($basearch)
baseurl=file:///media/disk/Server
gpgcheck=0
enabled=1
#------------------------------------------------

yum install -y cmake gcc gcc-c++ ncurses-devel bison zlib libxml openssl openssl-devel lrzsz
```

### 编译安装MySQL
``` perl
# mydb1/mydb2
mkdir -p /u01/mysql
mkdir -p /u01/conf
mkdir -p /u01/data/3306
mkdir -p /u01/log/3306/iblog
mkdir -p /u01/log/3306/binlog
mkdir -p /u01/run/3306
mkdir -p /u01/tmp/3306

groupadd -g 501 mysql
useradd -u 501 mysql -g mysql
echo "mysql123" | passwd --stdin mysql

chown -R mysql:mysql /u01
chmod -R 755 /u01

# 进入源码目录
cd /u01/mysql-5.6.35

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

### 配置MySQL参数配置
``` perl
cd /u01/conf
vi my3306.cnf
#----------------------------------------------------------
[client]
port=3306
socket=/u01/run/3306/mysql.sock

[mysql]
pid_file=/u01/run/3306/mysqld.pid

[mysqld]
autocommit=0
general_log=off
explicit_defaults_for_timestamp=true
transaction_isolation=read-committed

# system
basedir=/u01/mysql
datadir=/u01/data/3306
max_allowed_packet=1g
max_connections=3000
max_user_connections=2800
open_files_limit=65535
pid_file=/u01/run/3306/mysqld.pid
port=3306
server_id=101
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
#----------------------------------------------------------

# server_id=101 在mydb2上修改成102
```

### 初始化并启动数据库
``` perl
echo "export PATH=\$PATH:\$HOME/bin:/u01/mysql/bin" >> /home/mysql/.bash_profile
su - mysql
cd /u01/mysql/scripts
./mysql_install_db --defaults-file=/u01/conf/my3306.cnf \
--datadir=/u01/data/3306 --basedir=/u01/mysql --user=mysql

#启动数据库
mysqld_safe --defaults-file=/u01/conf/my3306.cnf &

#登陆数据库
mysql --socket=/u01/run/3306/mysql.sock

# 关闭数据库
mysqladmin --socket=/u01/run/3306/mysql.sock shutdown &
```

### 快速搭建MySQL一主一从
``` perl
# mydb1 mydb2
grant all privileges on *.* to 'root'@'localhost' identified by 'root123' with grant option;
grant all privileges on *.* to 'root'@'127.0.0.1' identified by 'root123' with grant option;
grant all privileges on *.* to 'root'@'10.245%' identified by 'root123' with grant option;

grant replication slave on *.* to rep@'10.245.231.202' identified by 'rep123';
grant replication slave on *.* to rep@'10.245.231.203' identified by 'rep123';

# mydb1
show master status\G;
*************************** 1. row ***************************
             File: binlog.000005
         Position: 540
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

# mydb2
CHANGE MASTER TO 
    MASTER_HOST='10.245.231.202',
    MASTER_PORT=3306,
    MASTER_USER='rep',
    MASTER_PASSWORD='rep123',
    MASTER_LOG_FILE='binlog.000005', 
    MASTER_LOG_POS=540;
start slave;
show slave status\G;
```

### 设置SSH公钥免密码登录
``` perl
# root用户在mymha/mydb1/mydb2
cd ~
ssh-keygen -t rsa
ll /root/.ssh
#----------------------------------------------------------
total 8
-rw------- 1 root root 1675 Apr  8 09:59 id_rsa
-rw-r--r-- 1 root root  392 Apr  8 09:59 id_rsa.pub
#----------------------------------------------------------

# mydb1
scp ~/.ssh/id_rsa.pub mymha:/root/.ssh/id_rsa.pub.mydb1
# mydb2
scp ~/.ssh/id_rsa.pub mymha:/root/.ssh/id_rsa.pub.mydb2
# mymha
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_rsa.pub.mydb1 >> ~/.ssh/authorized_keys
cat ~/.ssh/id_rsa.pub.mydb2 >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
rm -rf ~/.ssh/id_rsa.pub.mydb*
scp ~/.ssh/authorized_keys mydb1:/root/.ssh/
scp ~/.ssh/authorized_keys mydb2:/root/.ssh/

ssh mydb1
ssh mydb2
ssh mymha
```

### 安装mha4mysql-manager和mha4mysql-node
#### 安装依赖包
``` perl
# 在三个节点（node 和 manager）安装perl-DBD-MySQL，用光盘作yum源
yum -y install perl-DBD-MySQL perl-DBI mysql-libs

# 在管理节点安装，用光盘yum源
yum install perl-Time-HiRes

# 下载管理节点所需依赖包并安装
## http://mirrors.sohu.com/centos/6.8/os/x86_64/Packages/
## http://dl.fedoraproject.org/pub/epel/6/x86_64/
## 可以从以上两个网载中找到所需依赖包，下载后打包分享地址 http://pan.baidu.com/s/1eSzfAsq
rpm -ivh perl-Config-Tiny-2.12-7.1.el6.noarch.rpm
rpm -ivh perl-Params-Validate-0.92-3.el6.x86_64.rpm
rpm -ivh perl-MIME-Types-1.28-2.el6.noarch.rpm
rpm -ivh perl-Email-Date-Format-1.002-5.el6.noarch.rpm
rpm -ivh perl-Mail-Sender-0.8.16-3.el6.noarch.rpm
rpm -ivh perl-Mail-Sendmail-0.79-12.el6.noarch.rpm
rpm -ivh perl-TimeDate-1.16-13.el6.noarch.rpm
rpm -ivh perl-MailTools-2.04-4.el6.noarch.rpm
rpm -ivh perl-MIME-Lite-3.027-2.el6.noarch.rpm
rpm -ivh perl-Log-Dispatch-2.27-1.el6.noarch.rpm
rpm -ivh perl-Parallel-ForkManager-0.7.9-1.el6.noarch.rpm 
```

#### 在所有节点安装node包
``` perl
# 下载地址：http://olg2mr2vf.bkt.clouddn.com/mha4mysql-manager-0.56.tar.gz
tar xzvf mha4mysql-node-0.56.tar.gz
cd mha4mysql-node-0.56
perl Makefile.PL
make && make install

ll /usr/local/bin/
#----------------------------------------------------------
-r-xr-xr-x 1 root root 16367 Apr  8 11:10 apply_diff_relay_logs
-r-xr-xr-x 1 root root  4807 Apr  8 11:10 filter_mysqlbinlog
-r-xr-xr-x 1 root root  8261 Apr  8 11:10 purge_relay_logs
-r-xr-xr-x 1 root root  7525 Apr  8 11:10 save_binary_logs
#----------------------------------------------------------
```

#### 在管理节点安装manager包
``` perl
# 下载地址：http://olg2mr2vf.bkt.clouddn.com/mha4mysql-node-0.56.tar.gz
tar xzvf mha4mysql-manager-0.56.tar.gz
cd mha4mysql-manager-0.56
perl Makefile.PL
make && make install

ll /usr/local/bin/
#----------------------------------------------------------
-r-xr-xr-x. 1 root root 16367 Apr  8 11:08 apply_diff_relay_logs
-r-xr-xr-x. 1 root root  4807 Apr  8 11:08 filter_mysqlbinlog
-r-xr-xr-x. 1 root root  1995 Apr  8 11:12 masterha_check_repl
-r-xr-xr-x. 1 root root  1779 Apr  8 11:12 masterha_check_ssh
-r-xr-xr-x. 1 root root  1865 Apr  8 11:12 masterha_check_status
-r-xr-xr-x. 1 root root  3201 Apr  8 11:12 masterha_conf_host
-r-xr-xr-x. 1 root root  2517 Apr  8 11:12 masterha_manager
-r-xr-xr-x. 1 root root  2165 Apr  8 11:12 masterha_master_monitor
-r-xr-xr-x. 1 root root  2373 Apr  8 11:12 masterha_master_switch
-r-xr-xr-x. 1 root root  5171 Apr  8 11:12 masterha_secondary_check
-r-xr-xr-x. 1 root root  1739 Apr  8 11:12 masterha_stop
-r-xr-xr-x. 1 root root  8261 Apr  8 11:08 purge_relay_logs
-r-xr-xr-x. 1 root root  7525 Apr  8 11:08 save_binary_logs
#----------------------------------------------------------
```

#### mha 配置文件
``` perl
mkdir -p /u01/mha/etc
mkdir -p /u01/mha/log
vi /u01/mha/etc/app.cnf
#----------------------------------------------------------
[server default]
user = root
password = root123
ssh_user = root
repl_user = rep
repl_password = rep123
ping_interval = 1
ping_type = SELECT

manager_workdir=/u01/mha/etc/app
manager_log=/u01/mha/log/manager.log
remote_workdir=/u01/mha/etc/app
master_binlog_dir="/u01/log/3306/binlog"

master_ip_failover_script="/u01/mha/etc/master_ip_failover"
master_ip_online_change_script="/u01/mha/etc/master_ip_failover"

shutdown_script=""

report_script=""

#check_repl_delay=0

[server1]
hostname=mydb1
port=3306
master_binlog_dir="/u01/log/3306/binlog"
candidate_master=1
ignore_fail=1

[server2]
hostname=mydb2
port=3306
master_binlog_dir="/u01/log/3306/binlog"
candidate_master=1
ignore_fail=1
#----------------------------------------------------------
```

#### master_ip_failover脚本
``` perl
vi /u01/mha/etc/master_ip_failover
#----------------------------------------------------------
#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';
 
use Getopt::Long;
 
my (
    $command,          $ssh_user,        $orig_master_host, $orig_master_ip,
    $orig_master_port, $new_master_host, $new_master_ip,    $new_master_port
);
 
my $vip = '10.245.231.204/24';
# Virtual IP
my $key = "1";
my $int = "eth0";
my $ssh_start_vip = "/sbin/ifconfig $int:$key $vip";
my $ssh_stop_vip = "/sbin/ifconfig $int:$key down";
my $arp_effect = "/sbin/arping -Uq -s 10.245.231.204 -I $int 10.245.231.254 -c 3";
# Virtual IP and gateway
#my $test = "echo successfull >/tmp/test.txt";
$ssh_user = "root";
GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);
 
exit &main();
 
sub main {
 
    print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n";
 
    if ( $command eq "stop" || $command eq "stopssh" ) {
 
        # $orig_master_host, $orig_master_ip, $orig_master_port are passed.
        # If you manage master ip address at global catalog database,
        # invalidate orig_master_ip here.
        my $exit_code = 1;
        eval {
            print "Disabling the VIP on old master: $orig_master_host \n";
            &stop_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn "Got Error: $@\n";
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "start" ) {
 
        # all arguments are passed.
        # If you manage master ip address at global catalog database,
        # activate new_master_ip here.
        # You can also grant write access (create user, set read_only=0, etc) here.
        my $exit_code = 10;
        eval {
            print "Enabling the VIP - $vip on the new master - $new_master_host \n";
            &start_vip();
            $exit_code = 0;
        };
        if ($@) {
            warn $@;
            exit $exit_code;
        }
        exit $exit_code;
    }
    elsif ( $command eq "status" ) {
        print "Checking the Status of the script.. OK \n";
        #`ssh $ssh_user\@cluster1 \" $ssh_start_vip \"`;
        &status();
        exit 0;
    }
    else {
        &usage();
        exit 1;
    }
}
 
# A simple system call that enable the VIP on the new master
sub start_vip() {
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`;
    `ssh $ssh_user\@$new_master_host \" $arp_effect \"`;
#    `ssh $ssh_user\@$new_master_host \" $test \"`;
}
# A simple system call that disable the VIP on the old_master
sub stop_vip() {
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;
}

sub status() {
    print `ssh $ssh_user\@$orig_master_host \" ip add show $int \"`;
}
 
sub usage {
    print
    "Usage: master_ip_failover --command=start|stop|stopssh|status --orig_master_host=host --orig_maste
r_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";
}
#----------------------------------------------------------

chmod +x /u01/mha/etc/master_ip_failover
```

#### vip add 配置
``` perl
# 在master mydb1上执行下面命令（第一次启用可以手动配置vip address，不配置的话，自动切换后可生成）
ifconfig eth0:1 10.245.231.204/24
```

#### 检测/启动/关闭MHA
``` perl
/usr/local/bin/masterha_check_ssh --conf=/u01/mha/etc/app.cnf
/usr/local/bin/masterha_check_repl --conf=/u01/mha/etc/app.cnf
/usr/local/bin/masterha_manager --conf=/u01/mha/etc/app.cnf &
/usr/local/bin/masterha_check_status --conf=/u01/mha/etc/app.cnf
/usr/local/bin/masterha_stop --conf=/u01/mha/etc/app.cnf

# 检测ssh等价性配置
/usr/local/bin/masterha_check_ssh --conf=/u01/mha/etc/app.cnf
#----------------------------------------------------------
Sat Apr  8 11:42:09 2017 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Sat Apr  8 11:42:09 2017 - [info] Reading application default configuration from /u01/mha/etc/app.cnf..
Sat Apr  8 11:42:09 2017 - [info] Reading server configuration from /u01/mha/etc/app.cnf..
Sat Apr  8 11:42:09 2017 - [info] Starting SSH connection tests..
Sat Apr  8 11:42:10 2017 - [debug] 
Sat Apr  8 11:42:09 2017 - [debug]  Connecting via SSH from root@mydb1(10.245.231.202:22) to root@mydb2(10.245.231.203:22)..
Sat Apr  8 11:42:10 2017 - [debug]   ok.
Sat Apr  8 11:42:10 2017 - [debug] 
Sat Apr  8 11:42:10 2017 - [debug]  Connecting via SSH from root@mydb2(10.245.231.203:22) to root@mydb1(10.245.231.202:22)..
Sat Apr  8 11:42:10 2017 - [debug]   ok.
Sat Apr  8 11:42:10 2017 - [info] All SSH connection tests passed successfully.   # successfully代表SSH等价性配置成功
#----------------------------------------------------------

# 检测MHA
/usr/local/bin/masterha_check_repl --conf=/u01/mha/etc/app.cnf
## 检测问题和解决办法
#----------------------------------------------------------
# 问题1：
Sat Apr  8 11:44:15 2017 - [info]  read_only=1 is not set on slave mydb2(10.245.231.203:3306).
Sat Apr  8 11:44:15 2017 - [warning]  relay_log_purge=0 is not set on slave mydb2(10.245.231.203:3306)
# 问题2：
Can not exec "mysqlbinlog": No such file or directory at /usr/local/share/perl5/MHA/BinlogManager.pm line 106.
mysqlbinlog version command failed with rc 1:0, please verify PATH, LD_LIBRARY_PATH, and client options  at /usr/local/bin/apply_diff_relay_logs line 493
# 问题3：
Sat Apr  8 11:44:17 2017 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln205] Slaves settings check failed!
Sat Apr  8 11:44:17 2017 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln413] Slave configuration failed.
Sat Apr  8 11:44:17 2017 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln424] Error happened on checking configurations.  at /usr/local/bin/masterha_check_repl line 48
Sat Apr  8 11:44:17 2017 - [error][/usr/local/share/perl5/MHA/MasterMonitor.pm, ln523] Error happened on monitoring servers.
....
MySQL Replication Health is NOT OK!   # NOT OK，MHA有问题需要解决，以下是解决步骤
#----------------------------------------------------------

# mydb2
set global read_only=1;  
set global relay_log_purge=0;

# mydb1/mydb2
ln -s /u01/mysql/bin/mysqlbinlog /usr/bin/mysqlbinlog
ln -s /u01/mysql/bin/mysql /usr/bin/mysql

# 解决问题后再检测时状态是OK
#----------------------------------------------------------
Sat Apr  8 13:39:33 2017 - [info]  OK.
Sat Apr  8 13:39:33 2017 - [warning] shutdown_script is not defined.
Sat Apr  8 13:39:33 2017 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.
#----------------------------------------------------------

# 启动MHA后查看状态
/usr/local/bin/masterha_manager --conf=/u01/mha/etc/app.cnf &
/usr/local/bin/masterha_check_status --conf=/u01/mha/etc/app.cnf
#----------------------------------------------------------
app (pid:10163) is running(0:PING_OK), master:mydb1
#----------------------------------------------------------
/usr/local/bin/masterha_stop --conf=/u01/mha/etc/app.cnf
```


## MHA failover故障切换
### 模拟主库宕机
``` perl
ssh mydb1 "killall -r mysqld"
```

### 查看管理节点日志，可以看到VIP已经漂移
``` perl
cat /u01/mha/log/manager.log |grep -i vip
#----------------------------------------------------------
Disabling the VIP on old master: mydb1 
Enabling the VIP - 10.245.231.204/24 on the new master - mydb2 
#----------------------------------------------------------
```

### 验证VIP是否位于节点mydb2
``` perl
ssh mydb2 "ifconfig |grep 231.204 -B1"
#----------------------------------------------------------
eth0:1    Link encap:Ethernet  HWaddr 00:50:56:A0:35:C0  
          inet addr:10.245.231.204  Bcast:10.245.231.255  Mask:255.255.255.0
#----------------------------------------------------------

# 若VIP没有自动切换，可以使用以下命令手动切换（不建议）
ifconfig eth0:1 10.245.231.204/24
arping -Uq -s 10.245.231.204 -I eth0 10.245.231.254 -c 3
ifconfig eth0:1 down
```

### 查看管理节点MHA切换日志
``` perl
tail /u01/mha/log/manager.log
#----------------------------------------------------------
Started automated(non-interactive) failover.
Invalidated master IP address on mydb1(10.245.231.202:3306)
The latest slave mydb2(10.245.231.203:3306) has all relay logs for recovery.
Selected mydb2(10.245.231.203:3306) as a new master.
mydb2(10.245.231.203:3306): OK: Applying all logs succeeded.
mydb2(10.245.231.203:3306): OK: Activated master IP address.
Generating relay diff files from the latest slave succeeded.
mydb2(10.245.231.203:3306): Resetting slave info succeeded.
Master failover to mydb2(10.245.231.203:3306) completed successfully.
#----------------------------------------------------------
```

### new master(old slave)
``` perl
show master status\G
*************************** 1. row ***************************
             File: binlog.000007
         Position: 120
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```

### new slave(old:master)
``` perl
# 打开MySQL
mysqld_safe --defaults-file=/u01/conf/my3306.cnf &
mysql -uroot -proot123 --socket=/u01/run/3306/mysql.sock

# 检查数据库
mysql> show master status\G
*************************** 1. row ***************************
             File: binlog.000013
         Position: 120
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

mysql> show slave  status\G
Empty set (0.00 sec)

# 在管理节点日志中查主库的日志文件和位置
cat /u01/mha/log/manager.log |grep -i change
#----------------------------------------------------------
Sat Apr  8 14:55:24 2017 - [info]  All other slaves should start replication from here. 
Statement should be: CHANGE MASTER TO MASTER_HOST='mydb2 or 10.245.231.203', MASTER_PORT=3306, 
MASTER_LOG_FILE='binlog.000007', MASTER_LOG_POS=120, MASTER_USER='rep', MASTER_PASSWORD='xxx';
#----------------------------------------------------------

# 在slave连接master
CHANGE MASTER TO
  MASTER_HOST='10.245.231.203',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='binlog.000007',
  MASTER_LOG_POS=120,
  MASTER_USER='rep',
  MASTER_PASSWORD='rep123';

start slave;
```

### 启动管理节点
``` perl
/usr/local/bin/masterha_manager --conf=/u01/mha/etc/app.cnf &
/usr/local/bin/masterha_manager --conf=/u01/mha/etc/app.cnf  --ignore_last_failover &
```

## MHA switchover线上切换

### master：关闭event_scheduler
``` perl
# master mydb2
/usr/local/bin/masterha_check_status --conf=/u01/mha/etc/app.cnf
app (pid:3793) is running(0:PING_OK), master:mydb2

# mydb2
set global event_scheduler=off;
```

### manager：关闭管理进程
``` perl
/usr/local/bin/masterha_stop --conf=/u01/mha/etc/app.cnf
```

### manager：检查配置文件
``` perl
# 有没有被修改破坏。如果破坏需要重新编辑正确配置文件：/u01/mha/etc/app.cnf
cp /u01/mha/etc/app.cnf.bak /u01/mha/etc/app.cnf
```

### 开始切换
``` perl
/usr/local/bin/masterha_master_switch --master_state=alive --conf=/u01/mha/etc/app.cnf

at Apr  8 15:23:00 2017 - [info] MHA::MasterRotate version 0.56.
Sat Apr  8 15:23:00 2017 - [info] Starting online master switch..
Sat Apr  8 15:23:00 2017 - [info] 
Sat Apr  8 15:23:00 2017 - [info] * Phase 1: Configuration Check Phase..
Sat Apr  8 15:23:00 2017 - [info] 
Sat Apr  8 15:23:00 2017 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
Sat Apr  8 15:23:00 2017 - [info] Reading application default configuration from /u01/mha/etc/app.cnf..
Sat Apr  8 15:23:00 2017 - [info] Reading server configuration from /u01/mha/etc/app.cnf..
Sat Apr  8 15:23:00 2017 - [info] GTID failover mode = 0
Sat Apr  8 15:23:00 2017 - [info] Current Alive Master: mydb2(10.245.231.203:3306)
Sat Apr  8 15:23:00 2017 - [info] Alive Slaves:
Sat Apr  8 15:23:00 2017 - [info]   mydb1(10.245.231.202:3306)  Version=5.6.35-log (oldest major version between slaves) log-bin:enabled
Sat Apr  8 15:23:00 2017 - [info]     Replicating from 10.245.231.203(10.245.231.203:3306)
Sat Apr  8 15:23:00 2017 - [info]     Primary candidate for the new Master (candidate_master is set)

It is better to execute FLUSH NO_WRITE_TO_BINLOG TABLES on the master before switching. Is it ok to execute on mydb2(10.245.231.203:3306)? (YES/no): yes  # 输yes

Sat Apr  8 15:23:11 2017 - [info] Executing FLUSH NO_WRITE_TO_BINLOG TABLES. This may take long time..
Sat Apr  8 15:23:11 2017 - [info]  ok.
Sat Apr  8 15:23:11 2017 - [info] Checking MHA is not monitoring or doing failover..
Sat Apr  8 15:23:11 2017 - [info] Checking replication health on mydb1..
Sat Apr  8 15:23:11 2017 - [info]  ok.
Sat Apr  8 15:23:11 2017 - [info] Searching new master from slaves..
Sat Apr  8 15:23:11 2017 - [info]  Candidate masters from the configuration file:
Sat Apr  8 15:23:11 2017 - [info]   mydb1(10.245.231.202:3306)  Version=5.6.35-log (oldest major version between slaves) log-bin:enabled
Sat Apr  8 15:23:11 2017 - [info]     Replicating from 10.245.231.203(10.245.231.203:3306)
Sat Apr  8 15:23:11 2017 - [info]     Primary candidate for the new Master (candidate_master is set)
Sat Apr  8 15:23:11 2017 - [info]   mydb2(10.245.231.203:3306)  Version=5.6.35-log log-bin:enabled
Sat Apr  8 15:23:11 2017 - [info]  Non-candidate masters:
Sat Apr  8 15:23:11 2017 - [info]  Searching from candidate_master slaves which have received the latest relay log events..
Sat Apr  8 15:23:11 2017 - [info] 
From:
mydb2(10.245.231.203:3306) (current master)
 +--mydb1(10.245.231.202:3306)

To:
mydb1(10.245.231.202:3306) (new master)

Starting master switch from mydb2(10.245.231.203:3306) to mydb1(10.245.231.202:3306)? (yes/NO): yes   # 输yes
......
IN SCRIPT TEST====/sbin/ifconfig eth0:1 down==/sbin/ifconfig eth0:1 10.245.231.204/24===

Enabling the VIP - 10.245.231.204/24 on the new master - mydb1 
Sat Apr  8 15:23:54 2017 - [info]  ok.
Sat Apr  8 15:23:54 2017 - [info] 
Sat Apr  8 15:23:54 2017 - [info] * Switching slaves in parallel..
Sat Apr  8 15:23:54 2017 - [info] 
Sat Apr  8 15:23:54 2017 - [info] Unlocking all tables on the orig master:
Sat Apr  8 15:23:54 2017 - [info] Executing UNLOCK TABLES..
Sat Apr  8 15:23:54 2017 - [info]  ok.
Sat Apr  8 15:23:54 2017 - [info] All new slave servers switched successfully.
Sat Apr  8 15:23:54 2017 - [info] 
Sat Apr  8 15:23:54 2017 - [info] * Phase 5: New master cleanup phase..
Sat Apr  8 15:23:54 2017 - [info] 
Sat Apr  8 15:23:54 2017 - [info]  mydb1: Resetting slave info succeeded.
Sat Apr  8 15:23:54 2017 - [info] Switching master to mydb1(10.245.231.202:3306) completed successfully.
```

### new master(old slave)
``` perl
mysql> show master status\G
*************************** 1. row ***************************
             File: binlog.000013
         Position: 936
     Binlog_Do_DB: 
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)

mysql> show slave status\G
Empty set (0.00 sec)
```

### new slave(old master)
``` perl
CHANGE MASTER TO
  MASTER_HOST='10.245.231.202',
  MASTER_PORT=3306,
  MASTER_LOG_FILE='binlog.000013',
  MASTER_LOG_POS=936,
  MASTER_USER='rep',
  MASTER_PASSWORD='rep123';

mysql> start slave;
Query OK, 0 rows affected (0.01 sec)

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.245.231.202
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000013
          Read_Master_Log_Pos: 936
               Relay_Log_File: relaylog.000002
                Relay_Log_Pos: 280
        Relay_Master_Log_File: binlog.000013
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

### 启动管理节点
``` perl
/usr/local/bin/masterha_manager --conf=/u01/mha/etc/app.cnf &
/usr/local/bin/masterha_manager --conf=/u01/mha/etc/app.cnf --remove_dead_master_conf --ignore_last_failover &

/usr/local/bin/masterha_check_status --conf=/u01/mha/etc/app.cnf
app (pid:4463) is running(0:PING_OK), master:mydb1
```

## 杂记

MHA（Master High Availability）目前在MySQL高可用方面是一个相对成熟的解决方案，是日本的一位MySQL专家采用Perl语言编写的一个脚本管理工具，目的在于维持MySQL Replication中<font color='red'>Master库的高可用性</font>，其最大特点是可以修复多个Slave之间的差异日志，最终使所有Slave保持数据一致，然后从中选择一个充当新的Master，并将其它Slave指向它。

MHA集群架构图
![](http://oligvdnzp.bkt.clouddn.com/0406_mha_01.png)

### MHA监控
- MHA监控最主要的就是监控master，每`ping_interval`秒监控master一次。
- MHA自身提供了两种监控方式：SELECT和CONNECT，控制参数`ping_type`
- MHA调用SSH脚本对所有Node执行检查，包括：
  - Master服务器是否可以SSH连通
  - MySQL实例是否可以连接
  - 检查SQL Thread的状态
  - 检查哪些Server死掉了，哪些Server是活动的，以及活动的Slave实例
  - 检查Slave实例的配置及复制过滤规则
- MHA监控检查Master Node异常时操作：
  1. SQL Thread alive? NO: Restart it
  2. 调用`master_ip_failover_script`关闭VIP
  3. 检查各个Slave，获取最近的和最旧的`binary log file`和`position`，并检查各个Slave成为Master的优先级
  4. 若dead master所在服务器依然可以通过SSH连通，则提取dead master的binary log。另外，MHA还要对各个Slave节点SSH连通性进行检查。
  5. 调用`apply_diff_relay_logs`命令恢复Slave的差异日志，即各个Slave之间的relay log
  6. 差异日志恢复到New Master上，然后获取New Master的binlogname和position，最后会开启New Master的写权限
  7. 清理New Master其实就是重置slave info，即取消原来的Slave信息
