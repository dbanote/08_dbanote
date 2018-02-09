---
title: Docker学习笔记_01 Docker的相关概念
date: 2017-12-11
tags:
- docker
categories:
- Docker学习笔记
---

Docker就是虚拟化的一种轻量级替代技术。Docker的容器技术不依赖任何语言、框架或系统，可以将App变成一种标准化的、可移植的、自管理的组件，并脱离服务器硬件在任何主流系统中开发、调试和运行。

## Docker的核心技术
### Docker相关的核心技术之cgroups
Linux系统中经常有个需求就是希望能限制某个或者某些进程的分配资源。于是就出现了`cgroups`的概念，`cgroup`就是`controller group` ，在这个group中，有分配好的特定比例的cpu时间，IO时间，可用内存大小等。 `cgroups`是将任意进程进行分组化管理的Linux内核功能。最初由google的工程师提出，后来被整合进Linux内核中。

cgroups中的 重要概念是“子系统”，也就是资源控制器，每种子系统就是一个资源的分配器，比如cpu子系统是控制cpu时间分配的。首先挂载子系统，然后才有control group的。比如先挂载memory子系统，然后在memory子系统中创建一个cgroup节点，在这个节点中，将需要控制的进程id写入，并且将控制的属性写入，这就完成了内存的资源限制。

cgroups 被Linux内核支持，有得天独厚的性能优势，发展势头迅猛。在很多领域可以取代虚拟化技术分割资源。cgroup默认有诸多资源组，可以限制几乎所有服务器上的资源：cpu mem iops,iobandwide,net,device acess等
<!-- more -->

### Docker相关的核心技术之LXC
LXC是Linux containers的简称，是一种基于容器的操作系统层级的虚拟化技术。借助于namespace的隔离机制和cgroup限额功能，LXC提供了一套统一的API和工具来建立和管理container。LXC跟其他操作系统层次的虚拟化技术相比，最大的优势在于LXC被整合进内核，不用单独为内核打补丁。

LXC 旨在提供一个共享kernel的 OS 级虚拟化方法，在执行时不用重复加载Kernel, 且container的kernel与host共享，因此可以大大加快container的 启动过程，并显著减少内存消耗，容器在提供隔离的同时，还通过共享这些资源节省开销，这意味着容器比真正的虚拟化的开销要小得多。 在实际测试中，基于LXC的虚拟化方法的IO和CPU性能几乎接近 baremetal 的性能。 

虽然容器所使用的这种类型的隔离总的来说非常强大，然而是不是像运行在hypervisor上的虚拟机那么强壮仍具有争议性。如果内核停止，那么所有的容器就会停止运行。

* 性能方面：LXC>>KVM>>XEN
* 内存利用率：LXC>>KVM>>XEN
* 隔离程度： XEN>>KVM>>LXC

### Docker相关的核心技术之AUFS
什么是AUFS? AuFS是一个能透明覆盖一或多个现有文件系统的层状文件系统。 支持将不同目录挂载到同一个虚拟文件系统下，可以把不同的目录联合在一起，组成一个单一的目录。这种是一种虚拟的文件系统，文件系统不用格式化，直接挂载即可。

Docker一直在用AuFS作为容器的文件系统。当一个进程需要修改一个文件时，AuFS创建该文件的一个副本。AuFS可以把多层合并成文件系统的单层表示。这个过程称为写入复制`copy on write`。

AuFS允许Docker把某些镜像作为容器的基础。例如，你可能有一个可以作为很多不同容器的基础的CentOS系统镜像。多亏AuFS，只要一个CentOS镜像的副本就够了，这样既节省了存储和内存，也保证更快速的容器部署。

使用AuFS的另一个好处是Docker的版本容器镜像能力。每个新版本都是一个与之前版本的简单差异改动，有效地保持镜像文件最小化。但，这也意味着你总是要有一个记录该容器从一个版本到另一个版本改动的审计跟踪。

## Docker的核心组件
### Docker Image
* Docker Image是一个极度精简版的Linux程序运行环境，比如vi这种基本的工具没有，官网的Java镜像包括的东西更少，除非是镜像叠加方式的，如Centos+Java7
* Docker Image是需要定制化Build的一个“安装包”，包括基础镜像+应用的二进制部署包
* Docker Image内不建议有运行期需要修改的配置文件
* Dockerfile用来创建一个自定义的image,包含了用户指定的软件依赖等。当前目录下包含Dockerfile,使用命令build来创建新的image
* Docker Image的最佳实践之一是尽量重用和使用网上公开的基础镜像

### Docker Container
* Docker Container是Image的实例，共享内核
* Docker Container里可以运行不同Os的Image，比如Ubuntu的或者Centos
* Docker Container不建议内部开启一个SSHD服务，1.3版本后新增了docker exec命令进入容器排查问题。
* Docker Container没有IP地址，通常不会有服务端口暴露，是一个封闭的“盒子/沙箱”

**Docker Container的生命周期**
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_03.png)

### Docker Daemon
* Docker Daemon是创建和运行Container的Linux守护进程，也是Docker最主要的核心组件
* Docker Daemon 可以理解为Docker Container的Container
* Docker Daemon可以绑定本地端口并提供Rest API服务，用来远程访问和控制

### Docker Registry/Hub
Docker之所以这么吸引人，除了它的新颖的技术外，围绕官方Registry（Docker Hub）的生态圈也是相当吸引人眼球的地方。在Docker Hub上你可以很轻松下载到大量已经容器化好的应用镜像，即拉即用。这些镜像中，有些是Docker官方维护的，更多的是众多开发者自发上传分享的。而且你还可以在Docker Hub中绑定你的代码托管系统（目前支持Github和Bitbucket）配置自动生成镜像功能，这样Docker Hub会在你代码更新时自动生成对应的Docker镜像。
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_04.png)

### Docker 核心组件的关系
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_05.png)


## Docker原理之App打包
LXC的基础上, Docker额外提供的Feature包括：标准统一的打包部署运行方案

为了最大化重用Image，加快运行速度，减少内存和磁盘footprint, Docker container运行时所构造的运行环境，实际上是由具有依赖关系的多个Layer组成的。例如一个apache的运行环境可能是在基础的rootfs image的基础上，叠加了包含例如Emacs等各种工具的image，再叠加包含apache及其相关依赖library的image，这些image由AUFS文件系统加载合并到统一路径中，以只读的方式存在，最后再叠加加载一层可写的空白的Layer用作记录对当前运行环境所作的修改。

有了层级化的Image做基础，理想中，不同的APP就可以既可能的共用底层文件系统，相关依赖工具等，同一个APP的不同实例也可以实现共用绝大多数数据，进而以copy on write的形式维护自己的那一份修改过的数据等.
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_01.png)

## Docker全生命周期开发模式
Docker正在迅速改变云计算领域的运作规则，并彻底颠覆云技术的发展前景。从持续集成/持续交付到微服务、开源协作乃至DevOps，Docker一路走来已经给应用程序开发生命周期以及云工程技术实践带来了巨大变革。
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_02.png)

## Docker CE和Docker EE
Docker从17.03之后有两个可用版本：Community Edition (CE)社区版 and Enterprise Edition (EE) 企业版。社区版全功能免费使用，企业版强调安全，增加图形化管理等增强功能。功能对比如下图：
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_07.png)

两个版本支持的平台如下：
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_08.png)
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_09.png)

## Docker版本命名规则
从2017年3月1号开始，Docker的版本命名发生如下变化：
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_06.png)
Docker 会每月发布一个 edge 版本(17.03, 17.04, 17.05...)，每三个月发布一个 stable 版本(17.03, 17.06, 17.09, ...)，企业版(EE) 和 stable 版本号保持一致，但每个版本提供一年维护。
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_10.png)

## Containers vs. virtual machines
#### Virtual Machine diagram
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_13.png)

#### Container diagram
![](http://oligvdnzp.bkt.clouddn.com/1212_docker_14.png)
