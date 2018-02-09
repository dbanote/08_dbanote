---
title: Docker学习笔记_02 Docker的部署安装(CentOS)
date: 2017-12-12
tags:
- docker
categories:
- Docker学习笔记
---

## 环境准备
### 操作系统需求
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_15.png)
<!-- more -->

**为兼容企业级应用，学习选用Centos7做为部署安装Docker的系统平台**
```perl
# 通过以下命令可查看系统版本和内核版本等信息
cat /etc/redhat-release
#-----------------------------------
CentOS Linux release 7.4.1708 (Core) 
#-----------------------------------

uname -a
#---------------------------------------------------------------------------------------------------------
Linux docker01 3.10.0-693.el7.x86_64 #1 SMP Tue Aug 22 21:09:27 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
#---------------------------------------------------------------------------------------------------------

cat /etc/os-release
#------------------------------------------
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
#------------------------------------------
```


### 更换默认的yum源
Centos默认的yun源在国外，速度很慢有时间也无法访问
```perl
yum repolist
#------------------------------------------------------------------------------------------------------------------------------
Loaded plugins: fastestmirror
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=stock error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; Unknown error"
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=extras&infra=stock error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; Unknown error"
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=updates&infra=stock error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; Unknown error"
repo id                                                         repo name                                                          status
base/7/x86_64                                                   CentOS-7 - Base                                                    0
extras/7/x86_64                                                 CentOS-7 - Extras                                                  0
updates/7/x86_64                                                CentOS-7 - Updates                                                 0
repolist: 0
#------------------------------------------------------------------------------------------------------------------------------

# 公司内服务器域名解析总有问题，时好时不好，很烦，这里直接用hosts做解析
vi /etc/hosts
#-------------------------------------
221.206.129.236  mirrors.aliyun.com
#-------------------------------------

# 更换成aliyun yum源
cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 编译CentOS-Base.repo，把带mirrors.aliyuncs.com的行都删除
vi /etc/yum.repos.d/CentOS-Base.repo

# 运行以下命令生成缓存
yum clean all
yum makecache

# 查看已启用的repo，确保centos-extras repository是启用了，安装docker时需要
yum repolist
#--------------------------------------------------------------------------
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
repo id                                  repo name                                                        status
base/7/x86_64                            CentOS-7 - Base - mirrors.aliyun.com                             9,591
extras/7/x86_64                          CentOS-7 - Extras - mirrors.aliyun.com                             284
updates/7/x86_64                         CentOS-7 - Updates - mirrors.aliyun.com                          1,540
repolist: 11,415
#--------------------------------------------------------------------------
```

### 更新系统(可选)
```perl
yum update
```

### 删除docker旧版本
```perl
# 有旧版本的docker话，可以用下面命令删除
yum remove docker docker-common docker-selinux docker-engine
```

## 安装 Docker CE
```perl
yum install -y yum-utils device-mapper-persistent-data lvm2
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce
#--------------------------------------------------------------------------
.......
Installed:
  docker-ce.x86_64 0:17.09.1.ce-1.el7.centos                                                                                                                                    

Dependency Installed:
  audit-libs-python.x86_64 0:2.7.6-3.el7
  checkpolicy.x86_64 0:2.5-4.el7
  container-selinux.noarch 2:2.28-1.git85ce147.el7
  libcgroup.x86_64 0:0.41-13.el7
  libseccomp.x86_64 0:2.3.1-3.el7
  libsemanage-python.x86_64 0:2.5-8.el7
  policycoreutils-python.x86_64 0:2.5-17.1.el7
  python-IPy.noarch 0:0.75-6.el7    
  setools-libs.x86_64 0:3.3.8-1.1.el7       
......
#--------------------------------------------------------------------------
```

若需要安装指定的版本时，可参照以下命令
``` perl
# 根据需要选择是否开启edge和test repositories
yum-config-manager --enable docker-ce-edge
yum-config-manager --enable docker-ce-test
## 禁用命令
yum-config-manager --disable docker-ce-edge

## 安装指定的版本
yum list docker-ce --showduplicates | sort -r
#--------------------------------------------------------------------------
Loading mirror speeds from cached hostfile
Loaded plugins: fastestmirror
Installed Packages
docker-ce.x86_64            17.09.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.09.1.ce-1.el7.centos            @docker-ce-stable
docker-ce.x86_64            17.09.0.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.06.2.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.06.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.06.0.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.03.2.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.03.1.ce-1.el7.centos            docker-ce-stable 
docker-ce.x86_64            17.03.0.ce-1.el7.centos            docker-ce-stable 
Available Packages
#--------------------------------------------------------------------------

yum install docker-ce-17.06.1.ce
```

## 启动docker
```perl
systemctl start docker

# 查看docker的版本信息
docker version
#--------------------------------------------------------------------------
Client:
 Version:      17.09.1-ce       # 客户端版本
 API version:  1.32
 Go version:   go1.8.3
 Git commit:   19e2cf6
 Built:        Thu Dec  7 22:23:40 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.09.1-ce       # 服务端版本
 API version:  1.32 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   19e2cf6
 Built:        Thu Dec  7 22:25:03 2017
 OS/Arch:      linux/amd64
 Experimental: false
#--------------------------------------------------------------------------

# 查看网络信息
ip addr
#--------------------------------------------------------------------------
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP qlen 1000
    link/ether 00:50:56:ab:4c:50 brd ff:ff:ff:ff:ff:ff
    inet 10.240.4.185/24 brd 10.240.4.255 scope global ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feab:4c50/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN  # docker0 虚拟网桥
    link/ether 02:42:72:ac:05:bf brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:72ff:feac:5bf/64 scope link 
       valid_lft forever preferred_lft forever
#--------------------------------------------------------------------------

systemctl list-unit-files | grep docker
#--------------------------------------------------------------------------
docker.service                                disabled
#--------------------------------------------------------------------------

# 设置成自启服务
systemctl enable docker.service

# 查看状态
systemctl status docker
#--------------------------------------------------------------------------
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2017-12-12 17:24:31 CST; 13min ago
     Docs: https://docs.docker.com
 Main PID: 23479 (dockerd)
   CGroup: /system.slice/docker.service
           ├─23479 /usr/bin/dockerd
           └─23490 docker-containerd -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libco...

Dec 12 17:24:29 docker01 dockerd[23479]: time="2017-12-12T17:24:29.594209004+08:00" level=info msg="libcontainerd: new containerd process, pid: 23490"
Dec 12 17:24:30 docker01 dockerd[23479]: time="2017-12-12T17:24:30.596093094+08:00" level=warning msg="failed to rename /var/lib/docker/tmp for background deletio...chronously"
Dec 12 17:24:30 docker01 dockerd[23479]: time="2017-12-12T17:24:30.654014669+08:00" level=info msg="Graph migration to content-addressability took 0.00 seconds"
Dec 12 17:24:30 docker01 dockerd[23479]: time="2017-12-12T17:24:30.654714697+08:00" level=info msg="Loading containers: start."
Dec 12 17:24:30 docker01 dockerd[23479]: time="2017-12-12T17:24:30.852920366+08:00" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17...IP address"
Dec 12 17:24:30 docker01 dockerd[23479]: time="2017-12-12T17:24:30.996504508+08:00" level=info msg="Loading containers: done."
Dec 12 17:24:31 docker01 dockerd[23479]: time="2017-12-12T17:24:31.004149257+08:00" level=info msg="Docker daemon" commit=19e2cf6 graphdriver(s)=overlay version=17.09.1-ce
Dec 12 17:24:31 docker01 dockerd[23479]: time="2017-12-12T17:24:31.004282017+08:00" level=info msg="Daemon has completed initialization"
Dec 12 17:24:31 docker01 dockerd[23479]: time="2017-12-12T17:24:31.015479108+08:00" level=info msg="API listen on /var/run/docker.sock"
Dec 12 17:24:31 docker01 systemd[1]: Started Docker Application Container Engine.
Hint: Some lines were ellipsized, use -l to show in full.
#--------------------------------------------------------------------------
```

运行hello-world image验证docker安装是否成功
```perl
docker run hello-world
#--------------------------------------------------------------------------
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete 
Digest: sha256:be0cd392e45be79ffeffa6b05338b98ebb16c87b255f48e297ec7f98e123905c
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
#--------------------------------------------------------------------------
```

## 升级和卸载docker
``` perl
# 升级
yum -y upgrade docker-ce

# 卸载
yum remove docker-ce

# 删除Images, containers, volumes, or customized configuration files 
rm -rf /var/lib/docker
```


## 使用阿里镜像加速器
使用阿里云专属加速器加快获取Docker官方镜像，否则在国内速度会慢到你无法忍受哒。步骤如下：
1. 免费注册一个阿里云账号 [www.aliyun.com](https://www.aliyun.com/)
2. 进入加速器页面 [https://cr.console.aliyun.com/#/accelerator ](https://cr.console.aliyun.com/#/accelerator)
3. 选择`镜像加速器`
![](http://oligvdnzp.bkt.clouddn.com/1213_docker_02.png)

按图中进行相关配置
``` bash
# 下面的xxxxx要替换成你的专属加速器的地址哦
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxxx.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```


## 参考文档：
* [官方文档：Get Docker CE for CentOS](https://docs.docker.com/engine/installation/linux/docker-ce/centos/)