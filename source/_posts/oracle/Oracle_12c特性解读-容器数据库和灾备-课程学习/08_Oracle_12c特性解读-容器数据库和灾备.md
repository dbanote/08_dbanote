---
title: Oracle 12c特性解读-容器数据库和灾备-08 容灾简介和环境搭建
date: 2017-06-23
tags:
- oracle
- 12c
---

## 搭建12c的Data Guard环境
### 主备库环境配置
``` perl
# 主库已安数据，备库只安装了数据库软件，环境配置如下
cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.240.4.174  dgtp12  dgtp12.ydgwng.cn
10.240.4.175  dgts12  dgts12.ydgwng.cn

echo "export ORACLE_SID=dgt" >> /home/oracle/.bash_profile
echo "export ORACLE_BASE=/u01/app/oracle" >> /home/oracle/.bash_profile
echo "export ORACLE_HOME=\$ORACLE_BASE/product/12.2.0/db_1" >> /home/oracle/.bash_profile
echo "export LD_LIBRARY_PATH=\$ORACLE_HOME/lib" >> /home/oracle/.bash_profile
echo "export PATH=\$PATH:\$HOME/bin:\$ORACLE_HOME/bin:\$ORACLE_HOME/OPatch" >> /home/oracle/.bash_profile
echo "export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK" >> /home/oracle/.bash_profile
echo "export NLS_DATE_FORMAT='yyyy-mm-dd hh24:mi:ss'" >> /home/oracle/.bash_profile
echo "alias sqlplus='rlwrap sqlplus'" >> /home/oracle/.bash_profile
echo "alias asmcmd='rlwrap asmcmd'" >> /home/oracle/.bash_profile
echo "alias rman='rlwrap rman'" >> /home/oracle/.bash_profile
echo "alias dgmgrl='rlwrap dgmgrl'" >> /home/oracle/.bash_profile
```

<!-- more -->
### 查看/修改主库配置
``` perl
select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgt                            READ WRITE           PRIMARY

alter system set db_unique_name="dgtp" scope=spfile; 

select LOG_MODE,FORCE_LOGGING from v$database;

LOG_MODE     FORCE_LOGGING
------------ ---------------------------------------
NOARCHIVELOG NO

alter database force logging;
sthutdown immediate
startup mount
alter database archivelog;
alter database open;

select NAME,DB_UNIQUE_NAME,LOG_MODE,FORCE_LOGGING from v$database;

NAME      DB_UNIQUE_NAME                 LOG_MODE     FORCE_LOGGING
--------- ------------------------------ ------------ ---------------------------------------
DGT       DGTP                           ARCHIVELOG   YES

select MEMBERS,BYTES/1024/1024 from v$log;

   MEMBERS BYTES/1024/1024
---------- ---------------
         1             200
         1             200
         1             200

select MEMBER from v$logfile;

MEMBER
--------------------------------------------------
/u01/app/oracle/oradata/dgt/redo03.log
/u01/app/oracle/oradata/dgt/redo02.log
/u01/app/oracle/oradata/dgt/redo01.log        

alter database add standby logfile group 4 '/u01/app/oracle/oradata/dgt/redo04.log' size 200m;
alter database add standby logfile group 5 '/u01/app/oracle/oradata/dgt/redo05.log' size 200m;
alter database add standby logfile group 6 '/u01/app/oracle/oradata/dgt/redo06.log' size 200m;
alter database add standby logfile group 7 '/u01/app/oracle/oradata/dgt/redo07.log' size 200m;

select TYPE,MEMBER from v$logfile;

TYPE    MEMBER
------- --------------------------------------------------
ONLINE  /u01/app/oracle/oradata/dgt/redo03.log
ONLINE  /u01/app/oracle/oradata/dgt/redo02.log
ONLINE  /u01/app/oracle/oradata/dgt/redo01.log
STANDBY /u01/app/oracle/oradata/dgt/redo04.log
STANDBY /u01/app/oracle/oradata/dgt/redo05.log
STANDBY /u01/app/oracle/oradata/dgt/redo06.log
STANDBY /u01/app/oracle/oradata/dgt/redo07.log

alter system set standby_file_management=auto;
```

### 配置备库pfile和orapwd
``` perl
# 在主库上
create pfile from spfile;

cd $ORACLE_HOME/dbs
scp initdgt.ora 10.240.4.175:$ORACLE_HOME/dbs/
scp orapwdgt 10.240.4.175:$ORACLE_HOME/dbs/

# 在备库上
vi $ORACLE_HOME/dbs/initdgt.ora
#--------------------------------------------------------
# 修改
*.db_unique_name='dgts'
#--------------------------------------------------------

# 创建所需目录
mkdir -p /u01/app/oracle/admin/dgt/adump
mkdir -p /u01/app/oracle/oradata/dgt
```

### 配置主备库LISTENER和tnsname
``` perl
cd $ORACLE_HOME/network/admin
vi listener.ora
#--------------------------------------------------------
#添加
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = dgtp)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = dgt)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = dgtp_DGB)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = dgt)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = DGTP_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = dgt)
    )
    (SID_DESC =
      (GLOBAL_DBNAME = pdb1)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = dgt)
    )
)
#--------------------------------------------------------

lsnrctl reload

vi tnsnames.ora 
#--------------------------------------------------------
# 变为
DGTP =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.174)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = dgtp)
    )
  )

DGTS =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.175)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = dgts)
    )
  )

PDB1 = 
  (DESCRIPTION =
  (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.174)(PORT = 1521))
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.240.4.175)(PORT = 1521))
  )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb1)
    )
  )
#--------------------------------------------------------

scp tnsnames.ora 10.240.4.175:$ORACLE_HOME/network/admin/
scp listener.ora 10.240.4.175:$ORACLE_HOME/network/admin/

# 备库listener.ora中将dgtp改成dgts，然后重启监听

# 验证配置
tnsping dgtp
tnsping dgts

sqlplus sys/Center08@dgtp as sysdba
sqlplus sys/Center08@dgts as sysdba
```

### 在主库的PDB1下面创建测试数据
```
show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1    
alter pluggable database pdb1 open;
alter pluggable database pdb1 save state;
alter session set container=pdb1;
create user lyj identified by lyj default tablespace users;
grant connect,resource to lyj;
alter user lyj quota unlimited on users;

conn lyj/lyj@pdb1
create table test as select * from all_objects;
```

### 创建备库
``` perl
# 备库上，根据pfile创建spfile
sqlplus / as sysdba
create spfile from pfile;
startup nomount

# 在备库上执行
rman target sys/Center08@dgtp auxiliary sys/Center08@dgts nocatalog

Recovery Manager: Release 12.2.0.1.0 - Production on Fri Jun 23 18:05:14 2017

Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.

connected to target database: DGT (DBID=1555489009)
using target database control file instead of recovery catalog
connected to auxiliary database: DGT (not mounted)

duplicate target database for standby from active database nofilenamecheck;
#--------------------------------------------------------
ORACLE error from auxiliary database: ORA-19527: physical standby redo log must be renamed
ORA-00312: online log 1 thread 1: '/u01/app/oracle/oradata/dgt/redo01.log'

RMAN-05535: warning: All redo log files were not defined properly.
ORACLE error from auxiliary database: ORA-19527: physical standby redo log must be renamed
ORA-00312: online log 2 thread 1: '/u01/app/oracle/oradata/dgt/redo02.log'

RMAN-05535: warning: All redo log files were not defined properly.
ORACLE error from auxiliary database: ORA-19527: physical standby redo log must be renamed
ORA-00312: online log 3 thread 1: '/u01/app/oracle/oradata/dgt/redo03.log'

RMAN-05535: warning: All redo log files were not defined properly.
ORACLE error from auxiliary database: ORA-19527: physical standby redo log must be renamed
ORA-00312: online log 4 thread 0: '/u01/app/oracle/oradata/dgt/redo04.log'

RMAN-05535: warning: All redo log files were not defined properly.
ORACLE error from auxiliary database: ORA-19527: physical standby redo log must be renamed
ORA-00312: online log 5 thread 0: '/u01/app/oracle/oradata/dgt/redo05.log'

RMAN-05535: warning: All redo log files were not defined properly.
ORACLE error from auxiliary database: ORA-19527: physical standby redo log must be renamed
ORA-00312: online log 6 thread 0: '/u01/app/oracle/oradata/dgt/redo06.log'

RMAN-05535: warning: All redo log files were not defined properly.
ORACLE error from auxiliary database: ORA-19527: physical standby redo log must be renamed
ORA-00312: online log 7 thread 0: '/u01/app/oracle/oradata/dgt/redo07.log'

RMAN-05535: warning: All redo log files were not defined properly.
Finished Duplicate Db at 2017-06-23 18:06:15
#--------------------------------------------------------

# 解决办法
vi $ORACLE_HOME/dbs/initdgt.ora
#--------------------------------------------------------
添加
*.log_file_name_convert='/u01/app/oracle/oradata/dgt','/u01/app/oracle/oradata/dgt' 
*.db_file_name_convert='/u01/app/oracle/oradata/dgt','/u01/app/oracle/oradata/dgt'
#--------------------------------------------------------

# 删除相关文件后再次重新执行成功
cd /u01/app/oracle/oradata/dgt/
rm -rf *

sqlplus / as sysdba
create spfile from pfile;
startup nomount

rman target sys/Center08@dgtp auxiliary sys/Center08@dgts nocatalog
duplicate target database for standby from active database nofilenamecheck;

pwd
/u01/app/oracle/oradata/dgt
[oracle@dgts12 dgt]$ ll
total 2842396
-rw-r-----. 1 oracle oinstall  18726912 Jun 23 18:26 control01.ctl
-rw-r-----. 1 oracle oinstall  18726912 Jun 23 18:26 control02.ctl
drwxr-x---. 2 oracle oinstall        82 Jun 23 18:20 pdb1
drwxr-x---. 2 oracle oinstall        64 Jun 23 18:20 pdbseed
-rw-r-----. 1 oracle oinstall 209715712 Jun 23 18:20 redo01.log
-rw-r-----. 1 oracle oinstall 209715712 Jun 23 18:20 redo02.log
-rw-r-----. 1 oracle oinstall 209715712 Jun 23 18:20 redo03.log
-rw-r-----. 1 oracle oinstall 209715712 Jun 23 18:20 redo04.log
-rw-r-----. 1 oracle oinstall 209715712 Jun 23 18:20 redo05.log
-rw-r-----. 1 oracle oinstall 209715712 Jun 23 18:20 redo06.log
-rw-r-----. 1 oracle oinstall 209715712 Jun 23 18:20 redo07.log
-rw-r-----. 1 oracle oinstall 492838912 Jun 23 18:20 sysaux01.dbf
-rw-r-----. 1 oracle oinstall 838868992 Jun 23 18:20 system01.dbf
-rw-r-----. 1 oracle oinstall  68165632 Jun 23 18:20 undotbs01.dbf
-rw-r-----. 1 oracle oinstall   5251072 Jun 23 18:20 users01.dbf

# root用户下
vi /etc/oratab 
#-----------------------------------------------------------------
dgt:/u01/app/oracle/product/12.2.0/db_1:N
#-----------------------------------------------------------------
#
#
[oracle@dgts12 dgt]$ 

sqlplus / as sysdba
SQL> select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       DGTS                           MOUNTED              PHYSICAL STANDBY

alter database open;
```


### 配置dg_broker
``` perl
# 在主备库上同时设置
alter system set dg_broker_start=true;

dgmgrl /

DGMGRL> show configuration;
ORA-16532: Oracle Data Guard broker configuration does not exist

Configuration details cannot be determined by DGMGRL

DGMGRL> help create

Creates a broker configuration

Syntax:

  CREATE CONFIGURATION <configuration name> [AS] 
    PRIMARY DATABASE IS <database name>
    CONNECT IDENTIFIER IS <connect identifier>;

DGMGRL> help add

Adds a member to the broker configuration

Syntax:

  ADD { RECOVERY_APPLIANCE | DATABASE | FAR_SYNC } <object name>
    [AS CONNECT IDENTIFIER IS <connect identifier>];

# 创建配置文件
DGMGRL> create CONFIGURATION dgt_dg as PRIMARY DATABASE IS dgtp CONNECT IDENTIFIER IS dgtp;
Configuration "dgt_dg" created with primary database "dgtp"
DGMGRL> add DATABASE dgts AS CONNECT IDENTIFIER IS dgts maintained as physical;
Database "dgts" added

DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
DISABLED

DGMGRL> enable configuration;

DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 0 seconds ago)


SQL> select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgts                           READ ONLY WITH APPLY PHYSICAL STANDBY
```