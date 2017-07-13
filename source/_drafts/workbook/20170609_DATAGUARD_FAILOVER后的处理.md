---
title: DATAGUARD FAILOVER备库后，重新启用原主库并重建备库
date: 2017-06-09
categories: 
- workbook
tags:
- dataguard
---

## 环境
``` perl
原主库     dgtp      10.240.4.172
原备库     dgts      10.240.4.173

# 主库
shutdown abort

# 备库
dgmgrl /
failover to dgts;
```

<!-- more -->

## ORACLE 11G ACTIVE DATAGUARD FAILOVER后重新启用原主库
``` perl
# 原备库，新主库上
dgmgrl /
show configuration;
#------------------------------------------------
Configuration - dg_db

  Protection Mode: MaxPerformance
  Databases:
    dgts - Primary database
    dgtp - Physical standby database (disabled)
      ORA-16661: the standby database needs to be reinstated

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS
#------------------------------------------------

select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgts                           READ WRITE           PRIMARY

shutdown immediate


# 在原主库
startup
#------------------------------------------------------------------------------------
ORACLE instance started.

Total System Global Area 4409401344 bytes
Fixed Size                  2260408 bytes
Variable Size             855638600 bytes
Database Buffers         3539992576 bytes
Redo Buffers               11509760 bytes
Database mounted.
ORA-16649: possible failover to another database prevents this database from being opened
#------------------------------------------------------------------------------------

show parameter dg
#------------------------------------------------------------------------------------
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name               string
dg_broker_config_file1               string      /u01/app/oracle/product/11.2.0
                                                 .4/db_1/dbs/dr1dgtp.dat
dg_broker_config_file2               string      /u01/app/oracle/product/11.2.0
                                                 .4/db_1/dbs/dr2dgtp.dat
dg_broker_start                      boolean     TRUE
#------------------------------------------------------------------------------------

select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgtp                           MOUNTED              PRIMARY

show parameter service

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
service_names                        string      dgtp

alter system set dg_broker_start=false;

!mv /u01/app/oracle/product/11.2.0.4/db_1/dbs/dr1dgtp.dat /tmp
!mv /u01/app/oracle/product/11.2.0.4/db_1/dbs/dr2dgtp.dat /tmp

alter database open;

SQL> select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgtp                           READ WRITE           PRIMARY

SQL> show parameter service

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
service_names                        string      dgt
```

## dbca删除备库
``` perl
startup restrict
alter system set service_names='dgts';

export DISPLAY=10.240.4.150:0.0
dbca
```

## 重建备库前准备
``` perl
# 主库
alter system set dg_broker_start=true;
create pfile from spfile;

# 拷贝 pfile 和 passwd 文件到备库
cd $ORACLE_HOME/dbs/

# 主库
scp orapw$ORACLE_SID oracle@dgts:/u01/app/oracle/product/11.2.0.4/db_1/dbs/orapwdgts
scp init$ORACLE_SID.ora oracle@dgts:/u01/app/oracle/product/11.2.0.4/db_1/dbs/initdgts.ora

# 检查备库的tnsnames.ora和监听配置
vi $ORACLE_HOME/network/admin/tnsnames.ora

# 修改备库pfile
vi $ORACLE_HOME/dbs/initdgts.ora
#-----------------------------------------------------------------
#将其中的dgtp改成dgts
:%s/dgtp/dgts/g
#-----------------------------------------------------------------

# 创建必要的目录
mkdir -p /u01/app/oracle/admin/"$ORACLE_SID"/adump
```

## 创建备库
``` perl
# 备库上，根据pfile创建spfile
sqlplus / as sysdba
create spfile from pfile;
startup nomount

# 在备库执行以下命令开始创建备库
rman target sys/Center08@dgtp auxiliary sys/Center08@dgts nocatalog

duplicate target database for standby from active database nofilenamecheck;

# 退出rman登录sqlplus
exit
sqlplus / as sysdba

# 查看数据库状态
select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgts                           MOUNTED              PHYSICAL STANDBY

alter system set dg_broker_start=true;

# root用户下
vi /etc/oratab 
#-----------------------------------------------------------------
dgts:/u01/app/oracle/product/11.2.0.4/db_1:N            # line added by Agent
#-----------------------------------------------------------------
```

## 配置并启用 dg_broker
``` perl
dgmgrl /
create configuration dg_db as primary database is dgtp connect identifier is dgtp;
add database dgts as connect identifier is dgts maintained as physical;
enable configuration;

show configuration;

Configuration - dg_db

  Protection Mode: MaxPerformance
  Databases:
    dgtp - Primary database
    dgts - Physical standby database

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS
```

## 打开备库到 active dg
``` perl
alter database open;
select name,db_unique_name,open_mode,database_role from v$database;
NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgts                           READ ONLY WITH APPLY PHYSICAL STANDBY
```

## 切换测试
``` perl
dgmgrl sys/Center08@dgtp
dgmgrl sys/Center08@dgts
```