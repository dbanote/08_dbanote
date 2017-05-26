---
title: 在DELL R730物理机上安装配置Oracle Linux 7.3图解
date: 2017-05-24
tags: 
- Linux
---

Oracle Linux 7.3（Oracle Linux 7 Update 3）在2016.11.10发布。Oracle Linux全称为Oracle Enterprise Linux，是由Oracle公司提供支持的企业级Linux发行。这是第一个包含UEK版本 4（UEK R4）的 Oracle Linux 7 ISO，新安装的 Oracle Linux 7 Update 3 将默认安装和引导 UEK R4 内核。
**显著更新：**
UEFI 安全引导支持 - 此更新允许您在已启用 UEFI 安全引导的系统上安装和使用 Oracle Linux 7，这在 Oracle Linux 7 Update 3 上完全支持。
<!-- more -->

## 系统安装
Oracle Linux 7.X较6.X版本变动较大，图形化安装也有较大变化，默认使用英文
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_01.png)
所有配置项都集中放在一起，根据需求，依次配置相关设置
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_10.png)
操作系统安装位置/磁盘配置
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_02.png)
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_03.png)
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_04.png)
如果不想设置efi分区，需要在服务器BIOS里关闭UEFI功能
![](http://oligvdnzp.bkt.clouddn.com/0523_dell_idrac_10.png)

网络/主机名配置
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_06.png)
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_07.png)
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_08.png)
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_09.png)
在配置项集中页，点开始安装
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_11.png)
设置root密码
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_12.png)
安装完后重启
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_13.png)
安装成功，登陆
![](http://oligvdnzp.bkt.clouddn.com/0524_DellR730_Oracle_7_3_install_14.png)