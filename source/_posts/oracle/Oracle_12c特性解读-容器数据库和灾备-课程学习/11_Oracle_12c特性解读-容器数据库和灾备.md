---
title: Oracle 12c特性解读-容器数据库和灾备-11 灾备维护经验总结
date: 2017-07-14
tags:
- oracle
- 12c
categories:
- Oracle 12c特性解读-容器数据库和灾备
---

## 不影响主库，模拟备库Failover，恢复主备数据同步的过程
### 删除备库dg broker配置
``` perl
$ tnsping dgtp

TNS Ping Utility for Linux: Version 12.2.0.1.0 - Production on 13-JUL-2017 18:31:27

Copyright (c) 1997, 2016, Oracle.  All rights reserved.

Used parameter files:
/u01/app/oracle/product/12.2.0/db_1/network/admin/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.174)(PORT = 1521)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = dgtp)))
OK (0 msec)

$ vi $ORACLE_HOME/network/admin/tnsnames.ora
# 修改端口号为任意没开通的
DGTP =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.174)(PORT = 15211))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = dgtp)
    )
  )

$ tnsping dgtp

TNS Ping Utility for Linux: Version 12.2.0.1.0 - Production on 13-JUL-2017 18:34:37

Copyright (c) 1997, 2016, Oracle.  All rights reserved.

Used parameter files:
/u01/app/oracle/product/12.2.0/db_1/network/admin/sqlnet.ora


Used TNSNAMES adapter to resolve the alias
Attempting to contact (DESCRIPTION = (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.174)(PORT = 15211)) (CONNECT_DATA = (SERVER = DEDICATED) (SERVICE_NAME = dgtp)))
TNS-12541: TNS:no listener

# 关闭dg broker
SQL> show parameter dg

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name               string
dg_broker_config_file1               string      /u01/app/oracle/product/12.2.0
                                                 /db_1/dbs/dr1DGTS.dat
dg_broker_config_file2               string      /u01/app/oracle/product/12.2.0
                                                 /db_1/dbs/dr2DGTS.dat
dg_broker_start                      boolean     TRUE
inmemory_adg_enabled                 boolean     TRUE
SQL> alter system set dg_broker_start=false;

# 删除DG broker配置文件
! rm /u01/app/oracle/product/12.2.0/db_1/dbs/dr1DGTS.dat
! rm /u01/app/oracle/product/12.2.0/db_1/dbs/dr2DGTS.dat
```

<!-- more -->

### 手工failover
``` perl
# 在备库上记录current_scn
SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    5651932

SQL> select open_mode from v$database;

OPEN_MODE
--------------------
READ ONLY WITH APPLY

# 手工failover
SQL> recover managed standby database finish force;
Media recovery complete.

SQL> select open_mode from v$database;

OPEN_MODE
--------------------
MOUNTED

SQL> alter database commit to switchover to primary;

Database altered.

SQL> alter database open;

SQL> select open_mode, database_role from v$database;

OPEN_MODE            DATABASE_ROLE
-------------------- ----------------
READ WRITE           PRIMARY

# 这时备库就可以做读写操作了
SQL> create table test as select * from all_objects;
```

### 恢复主备数据同步
``` perl
SQL> alter database close;

Database altered.

SQL> select open_mode, database_role from v$database;

OPEN_MODE            DATABASE_ROLE
-------------------- ----------------
MOUNTED              PRIMARY

SQL> flashback database to scn 5651932;

Flashback complete.

SQL> alter database convert to physical standby;

Database altered.

SQL> select open_mode, database_role from v$database;

OPEN_MODE            DATABASE_ROLE
-------------------- ----------------
MOUNTED              PHYSICAL STANDBY

SQL> recover managed standby database disconnect from session;
Media recovery complete.

SQL> alter system set dg_broker_start=true;

$ vi $ORACLE_HOME/network/admin/tnsnames.ora
# 修改端口号修改成原先的1521
DGTP =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.174)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = dgtp)
    )
  )

SQL> alter database open;

Database altered.

SQL> select open_mode, database_role from v$database;

OPEN_MODE            DATABASE_ROLE
-------------------- ----------------
READ ONLY WITH APPLY PHYSICAL STANDBY

# 查看ADG配置已自已应用成功
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 58 seconds ago)
```

## 12cR2 SQL/PLUS里新增命令HISTORY
``` perl
show hist
set hist off
set hist on
set hist 500
hist
hist 3 del
hist 1 run
hist 1 edit
hist clear
```

## 主库闪回数据库后备库恢复同步
``` perl
# 主库上执行
create restore point fb_recover_trunc guarantee flashback database;

col NAME for a20
col TIME for a35
set lines 200
col STORAGE_SIZE for a50
SELECT NAME, SCN, TIME, DATABASE_INCARNATION# DI,GUARANTEE_FLASHBACK_DATABASE, STORAGE_SIZE/1024/1024/1024 
  FROM V$RESTORE_POINT WHERE GUARANTEE_FLASHBACK_DATABASE='YES';

NAME                        SCN TIME                                        DI GUA STORAGE_SIZE/1024/1024/1024
-------------------- ---------- ----------------------------------- ---------- --- ---------------------------
FB_RECOVER_TRUNC        5633353 13-JUL-17 04.45.02.000000000 PM              5 YES                    .1953125

show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO

conn lyj/lyj@pdb1
select * from cat;

TABLE_NAME           TABLE_TYPE
-------------------- -----------
TEST                 TABLE
TEST2                TABLE

create table test_trunc as select * from all_objects;
select count(*) from test_trunc;

  COUNT(*)
----------
     56327

drop table TEST purge;
drop table TEST2 purge;

select * from cat;

TABLE_NAME           TABLE_TYPE
-------------------- -----------
TEST_TRUNC           TABLE

# 主库做闪回数据库操作
conn / as sysdba
shutdown immediate
startup mount
flashback database to restore point fb_recover_trunc;
alter database open resetlogs;

alter session set container=pdb1;
select TABLE_NAME from all_tables where OWNER='LYJ';

TABLE_NAME
--------------------------------------------------------------------------------
TEST
TEST2

# 备库状态
SQL> select database_role,open_mode from v$database;

DATABASE_ROLE    OPEN_MODE
---------------- --------------------
PHYSICAL STANDBY READ ONL      # ADG状态已停止

# 主备库数据不同步
alter session set container=pdb1;
select TABLE_NAME from all_tables where OWNER='LYJ';

TABLE_NAME
--------------------------------------------------------------------------------
TEST_TRUNC

# 在备库执行闪回数据库（用闪回点的SCN号5633353）
conn / as sysdba
shutdown immediate
startup mount
recover managed standby database cancel;  # 视情况可不执行
flashback database to scn 5633353;

dgmgrl sys/Center08@dgtp
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 
      Error: ORA-16810: multiple errors or warnings detected for the member

Fast-Start Failover: DISABLED

Configuration Status:
ERROR   (status updated 13 seconds ago)

DGMGRL> enable configuration;

DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 1 second ago)

SQL> select database_role,open_mode from v$database;

DATABASE_ROLE    OPEN_MODE
---------------- --------------------
PHYSICAL STANDBY MOUNTED

SQL> alter database open;

Database altered.

SQL> select database_role,open_mode from v$database;

DATABASE_ROLE    OPEN_MODE
---------------- --------------------
PHYSICAL STANDBY READ ONLY WITH APPLY
```

## 闪回数据库是无法闪回到datafile收缩前的状态
``` perl
# 将已有的数据文件收缩到190M
alter session set container=pdb1;
col TABLESPACE_NAME for a10
col FILE_NAME for a50
select TABLESPACE_NAME,FILE_NAME,BYTES/1024/1024 size_m from dba_data_files;

TABLESPACE FILE_NAME                                              SIZE_M
---------- -------------------------------------------------- ----------
SYSTEM     /u01/app/oracle/oradata/dgt/pdb1/system01.dbf             260
SYSAUX     /u01/app/oracle/oradata/dgt/pdb1/sysaux01.dbf             420
UNDOTBS1   /u01/app/oracle/oradata/dgt/pdb1/undotbs01.dbf            100
USERS      /u01/app/oracle/oradata/dgt/pdb1/users01.dbf             22.5

alter tablespace USERS add datafile '/u01/app/oracle/oradata/dgt/pdb1/users02.dbf' size 200m;

select sysdate from dual;

SYSDATE
-------------------
2017-07-14 13:35:18

alter database datafile '/u01/app/oracle/oradata/dgt/pdb1/users02.dbf' resize 190M;

select sysdate from dual;

SYSDATE
-------------------
2017-07-14 13:38:07

# 按以下步骤是无法闪回数据库的
recover managed standby database cancel;
alter database close;
flashback database to timestamp to_timestamp('2017-07-14 13:35:18','yyyy-mm-dd hh24:mi:ss');
flashback database to timestamp to_timestamp('2017-07-14 13:35:18','yyyy-mm-dd hh24:mi:ss')
*
ERROR at line 1:
ORA-38766: cannot flashback data file 13; file resized smaller
ORA-01110: data file 13: '/u01/app/oracle/oradata/dgt/pdb1/users02.dbf'
```

## 开启12c Data Guard压缩归档
``` perl
edit database dgtp set property RedoCompression = 'ENABLE';
# 关闭压缩归档
edit database dgtp set property RedoCompression = 'DISABLE';
```

## 脚本
``` perl
sqlplus -s / as sysdba <<EOF
select name, decode(cdb, 'YES', 'Multitenant Option enabled', 'Regular 12c Database: ') "Multitenant Option" , open_mode, con_id from v\$database;
show pdbs;
set linesize 200
col name format a10
col open_time format a35
select con_id,dbid,con_uid,name,open_mode,open_time,trunc(total_size/1024/1024) size_MB from v\$pdbs;
EOF

sqlplus -s / as sysdba <<EOF
select sess.con_id,pdbs.name,count(*)from v\$session sess,v\$pdbs pdbs where pdbs.con_id=sess.con_id group by sess.con_id,pdbs.name;
EOF
```
