---
title: MySQL DBA从小白到大神实战-08 MySQL监控系统之Zabbix
tags:
- mysql
- zabbix
---

## 部署zabbix监控
### 简介
zabbix（音同 zbix）是一个基于WEB界面的提供分布式系统监视以及网络监视功能的企业级的开源解决方案。
zabbix能监视各种网络参数，保证服务器系统的安全运营；并提供灵活的通知机制以让系统管理员快速定位/解决存在的各种问题。
zabbix由2部分构成，zabbix server与可选组件zabbix agent。zabbix server可以通过SNMP，zabbix agent，ping，端口监视等方法提供对远程服务器/网络状态的监视，数据收集等功能，它可以运行在Linux，Solaris，HP-UX，AIX，Free BSD，Open BSD，OS X等平台上。

为了使用最新3.2版本的Zabbix，本次部署选用的Linux OS是最新的CentOS7.3（php版本为5.4）。 
CentOS6.X和CentOS7.X的安装和使用均有较大变化，为便于今后的学习和工作，下面将CentOS7.3的安装和基本配置也记录到本文中。

### 安装CentOS7.3
详见：[CentOS7.3安装图解](https://dbanote.github.io/2017/03/18/linux/20170318_CentOS7.3安装图解/)
<!-- more -->

### CentOS7.X系统的基本配置和使用
``` perl
# 查看系统版本
cat /etc/redhat-release
-----------------------------------------------
CentOS Linux release 7.3.1611 (Core) 
-----------------------------------------------

# 配置IP
vi /etc/sysconfig/network-scripts/ifcfg-ens160
-----------------------------------------------
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=no
NAME=ens160
UUID=38e983c8-b6ad-40cf-9592-3a3652cf8c6d
DEVICE=ens160
ONBOOT=yes
IPADDR=10.245.231.202
PREFIX=24
GATEWAY=10.245.231.254
-----------------------------------------------

# 重启网络，使配置生效
service network restart

# 配置主机名
hostnamectl status
-----------------------------------------------
   Static hostname: localhost.localdomain
         Icon name: computer-vm
           Chassis: vm
        Machine ID: 5f66fc1b61374082a7e6e595fc7669c5
           Boot ID: 269679da89c54fd4a736c6968b8644f3
    Virtualization: vmware
  Operating System: CentOS Linux 7 (Core)
       CPE OS Name: cpe:/o:centos:centos:7
            Kernel: Linux 3.10.0-514.el7.x86_64
      Architecture: x86-64
-----------------------------------------------
hostnamectl --static set-hostname zabbix

# 禁用SELINUX
setenforce 0
vi /etc/selinux/config
-----------------------------------------------
SELINUX=disabled
-----------------------------------------------

# 禁用防火墙
systemctl disable firewalld
systemctl status firewalld

# 设置内核参数(mysql需要)
vi /etc/sysctl.d/99-sysctl.conf
-----------------------------------------------
kernel.shmmax = 2199023255552
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
-----------------------------------------------

# 使内核参数生效
sysctl -p

# 常用CentOS7.x命令
## 查看所有系统服务状态
systemctl list-unit-files -t service 

## 查看系统启动以来的message信息(类似dmesg)
journalctl -b

# 查看网络配置
ip a
-----------------------------------------------
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:50:56:a0:1d:b1 brd ff:ff:ff:ff:ff:ff
    inet 10.245.231.202/24 brd 10.245.231.255 scope global ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fea0:1db1/64 scope link 
       valid_lft forever preferred_lft forever
-----------------------------------------------
```


### 源码编译安装mysql
参见博文：[MySQL DBA从小白到大神实战-02 MySQL标准化、自动化部署](https://dbanote.github.io/2017/01/26/mysql/课程学习/02-MySQL-DBA从小白到大神实战/#详细描述mysql编译安装的过程截图安装步骤)

### 安装和配置zabbix server
#### 建zabbix用户、组和安装目录
``` perl
groupadd zabbix
useradd -g zabbix zabbix
echo "zabbix" | passwd --stdin zabbix

mkdir -p /u01/zabbix
chown -R zabbix:zabbix /u01/zabbix
```

#### 安装编译基础依赖包
``` perl
mkdir /media/disk
mount /dev/cdrom /media/disk

cd /etc/yum.repos.d/
-----------------------------------------------
cp CentOS-Base.repo CentOS-Base.repo.bak
vi /etc/yum.repos.d/CentOS-Base.repo
[base]
name=CentOS-$releasever - Base
baseurl=file:///media/disk
gpgcheck=0
enabled=1
-----------------------------------------------

yum -y install wget unzip libxml2 libxml2-devel httpd php php-mysql php-common php-gd \
php-odbc php-pear curl curl-devel net-snmp net-snmp-devel perl-DBI php-xml ntpdate  \
zlib-devel glibc-devel curl-devel gcc automake openssl-devel net-snmp-devel rpm-devel \
lrzsz ncurses-devel readline-devel

# http://mirrors.sohu.com/centos/7.3.1611/os/x86_64/Packages/ 下载以下包并上传到服务器/tmp目录下
-rw-r--r--. 1 root root 126576 Mar 14 11:58 libidn-devel-1.28-4.el7.x86_64.rpm
-rw-r--r--. 1 root root 176988 Mar 14 13:46 OpenIPMI-2.0.19-15.el7.x86_64.rpm
-rw-r--r--. 1 root root 136452 Mar 14 11:58 OpenIPMI-devel-2.0.19-15.el7.x86_64.rpm
-rw-r--r--. 1 root root 513864 Mar 14 13:42 OpenIPMI-libs-2.0.19-15.el7.x86_64.rpm
-rw-r--r--. 1 root root  15440 Mar 14 13:45 OpenIPMI-modalias-2.0.19-15.el7.x86_64.rpm
-rw-r--r--. 1 root root  58488 Mar 14 11:57 php-bcmath-5.4.16-42.el7.x86_64.rpm
-rw-r--r--. 1 root root 516620 Mar 14 11:57 php-mbstring-5.4.16-42.el7.x86_64.rpm

# 手动安装包
cd /tmp
rpm -ivh libidn*.rpm
rpm -ivh php*.rpm
rpm -ivh OpenIPMI-modalias-2.0.19-15.el7.x86_64.rpm
rpm -ivh OpenIPMI-libs-2.0.19-15.el7.x86_64.rpm
rpm -ivh OpenIPMI-2.0.19-15.el7.x86_64.rpm
rpm -ivh OpenIPMI-devel-2.0.19-15.el7.x86_64.rpm

# 查看php和apache版本
php --version
-----------------------------------------------
PHP 5.4.16 (cli) (built: Nov  6 2016 00:29:02) 
Copyright (c) 1997-2013 The PHP Group
Zend Engine v2.4.0, Copyright (c) 1998-2013 Zend Technologies
-----------------------------------------------

httpd -version
-----------------------------------------------
Server version: Apache/2.4.6 (CentOS)
Server built:   Nov 14 2016 18:04:44
-----------------------------------------------
```

#### 源码编译安装zabbix
从官岗下载zabbix源码包：[http://www.zabbix.com/download ](http://www.zabbix.com/download)  
上传到服务器/tmp目录后解压
``` perl
cd /tmp/
tar -zxvf zabbix-3.2.4.tar.gz
```

编译安装，相关步骤参考[官网安装手册](https://www.zabbix.com/documentation/3.2/manual/installation/install)
``` perl
cd /tmp/zabbix-3.2.4
./configure --prefix=/u01/zabbix --enable-server --enable-agent \
--with-mysql=/u01/mysql/bin/mysql_config \
--with-net-snmp --with-libcurl --with-libxml2

make && make install
```

#### 创建所需数据库并授权用户
``` perl
su - mysql
mysql

create database zabbix default character set utf8;
grant all privileges on *.* to 'zabbix'@'%' identified by 'zabbix';
grant all privileges on *.* to 'zabbix'@'localhost' identified by 'zabbix';

show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
| zabbix             |
+--------------------+

use zabbix
select database();
+------------+
| database() |
+------------+
| zabbix     |
+------------+
```

#### 导入zabbix定义的表结构和数据
``` perl
source /tmp/zabbix-3.2.4/database/mysql/schema.sql
source /tmp/zabbix-3.2.4/database/mysql/data.sql
source /tmp/zabbix-3.2.4/database/mysql/images.sql
```

#### 修改相关配置文件
``` perl
# root用户下
# 修改zabbix server配置文件
vi /u01/zabbix/etc/zabbix_server.conf
-----------------------------------------------
ListenPort=10051
DBHost=10.245.231.202
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
DBPort=3306
PidFile=/tmp/zabbix_server.pid
-----------------------------------------------

# 修改php配置文件
cp /etc/php.ini /etc/php.ini.zabbixbak
sed -i 's/max_execution_time = 30/max_execution_time = 300/g' /etc/php.ini
sed -i '/date.timezone =/a\date.timezone = Asia/Shanghai' /etc/php.ini
sed -i '/max_input_time =/s/60/300/' /etc/php.ini
sed -i '/mbstring.func_overload = 0/a\mbstring.func_overload = 1' /etc/php.ini
sed -i '/post_max_size =/s/8M/32M/' /etc/php.ini

# 修改apache配置文件
sed -i '/ServerName/a\ServerName zabbix:80' /etc/httpd/conf/httpd.conf
```

#### 拷贝zabbix的web界面程序文件
```
mkdir /var/www/html/zabbix
cp -rf /tmp/zabbix-3.2.4/frontends/php/* /var/www/html/zabbix
chown -R apache.apache /var/www/html/zabbix
```

#### 启动apache和zabbix server
``` perl
systemctl enable httpd.service
systemctl start httpd.service
systemctl status httpd.service
-----------------------------------------------
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2017-03-16 15:12:59 CST; 15s ago
     Docs: man:httpd(8)
           man:apachectl(8)
  Process: 762 ExecStop=/bin/kill -WINCH ${MAINPID} (code=exited, status=0/SUCCESS)
 Main PID: 773 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─773 /usr/sbin/httpd -DFOREGROUND
           ├─775 /usr/sbin/httpd -DFOREGROUND
           ├─776 /usr/sbin/httpd -DFOREGROUND
           ├─777 /usr/sbin/httpd -DFOREGROUND
           ├─779 /usr/sbin/httpd -DFOREGROUND
           └─780 /usr/sbin/httpd -DFOREGROUND

Mar 16 15:12:59 zabbix systemd[1]: Starting The Apache HTTP Server...
Mar 16 15:12:59 zabbix systemd[1]: Started The Apache HTTP Server.
-----------------------------------------------

# 启动zabbix server
/u01/zabbix/sbin/zabbix_server -c /u01/zabbix/etc/zabbix_server.conf

# 添加到系统启动
cd /tmp/zabbix-3.2.4
cp misc/init.d/fedora/core/zabbix_server /etc/init.d/zabbix_server
cp misc/init.d/fedora/core/zabbix_agentd /etc/init.d/zabbix_agentd

chkconfig --add zabbix_server
chkconfig --add zabbix_agentd
chkconfig zabbix_server on
chkconfig zabbix_agentd on

vi /etc/init.d/zabbix_server
-----------------------------------------------
BASEDIR=/u01/zabbix
-----------------------------------------------

vi /etc/init.d/zabbix_agentd
-----------------------------------------------
BASEDIR=/u01/zabbix
-----------------------------------------------

# 启动zabbix_server
systemctl start zabbix_server.service
```

#### 通过WEB界面安装配置zabbix
在浏览器中输入：[http://10.245.231.202/zabbix](http://10.245.231.202/zabbix)，打开图形界面 -> 下一步
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_01.png)
确保所有项都是OK状态 -> 下一步
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_02.png)
配置数据库连接 -> 下一步
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_03.png)
配置zabbix server信息 -> 下一步
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_04.png)
确认信息 -> 下一步
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_05.png)
下载配置文件，按要求保存到指定位置
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_06.png)
配置文件内容如下：
``` perl
vi /var/www/html/zabbix/conf/zabbix.conf.php
-----------------------------------------------
<?php
// Zabbix GUI configuration file.
global $DB;

$DB['TYPE']     = 'MYSQL';
$DB['SERVER']   = '10.245.231.202';
$DB['PORT']     = '3306';
$DB['DATABASE'] = 'zabbix';
$DB['USER']     = 'zabbix';
$DB['PASSWORD'] = 'zabbix';

// Schema name. Used for IBM DB2 and PostgreSQL.
$DB['SCHEMA'] = '';

$ZBX_SERVER      = '10.245.231.202';
$ZBX_SERVER_PORT = '10051';
$ZBX_SERVER_NAME = 'zabbix';

$IMAGE_FORMAT_DEFAULT = IMAGE_FORMAT_PNG;

systemctl restart httpd
The default user name is Admin, password zabbix.
-----------------------------------------------
```

结束安装
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_07.png)
默认的用户名是`admin`，密码是`zabbix`
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_08.png)
登陆后，可以将显示语言设置为中文(英文好的，不建议修改，而且zabbix汉化不是很完全和准确)
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_09.png)
选择中文，点update
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_10.png)

### 配置启动zabbix agent
修改配置文件并启动agent
``` perl
vi /u01/zabbix/etc/zabbix_agentd.conf
# 修改以下内容
-----------------------------------------------
Server=10.245.231.202
ServerActive=10.245.231.202
Hostname=zabbix
-----------------------------------------------
# 启动
/u01/zabbix/sbin/zabbix_agentd
# 或者
systemctl restart zabbix_agentd.service
```

在zabbix web界面中配置主机中，点击【停用的】
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_11.png)
确定，启用主机
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_12.png)
这时状态已变为【启用】，但可用状态异常
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_16.png)
点击主机名
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_14.png)
将主机IP变更为实际的 -> 更新
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_15.png)
重新刷新页面，稍等一会可用性状态正常了
![](http://oligvdnzp.bkt.clouddn.com/0316_zabbix_17.png)


## 监控MySQL实例状态
> **监控思路：**
在要监控的MySQL服务器上安装zabbix agent，然后在zabbix主机web界面里配置要监控的MySQL服务器的信息，添加zabbix自带的Template App MySQL模版。

### 在监控的mysql服务器上安装并启动zabbix agent
``` perl
# 系统环境
yum -y install gcc* make vim
setenforce 0
systemctl stop firewalld.service

# 添加zabbix帐号
groupadd zabbix
useradd zabbix -g zabbix -s /sbin/nologin

# 配置hosts(非必须)
vi /etc/hosts
-----------------------------------------------
10.245.231.201 mysql
10.245.231.202 zabbix
-----------------------------------------------

# 上传rpm包到/tmp目录后开始安装
rpm -ivh zabbix-agent-3.2.4-1.el6.x86_64.rpm

# 也可以按以下步骤在解压后的源码目录编译安装
./configure --enable-agent
make && make install
cp misc/init.d/fedora/core/zabbix_agentd /etc/init.d/zabbix_agentd
chkconfig --add zabbix_agentd
chkconfig zabbix_agentd on

# 修改agent配置文件(注意下面都是zabbix_agent rpm包安装后的目录，源码安装的话请注意修改)
vi /etc/zabbix/zabbix_agentd.conf
-----------------------------------------------
Server=10.245.231.202
ServerActive=10.245.231.202
Hostname=zabbix
-----------------------------------------------

# 启动agent
/usr/sbin/zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf 
```

### 建立mysql host groups组
模板是zabbix系统提供的，进入zabbix web后台，Configuration-->Hosts groups-->点击Create host group-->选择template选项卡，选择模板TemplateApp MySQL，Templdate OS Linux，Group name设置成Mysql_Servers，最后点击add
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_01.png)

### 建立hosts
configuration-->hosts-->Create hosts，然后按图配置
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_02.png)
选择template选项卡-->select-->选择模板Template App MySQL,Templdate OS Linux，最后点击下边的Add按钮
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_03.png)
确认模板已添加，点Add按钮
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_04.png)
稍等一会，确保状态和可用性都正常
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_05.png)

### 使用自带模板监控mysql
在监控的mysql服务器上给zabbix agent配置数据库账号
``` perl
GRANT PROCESS,SUPER,REPLICATION CLIENT ON *.* TO zabbix@'127.0.0.1' IDENTIFIED BY 'zabbix123456';
GRANT PROCESS,SUPER,REPLICATION CLIENT ON *.* TO zabbix@'localhost' IDENTIFIED BY 'zabbix123456';
```
创建/etc/zabbix/.my.cnf,并在里面写入mysql服务器的密码
``` perl
vi /etc/zabbix/.my.cnf
-----------------------------------------------
[mysql]
user=zabbix
password=zabbix123456

[mysqladmin]
user=zabbix
password=zabbix123456
-----------------------------------------------
```
修改监控mysql相关的用户参数文件
``` perl
cd /etc/zabbix/zabbix_agentd.d/
vi userparameter_mysql.conf
# 修改以下方面：
-----------------------------------------------
HOME=/var/lib/zabbix    替换成     HOME=/etc/zabbix
mysql                   补全路径   /u01/mysql/bin/mysql
-----------------------------------------------

# 重启zabbix agent
ps -ef | grep 'zabbix_agentd' | grep -v grep | awk '{print $2}' | xargs kill -9
/usr/sbin/zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf
```
然后在zabbix-server服务器上测试是否可以获取到mysql的status信息
``` perl
cd /u01/zabbix/bin/
./zabbix_get -s 10.245.231.201 -p10050 -k mysql.ping
1

./zabbix_get -s 10.245.231.201 -p10050 -k mysql.version
/u01/mysql/bin/mysql  Ver 14.14 Distrib 5.6.35, for Linux (x86_64) using  EditLine wrapper

./zabbix_get -s 10.245.231.201 -p10050 -k mysql.status[Com_insert]
3
```

zabbix自带的mysql模板共可监测以下14项
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_06.png)
按下图可以查看监控mysql的情况，也可以图形化显示
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_07.png)
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_08.png)

### 自定义zabbix模板监控mysql
点击Configuration-->Templates-->create template，参照下图中配置
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_09.png)
点Applications-->Create Applications
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_10.png)
输入监控的应用名称Mysql
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_11.png)
添加成功，点【Items】，然后点击左上的【Create Items】
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_12.png)
按下图所示填写后点下方add
![](http://oligvdnzp.bkt.clouddn.com/0317_zabbix_mysql_13.png)
创建监控脚本文件并参数文件
``` perl
mkdir /etc/zabbix/scripts
vi /etc/zabbix/scripts/check_mysql_status_3306.sh
-----------------------------------------------
#!/bin/bash
host=localhost
username=zabbix
password=zabbix
port=3306
CHECK_TIME=3
#mysql  is working MYSQL_IS_OK is 1 , mysql down MYSQL_IS_OK is 0
MYSQL_IS_OK=1
function check_mysql_status (){
    $MYSQL -u$username -p"$password" -P$port -e "select user();" >/dev/null 2>&1
    if [ $? = 0 ] ;then
    MYSQL_IS_OK=1
    else
    MYSQL_IS_OK=0
    fi
    return $MYSQL_IS_OK 
}
while [ $CHECK_TIME -ne 0 ]
do
    let "CHECK_TIME -= 1"
    
    check_mysql_status
if [ $MYSQL_IS_OK = 1 ] ; then
    CHECK_TIME=0
    echo 0
    exit 0
fi
if [ $MYSQL_IS_OK -eq 0 ] &&  [ $CHECK_TIME -eq 0 ]
then
    echo 1
    exit 1
fi
sleep 3
done
-----------------------------------------------

chmod +x /etc/zabbix/scripts/check_mysql_status_3306.sh
cd /etc/zabbix/zabbix_agentd.d/
vi userparameter_mysql.conf
# 添加以下内容
-----------------------------------------------
UserParameter=my3306.check_mysql_status,/etc/zabbix/scripts/check_mysql_status_3306.sh
-----------------------------------------------
```

参考：[Centos7.2下搭建Zabbix3.2](http://www.iyunv.com/forum.php?mod=viewthread&tid=351608&highlight=zabbix) 
