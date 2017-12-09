---
title: MySQL性能优化最佳实践 - 04 MySQL优化之Linux系统层面调优
date: 2017-08-28
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

## MySQL中出现存SWAP,主要会是哪些原因？
1. 没设置内核参数`sysctl.conf`中的`vm.swappiness=0`
``` perl
# 在CentOS 6.4及更新版本的内核中设置vm.swappiness=0，有可能会导致MySQL数据库所在的系统出现内存溢出(OOM)，
# 可将vm.swappiness设置成1
vm.swappiness = 1

# CentOS 7.x中
## 查找到存在vm.swappiness参数的配置文件
find /usr/lib/tuned -name '*.conf' -type f -exec grep "vm.swappiness" {} \+
#----------------------------------------------------------
/usr/lib/tuned/latency-performance/tuned.conf:vm.swappiness=10
/usr/lib/tuned/throughput-performance/tuned.conf:vm.swappiness=10
/usr/lib/tuned/virtual-guest/tuned.conf:vm.swappiness = 30
#----------------------------------------------------------
# 把以上3个文件中的vm.swappiness的值都修改为0或1
```

2. 没配置MySQL的配置参数memlock
  这个参数会强迫mysqld进程的地址空间一直被锁定在物理内存上，对于os来说是非常霸道的一个要求。要用root帐号来启动MySQL才能生效。
3. 没关闭NUMA
<!-- more -->

## MySQL中CPU负载很高，是什么原因？给出查找的步骤及解决思路？
一般来说MySQL占用CPU高，多半是数据库有变态SQL、大表无索引，或有大量的并发任务导致。

### 问题排查思路 
1. 确定高负载的类型，top命令看负载高是CPU还是IO。 
2. MySQL下执行查看当前的连接数与执行的sql语句。 
3. 检查慢查询日志，可能是慢查询引起负载高。 
4. 检查硬件问题，是否磁盘故障问题造成的。 
5. 检查监控平台，对比此机器不同时间的负载。 

### 确定负载类型(top)
``` perl
top
#----------------------------------------------------------------------------
top - 11:33:17 up 142 days, 20:57,  2 users,  load average: 1.85, 0.80, 0.62
Tasks: 174 total,   1 running, 173 sleeping,   0 stopped,   0 zombie
Cpu(s): 33.8%us,  8.3%sy,  0.0%ni, 52.1%id,  3.0%wa,  0.0%hi,  2.8%si,  0.0%st
Mem:   8174352k total,  3291308k used,  4883044k free,   151272k buffers
Swap: 16531452k total,    19584k used, 16511868k free,  2034864k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                      
11420 mysql     20   0 5786m 819m  13m S 372.1 10.3   0:57.47 mysqld 
.....
#----------------------------------------------------------------------------
```

### 查看当前的连接数与执行的sql语句
``` perl
mysql> show processlist;
show processlist;
+----+------+----------------------+------+-------------+------+-----------------------------------------------------------------------+------------------------------------------+
| Id | User | Host                 | db   | Command     | Time | State                                                                 | Info                                     |
+----+------+----------------------+------+-------------+------+-----------------------------------------------------------------------+------------------------------------------+
|  1 | root | localhost            | NULL | Query       |    0 | init                                                                  | show processlist                         |
| 12 | rep  | 10.245.231.203:41161 | NULL | Binlog Dump |  120 | Master has sent all binlog to slave; waiting for binlog to be updated | NULL                                     |
| 26 | root | 10.245.231.201:46306 | test | Query       |    0 | Writing to net                                                        | SELECT pad FROM sbtest11 WHERE id=100233 |
| 27 | root | 10.245.231.201:46308 | test | Sleep       |    0 | NULL                                                                  | NULL                                     |
| 28 | root | 10.245.231.201:46309 | test | Query       |    0 | statistics                                                            | SELECT pad FROM sbtest3 WHERE id=100176  |
| 29 | root | 10.245.231.201:46307 | test | Query       |    0 | Sending data                                                          | SELECT pad FROM sbtest15 WHERE id=63746  |
+----+------+----------------------+------+-------------+------+-----------------------------------------------------------------------+------------------------------------------+
```

### 记录慢查询
编辑Mysql 配置文件(my.cnf),在[mysqld]字段添加以下几行
``` perl
log_slow_queries = /usr/local/mysql/var/slow_queries.log   # 慢查询日志路径
long_query_time = 10                                       # 记录SQL查询超过10s的语句
log-queries-not-using-indexes = 1                          # 记录没有使用索引的sql
```

### 查看慢查询日志
``` perl
tail /usr/local/mysql/var/slow_queries.log 
# Time: 130305  9:48:13
# User@Host: biotherm[biotherm] @  [8.8.8.45]
# Query_time: 1294.881407  Lock_time: 0.000179 Rows_sent: 4  Rows_examined: 1318033
## 这4个参数意思分别是：
## 查询时间                 锁定时间            查询结果行数   扫描行数
## 主要看扫描行数多的语句,然后去数据库加上对应的索引,再优化下变态的sql 语句
SET timestamp=1363916893; 
SELECT * FROM xxx_list WHERE tid = '11xx'  AND del = 0  ORDER BY  id DESC  LIMIT 0, 4;
.....
```

### 极端情况kill sql进程
``` perl
# 找出占用cpu时间过长的sql，在mysql 下执行如下命令： 
show processlist; 
# 确定后一条sql处于Query状态，且Time时间过长，锁定它的ID，执行如下命令： 
kill QUERY  269815764; 
#注意：杀死 sql进程，可能导致数据丢失，所以执行前要衡量数据的重要性。
```

## 课程笔记
### 设定主机名
``` perl
vi /etc/sysconfig/network
```

### 设定时区
``` perl
vi /etc/sysconfig/clock
ZONE=Asia/Shanghai
UTC=false
ARC=false

rm /etc/localtime
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

### 设定NTP同步
``` perl
sudocrontab-l
0 * * * * /usr/sbin/ntpdate192.168.50.21; hwclock -w 1>/dev/null 2>&1
30 23 * * * /usr/sbin/ntpdate192.168.50.22; hwclock -w 1>/dev/null 2>&1
```

### 删除操作系统多余组和用户
``` perl
groupdel adm
groupdel lp
groupdel news
groupdel uucp
groupdel games
groupdel dip
groupdel pppusers
groupdel popusers
groupdel slipusers

userdel adm
userdel lp
userdel sync
userdel shutdown
userdel halt
userdel news
userdel uucp
userdel operator
userdel games
userdel gopher
userdel ftp
```

### 清理不需要的服务
``` perl
#一般只留下以下服务即可
crond
haldaemon
irqbalance
messagebus
network
sshd
syslog
```

### 设置口令有效期和最少长度
``` perl
vi /etc/login.defs
PASS_MAX_DAYS=90
# 设置口令有效期为90（天）
PASS_MIN_LEN 8
# 设置口令最小长度为8
```

### 设置密码复杂度
``` perl
vi /etc/pam.d/login
# 增加如下内容
password required pam_cracklib.so difok=5 dcredit=-1 retry=1
# 新密码与旧密码必须有5个不同字符，必须有一个数字，一次失败后passwd程序返回错误信息
```

### 设置缺省密码长度限制
``` perl
vi /etc/login.defs
PASS_MIN_LEN 8
```

### SSH安全配置
``` perl
vi /etc/ssh/sshd_config
Procotol2         # 强制使用协议
PermitRootLoginno # 不允许root远程登录
```

### 设置历史命令记录增加时间戳
``` perl
vi /etc/profile
HISTTIMEFORMAT="%Y:%:M:%D %H-%m-%s"

export=HISTTIMEFORMAT

# 修改用户当前目录下.bash_history文件权限
chmod 600 .bash_history
```

### sysctl内核参数配置最佳实践
``` perl
sysctl -p
#----------------------------------------------
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 4398046511104
kernel.shmall = 1073741824
fs.file-max = 6815744
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_tw_reuse = 1
net.core.rmem_max = 4194304
net.core.wmem_max = 1048576
net.core.rmem_default = 262144
net.core.wmem_default = 262144
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_max_syn_backlog = 4096
net.core.netdev_max_backlog = 2048
net.ipv4.tcp_fin_timeout = 15
net.core.somaxconn = 1024
vm.swappiness = 10
vm.zone_reclaim_mode = 0
vm.extra_free_kbytes = 2097152

vm.nr_hugepages = 20550
#----------------------------------------------

net.ipv4.ip_forward = 0                          # 禁止IP包转发
net.ipv4.conf.default.rp_filter = 0              # 避免伪装IP的攻击
kernel.shmmax = 4398046511104                    # 对mysql不生效，oracle用
kernel.shmall = 1073741824                       # 对mysql不生效，oracle用，不能小于SGA

# 查看os系统页的大小
getconf PAGESIZE
#----------------------------------------------
4096
#----------------------------------------------

# 需要共享内存页数(假设SGA=16G)
16G/4KB=16*1024*1024/4=16777216KB/4=4194304
```

### 调整操作系统预读
``` perl
lspci -nn | grep -i sata

# 查看预读设置
cat /sys/block/sda/queue/read_ahead_kb 
4096
# 这个参数没有准确的答案，针对不同的应用场景有不同的合理值，这个不大适合统一标准化。
```

### 文件系统优化
``` perl
# 格式化标准命令
mkfs.xfs -f -iattr=2 -l lazy-count=1,sectsize=4096 -b size=4096 -d sectsize=4096 -L data /dev/<XXX>
# mount标准命令
mount -o rw,noatime,nodiratime,noikeep,nobarrier,allocsize=100M,attr2,largeio,inode64,swalloc /dev/<XXX> /apps
```

### 参考：
[Linux I/O调度](http://www.cnblogs.com/sopc-mc/archive/2011/10/09/2204858.html)