---
title: Oracle Database 11g工作中常用的命令和脚本整理
categories:
- script
tags:
- oracle
---

## 设置Oracle用户密码永不过期
``` perl
alter profile default limit PASSWORD_LIFE_TIME unlimited;

# 查看设置
select * from dba_profiles where resource_name like 'PASSWORD_LIFE_TIME%';
```

## 设置Oracle用户登陆失败次数限制
``` perl
# 设置登陆失败次数限制为100，错误登陆数超过100会导致用户被锁
alter profile default limit FAILED_LOGIN_ATTEMPTS 100;

# 无限制
alter profile default limit FAILED_LOGIN_ATTEMPTS unlimited;

# 查看登陆失败限制设置
select * from dba_profiles where resource_name like 'FAILED_LOGIN_ATTEMPTS%';
```

<!-- more -->

## 系统进程PID查找对应SQLTEXT
``` sql
SELECT sql_text FROM v$sqltext a 
  WHERE (a.hash_value, a.address) IN (SELECT DECODE (sql_hash_value,0, prev_hash_value,sql_hash_value),
        DECODE (sql_hash_value, 0, prev_sql_addr, sql_address)
    FROM v$session b WHERE b.paddr = (SELECT addr FROM v$process c WHERE c.spid =&pid)) ORDER BY piece ASC;
```

## 查找长事务对应的SQLTEXT
``` sql
with ltr as ( 
select to_char(sysdate,'YYYYMMDDHH24MISS') TM, 
       s.sid, 
       s.sql_id, 
       s.sql_child_number, 
       s.prev_sql_id, 
       xid, 
       to_char(t.start_date,'YYYYMMDDHH24MISS') start_time, 
       e.TYPE,e.block, 
       e.ctime, 
       decode(e.CTIME, 0, (sysdate - t.start_date) * 3600*24, e.ctime) el_second 
  from v$transaction t, v$session s,v$transaction_enqueue e
 where t.start_date <= sysdate - interval '200' second
   and t.addr = s.taddr 
   and t.addr = e.addr(+) ) 
  select ltr.* , (select q1.sql_text from v$sql q1 where ltr.prev_sql_id = q1.sql_id(+)
   and rownum = 1) prev_sql_text , 
  (select q1.sql_text from v$sql q1 where ltr.sql_id = q1.sql_id(+) 
   and ltr.sql_child_number = q1.CHILD_NUMBER(+)) sql_text 
   from ltr ltr;
```

## 永久设置sql*plus的环境变量
``` perl
echo "set pagesize 9999" >> $ORACLE_HOME/sqlplus/admin/glogin.sql
echo "set line 150" >> $ORACLE_HOME/sqlplus/admin/glogin.sql
echo "set long 5000" >> $ORACLE_HOME/sqlplus/admin/glogin.sql
```

## 查找非系统用户中无主键表
``` sql
select distinct at.TABLE_NAME, at.OWNER, at.NUM_ROWS
from 
  (SELECT owner,table_name FROM all_tables WHERE owner in 
   (select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE'))
MINUS
  SELECT owner,table_name FROM all_constraints WHERE owner in 
   (select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE')) 
  AND constraint_type = 'P' ) vn, all_tables at
where vn.TABLE_NAME=at.TABLE_NAME and vn.OWNER=at.OWNER and at.TABLE_NAME not like '%$%' order by 2,1;
```

## 查询有Object的schema
``` sql
select OWNER,count(OBJECT_NAME) OBJECT_NUM from dba_objects where OWNER in
    (select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE')) and OBJECT_TYPE='TABLE'
    group by OWNER;
```