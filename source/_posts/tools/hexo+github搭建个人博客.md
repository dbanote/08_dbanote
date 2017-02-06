---
title: hexo+github搭建个人博客
date: 2017-02-06
tags:
- tools
---

## 前言：我的博客之旅
我用wordpress写过一段时间的博客，因租用的外国空间访问速度不理想，放弃！  
我用`vimwiki`写过一段时间博客，因其配置、更新博客比较麻烦，且生成的html也需要存放到租用的空间上才能共享，放弃！  
我在类似CSDN免费空间上也写过一段时间博客，因没有客户端工具，在线编辑功能受限，且本地不能保存备份博客，放弃！  
这两年我一直用有道云笔记写博客，其本地编辑功能比较强大，有易用的分类目录和标签功能，单条博客也易于分享，但缺乏一个整体对外的窗口，文章中的代码也无法实现高亮显示，不够美观！  
2017年初，偶然的机会知道了hexo，因有较强的markdown和git功底，决定开启hexo写博之旅。

## hexo是什么
hexo是一款基于Node.js的静态博客框架。 hexo官网地址：[https://hexo.io](https://hexo.io)

## 为什么选择Hexo
目前比较流行的静态博客框架有[Jekyll](http://jekyllrb.com/)，Hexo，Simple，Octopress，Pelican以及Lo·gecho等。 些静态博客框架各有各的好处，之所以选择Hexo，最主要的原因如下：
- Hexo基于Node.js实现，在Windows上安装Node.js环境简单；而其他的静态博客框架如Jekyll基于Ruby实现，不建议在Windows下搭建的。
- Hexo有本地WEB服务器，能实现本地预览，并直接发布到WEB容器(github)中实现同步；而Jekyll没有本地服务器，无法实现本地博文预览。
- Hexo主题丰富，基本直接就可以用，不需要太多的修改。
- 支持Markdown语法。

## hexo安装部署

###### 1. 下载并默认安装git和Node.js
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

###### 2. 配置Git Bash样式（可选）
新建D:\08_dbanote目录做为博客根目录，进入目录，右键选择Git Bash Here
![git-bash-here.jpg](/img/git-bash-here.jpg)

在弹出的窗口上右键选择Options，设置窗口样式
![git-bash-01.jpg](/img/git-bash-01.png)

设置显示字体
![git-bash-02.jpg](/img/git-bash-02.jpg)

设置窗口大小(需重新开启Git Bash方可生效)
![git-bash-03.jpg](/img/git-bash-03.jpg)

设置鼠标右健直接粘贴
![git-bash-04.jpg](/img/git-bash-04.jpg)

###### 3. 安装淘宝的cnpm源
考虑到国内访问速度慢，建议安装淘宝的cnpm源，以后使用`cnpm`命令代替`npm`
```
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

###### 4. 安装hexo (-g 是全局化安装)
```
cnpm install hexo-cli -g
```

###### 5. 初始化hexo博客
```
# 确保当前目录为博客根目录
cd D:\08_dbanote
hexo init
cnpm install
```

###### 6. 安装git部署包
作用：通过`hexo d`这一条命令，将博客部署到git服务器上，如github
```
cnpm install hexo-deployer-git --save
```


生成SSH密码
ssh-keygen -t rsa -C "foxbei@163.com"
ssh-keygen -t rsa -C "15004618839@139.com"

Clone Existing Repository

https://github.com/foxbei/hexo.git
D:/06_MyBlog

git config --global user.name "dbanote"
git config --global user.email "15004618839@139.com"


git config --global user.email foxbei@163.com

git config --global user.name "

git init


git remote add origin https://github.com/foxbei/hexo.git

git pull origin


liuyajun@liuyajun-PC MINGW64 /d
$ git remote add origin https://github.com/foxbei/hexo.git
fatal: Not a git repository (or any of the parent directories): .git

liuyajun@liuyajun-PC MINGW64 /d
$ echo "# hexo" >> README.md

liuyajun@liuyajun-PC MINGW64 /d
$ 
Initialized empty Git repository in D:/.git/

liuyajun@liuyajun-PC MINGW64 /d (master)

修改_config.yml文件中以下内容：
```
# Site
title: DBA工作笔记
subtitle: 欲事之无繁，则必劳于始而逸于终
description: 
author: 刘雅君
language:
timezone:


# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: git@github.com:dbanote/dbanote.github.io.git
  branch: master
```

生成静态页
hexo g

启动web server
hexo s

部署到github上
hexo d

ssh -T git@github.com


1、在博客根目录（注意不是yilia根目录）执行以下命令：
cnpm i hexo-generator-json-content --save

2、在根目录_config.yml里添加配置：
  jsonContent:
    meta: false
    pages: false
    posts:
      title: true
      date: true
      path: true
      text: true
      raw: false
      content: falsef
      slug: false
      updated: false
      comments: false
      link: false
      permalink: false
      excerpt: false
      categories: false
      tags: true