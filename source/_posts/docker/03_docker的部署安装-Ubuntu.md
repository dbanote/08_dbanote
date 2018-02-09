---
title: Docker学习笔记_03 Docker的部署安装(Ubuntu)
date: 2018-01-11
tags:
- docker
categories:
- Docker学习笔记
---

## 为什么生产环境中Docker要部署在Ubuntu上
原来为了兼容企业级应用，之前学习Docker选用了Centos7做为部署安装Docker的系统平台，但在阅读官方文档后发现，Centos7下默认的Docker存储驱动在生产环境中部署是会有一些问题。
![](http://p2c0rtsgc.bkt.clouddn.com/0111_docker_01.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0111_docker_02.png)

详细可参考这篇文章：http://blog.csdn.net/csdn_duomaomao/article/details/78499639

<!-- more -->

## 在Ubuntu上安装Docker

### 查看系统版本
```perl
uname -a
Linux rancher1 4.4.0-87-generic #110-Ubuntu SMP Tue Jul 18 12:55:35 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux

cat /proc/version
[sudo] password for ubuntu: 
Linux version 4.4.0-87-generic (buildd@lcy01-31) (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.4) ) #110-Ubuntu SMP Tue Jul 18 12:55:35 UTC 2017

lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.3 LTS
Release:        16.04
Codename:       xenial
```

### 移除旧的Docker版本
``` perl
sudo apt-get remove docker docker-engine docker.io
```

### 安装必要的一些系统工具
``` perl
# Update the apt package index
sudo apt-get update

# Install packages to allow apt to use a repository over HTTPS
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
```

### 安装GPG证书并使用阿里源
``` perl
# Add Docker’s official GPG key
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

sudo apt-key fingerprint 0EBFCD88
#--------------------------------------------------------------
pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
#--------------------------------------------------------------

sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
```

### 安装DOCKER CE
为了之后配置使用rancher和kubernetes，这里安装指定版本的docker(17.03.2-ce)
版本的选择参考：https://rancher.com/docs/rancher/v1.6/en/hosts/#supported-docker-versions
``` perl
# 查找可用Docker-CE的版本
sudo apt-cache madison docker-ce
#---------------------------------------------------------------------------------------------------------
 docker-ce | 17.12.0~ce-0~ubuntu | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.09.1~ce-0~ubuntu | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.09.0~ce-0~ubuntu | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.06.2~ce-0~ubuntu | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.06.1~ce-0~ubuntu | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.06.0~ce-0~ubuntu | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.03.2~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.03.1~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
 docker-ce | 17.03.0~ce-0~ubuntu-xenial | http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#---------------------------------------------------------------------------------------------------------

# 安装指定版本的Docker-CE: (VERSION 例如上面的 17.03.2~ce-0~ubuntu-xenial)
sudo apt-get -y install docker-ce=17.03.2~ce-0~ubuntu-xenial

# The Docker daemon starts automatically.
```

### 查看docker版本及运行信息
``` perl
sudo docker version
#-----------------------------------
Client:
 Version:      17.03.2-ce
 API version:  1.27
 Go version:   go1.7.5
 Git commit:   f5ec1e2
 Built:        Tue Jun 27 03:35:14 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.03.2-ce
 API version:  1.27 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   f5ec1e2
 Built:        Tue Jun 27 03:35:14 2017
 OS/Arch:      linux/amd64
 Experimental: false
#-----------------------------------

sudo docker info
#-----------------------------------
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 17.03.2-ce
Storage Driver: aufs
 Root Dir: /var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 0
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins: 
 Volume: local
 Network: bridge host macvlan null overlay
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Init Binary: docker-init
containerd version: 4ab9917febca54791c5f071a9d1f404867857fcc
runc version: 54296cf40ad8143b62dbcaa1d90e520a2136ddfe
init version: 949e6fa
Security Options:
 apparmor
 seccomp
  Profile: default
Kernel Version: 4.4.0-87-generic
Operating System: Ubuntu 16.04.3 LTS
OSType: linux
Architecture: x86_64
CPUs: 4
Total Memory: 3.859 GiB
Name: rancher1
ID: 7IDE:C4U7:SPVA:AQPL:KCPV:XCWC:ESIT:JAQL:NIVJ:FTZB:2WNH:BTZJ
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Experimental: false
Insecure Registries:
 127.0.0.0/8
Live Restore Enabled: false

WARNING: No swap limit support
#-----------------------------------
# WARNING: No swap limit support 这个警告不影响正常使用
```

### 设置不更新指定版本的docker-ce
``` perl
sudo echo "docker-ce hold" | sudo dpkg --set-selections

# 查询当前系统内所有软件包的状态，命令为：
sudo dpkg --get-selections | more

# 查询当前系统被锁定不更新的软件包状态(hold)，命令为
sudo dpkg --get-selections | grep hold
```

## 卸载Docker CE
``` perl
sudo apt-get purge docker-ce
sudo rm -rf /var/lib/docker
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
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxxxx.mirror.aliyuncs.com"]
}
EOF

# 重启docker
sudo /etc/init.d/docker restart
```


## 更改ubuntu源为阿里源
``` perl
sudo vi /etc/apt/sources.list

# 输入ggdG删除所有内容，添加以下内容
#----------------------------------------------------------------------
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse  
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse  
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse  
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse  
deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse  
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse  
deb http://archive.canonical.com/ubuntu/ xenial partner  
#----------------------------------------------------------------------

# 更新获取阿里云软件源 提供的软件列表
sudo apt-get update

# 更新软件
sudo apt-get -y upgrade

# 公司局域网域名解析有问题，手动配置hosts
sudo vi /etc/hosts
#----------------------------------------------------------------------
221.206.129.236  mirrors.aliyun.com
91.189.88.162  archive.ubuntu.com
91.189.92.150  archive.canonical.com
#----------------------------------------------------------------------
```