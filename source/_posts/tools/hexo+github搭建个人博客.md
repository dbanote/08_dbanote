---
title: hexo+github搭建个人博客
tags:
- tools
- hexo
---

## 前言：我的博客之旅
我用`wordpress`写过一段时间博客，因租用的外国空间访问速度不理想，放弃！  
我用`vimwiki`写过一段时间博客，因其配置、更新博客比较麻烦，且生成的html也需要存放到租用的空间上才能共享，放弃！  
我在类似CSDN免费空间上也写过一段时间博客，因没有客户端工具，在线编辑功能受限，且本地不能保存备份博客，放弃！  
这两年我一直用有道云笔记写博客，其本地编辑功能比较强大，有易用的分类目录和标签功能，单条博客也易于分享，但缺乏一个整体对外的窗口，文章中的代码也无法实现高亮显示，不够美观！  
2017年初，偶然的机会知道了hexo，因有较强的markdown和git功底，决定开启hexo写博之旅。

>**hexo是什么?**
hexo是一款基于Node.js的静态博客框架。 hexo官网地址：[https://hexo.io](https://hexo.io)

<!--more-->

## 为什么选择Hexo
目前比较流行的静态博客框架有[Jekyll](http://jekyllrb.com/)，Hexo，Simple，Octopress，Pelican以及Lo·gecho等。 这些静态博客框架各有各的好处，之所以选择Hexo，最主要的原因如下：
1. Hexo基于Node.js实现，在Windows上安装Node.js环境简单；而其他的静态博客框架如Jekyll基于Ruby实现，不建议在Windows下搭建的。
2. Hexo有本地WEB服务器，能实现本地预览，并直接发布到WEB容器(github)中实现同步；而Jekyll没有本地服务器，无法实现本地博文预览。
3. Hexo主题丰富，基本直接就可以用，不需要太多的修改。
4. 支持Markdown语法。

## 搭建本地博客

### 下载并安装git和Node.js
从以下地址下载所需的Windows版安装包，使用默认设置安装：
- Node.js: [https://nodejs.org](https://nodejs.org)
- git: [https://git-scm.com](https://git-scm.com)

安装成功后，可通过以下命令查看安装版本：
```
$ node -v
v6.9.4

$ git --version
git version 2.11.1.windows.1
```

### 配置Git Bash样式（可选）
新建`D:\08_dbanote`目录做为博客根目录，进入目录，右键选择Git Bash Here
![git-bash-here.jpg](/img/2017/git-bash-here.jpg)

在弹出的窗口上右键选择Options，设置窗口样式
![git-bash-01.jpg](/img/2017/git-bash-01.png)

设置显示字体
![git-bash-02.jpg](/img/2017/git-bash-02.jpg)

设置窗口大小(需重新开启Git Bash方可生效)
![git-bash-03.jpg](/img/2017/git-bash-03.jpg)

设置鼠标右健直接粘贴
![git-bash-04.jpg](/img/2017/git-bash-04.jpg)

### 安装淘宝的cnpm源
访问国外源速度较慢，建议安装淘宝的cnpm源，以后使用`cnpm`命令代替`npm`
```
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

### 安装hexo (-g 是全局化安装)
```
cnpm install hexo-cli -g
```

### 初始化hexo博客
``` perl
# 确保当前目录为博客根目录
cd D:\08_dbanote
hexo init
cnpm install
```

### 安装git部署包
作用：通过`hexo d`这一条命令，将博客部署到git服务器上，如github
```
cnpm install hexo-deployer-git --save
```

### 生成博客静态文件
```
hexo g
```

### 启动本地服务器
确保4000端口没有被占用
``` perl
# 在windows命令行下执行
netstat -ano | findstr "4000"
  TCP    127.0.0.1:4000         0.0.0.0:0              LISTENING       16360
  TCP    127.0.0.1:4000         127.0.0.1:53443        ESTABLISHED     16360
  TCP    127.0.0.1:4000         127.0.0.1:63485        CLOSE_WAIT      16360
  TCP    127.0.0.1:53443        127.0.0.1:4000         ESTABLISHED     14256
  TCP    127.0.0.1:63485        127.0.0.1:4000         FIN_WAIT_2      14256

# 4000端口被占用，查找进程号对应的程序
tasklist | findstr "16360"
FoxitProtect.exe             16360 Services                   0     11,628 K

# 结束该进程
taskkill /f /t /im FoxitProtect.exe
```

启动本地服务器
``` perl
hexo s
#---------------------------------------------------------------------
INFO  Start processing
INFO  Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
#---------------------------------------------------------------------
```

### 本地预览博客
打开浏览器，输入[http://localhost:4000](http://localhost:4000)
![hexo_init.png](/img/2017/hexo_init.png)


### 配置博客网站基本信息
编辑修改根目录下的`_config.yml`文件
``` perl
# 站点基本信息修改
# Site
title: 一个DBA的工作学习笔记
subtitle: 欲事之无繁，则必劳于始而逸于终
author: 刘雅君

# 设置网站url
# URL
url: http://dbanote.github.io

# 取消代码段行号显示
# Writing
highlight:
  enable: true
  line_number: false
```


## 部署远程博客

### 注册Github账号
因为是托管到[Github](https://github.com/)上，所以第一步需要注册一个账号。这里我新注册了一个dbanote的帐号，步骤很简单，这里不做赘述。

### 建立和用户名对应的仓库
建立和用户名相对应的仓库，这是什么意思呢？以我的例子来说，我的用户名是dbanote,那么我的博客仓库就必须是dbanote.github.io。
![创建新仓库](/img/2017/github_01.png)
![创建新仓库](/img/2017/github_02.png)

### 配置SSH公钥
远程代码是基于SSH的，所以需要SSH的相关配置。方法是现在本地生成SSH公钥，然后添加到Github上面。打开`git bash`，具体的操作如下：

#### 1. 设置你的邮箱和用户名
``` perl
git config --global user.name "dbanote"
git config --global user.email "15004618839@139.com"
```

#### 2. 生成密钥，设置密码，输入的密码不显示（也可以不设置，按三次回车，密码为空）
``` perl
ssh-keygen -t rsa -C "15004618839@139.com"
```
上述的命令成功后，会得到`id_rsa`和`id_rsa.pub`两个文件，一般在`C:\Users\<用户名>\.ssh`文件夹里，没有的话，就用Everything搜一下。

#### 3. 把SSH密钥添加到Github上
登陆`Github`后，点击`settings`，然后进入`SSH keys`，把`id_rsa.pub`文件里内容添加进去就好了。 测试配置是否正确
``` perl
$ ssh -T git@github.com
Hi dbanote! You’ve successfully authenticated, but GitHub does not provide shell access.
```


### 部署远程博客
#### 1. 编辑修改根目录下的`_config.yml`文件
``` perl
# 设置部署到github(https方式出现错误，使用ssh方式成功)
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: git@github.com:dbanote/dbanote.github.io.git
  branch: master
```

#### 2. 部署远程博客，输入以下命令
``` perl
# 生成静态页
hexo g

# 部署到github上
hexo d
```

部署好了后，在浏览器地址栏中输入你的仓库名来访问，我的是[dbanote.github.io](http://dbanote.github.io)。注意一点，第一次部署的话，可能需要等待一会（一般不到10分钟就好了）才能生效，以后每次部署就可以直接访问。到这里基本的博客就搭建好了。

## Tips
1. 注意一定要验证Github的验证邮件。
2. 出现其他任何的问题，先删除博客目录下的`db.json`文件，然后清理再部署远程博客，操作时输入以下的命令
``` perl
hexo clean
hexo d -g
```
3. Hexo的基本命令
``` perl
hexo g = hexo generate  #生成
hexo s = hexo server  #启动本地预览
hexo d = hexo deploy  #远程部署
hexo n "文章标题" = hexo new "文章标题"  #新建一篇博文

hexo s -g  #等同先输入hexo g，再输入hexo s
hexo d -g  #等同先输入hexo g，再输入hexo d
```