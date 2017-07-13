---
title: Oracle 12c特性解读-容器数据库和灾备-09 12c 灾备环境搭建
date: 2017-07-02
tags:
- oracle
- 12c
---

## Snapshot Standby
### 查看数据库状态
``` perl
# 主库状态
set line 150
col DB_UNIQUE_NAME for a15
col FORCE_LOGGING for a15
select name,db_unique_name,open_mode,database_role,flashback_on,force_logging,log_mode from v$database;

NAME      DB_UNIQUE_NAME  OPEN_MODE            DATABASE_ROLE    FLASHBACK_ON       FORCE_LOGGING   LOG_MODE
--------- --------------- -------------------- ---------------- ------------------ --------------- ------------
DGT       DGTP            READ WRITE           PRIMARY          YES                YES             ARCHIVELOG

# 查看备库状态
set line 150
col DB_UNIQUE_NAME for a15
col FORCE_LOGGING for a15
select name,db_unique_name,open_mode,database_role,flashback_on,force_logging,log_mode from v$database;

NAME      DB_UNIQUE_NAME  OPEN_MODE            DATABASE_ROLE    FLASHBACK_ON       FORCE_LOGGING   LOG_MODE
--------- --------------- -------------------- ---------------- ------------------ --------------- ------------
DGT       DGTS            READ ONLY WITH APPLY PHYSICAL STANDBY YES                YES             ARCHIVELOG
```

<!-- more -->

### 切换到Snapshot Standby
``` perl
# 在主库上执行
dgmgrl sys/Center08@dgtp
DGMGRL for Linux: Release 12.2.0.1.0 - Production on Fri Jun 30 11:18:13 2017

Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "dgts"
Connected as SYSDG.
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 49 seconds ago)

DGMGRL> help convert

Converts a database from one type to another

Syntax:

  CONVERT DATABASE <database name> TO
     { SNAPSHOT STANDBY | PHYSICAL STANDBY };

DGMGRL> convert database dgts to snapshot standby;
Converting database "dgts" to a Snapshot Standby database, please wait...
Database "dgts" converted successfully

DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Snapshot standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 33 seconds ago)
```

### 在SNAPSHOT STANDBY上增加测试数据
``` perl
# 主库PDB1下表
conn lyj/lyj@pdb1
select * from cat;

TABLE_NAME                                         TABLE_TYPE
-------------------------------------------------- -----------
TEST                                               TABLE

# 备库tnsnames中添加pdb1备库地址
SPDB1 =
  (DESCRIPTION =
  (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.175)(PORT = 1521))
  )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb1)
    )
  )


# 查看备库状态
conn / as sysdba
set line 150
col DB_UNIQUE_NAME for a15
col FORCE_LOGGING for a15
select name,db_unique_name,open_mode,database_role,flashback_on,force_logging,log_mode from v$database;

NAME      DB_UNIQUE_NAME  OPEN_MODE            DATABASE_ROLE    FLASHBACK_ON       FORCE_LOGGING   LOG_MODE
--------- --------------- -------------------- ---------------- ------------------ --------------- ------------
DGT       DGTS            READ WRITE           SNAPSHOT STANDBY YES                YES             ARCHIVELOG

show pdbs

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO

conn lyj/lyj@spdb1

create table test1 as select * from all_objects;
select count(*) from test1;

  COUNT(*)
----------
     56326

drop table test purge;
Table dropped.

select * from cat;

TABLE_NAME                                         TABLE_TYPE
-------------------------------------------------- -----------
TEST1                                              TABLE
```

### 备库切换回PHYSICAL STANDBY
``` perl
# 在主库上执行
dgmgrl sys/Center08@dgtp

DGMGRL> convert database dgts to physical standby;
Converting database "dgts" to a Physical Standby database, please wait...
Operation requires shut down of instance "dgt" on database "dgts"
Shutting down instance "dgt"...
Connected to "DGTS"
Database closed.
Database dismounted.
ORACLE instance shut down.
Operation requires start up of instance "dgt" on database "dgts"
Starting instance "dgt"...
ORACLE instance started.
Database mounted.
Connected to "DGTS"
Continuing to convert database "dgts" ...
Database "dgts" converted successfully

# 查看备库状态
set line 150
col DB_UNIQUE_NAME for a15
col FORCE_LOGGING for a15
select name,db_unique_name,open_mode,database_role,flashback_on,force_logging,log_mode from v$database;

NAME      DB_UNIQUE_NAME  OPEN_MODE            DATABASE_ROLE    FLASHBACK_ON       FORCE_LOGGING   LOG_MODE
--------- --------------- -------------------- ---------------- ------------------ --------------- ------------
DGT       DGTS            READ ONLY WITH APPLY PHYSICAL STANDBY YES                YES             ARCHIVELOG

# 查看刚才创建和删除的表都没有了，数据和主库保持一致
conn lyj/lyj@spdb1

TABLE_NAME                                         TABLE_TYPE
-------------------------------------------------- -----------
TEST                                               TABLE
```

## 跳归档恢复

### 查看主库redo和归档状态
``` perl
SQL> select GROUP#,STATUS from v$log;

    GROUP# STATUS
---------- ----------------
         1 ACTIVE
         2 CURRENT
         3 INACTIVE

# 最后一个归档编号40
RMAN> crosscheck archivelog all;
......
archived log file name=/u01/app/oracle/fast_recovery_area/dgt/DGTP/archivelog/2017_07_02/o1_mf_1_40_dokd56wz_.arc RECID=71 STAMP=948299815
Crosschecked 39 objects
```

### 停止备库模拟主库归档丢失
``` perl
shutdown immediate

# 在主库创建测试数据
conn lyj/lyj@pdb1
create table test2 as select * from test;
conn / as sysdba
alter system switch logfile;
SQL> select GROUP#,STATUS from v$log;

    GROUP# STATUS
---------- ----------------
         1 INACTIVE
         2 ACTIVE
         3 CURRENT
alter system switch logfile;

# 模拟归档丢失
cd /u01/app/oracle/fast_recovery_area/dgt/DGTP/archivelog/2017_07_02
ll
total 634884
-rw-r-----. 1 oracle oinstall 177589760 Jul  2 02:33 o1_mf_1_37_dohtqjkb_.arc
-rw-r-----. 1 oracle oinstall 174428672 Jul  2 08:13 o1_mf_1_38_dojgp4kz_.arc
-rw-r-----. 1 oracle oinstall 174482432 Jul  2 14:00 o1_mf_1_39_dok2znlx_.arc
-rw-r-----. 1 oracle oinstall 109455360 Jul  2 16:36 o1_mf_1_40_dokd56wz_.arc
-rw-r-----. 1 oracle oinstall  14153728 Jul  2 16:48 o1_mf_1_41_dokdvmw8_.arc  # 模拟丢失这个归档
-rw-r-----. 1 oracle oinstall    198656 Jul  2 16:52 o1_mf_1_42_dokf32hs_.arc

mv o1_mf_1_41_dokdvmw8_.arc o1_mf_1_41_dokdvmw8_.arc.bak

# 备库打开mount状态
startup mount

# 查看备库归档目录，发现41号归档没有传输到备库
ll
total 625064
-rw-r-----. 1 oracle oinstall 177589760 Jul  2 02:33 o1_mf_1_37_dohtqjnk_.arc
-rw-r-----. 1 oracle oinstall 174428672 Jul  2 08:13 o1_mf_1_38_dojgp4o6_.arc
-rw-r-----. 1 oracle oinstall 174482432 Jul  2 14:00 o1_mf_1_39_dok2znov_.arc
-rw-r-----. 1 oracle oinstall 109455360 Jul  2 16:36 o1_mf_1_40_dokd570y_.arc
-rw-r-----. 1 oracle oinstall    198656 Jul  2 16:55 o1_mf_1_42_dokf85kk_.arc
-rw-r-----. 1 oracle oinstall   3896832 Jul  2 16:55 o1_mf_1_43_dokf85jr_.arc

# 查看DG BROKER状态，有GAP Error
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    Error: ORA-16810: multiple errors or warnings detected for the member

    dgts - Physical standby database 
      Warning: ORA-16809: multiple warnings detected for the member

Fast-Start Failover: DISABLED

Configuration Status:
ERROR   (status updated 39 seconds ago)

DGMGRL> show database verbose dgtp;

Database - dgtp

  Role:               PRIMARY
  Intended State:     TRANSPORT-ON
  Instance(s):
    dgt
      Error: ORA-16737: the redo transport service for member "dgts" has an error

  Database Error(s):
    ORA-16783: cannot resolve gap for member dgts
......


# 查看备库当前SCN号
SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    2606427

# 查看主库当前SCN号
SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    2615063
```

### 跟归档恢复
``` perl
# SCN增量备份，同时创建备库控制文件
rman target /
backup incremental from scn 2606427 database format '/tmp/incre_%U' tag 'for_standby';
alter database create standby controlfile as '/tmp/std_con.ctl';

# 将备份集和备库控制文件拷贝到备份
cd /tmp
scp incre* dgts12:/tmp/incre_backup  # 没有这个目录的话可以先创建
scp std_con.ctl dgts12:/tmp

# 启动备库到nomount
shutdown immediate
startup nomount

# 恢复控制文件
RMAN> restore controlfile from '/tmp/std_con.ctl';

Starting restore at 2017-07-02 17:26:37
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=743 device type=DISK

channel ORA_DISK_1: copied control file copy
output file name=/u01/app/oracle/oradata/dgt/control01.ctl
output file name=/u01/app/oracle/fast_recovery_area/dgt/control02.ctl
Finished restore at 2017-07-02 17:26:39

RMAN> alter database mount;

Statement processed
released channel: ORA_DISK_1

RMAN> catalog start with '/tmp/incre_backup';
......
File Name: /u01/app/oracle/fast_recovery_area/dgt/DGTS/autobackup/2017_06_30/o1_mf_s_948045937_dod8mk3g_.bkp

searching for all files that match the pattern /tmp/incre_backup

List of Files Unknown to the Database

File Name: /tmp/incre_backup/incre_3gs8bs72_1_1
File Name: /tmp/incre_backup/incre_3hs8bs79_1_1
File Name: /tmp/incre_backup/incre_3js8bs7c_1_1

Do you really want to catalog the above files (enter YES or NO)? yes
cataloging files...
cataloging done

List of Cataloged Files

File Name: /tmp/incre_backup/incre_3gs8bs72_1_1
File Name: /tmp/incre_backup/incre_3hs8bs79_1_1
File Name: /tmp/incre_backup/incre_3js8bs7c_1_1

RMAN> recover database noredo;

# 开启恢复模式
recover managed standby database disconnect from session;
```

