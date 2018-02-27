---
title: Docker学习笔记_06 使用rancher搭建gitlab服务并启用https
date: 2018-01-22
tags:
- docker
- rancher
- gitlab
categories:
- Docker学习笔记
---

## 添加一个服务
添加应用(使用rancher默认的编排工具Cattle，docker-ce升级到最新版本)
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_01.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_02.png)

为应用添加服务
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_03.png)
设置名称、镜像，添加端口映射：80、22、443
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_04.png)

持久化本地存储：
/data/gitlab/app-data:/var/opt/gitlab
/data/gitlab/log-data:/var/log/gitlab
/data/gitlab/conf-files:/etc/gitlab
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_05.png)

设置容器主机名 gitlab
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_06.png)

设置健康检查
![](http://p2c0rtsgc.bkt.clouddn.com/0207_rancher_05.png)

调度安装到指定的主机上，并创建
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_08.png)

创建成功
![](http://p2c0rtsgc.bkt.clouddn.com/0205_rancher_09.png)

## gitlab启用https
### 配置HTTPS所需证书
```perl
mkdir /root/data
cd /root/data

# 创建自已的CA证书
openssl req -newkey rsa:4096 -nodes -sha256 -keyout ca.key -x509 -days 365 -out ca.crt
#------------------------------------------------------------
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Harbin
Locality Name (eg, city) []:Harbin
Organization Name (eg, company) [Internet Widgits Pty Ltd]:ydgw
Organizational Unit Name (eg, section) []:ydgw
Common Name (e.g. server FQDN or YOUR name) []:10.240.4.160
Email Address []:liuyajun@ydgw.cn
#------------------------------------------------------------

# 生成一个证书签名请求
openssl req -newkey rsa:4096 -nodes -sha256 -keyout 10.240.4.160.key -out 10.240.4.160.csr
#------------------------------------------------------------
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Harbin
Locality Name (eg, city) []:Harbin
Organization Name (eg, company) [Internet Widgits Pty Ltd]:ydgw
Organizational Unit Name (eg, section) []:ydgw
Common Name (e.g. server FQDN or YOUR name) []:10.240.4.160
Email Address []:liuyajun@ydgw.cn

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:   #密码留空即可
An optional company name []:
#------------------------------------------------------------

# 创建文件夹和辅助内容
mkdir demoCA
cd demoCA
touch index.txt
echo '01' > serial
cd ..

ll
#------------------------------------------------------------
total 28
drwxr-xr-x 3 root root 4096 Feb  5 17:03 ./
drwx------ 6 root root 4096 Feb  5 17:00 ../
-rw-r--r-- 1 root root 1740 Feb  5 17:02 10.240.4.160.csr
-rw-r--r-- 1 root root 3272 Feb  5 17:02 10.240.4.160.key
-rw-r--r-- 1 root root 2098 Feb  5 17:01 ca.crt
-rw-r--r-- 1 root root 3276 Feb  5 17:01 ca.key
drwxr-xr-x 2 root root 4096 Feb  5 17:03 demoCA/
#------------------------------------------------------------

# 签名证书
echo subjectAltName = IP:10.240.4.160 > extfile.cnf
openssl ca -in 10.240.4.160.csr -out 10.240.4.160.crt -cert ca.crt -keyfile ca.key -extfile extfile.cnf -outdir .
#------------------------------------------------------------
Using configuration from /usr/lib/ssl/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: Feb  5 09:03:38 2018 GMT
            Not After : Feb  5 09:03:38 2019 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = Harbin
            organizationName          = ydgw
            organizationalUnitName    = ydgw
            commonName                = 10.240.4.160
            emailAddress              = liuyajun@ydgw.cn
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                IP Address:10.240.4.160
Certificate is to be certified until Feb  5 09:03:38 2019 GMT (365 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
#------------------------------------------------------------

ll
#------------------------------------------------------------
total 48
drwxr-xr-x 3 root root 4096 Jan 30 22:20 ./
drwx------ 5 root root 4096 Jan 30 22:09 ../
-rw-r--r-- 1 root root 6873 Feb  5 17:03 01.pem
-rw-r--r-- 1 root root 6873 Feb  5 17:03 10.240.4.160.crt
-rw-r--r-- 1 root root 1740 Feb  5 17:02 10.240.4.160.csr
-rw-r--r-- 1 root root 3272 Feb  5 17:02 10.240.4.160.key
-rw-r--r-- 1 root root 2098 Feb  5 17:01 ca.crt
-rw-r--r-- 1 root root 3276 Feb  5 17:01 ca.key
drwxr-xr-x 2 root root 4096 Feb  5 17:03 demoCA/
-rw-r--r-- 1 root root   33 Feb  5 17:03 extfile.cnf
#------------------------------------------------------------

# 证书加入本机信任
cp 10.240.4.160.crt /usr/local/share/ca-certificates/
update-ca-certificates

# 重启docker使证书生效
systemctl daemon-reload
systemctl restart docker

# 将证书放到指定的路径
mkdir /data/gitlab/conf-files/ssl
chmod 700 /data/gitlab/conf-files/ssl
cp /root/data/10.240.4.160.crt /data/gitlab/conf-files/ssl/
cp /root/data/10.240.4.160.key /data/gitlab/conf-files/ssl/


```

## 升级服务
![](http://p2c0rtsgc.bkt.clouddn.com/0207_rancher_01.png)
添加环境变量GITLAB_OMNIBUS_CONFIG
![](http://p2c0rtsgc.bkt.clouddn.com/0207_rancher_02.png)
也可以直接修改配置文件
``` perl
vi /data/gitlab/conf-files/gitlab.rb
# 添加以下内容在文件的最后
#------------------------------------------------------------
external_url "https://10.240.4.160"
nginx['enable'] = true
nginx['redirect_http_to_https'] = true
nginx['ssl_certificate'] = "/etc/gitlab/ssh/10.240.4.160.crt"
nginx['ssl_certificate_key'] = "/etc/gitlab/ssh/10.240.4.160.key"
#------------------------------------------------------------
```

## 登陆gitlab
https://10.240.4.160
第一次登陆需要设置root密码
![](http://p2c0rtsgc.bkt.clouddn.com/0207_rancher_03.png)
登陆后界面
![](http://p2c0rtsgc.bkt.clouddn.com/0207_rancher_04.png)