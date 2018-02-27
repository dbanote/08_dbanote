---
title: Docker学习笔记_08使用Rancher pipeline搭建基于容器的CICD
date: 2018-02-27
tags:
- docker
- rancher
categories:
- Docker学习笔记
---

## CICD概述
* CI-持续集成(Continuous Integration)：频繁地将代码集成到主干的一种开发实践，每次集成都通过自动化的构建（包括编译，发布，自动化测试）来验证，从而尽早地发现集成错误。
* CD-持续部署(Continuous Deployment)：从代码提交，自动化完成测试、构建及到生产环境的部署

## 在Rancher中做CI/CD的方法
1. 配合第三方工具，Drone/Travis/Jenkins，配合webhook，rancher cli等触发部署更新 
2. 使用`Rancher pipeline`构建从源码提交到Rancher中应用部署的一套流水线

<!-- more -->
## Rancher pipeline的部署
Ranche Pipeline 是Rancher V1.6.13更新发布的新功能。所以如果不是V1.6.13首先要进行 Rancher的升级。
Rancher pipeline的安装非常简单，在应用商店搜索pipeline
![](http://p2c0rtsgc.bkt.clouddn.com/0213_rancher_01.png)

用默认的配置一键部署
![](http://p2c0rtsgc.bkt.clouddn.com/0213_rancher_02.png)

等基础设施应用中pipeline中的服务都启动后，就会在上方看到流水线的菜单出现
![](http://p2c0rtsgc.bkt.clouddn.com/0213_rancher_03.png)

第一次打流水线时可能会因加载UI文件会慢一些，打开后的效果如下图
![](http://p2c0rtsgc.bkt.clouddn.com/0213_rancher_04.png)

## 授权git仓库
![](http://p2c0rtsgc.bkt.clouddn.com/0213_rancher_05.png)

选择gitlab
![](http://p2c0rtsgc.bkt.clouddn.com/0213_rancher_06.png)

按提示步骤设置gitlab
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_01.png)

配置GitLab进行基于OAuth的身份验证
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_02.png)

生成了客户端ID和秘钥
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_03.png)

填写刚生成的客户端ID和秘钥，并添写gitlab信息后验证
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_04.png)

授权
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_09.png)

等待验证
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_06.png)

验证成功
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_08.png)

可以添加其他更多的帐号（gitlab要退出重新用其他帐号登陆）
![](http://p2c0rtsgc.bkt.clouddn.com/0227_rancher_10.png)


## GO DEMO
### 添加流水线




https://github.com/golang/example
## 
```perl

```


## 参考文档：
* [Rancher之Pipeline JAVA demo](https://www.cnblogs.com/xzkzzz/p/8125389.html)


# 将证书拷贝到如10.240.4.158客户机上并信任
scp -P 50022 10.240.4.160.crt 10.240.4.158:/usr/local/share/ca-certificates/



mattermost_external_url 'https://10.240.4.160'
mattermost_nginx['redirect_http_to_https'] = true
mattermost['gitlab_auth_endpoint'] = "https://10.240.4.160/oauth/authorize"
mattermost['gitlab_token_endpoint'] = "https://10.240.4.160/oauth/token"
mattermost['gitlab_user_api_endpoint'] = "https://10.240.4.160/api/v4/user"




## 
```perl
openssl req -new -newkey rsa:2048 -nodes -out 10.240.4.160.csr -keyout 10.240.4.160.key -subj "/C=CN/ST=Harbin/L=Harbin/O=ydgw/OU=IT/CN=10.240.4.160"

openssl x509 -in 10.240.4.160.crt -text -noout


cp 10.240.4.160.crt /usr/local/share/ca-certificates/


/data/dns-etc/resolv.dnsmasq:/etc/resolv.dnsmasq
/data/dns-etc/dnsmasqhosts:/etc/dnsmasqhosts
/data/dns-etc/dnsmasq.conf:/etc/dnsmasq.conf
/etc/localtime:/etc/localtime:ro

dns (Expected state running but got error: Error response from daemon: OCI runtime create failed: container_linux.go:296: starting container process caused "process_linux.go:398: container init caused \"rootfs_linux.go:58: mounting \\\"/data/docker-dns/dnsmasq.conf\\\" to rootfs \\\"/var/lib/docker/aufs/mnt/3daf5708bcea4ec8da7108e5c8d6b2d030010e5a1fbe7d86349dc3db1a3fd774\\\" at \\\"/var/lib/docker/aufs/mnt/3daf5708bcea4ec8da7108e5c8d6b2d030010e5a1fbe7d86349dc3db1a3fd774/etc/dnsmasq.conf\\\" caused \\\"not a directory\\\"\"": unknown: Are you trying to mount a directory onto a file (or vice-versa)? Check if the specified host path exists and is the expected type) 



docker run -d -p 53:53/tcp -p 53:53/udp --cap-add=NET_ADMIN --name dns-server andyshinn/dnsmasq


docker pull andyshinn/dnsmasq
mkdir -p /data/docker-dns
cd /data/docker-dns

vi resolv.dnsmasq
nameserver 202.97.224.68
nameserver 114.114.114.114
nameserver 8.8.8.8

vi dnsmasqhosts
10.240.4.160  gitlab  gitlab.ydgw.cn


vi dnsmasq.conf
resolv-file=/etc/dnsmasq.d/resolv.dnsmasq
addn-hosts=/etc/dnsmasq.d/dnsmasqhosts

resolv-file=/etc/dnsmasq.d/resolv.dnsmasq
addn-hosts=/etc/dnsmasq.d/dnsmasqhosts

docker tag SOURCE_IMAGE[:TAG] 10.240.4.159/app/IMAGE[:TAG]
docker-compose -f ./dns.yaml up -d
```



