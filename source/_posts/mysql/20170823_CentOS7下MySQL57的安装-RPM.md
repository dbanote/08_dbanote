---
title: CentOS7下MySQL5.7的安装-RPM方式
date: 2017-08-23
tags:
- mysql
---

## Installing MySQL on Linux Using RPM Packages
### 下载安装包
mysql下载地址：https://dev.mysql.com/downloads/mysql/
![](http://oligvdnzp.bkt.clouddn.com/0823_mysql_01.png)

<!-- more -->

### 环境配置
``` perl
# 禁用SELINUX
setenforce 0
vi /etc/selinux/config
#-----------------------------------------------
SELINUX=disabled
#-----------------------------------------------

# 禁用防火墙
systemctl disable firewalld
systemctl stop firewalld
systemctl status firewalld

# 设置内核参数(mysql需要)
vi /etc/sysctl.d/99-sysctl.conf
#-----------------------------------------------
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
#-----------------------------------------------
# kernel.shmmax算法：修改为物理内容的50%、60%
# 4G:kernel.shmmax = (4G*1024*1024*1024*1024)*50% = 2199023255552

# 使内核参数生效
sysctl -p
```

### 使用RPM包安装mysql
``` perl
# 安装新版mysql之前，我们需要将系统自带的mariadb-lib卸载
rpm -qa | grep mariadb-lib
rpm -e --nodeps mariadb-libs-5.5.52-1.el7.x86_64

# 安装需要的工具包
mkdir /media/disk
mount /dev/cdrom /media/disk

cd /etc/yum.repos.d/
cp CentOS-Base.repo CentOS-Base.repo_bak
vi CentOS-Base.repo
[local]
name=CentOS-$releasever - Local
baseurl=file:///media/disk
gpgcheck=0
enabled=1

yum -y install lrzsz unzip

# 使用rz工具上传安装包后
cd /tmp
rz

tar xvpf  mysql-5.7.19-1.el7.x86_64.rpm-bundle.tar
mysql-community-embedded-devel-5.7.19-1.el7.x86_64.rpm
mysql-community-client-5.7.19-1.el7.x86_64.rpm
mysql-community-server-5.7.19-1.el7.x86_64.rpm
mysql-community-test-5.7.19-1.el7.x86_64.rpm
mysql-community-embedded-compat-5.7.19-1.el7.x86_64.rpm
mysql-community-minimal-debuginfo-5.7.19-1.el7.x86_64.rpm
mysql-community-server-minimal-5.7.19-1.el7.x86_64.rpm
mysql-community-libs-compat-5.7.19-1.el7.x86_64.rpm
mysql-community-common-5.7.19-1.el7.x86_64.rpm
mysql-community-embedded-5.7.19-1.el7.x86_64.rpm
mysql-community-devel-5.7.19-1.el7.x86_64.rpm
mysql-community-libs-5.7.19-1.el7.x86_64.rpm

# 把不需要安装的包删除
rm -rf mysql-community-embedded-devel-5.7.19-1.el7.x86_64.rpm
rm -rf mysql-community-test-5.7.19-1.el7.x86_64.rpm
rm -rf mysql-community-embedded-compat-5.7.19-1.el7.x86_64.rpm
rm -rf mysql-community-minimal-debuginfo-5.7.19-1.el7.x86_64.rpm
rm -rf mysql-community-server-minimal-5.7.19-1.el7.x86_64.rpm
rm -rf mysql-community-embedded-5.7.19-1.el7.x86_64.rpm
rm -rf mysql-community-devel-5.7.19-1.el7.x86_64.rpm

# 开始安装，依赖包会自动安装
yum install mysql-community-{server,client,common,libs}-*

yum install mysql-commercial-{server,client,common,libs}-*

```

### 使用RPM包标准安装mysql，相关文件在以下系统目录
|Files or Resources|Location|
| :---- | :----|
|Client programs and scripts| /usr/bin|
|mysqld server|/usr/sbin|
|Configuration file|/etc/my.cnf|
|Data directory|/var/lib/mysql|
|Error log file|For RHEL, Oracle Linux, CentOS or Fedora platforms: <br>/var/log/mysqld.log<br>For SLES: /var/log/mysql/mysqld.log|
|Value of secure_file_priv|/var/lib/mysql-files|
|System V init script|For RHEL, Oracle Linux, CentOS or Fedora platforms: <br>/etc/init.d/mysqld<br>For SLES: /etc/init.d/mysql|
|Systemd service|For RHEL, Oracle Linux, CentOS or Fedora platforms: mysqld<br>For SLES: mysql|
|Pid file|/var/run/mysql/mysqld.pid|
|Socket|/var/lib/mysql/mysql.sock|
|Keyring directory|/var/lib/mysql-keyring|
|Unix manual pages|/usr/share/man|
|Include (header) files|/usr/include/mysql|
|Libraries|/usr/lib/mysql|
|Miscellaneous support files <br>(for example, error messages, and character set files)|/usr/share/mysql|

### 启动mysql
``` perl
# 以下两个命令都可以
service mysqld start
systemctl start mysqld.service
```

### 查看并登陆修改初始的root密码
``` perl
grep 'temporary password' /var/log/mysqld.log
#-----------------------------------------------
2017-08-23T05:56:38.197060Z 1 [Note] A temporary password is generated for root@localhost: gi._hyg/f2oJ
#-----------------------------------------------

mysql -uroot -p

# 以下两个命令都可以修改root密码
alter user 'root'@'localhost' identified by 'Center08+';
set password for root@localhost = password('Center08+');
```

