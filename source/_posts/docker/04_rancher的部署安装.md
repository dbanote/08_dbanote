---
title: Docker学习笔记_04 Rancher的部署安装(编排选用K8S)
date: 2018-01-12
tags:
- docker
- rancher
- kubernetes
categories:
- Docker学习笔记
---

## 为什么要使用Rancher
Rancher是一个开源的企业级容器管理平台。通过Rancher，企业再也不必自己使用一系列的开源软件去从头搭建容器服务平台。Rancher提供了在生产环境中使用的管理Docker和Kubernetes的全栈化容器部署与管理平台。

![](http://p2c0rtsgc.bkt.clouddn.com/0111_rancher_01.png)

Rancher的官方文档：https://rancher.com/docs/rancher/latest/en/

下图展示了Rancher的主要组件和功能：
![](http://p2c0rtsgc.bkt.clouddn.com/0111_rancher_02.png)

<!-- more -->

## 版本选择
版本选择参照官方文档：[supported version of Docker](https://rancher.com/docs/rancher/v1.6/en/hosts/#supported-docker-versions)

![](http://p2c0rtsgc.bkt.clouddn.com/0116_rancher_01.png)

**根据上图，本文选用以下符合要求的最新版本**

| 软件 | 版本 | github项目地址 |
|-|-|
| rancher | 1.6.14  | https://github.com/rancher/rancher |
| docker | 17.03.2-ce | https://github.com/docker/docker-ce |
| kubernetes | 1.8.5 | https://github.com/kubernetes/kubernetes |

## 系统准备

| 主机名 | IP地址  | 用途说明 | 操作系统 |
|-|-|
| rancher1 | 10.245.231.119  | server管理节点 | Ubuntu 16.04.3 LTS |
| docker201 | 10.245.231.201 | agent工作节点 | Ubuntu 16.04.3 LTS |
| docker201 | 10.245.231.202 | agent工作节点 | Ubuntu 16.04.3 LTS |

## 禁用IPV6
``` perl
sudo vi /etc/sysctl.d/99-sysctl.conf
# 添加以下内容
#------------------------------------------
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
#------------------------------------------

sudo sysctl -p
```

## 禁用虚拟内存swap
``` perl
# 不重启电脑，禁用启用swap，立刻生效
sudo swapoff -a

# 启用命令
sudo swapon -a

# 查看交换分区的状态
sudo free -m

# 永久禁用Swap，在/etc/fstab中swap分区这行前加 #
sudo vi /etc/fstab
#------------------------------------------
# /dev/mapper/docker201--vg-swap_1 none            swap    sw              0       0
#------------------------------------------
```

## 安装指定版本的docker
在以上三台主机上，安装指定版本的docker，参照[docker的部署安装-Ubuntu](/2018/01/11/docker/03_docker%E7%9A%84%E9%83%A8%E7%BD%B2%E5%AE%89%E8%A3%85-Ubuntu/#安装DOCKER-CE)并[配置阿里镜像加速器](/2018/01/11/docker/03_docker%E7%9A%84%E9%83%A8%E7%BD%B2%E5%AE%89%E8%A3%85-Ubuntu/#使用阿里镜像加速器) 

查看docker信息能看到以下内容：
``` perl
sudo docker info 
#------------------------------------------
Registry Mirrors:
 https://lwdxerv9.mirror.aliyuncs.com
#------------------------------------------

# 重启docker服务
sudo systemctl daemon-reload
sudo systemctl restart docke

# 检查防火墙是否启动
systemctl list-unit-files | grep enable | grep ufw
ufw.service                                enabled 
```


## 安装Rancher管理端
在rancher1上操作
``` perl
sudo docker run -d --name rancher -v /etc/localtime:/etc/localtime -v /opt/rancher/mysql:/var/lib/mysql --restart=unless-stopped -p 8080:8080 rancher/server

$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                              NAMES
4aba45218a7a        rancher/server      "/usr/bin/entry /u..."   6 minutes ago       Up 5 minutes        3306/tcp, 0.0.0.0:8080->8080/tcp   rancher

$ sudo docker exec -it rancher /bin/bash

root@4aba45218a7a:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 15:03 ?        00:00:00 /usr/bin/s6-svscan /service
root         7     1  0 15:03 ?        00:00:00 s6-supervise cattle
root         8     1  0 15:03 ?        00:00:00 s6-supervise graphite_exporter
root         9     1  0 15:03 ?        00:00:00 s6-supervise mysql
root        10     7 22 15:03 ?        00:01:41 java -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled -Xms128m -Xmx2g -XX:+HeapDumpOnOutOfMemoryErr
mysql      113     9  1 15:03 ?        00:00:07 /usr/sbin/mysqld --basedir=/usr --datadir=/var/lib/mysql --plugin-dir=/usr/lib/mysql/plugin --user=mysql
root       249    10  0 15:03 ?        00:00:03 rancher-catalog-service --config repo.json --refresh-interval 300
root       284    10  0 15:04 ?        00:00:01 websocket-proxy
root       298    10  1 15:04 ?        00:00:04 go-machine-service
root       306    10  0 15:04 ?        00:00:00 secrets-api server --enc-key-path .
root       307    10  0 15:04 ?        00:00:00 webhook-service
root       325    10  0 15:04 ?        00:00:00 rancher-compose-executor
root       332    10  0 15:04 ?        00:00:00 rancher-auth-service --auth-config-file authConfigFile.txt
root       425     0  0 15:07 ?        00:00:00 /bin/bash
root       440     0  0 15:07 ?        00:00:00 /bin/bash
root       507     0  0 15:10 ?        00:00:00 /bin/bash
root       523   507  0 15:10 ?        00:00:00 ps -ef

# 使用以下命令可以查看rancher容器日志
sudo docker logs -f rancher
```

访问 http://10.245.231.119:8080
![](http://p2c0rtsgc.bkt.clouddn.com/0116_rancher_08.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0116_rancher_09.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0116_rancher_03.png)

## 配置环境
进入环境管理，准备添加环境模板
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_01.png)
点击添加环境模板
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_02.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_03.png)

点击编辑设置后，在弹出的页面中，更改如下几个参数：
Private Registry for Add-Ons and Pod infra Container Images(修改私有仓库地址)：registry.cn-shenzhen.aliyuncs.com
Image namespace for Add-ons and Pod infra Container Images(修改AAONS组件命名空间)：rancher_cn
Image namespace for kubernetes-helm (修改kubernetes-helm命名空间)：rancher_cn
Pod Infra Container Image (修改默认的pause镜像名):rancher_cn/pause-amd64:3.0
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_04.png)

参数设置完，点击页面下方的设置按钮返回环境模板编辑页面，保持环境模板其他参数不变，点击页面下方的创建按钮。
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_05.png)

回到环境管理，点击添加环境
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_06.png)

选择新创建的环境模板后点击创建
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_07.png)

这样就用刚刚创建的模板创建了一个K8S环境
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_08.png)

设置为默认环境并切换到此环境
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_09.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_10.png)


## 添加主机
因为是第一次添加主机，系统会要求你确认节点注册地址，我们直接点击保存。
![](http://p2c0rtsgc.bkt.clouddn.com/0116_rancher_05.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0116_rancher_06.png)

依次登陆各个宿主机，执行5里面的脚本。

如果需要把rancher1加为宿主机，那么需要在4里面填写管理端和宿主主机之间互通的内网IP地址，建议不要添加rancher1为宿主机，方便后续做rancher server集群高可用。
``` bash
sudo docker run --rm --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/rancher:/var/lib/rancher rancher/agent:v1.2.9 http://10.245.231.119:8080/v1/scripts/48FC1D196FBDD2EC666B:1514678400000:p9K0flJKUBDcKxpnOvJNoXAadU

# 在所有的机器上执行，开启所需防火墙端口
sudo ufw allow 500/udp
sudo ufw allow 4500/udp
```

经历较长时间后，添加成功
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_11.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_12.png)

按此办法可添加多台宿主机

k8s仪表板
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_14.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_15.png)

k8s命令行
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_16.png)

k8s基础设施（点击+可查看详细内容）
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_13.png)

主机视图
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_17.png)

正常状态，非系统容器应该有14个
![](http://p2c0rtsgc.bkt.clouddn.com/0117_rancher_18.png)


> 因k8s管理复杂，本文只是演示如何配置k8s，后续文章的环境将继续使用rancher默认的Cattle，并且将docker升级到最新版本

## 参考文档
* [使用RancherServer:v1.6.12部署K8S-v1.8.3](http://blog.csdn.net/csdn_duomaomao/article/details/78746034)
* [原生加速中国区Kubernetes安装](https://www.cnrancher.com/kubernetes-installation/)
