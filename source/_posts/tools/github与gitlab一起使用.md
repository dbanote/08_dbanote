---
title: 一台电脑同时使用GitHub和GitLab仓库
date: 2018-02-27
tags:
- gitlab
- gitlub
---

github已在用，后公司搭建私有gitlab仓库，现要在一台电脑上同时使用GitHub和GitLab仓库
![](http://p2c0rtsgc.bkt.clouddn.com/0227_gitlab_01.png)

<!-- more -->
进入git bash生成gitlab钥匙
```
$ cd ~/.ssh
$ ssh-keygen -t rsa -C "liuyajun@ydgw.cn"
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/liuyajun/.ssh/id_rsa): /c/Users/liuyajun/.ssh/gitlab_id_rsa
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/liuyajun/.ssh/gd_rgitlab_id_rsa.
Your public key has been saved in /c/Users/liuyajun/.ssh/gd_rgitlab_id_rsa.pub.
The key fingerprint is:
SHA256:IBKC+H7rm/4f19CViQ3qtbDXHGNGufG4GCx33ykCJdQ liuyajun@ydgw.cn
The keys randomart image is:
+---[RSA 2048]----+
|+ .     ... . .. |
|o. .     . E =oo |
| .. . .   =.o X= |
|  .. . . o.=+B+o.|
| .      S =o++ooo|
|  . .      =....o|
|   . .  . . o .  |
|    ..   o       |
|   o=o...        |
+----[SHA256]-----+

$ ll
total 17
-rw-r--r-- 1 liuyajun 197121 1679 二月 27 11:50 gitlab_id_rsa
-rw-r--r-- 1 liuyajun 197121  398 二月 27 11:50 gitlab_id_rsa.pub
-rw-r--r-- 1 liuyajun 197121 1675 一月 22  2017 id_rsa
-rw-r--r-- 1 liuyajun 197121  401 一月 22  2017 id_rsa.pub
-rw-r--r-- 1 liuyajun 197121 2351 二月 27 11:34 known_hosts

eval $(ssh-agent -s)
ssh-add ~/.ssh/gitlab_id_rsa
```

用gitlab_id_rsa.pub里的内容配置gitlab帐号SSH KEY
```
cat ~/.ssh/gitlab_id_rsa.pub
```

![](http://p2c0rtsgc.bkt.clouddn.com/0227_gitlab_02.png)

配置config
```
cat ~/.ssh/config
# GitLab.com server
Host gitlab.com
IdentityFile ~/.ssh/id_rsa

# Private GitLab server
Host 10.240.4.160
IdentityFile ~/.ssh/gitlab_id_rsa
```

测试
```
ssh -T git@github.com
ssh -T git@10.240.4.160
```