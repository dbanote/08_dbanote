---
title: ORACLE审计
date: 2017-09-13
tags:
- oracle
- audit
---

Oracle审计(Audit)主要用于记录用户对数据库所做的操作，基于配置的不同，审计的结果会放在操作系统文件中或者系统表中。
默认情况下，使用管理员权限连接实例，开启及关闭数据库是会强制进行审计的，其它的基础的操作则没有进行审计。
在一些安全性要求比较高的环境是需要做一些审计的配置的。

<!-- more -->

``` perl
show parameter audit_trail;


NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
audit_trail                          string      DB_EXTENDED

# 将审计结果放在数据库表中，并额外记录SQL_BIND和SQL_TEXT(具体执行语句)
alter system set audit_trail=db_extended scope=spfile;

# 重启数据库
shutdown immediate
startup

# 设置审计内容
audit all by lyj by access;
audit select table, update table, insert table, delete table by lyj by access;
audit execute procedure by lyj by access;

# 取消设置
noaudit all by lyj;
noaudit select table, update table, insert table, delete table by lyj;
noaudit execute procedure by lyj;

# 审计dept表的查询，删除不论它是否成功
audit select ,delete on scott.dept by access whenever not successful;

# 审计emp表的所有操作，只有当操作成功的时候
audit all on scott.emp by access whenever successful;

# 取消对表的审计
noaudit all on scott.dept;

# 查看审计记录
select USERNAME,USERHOST,TIMESTAMP,SQL_TEXT FROM dba_audit_trail order by timestamp;

# 清空审计表内容
truncate table aud$;

## by access  每个被审计的操作都会生成一条记录
## by session 默认值，每个会话里同类型操作只会生成一条audit trail
## whenever successful 操作成功才审计
## whenever not successful 操作不成功才审计
```