---
title: CRM库DATAGUARD_FAILOVER后的恢复处理
date: 2017-06-12
categories: 
- workbook
tags:
- dataguard
---

## 环境
``` perl
主库     crmpn      10.240.3.123
备库     crmsn      10.240.4.124
```

## 备份(主库和备库都做)
``` perl
expdp system/12jca7or1B \
DIRECTORY=dump_dir \
DUMPFILE="$ORACLE_SID"_expdp_`date +%Y%m%d`_0a_%U.dmp \
LOGFILE="$ORACLE_SID"_expdp_`date +%Y%m%d`_0a.log \
PARALLEL=4 \
compression=ALL \
SCHEMAS=BFAPP8,BFCRM8,BFPUB8,CRMDR,CRMQL_RW,DNB_HSS,DNB_XY,DNB_YX,DNB_ZCM,YDJK,CY_TEST
```

<!-- more -->

## ORACLE 11G ACTIVE DATAGUARD FAILOVER后重新启用原主库
``` perl
select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
CRM       crmpn                          READ WRITE           PRIMARY

show parameter dg
#------------------------------------------------------------------------------------
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name               string
dg_broker_config_file1               string      +DATA/dr1crmpn.dat
dg_broker_config_file2               string      +DATA/dr2crmpn.dat
dg_broker_start                      boolean     FALSE
#------------------------------------------------------------------------------------

ASMCMD> rm dr1crmpn.dat
ASMCMD> rm dr2crmpn.dat

SQL> show parameter service

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
service_names                        string      crm

```

## dbca删除备库
``` perl
export DISPLAY=10.240.4.150:0.0
dbca
```

## 重建备库前准备
``` perl
# 主库
create pfile from spfile;
alter system set dg_broker_start=true;

# 拷贝 pfile 和 passwd 文件到备库
cd $ORACLE_HOME/dbs/

# 主库
scp orapw$ORACLE_SID oracle@crmsn:/u01/app/oracle/product/11.2.0.4/db_1/dbs/orapwcrmsn
scp init$ORACLE_SID.ora oracle@crmsn:/u01/app/oracle/product/11.2.0.4/db_1/dbs/initcrmsn.ora

# 检查备库的tnsnames.ora和监听配置
vi $ORACLE_HOME/network/admin/tnsnames.ora

CRMPN =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.3.123)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = crmpn)
    )
  )

CRMSN =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.3.124)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = crmsn)
    )
  )

# 修改备库pfile
vi $ORACLE_HOME/dbs/initcrmsn.ora
#-----------------------------------------------------------------
#将其中的dgtp改成dgts
:%s/crmpn/crmsn/g
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
rman target sys/12jca7or1B@crmpn auxiliary sys/12jca7or1B@crmsn nocatalog

duplicate target database for standby from active database nofilenamecheck;

# 退出rman登录sqlplus
exit
sqlplus / as sysdba

# 查看数据库状态
select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
CRM       crmsn                          MOUNTED              PHYSICAL STANDBY

SQL> show parameter service

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
service_names                        string      crmsn
SQL> show parameter dg

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
cell_offloadgroup_name               string
dg_broker_config_file1               string      +DATA/dr1crmsn.dat
dg_broker_config_file2               string      +DATA/dr2crmsn.dat
dg_broker_start                      boolean     TRUE

# root用户下
vi /etc/oratab 
#-----------------------------------------------------------------
crmsn:/u01/app/oracle/product/11.2.0.4/db_1:N            # line added by Agent
#-----------------------------------------------------------------
```

## 配置并启用 dg_broker
``` perl
dgmgrl /
create configuration dg_db as primary database is crmpn connect identifier is crmpn;
add database crmsn as connect identifier is crmsn maintained as physical;
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
dgmgrl sys/12jca7or1B@crmpn
dgmgrl sys/12jca7or1B@crmsn
```

## 遇到的问题
``` perl
# 据说是ORACLE的一个BUG，可以忽略不影响使用
ERROR: failed to establish dependency between database crmsn and diskgroup resource ora.DATA.dg
ERROR: failed to establish dependency between database crmsn and diskgroup resource ora.FRA.dg

# 应该是tnsnames.ora配置有问题，使用OEM做DG时，会在原有的基础上加项
PING[ARC2]: Heartbeat failed to connect to standby 'crmsn'. Error is 16057.

# 大量的报错，从trace中看，发出原因是oracle agent所引起，因为数据库或者监听在agent启动之后才启动起来，导致这个故障
# 解决的办法：先启动数据库实例和监听，然后再启动agent,或者将agent停掉，等数据库和监听启动以后，再启动agent
Errors in file /u01/app/oracle/diag/rdbms/crmsn/crmsn/trace/crmsn_ora_21152.trc:
vi /u01/app/oracle/diag/rdbms/crmsn/crmsn/trace/crmsn_ora_21152.trc
#------------------------------------------------------------
*** 2017-06-13 01:39:29.809
*** SESSION ID:(561.1) 2017-06-13 01:39:29.809
*** CLIENT ID:() 2017-06-13 01:39:29.809
*** SERVICE NAME:(SYS$USERS) 2017-06-13 01:39:29.809
*** MODULE NAME:(emagent_SQL_oracle_database) 2017-06-13 01:39:29.809
*** ACTION NAME:(incident_meter) 2017-06-13 01:39:29.809

psdgbt: bind csid (1) does not match session csid (852)
#------------------------------------------------------------
```