---
title: hexo解决代码块空行自动删除问题
tags:
- tools
- hexo
---

## 修改hexo源码
将`node_modules\hexo-util\lib\highlight.js`文件中第`35`行
``` perl
# 原
    numbers += '<div class="line">' + (firstLine + i) + '</div>\n';
    content += '<div class="line';
    content += (mark.indexOf(firstLine + i) !== -1) ? ' marked' : '';
    content += '">' + line + '</div>\n';

# 改后
    numbers += '<span class="line">' + (firstLine + i) + '</span>\n';
    content += '<span class="line';
    content += (mark.indexOf(firstLine + i) !== -1) ? ' marked' : '';
    content += '">' + line + '</span>\n';
```

## 清理临时数据库并重启本地服务
``` perl
hexo clean
hexo s -g
```

<!--more-->

## 查看hexo版本信息
``` perl
$ hexo version
hexo: 3.2.2
hexo-cli: 1.0.2
os: Windows_NT 6.1.7601 win32 x64
http_parser: 2.7.0
node: 6.9.4
v8: 5.1.281.89
uv: 1.9.1
zlib: 1.2.8
ares: 1.10.1-DEV
icu: 57.1
modules: 48
openssl: 1.0.2j
```

