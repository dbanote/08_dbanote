---
title: hexo更换主题并个性化定制
tags:
- tools
- hexo
---

>hexo系统默认的主题虽然也不错，但我更喜欢[yelee](http://moxfive.xyz/yelee)和[yilia](https://github.com/litten/hexo-theme-yilia)这两款主题

## 安装yelee主题

1. 主题说明文档：[`http://moxfive.coding.me/yelee/`](http://moxfive.coding.me/yelee)

2. 在博客根目录下执行以下命令
``` perl
# 克隆最新一次提交（git clone速度很慢，可只克隆最新一次提交）
git clone --depth 1 https://github.com/MOxFIVE/hexo-theme-yelee.git themes/yelee
```

3. 编辑修改根目录下的`_config.yml`文件
``` perl
# theme: landscape
theme: yelee
```

4. 更换主题后需要清理并重新生成静态页
```
hexo clean && hexo g && hexo s
```

5. 安装搜索插件
```
cnpm install hexo-generator-search --save
```

7. 参照文档修改`themes\yelee`目录下的`_config.yml`文件

<!--more-->

## 安装yilia主题

1. 在博客根目录下执行以下命令
``` perl
git clone https://github.com/litten/hexo-theme-yilia.git themes/yilia
# 或者
git clone git@github.com:litten/hexo-theme-yilia.git themes/yilia

# 克隆最新一次提交（git clone速度很慢，可只克隆最新一次提交）
git clone --depth 1 https://github.com/litten/hexo-theme-yilia.git themes/yilia
```

2. 在博客根目录（注意不是yilia根目录）执行以下命令：
```
cnpm i hexo-generator-json-content --save
```

3. 编辑修改根目录下的`_config.yml`文件
``` perl
# theme: landscape
theme: yilia

# 添加以下内容
# jsonContent
jsonContent:
    meta: false
    pages: false
    posts:
      title: true
      date: true
      path: true
      text: true
      raw: false
      content: false
      slug: false
      updated: false
      comments: false
      link: false
      permalink: false
      excerpt: false
      categories: false
      tags: true
```

4. 更换主题后需要清理并重新生成静态页
```
hexo clean
hexo g
hexo s
```

5. 更换后效果
![update_theme.png](/img/2017/update_theme.png)

6. 编辑修改`themes\yilia`目录下的`_config.yml`文件
``` perl
# 修改menu
menu:
  主页: /
  归档: /archives/
  #随笔: /tags/随笔/

# 修改并注释掉不需要的项
# SubNav
subnav:
  github: "https://github.com/dbanote"
  weibo: "http://weibo.com/foxbei"
  #rss: "#"
  #zhihu: "#"
  qq: "http://sighttp.qq.com/msgrd?v=1&uin=9320802"
  #weixin: "#"
  #jianshu: "#"
  #douban: "#"
  mail: "mailto:15004618839@139.com"

# 文章太长，截断按钮文字
# 截断需要在文章中添加 <!--more-->
#excerpt_link: more
excerpt_link: false

# 设置打赏二维码
# 打赏基础设定：0-关闭打赏； 1-文章对应的md文件里有reward:true属性，才有打赏； 2-所有文章均有打赏
reward_type: 2
# 打赏wording
reward_wording: '喜欢的话，支付1元请我吃糖吧！'

# 支付宝二维码图片地址，跟你设置头像的方式一样。比如：/assets/img/alipay.jpg
# 支付宝收款1元二维码图片
alipay: /img/zhifubao01.png
# 支付宝收款无金额设置二维码图片
# alipay: /img/zhifubao.png

# 微信二维码图片地址
# 微信收款1元二维码图片
weixin: /img/weixin01.png
# 微信收款无金额设置二维码图片
# alipay: /img/weixin.png

# 修改网站收藏夹图标
favicon: /img/favicon.png

#你的头像url
avatar: /img/avatar.jpg

# 根据需要修改以下内容
# 智能菜单
# 如不需要，将该对应项置为false
# 比如
#smart_menu:
#  friends: false
smart_menu:
  innerArchive: '所有文章'
  friends: '常用网站'
  aboutme: '关于我'

friends:
  My Oracle Support: https://support.oracle.com/
  Oracle Edelivery: https://edelivery.oracle.com/
  Oracle Help Center: http://docs.oracle.com/
  练数成金: http://www.dataguru.cn/
  三通IT论坛: http://www.santongit.com/
  一起自学吧: http://www.17zixueba.com/

aboutme: DBA<br><br>工作学习笔记<br>谢谢大家
```

