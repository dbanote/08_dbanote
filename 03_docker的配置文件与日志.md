---
title: Docker学习笔记_03 Docker的配置文件与日志
date: 2017-12-13
tags:
- docker
categories:
- Docker学习笔记
---

### Docker的配置文件
``` perl
vi /usr/lib/systemd/system/docker.service
```

<!-- more -->
### Docker的日志文件
``` perl
# Docker的日志文件写入到 /var/log/messages中
vi /var/log/messages
#---------------------------------------------------------------------------------------------------------
......
Dec 12 18:15:48 localhost systemd: Stopping Docker Application Container Engine...
Dec 12 18:15:48 localhost dockerd: time="2017-12-12T18:15:48.483010203+08:00" level=info msg="Processing signal 'terminated'"
Dec 12 18:15:48 localhost dockerd: time="2017-12-12T18:15:48.507640035+08:00" level=info msg="stopping containerd after receiving terminated"
Dec 12 18:15:49 localhost systemd: Starting Docker Application Container Engine...
Dec 12 18:15:49 localhost dockerd: time="2017-12-12T18:15:49.546613547+08:00" level=info msg="libcontainerd: new containerd process, pid: 23932"
Dec 12 18:15:50 localhost dockerd: time="2017-12-12T18:15:50.554965941+08:00" level=info msg="[graphdriver] using prior storage driver: overlay"
Dec 12 18:15:50 localhost dockerd: time="2017-12-12T18:15:50.557292503+08:00" level=info msg="Graph migration to content-addressability took 0.00 seconds"
Dec 12 18:15:50 localhost dockerd: time="2017-12-12T18:15:50.558168972+08:00" level=info msg="Loading containers: start."
Dec 12 18:15:50 localhost firewalld[776]: WARNING: COMMAND_FAILED: '/usr/sbin/iptables -w2 -t nat -D OUTPUT -m addrtype --dst-type LOCAL -j DOCKER' failed: iptables: No chain/target/match by that name.
Dec 12 18:15:50 localhost firewalld[776]: WARNING: COMMAND_FAILED: '/usr/sbin/iptables -w2 -t nat -D PREROUTING' failed: iptables: Bad rule (does a matching rule exist in that chain?).
Dec 12 18:15:50 localhost firewalld[776]: WARNING: COMMAND_FAILED: '/usr/sbin/iptables -w2 -t nat -D OUTPUT' failed: iptables: Bad rule (does a matching rule exist in that chain?).
......
#---------------------------------------------------------------------------------------------------------

# 关闭防火墙后重启docker上面的警告没有了
systemctl stop firewalld
systemctl restart docker

vi /var/log/messages
#---------------------------------------------------------------------------------------------------------
Dec 13 14:15:40 localhost systemd: Starting Docker Application Container Engine...
Dec 13 14:15:40 localhost dockerd: time="2017-12-13T14:15:40.521527114+08:00" level=info msg="libcontainerd: new containerd process, pid: 27895"
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.529055212+08:00" level=info msg="[graphdriver] using prior storage driver: overlay"
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.531992896+08:00" level=info msg="Graph migration to content-addressability took 0.00 seconds"
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.532699034+08:00" level=info msg="Loading containers: start."
Dec 13 14:15:41 localhost kernel: nf_conntrack version 0.5.0 (65536 buckets, 262144 max)
Dec 13 14:15:41 localhost kernel: ip_tables: (C) 2000-2006 Netfilter Core Team
Dec 13 14:15:41 localhost kernel: ctnetlink v0.93: registering with nfnetlink.
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.666796126+08:00" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used to set a preferred IP address"
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.703565410+08:00" level=info msg="Loading containers: done."
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.711707839+08:00" level=info msg="Docker daemon" commit=19e2cf6 graphdriver(s)=overlay version=17.09.1-ce
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.712164063+08:00" level=info msg="Daemon has completed initialization"
Dec 13 14:15:41 localhost dockerd: time="2017-12-13T14:15:41.721846967+08:00" level=info msg="API listen on /var/run/docker.sock"
Dec 13 14:15:41 localhost systemd: Started Docker Application Container Engine.
#---------------------------------------------------------------------------------------------------------

# 建议docker运行后，开启以下命令，实时观察docker运行状态
tail -f /var/log/messages | grep docker
```


>CentOS-7 中介绍了 firewalld，firewall的底层是使用iptables进行数据过滤，建立在iptables之上，这可能会与 Docker 产生冲突。
当 firewalld 启动或者重启的时候，将会从 iptables 中移除 DOCKER 的规则，从而影响了 Docker 的正常工作。
当你使用的是 Systemd 的时候， firewalld 会在 Docker 之前启动，但是如果你在 Docker 启动之后再启动 或者重启 firewalld ，你就需要重启 Docker 进程了。