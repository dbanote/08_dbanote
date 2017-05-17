---
title: Oracle 12c特性解读-容器数据库和灾备-02 数据库环境初始化
date: 2017-05-08
tags:
- oracle
- 12c
---

## 静默安装12c软件
### 系统检查
``` perl
uname –a
cat /etc/redhat-release
grep MemTotal /proc/meminfo
grep SwapTotal /proc/meminfo
df -hT
```

<!-- more -->

### 安装前的准备
``` perl
# 安装依赖包
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

# 配置hosts
vi /etc/hosts

# 禁用防火墙和selinux
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
vi /etc/selinux/config
#-----------------------------------------------
SELINUX=disabled
#-----------------------------------------------

# 内核参数
sysctl -p
#------------------------------------------------------------------------------------
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104      # 物理内存的50%、60%  如(8G*1024*1024*1024*1024)*50%=
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

sysctl -p

# 配置limit
vi /etc/security/limits.conf
#------------------------------------------------------------------------------------
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc     2047
oracle   hard   nproc     16384
oracle   soft   stack     10240
oracle   hard   stack     32768
oracle   hard   memlock   134217728
oracle   soft   memlock   134217728
#------------------------------------------------------------------------------------

# 大页设置
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /proc/meminfo | grep Huge
vi /etc/sysctl.conf 
#------------------------------------------------------------------------------------
vm.nr_hugepages=2564
# sga_target=5G  大页的内存要大于SGA大小  计算使用Oracle官网提供的脚本，文档 ID 401749.1
#------------------------------------------------------------------------------------

# 安装常用工具
yum -y install kernel-devel readline-devel.x86_64  tigervnc-server.x86_64 lrzsz gcc bc
yum -y localinstall --nogpgcheck jdk-8u131-linux-x64.rpm
tar xvpf rlwrap-0.42.tar.gz 
cd rlwrap-0.42
./configure
make && make install
```

### 用户环境配置
``` perl
id oracle
#------------------------------------------------------------------------------------
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba)
#------------------------------------------------------------------------------------

echo "oracle" | passwd --stdin oracle

chattr +i /etc/passwd /etc/shadow    # 增加安全性
chattr -i /etc/passwd /etc/shadow    # 需要修改时执行

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

### 配置静默安装响应文件
``` perl
cd database/response/
cp db_install.rsp db_install_new.rsp
vi db_install_new.rsp
#------------------------------------------------------------------------------------
oracle.install.option=INSTALL_DB_SWONLY
UNIX_GROUP_NAME=dba
INVENTORY_LOCATION=/u01/app/oraInventory
ORACLE_HOME=/u01/app/oracle/product/12.2.0/db_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.OSDBA_GROUP=dba
oracle.install.db.OSOPER_GROUP=dba
oracle.install.db.OSBACKUPDBA_GROUP=dba
oracle.install.db.OSDGDBA_GROUP=dba
oracle.install.db.OSKMDBA_GROUP=dba
oracle.install.db.OSRACDBA_GROUP=dba
DECLINE_SECURITY_UPDATES=true
#------------------------------------------------------------------------------------
```

### 静默安装12c软件
``` perl
cd ..
./runInstaller -silent -responseFile /tmp/database/response/db_install_new.rsp -ignoreSysPrereqs
#------------------------------------------------------------------------------------
As a root user, execute the following script(s):
        1. /u01/app/oraInventory/orainstRoot.sh
        2. /u01/app/oracle/product/12.2.0/db_1/root.sh   # 在root用户下执行这两个脚本

Successfully Setup Software.
#------------------------------------------------------------------------------------
```

## dbca静默安装12c容器数据库
``` perl
dbca -silent -createDatabase -templateName $ORACLE_HOME/assistants/dbca/templates/General_Purpose.dbc \
-gdbname orcl12 -sid orcl12 -characterSet ZHS16GBK \
-createAsContainerDatabase true \
-sysPassword Center08 -systemPassword Center08
```

## 对于环境安装过程中的问题总结


### 卸载已有的数据库软件
#### 使用deinstall工具
``` perl
cd $ORACLE_HOME/deinstall
./deinstall
```

#### 手工删除
``` perl
ep -ef | grep smon

# shutdown数据库
sqlplus / as sysdba
shutdown abort

# 删除相应的数据文件和参数文件（可选）
$ORACLE_BASE/oradata
$ORACLE_HOME/dbs

# 删除相应的配置文件(root用户)，多实例时作相应的修改
/etc/oratab
/etc/oraInst.loc

# 删除ORACLE_HOME
# 修改oraInventory
```

### ORACLE的补丁CPU/PSU
* Oracle CPU的全称是Critical Patch Update, Oracle对于其产品每个季度发行一次安全补丁包，通常是为了修复产品中的安全隐患。
* Oracle PSU的全称是Patch Set Update，Oracle对于其产品每个季度发行一次的补丁包，包含了bug的修复。Oracle选取被用户下载数量多，且被验证过具有较低风险的补丁放入到每个季度的PSU中。在每个PSU中不但包含Bug的修复而且还包含了最新的CPU。PSU通常随CPU一起发布。
* 2016年1月份Oracle推出了对 PSU、SPU、Bundle Patch 新的命名规则。新的命名规则为（以11.2.0.4为例）：11.2.0.4.YYMMDD。YYMMDD 会对应为patch （PSU、SPU、Bundle）发布的具体日期。具体可以参见MOS文档（Doc ID 1454618.1）

### 数据库软件克隆安装方法
``` perl
# 解压压缩包后(物理拷贝)进行编译(relink)，方法如下
## 10g
$ORACLE_HOME/oui/bin/runInstaller -clone -silent -ignorePreReq ORACLE_HOME="$ORACLE_HOME" ORACLE_HOME_NAME="OraDb12c_home1"

## 11g,12c
cd $ORACLE_HOME/clone/bin
perl clone.pl ORACLE_BASE=$ORACLE_BASE ORACLE_HOME=$ORACLE_HOME ORACLE_HOME_NAME=OraDb12c_home1
```

### 建库模板
模板类型主要有seed和noseed两种，主要的区别在于是否包含数据文件。简单来说，seed就是从RMAN备份中恢复数据库，由于是恢复所以也不能做其他更多的定制修改，但是最大的特点是创建速度快，OLTP和OLAP的模板就属于seed模板类型；而“定制数据库”则属于nonseed模板，不包含数据文件，需要使用create database命令创建数据库，创建时间要稍微长一些，对于大部分系统业务来说，需要根据自己的需求来选择合适的模板类型。