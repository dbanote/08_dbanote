---
title: GIT常用命令整理
tags:
- git
---

# GIT常用命令整理

##### 基本信息设置
```
git config --global user.name 'dbanote'
git config --global user.email '15004618839@139.com'
```

##### 初始化git仓库
```
git init
```

##### 向本地仓库中添加/修改/删除文件
```
# 添加/修改/删除工作区文件后，保存变动内容到暂存区
git add <file>
git add *

# 查看暂存区状态
git status

# 暂存区内容提交到git仓库
git commit -m "git init"
```



##### PUSH远程仓库
```
git remote add origin git@github.com:dbanote/08_dbanote.git
git push -u origin master
```

##### 克隆远程仓库
```
git clone git@github.com:dbanote/08_dbanote.git
```


…or create a new repository on the command line

修改测试

echo "# hexo" >> README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/dbanote/hexo.git
git push -u origin master
…or push an existing repository from the command line

git remote add origin https://github.com/dbanote/hexo.git
git push -u origin master
…or import code from another repository
You can initialize this repository with code from a Subversion, Mercurial, or TFS project.