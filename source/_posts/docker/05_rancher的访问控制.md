---
title: Docker学习笔记_05 Rancher的访问控制
date: 2018-01-20
tags:
- docker
- rancher
categories:
- Docker学习笔记
---


## 设置rancher访问控制
### 启用本地验证
安装完rancher之后，正式使用前一定要配置管理员权限
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_01.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_02.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_03.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_05.png)

设置后，以后登陆Rancher server就需要输入用户名和密码了
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_04.png)

### 创建普通帐户
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_06.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_07.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_08.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0118_rancher_09.png)

### 设置环境访问控制
![](http://p2c0rtsgc.bkt.clouddn.com/0207_rancher_06.png)
角色说明[官方文档](http://rancher.com/docs/rancher/v1.6/zh/environments/#成员角色)中有详细说明
![](http://p2c0rtsgc.bkt.clouddn.com/0207_rancher_07.png)