---
title: Oracle 12c特性解读-容器数据库和灾备-01 Oracle 12c体系结构
date: 2017-05-05
tags:
- oracle
- 12c
categories:
- Oracle 12c特性解读-容器数据库和灾备
---

## 安装部署12c
### 官网下载12cr2的安装包
下载地址：[http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html ](http://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_01.png)

有MOS帐号也可以去edelivery下载：[https://edelivery.oracle.com](https://edelivery.oracle.com)

<!-- more -->

### 系统环境要求RHEL6或者以上，Oracle Enterprise Linux也可以。
``` perl
cat /etc/redhat-release
#------------------------------------------------------------------------------------
Red Hat Enterprise Linux Server release 7.3 (Maipo)
#------------------------------------------------------------------------------------
```

### 使用图形方式安装部署，给出基本的步骤和错误总结
#### Hardware Checklist
``` perl
df -hT
#------------------------------------------------------------------------------------
Filesystem           Type   Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                     ext4    43G  3.2G   38G   8% /
tmpfs                tmpfs  3.9G   72K  3.9G   1% /dev/shm
/dev/sda1            ext4   477M   82M  366M  19% /boot
#------------------------------------------------------------------------------------

free
#------------------------------------------------------------------------------------
              total        used        free      shared  buff/cache   available
Mem:        8174072      124584     7913700        8676      135788     7973288
Swap:      16777212           0    16777212
#------------------------------------------------------------------------------------

cat /proc/cpuinfo

cat /sys/kernel/mm/transparent_hugepage/enabled
#------------------------------------------------------------------------------------
always madvise [never]      # 停用transparent hugepage
#------------------------------------------------------------------------------------
```

#### Operating System Checklist
``` perl
# 检查安装依赖包
mkdir /media/disk
mount /dev/cdrom /media/disk
cp /etc/yum.repos.d/public-yum-ol7.repo /etc/yum.repos.d/public-yum-ol7.repo_old
vi /etc/yum.repos.d/public-yum-ol7.repo
#------------------------------------------------------------------------------------
[ol7_latest]
name=Oracle Linux $releasever Latest ($basearch)
baseurl=file:///media/disk
gpgcheck=0
enabled=1
#------------------------------------------------------------------------------------

yum -y install oracle-database-server-12cR2-preinstall.x86_64

# 配置IP/HOSTS
vi /etc/sysconfig/network-scripts/ifcfg-ens160
#------------------------------------------------------------------------------------
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens160
UUID=198f69b3-91b8-4b19-9312-50b3a75820a7
DEVICE=ens160
ONBOOT=yes
IPADDR=10.245.231.205
PREFIX=24
GATEWAY=10.245.231.254
#------------------------------------------------------------------------------------

vi /etc/hosts
#------------------------------------------------------------------------------------
10.245.231.205 orcl12
#------------------------------------------------------------------------------------

# 配置主机名
hostnamectl --static set-hostname orcl12

# 禁用SELINUX
setenforce 0
vi /etc/selinux/config
#-----------------------------------------------
SELINUX=disabled
#-----------------------------------------------

# 禁用防火墙
systemctl stop firewalld
systemctl disable firewalld
systemctl status firewalld

# 配置内核参数
vi /etc/sysctl.conf 
#------------------------------------------------------------------------------------
vm.nr_hugepages=3072
# sga_target=5G  大页的内存要大于SGA大小  计算方法：5G+1G=6G=6*1024M / 2=3072
#------------------------------------------------------------------------------------

sysctl -p
#------------------------------------------------------------------------------------
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
#------------------------------------------------------------------------------------

vi /etc/security/limits.conf
#------------------------------------------------------------------------------------
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc     2047
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   hard   memlock   unlimited
oracle   soft   memlock   unlimited
#------------------------------------------------------------------------------------

# 安装常用工具
yum -y install kernel-devel readline-devel.x86_64  tigervnc-server.x86_64 lrzsz gcc
yum -y localinstall --nogpgcheck jdk-8u131-linux-x64.rpm
java -version
#------------------------------------------------------------------------------------
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
#------------------------------------------------------------------------------------

tar xvpf rlwrap-0.42.tar.gz 
cd rlwrap-0.42
./configure
make && make install
```

#### Oracle User Environment Configuration
``` perl
id oracle
#------------------------------------------------------------------------------------
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba)
#------------------------------------------------------------------------------------

echo "oracle" | passwd --stdin oracle

mkdir -p /u01/app/oracle
chown -R oracle:oinstall /u01
chmod -R 775 /u01/

vi /home/oracle/.bash_profile
#------------------------------------------------------------------------------------
export ORACLE_SID=orcl12
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/12.2.0/db_1
export PATH=$PATH:$HOME/bin:$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:/usr/local/bin
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
export NLS_DATE_FORMAT='yyyy/mm/dd hh24:mi:ss'

alias sqlplus='rlwrap sqlplus'
alias rman='rlwrap rman'
#------------------------------------------------------------------------------------
```

#### Installing and Configuring Oracle Database
``` perl
su - oracle

# 进入解压后的数据库安装文件目录，客户端打开xmanager
cd database
export DISPLAY=10.245.4.90:0.0
./runInstaller
```

先安装数据库软件
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_02.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_03.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_04.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_05.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_06.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_07.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_08.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_09.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_10.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_11.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_12.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_13.png)

创建数据库
```
dbca
```

![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_14.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_15.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_16.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_17.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_18.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_19.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_20.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_21.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_22.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_23.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_24.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_25.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_26.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_27.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_28.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_29.png)
![](http://oligvdnzp.bkt.clouddn.com/0505_oracle12c_install_30.png)

### 安装成功的基本检查
``` perl
[oracle@orcl12 database]$ sqlplus / as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Fri May 5 18:57:32 2017

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select name, cdb from v$database;

NAME      CDB
--------- ---
ORCL12    YES

SQL> set line 100
SQL> col NAME for a50
SQL> select con_id, name from v$containers;

    CON_ID NAME
---------- --------------------------------------------------
         1 CDB$ROOT
         2 PDB$SEED
         3 POS

SQL> col file_name for a80
SQL> select con_id, file_name from cdb_data_files order by 1;

    CON_ID FILE_NAME
---------- --------------------------------------------------------------------------------
         1 /u01/app/oracle/oradata/orcl12/system01.dbf
         1 /u01/app/oracle/oradata/orcl12/users01.dbf
         1 /u01/app/oracle/oradata/orcl12/undotbs01.dbf
         1 /u01/app/oracle/oradata/orcl12/sysaux01.dbf
         3 /u01/app/oracle/oradata/orcl12/pos/system01.dbf
         3 /u01/app/oracle/oradata/orcl12/pos/users01.dbf
         3 /u01/app/oracle/oradata/orcl12/pos/undotbs01.dbf
         3 /u01/app/oracle/oradata/orcl12/pos/sysaux01.dbf
```