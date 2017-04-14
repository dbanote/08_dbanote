---
title: Linux常用命令整理
date: 2017-04-05
tags:
- linux
---

## VI命令
``` perl
# 替换每一行的第一个35 09为35 10
: %s/35 09/35 10/
```

## 重新加载fstab中指定的挂载点
``` perl
mount -o remount /oradata/nfs
```

## find命令
### 查出近期日志并复制到其他地方
``` perl
find /oradata/hknfs/check_report -type f -name "*" -mtime -19 -exec cp {} /oradata/ngnfs/check_report \;
```

### 查看30天前的文件删除
``` perl
find /oradata/hknfs/"$ORACLE_SID"_expdp -type f -name "*" -mtime +30 -exec rm -rf {} \; 
```

