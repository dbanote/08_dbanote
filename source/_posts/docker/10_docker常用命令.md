---
title: Docker学习笔记_10 Docker常用命令
date: 2018-01-22
tags:
- docker
categories:
- Docker学习笔记
---

## docker search
命令示例：
``` perl
docker search oracle
#---------------------------------------------------------------------------------------------------------
NAME                                DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
oraclelinux                         Oracle Linux is an open-source operating s...   404                 [OK]                
frolvlad/alpine-oraclejdk8          The smallest Docker image with OracleJDK 8...   270                                     [OK]
alexeiled/docker-oracle-xe-11g      This is a working (hopefully) Oracle XE 11...   223                                     [OK]
sath89/oracle-12c                   Oracle Standard Edition 12c Release 1 with...   222                                     [OK]
sath89/oracle-xe-11g                Oracle xe 11g with database files mount su...   136                                     [OK]
isuper/java-oracle                  This repository contains all java releases...   55                                      [OK]
jaspeen/oracle-11g                  Docker image for Oracle 11g database            55                                      [OK]
oracle/glassfish                    GlassFish Java EE Application Server on Or...   31                                      [OK]
oracle/openjdk                      Docker images containing OpenJDK Oracle Linux   26                                      [OK]
airdock/oracle-jdk                  Docker Image for Oracle Java SDK (8 and 7)...   23                                      [OK]
wnameless/oracle-xe-11g             Dockerfile of Oracle Database Express Edit...   22                                      [OK]
ingensi/oracle-jdk                  Official Oracle JDK installed on centos.        21                                      [OK]
cogniteev/oracle-java               Oracle JDK 6, 7, 8, and 9 based on Ubuntu ...   20                                      [OK]
n3ziniuka5/ubuntu-oracle-jdk        Ubuntu with Oracle JDK. Check tags for ver...   14                                      [OK]
oracle/nosql                        Oracle NoSQL on a Docker Image with Oracle...   13                                      [OK]
sgrio/java-oracle                   Docker images of Java 7/8 provided by Orac...   7                                       [OK]
andreptb/oracle-java                Debian Jessie based image with Oracle JDK ...   7                                       [OK]
openweb/oracle-tomcat               A fork off of Official tomcat image with O...   7                                       [OK]
flurdy/oracle-java7                 Base image containing Oracle's Java 7 JDK       5                                       [OK]
martinseeler/oracle-server-jre      Oracle's Java 8 as 61 MB Docker container.      4                                       [OK]
davidcaste/debian-oracle-java       Oracle Java 8 (and 7) over Debian Jessie        3                                       [OK]
teradatalabs/centos6-java8-oracle   Docker image of CentOS 6 with Oracle JDK 8...   3                                       
spansari/nodejs-oracledb            nodejs with oracledb installed globally on...   1                                       
publicisworldwide/oracle-core       This is the core image based on Oracle Lin...   1                                       [OK]
sigma/nimbus-lock-oracle                                                            0                                       [OK]
#---------------------------------------------------------------------------------------------------------
```

<!-- more -->

也可以去[https://hub.docker.com/ ](https://hub.docker.com/)里去搜索，更直观，以及dockfile等内容
![](http://oligvdnzp.bkt.clouddn.com/1213_docker_03.png)
![](http://oligvdnzp.bkt.clouddn.com/1213_docker_04.png)
![](http://oligvdnzp.bkt.clouddn.com/1213_docker_05.png)


## docker pull
pull镜像，命令示例：
``` perl
docker pull java
#---------------------------------------------------------------------------------------------------------
Using default tag: latest
latest: Pulling from library/java
5040bd298390: Pull complete 
fce5728aad85: Pull complete 
76610ec20bf5: Pull complete 
60170fec2151: Pull complete 
e98f73de8f0d: Pull complete 
11f7af24ed9c: Pull complete 
49e2d6393f32: Downloading [============================>                      ]  72.97MB/130.1MB   # 分层Downloading、Extracting images
bb9cdec9c7f3: Verifying Checksum

Digest: sha256:c1ff613e8ba25833d2e1940da0940c3824f03f802c449f3d1815a66b7f8c0e9d
Status: Downloaded newer image for java:latest      # 完成
#---------------------------------------------------------------------------------------------------------

# sgrio/java-oracle latest: pointed to sgrio/java-oracle:server_jre_9
docker pull sgrio/java-oracle

# pull java 8 image用以下命令
docker pull sgrio/java-oracle:server_jre_8
```

## docker images
查看本地已经装载的镜像，命令示例：
``` perl
docker images
#---------------------------------------------------------------------------------------------------------
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
sgrio/java-oracle   server_jre_8        6d10e204fc36        2 weeks ago         292MB
hello-world         latest              f2a91732366c        3 weeks ago         1.85kB
ubuntu              latest              20c44cd7596f        3 weeks ago         123MB
lwieske/java-8      jdk-8u131           0ff57dff9137        5 months ago        590MB
java                latest              d23bdf5b1b1b        11 months ago       643MB
#---------------------------------------------------------------------------------------------------------
```

## docker run
docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...]
* docker run后面追加-d=true或者-d，那么容器将会运行在后台模式。
* docker exec来进入到到该容器中，或者attach重新连接容器的会话
* 进行交互式操作（例如Shell脚本），须使用-i -t参数同容器进行数据交互
* docker run时没有指定--name，那么deamon会自动生成一个随机字符串UUID
* Docker时有自动化的需求，可以将containerID输出到指定的文件中（PIDfile）： --cidfile="" 
* Docker的容器是没有特权的，例如不能在容器中再启动一个容器。这是因为默认情况下容器是不能访问任何其它设备的。但是通过"privileged"，容器就拥有了访问任何其它设备的权限。

命令示例：
```perl
docker run -it java java -version
#---------------------------------------------------------------------------------------------------------
openjdk version "1.8.0_111"
OpenJDK Runtime Environment (build 1.8.0_111-8u111-b14-2~bpo8+1-b14)
OpenJDK 64-Bit Server VM (build 25.111-b14, mixed mode)
#---------------------------------------------------------------------------------------------------------

docker run -it sgrio/java-oracle:server_jre_8 java -version
#---------------------------------------------------------------------------------------------------------
java version "1.8.0_152"
Java(TM) SE Runtime Environment (build 1.8.0_152-b16)
Java HotSpot(TM) 64-Bit Server VM (build 25.152-b16, mixed mode)
#---------------------------------------------------------------------------------------------------------

docker run -it java uname
#---------------------------------------------------------------------------------------------------------
Linux
#---------------------------------------------------------------------------------------------------------

docker run -it java ps
#---------------------------------------------------------------------------------------------------------
  PID TTY          TIME CMD
    1 pts/0    00:00:00 ps
#---------------------------------------------------------------------------------------------------------

docker run -it java ip addr
#---------------------------------------------------------------------------------------------------------
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
52: eth0@if53: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever    
#---------------------------------------------------------------------------------------------------------


docker run -it java ip addr
#---------------------------------------------------------------------------------------------------------
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
58: eth0@if59: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
#---------------------------------------------------------------------------------------------------------       

docker run -it java env
#---------------------------------------------------------------------------------------------------------
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=126a22eb2c97
TERM=xterm
LANG=C.UTF-8
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
JAVA_VERSION=8u111
JAVA_DEBIAN_VERSION=8u111-b14-2~bpo8+1
CA_CERTIFICATES_JAVA_VERSION=20140324
HOME=/root
#---------------------------------------------------------------------------------------------------------

# docker run 里面的命令结束了，container就结束了
```


## 批量停止所有运行中的容器
``` bash
tee stop_all_container.sh <<-'EOF'
sudo docker ps | grep -v CONTAINER | awk '{print $1}'| xargs sudo docker stop
EOF

chmod +x stop_all_container.sh
```

## 批量删除所有容器
``` bash
tee rm_all_container.sh <<-'EOF'
sudo docker ps -a | grep -v CONTAINER | awk '{print $1}'| xargs sudo docker rm
EOF

chmod +x rm_all_container.sh
```