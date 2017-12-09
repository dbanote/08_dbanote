---
title: Oracle职业直通车-01 轻松带你走进Oracle数据库的世界
date: 2015-02-10
tags:
- oracle
categories:
- Oracle职业直通车
---

## 使用sqlplus 启动和关闭数据库
``` perl
$ sqlplus / as sysdba

SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.

SQL> startup
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             905970328 bytes
Database Buffers         3456106496 bytes
Redo Buffers               11661312 bytes
Database mounted.
Database opened.
```

## 创建用户test，密码test
``` sql
create user test identified by test default tablespace users;
```