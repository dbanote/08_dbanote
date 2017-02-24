---
title: hexo博客添加常用插件
tags:
- tools
- hexo
---

Hexo提供了诸多插件来增强博客体验，地址[`http://hexo.io/plugins`](http://hexo.io/plugins/)

## 添加pdf浏览插件

### 安装[hexo-pdf](https://github.com/superalsrk/hexo-pdf)
``` perl
cnpm install --save hexo-pdf
```

### 使用
``` perl
{% pdf http://7xov2f.com1.z0.glb.clouddn.com/bash_freshman.pdf %}
{% pdf ./bash_freshman.pdf %}
```

<!-- more -->

## 添加生成文章目录插件hexo-toc

1. 安装
```
cnpm install hexo-toc --save
```

2. 配置
在博客根目录下的`_config.yml`中如下配置：
```
toc:
  maxdepth: 3
```

3. 使用
在Markdown中需要显示文章目录的地方添加 
```
<!-- toc -->
```

## 添加百度统计支持

`yilia`主题自带统计支持
1. 打开[百度统计](http://tongji.baidu.com/web/welcome/login)网站，注册账号
![baidu_stat_01.png](http://oligvdnzp.bkt.clouddn.com/baidu_stat_01.png)

2. 填写注册信息
![baidu_stat_02.png](http://oligvdnzp.bkt.clouddn.com/baidu_stat_02.png)

3. 记录百度统计ID号码
![baidu_stat_03.png](http://oligvdnzp.bkt.clouddn.com/baidu_stat_03.png)

4. 编辑修改`themes\yilia`目录下的`_config.yml`文件
``` perl
# Miscellaneous
baidu_analytics: 'xxxxxxxxxxxx'   # xxxxxx是刚获得的百度统计ID号码
```

5. 统计代码安装检查，确保代码安装正确
![baidu_stat_04.png](http://oligvdnzp.bkt.clouddn.com/baidu_stat_04.png)

## 添加多说评论支持
1. 打开[多说](http://duoshuo.com/)网站
![duoshuo_01.png](http://oligvdnzp.bkt.clouddn.com/duoshuo_01.png)

2. 选择微信登陆
![duoshuo_02.png](http://oligvdnzp.bkt.clouddn.com/duoshuo_02.png)

3. 点击`我要安装`
![duoshuo_03.png](http://oligvdnzp.bkt.clouddn.com/duoshuo_03.png)

4. 填写相应信息
![duoshuo_04.png](http://oligvdnzp.bkt.clouddn.com/duoshuo_04.png)

5. 编辑修改`themes\yilia`目录下的`_config.yml`文件
``` perl
#是否开启多说评论，开启填写你在多说申请的项目的short_name: dbanote
#duoshuo: false
duoshuo: dbanote
```

## 参考链接

- [Hexo站点中添加文章目录以及归档](http://www.ituring.com.cn/article/199624)
- [hexo优化目录](http://www.cnblogs.com/peihao/p/5269131.html)

