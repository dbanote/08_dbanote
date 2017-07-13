---
title: Oracle 12c特性解读-容器数据库和灾备-10 容灾切换和故障演练实践总结
date: 2017-07-04
tags:
- oracle
- 12c
---

## 练习DG Broker switchover和failover的过程
### 练习switchover
``` perl
# switchover可以在主或备库上运行
# 使用TNS的方式登陆DG Broker
$ dgmgrl sys/Center08@dgtp
DGMGRL for Linux: Release 12.2.0.1.0 - Production on Tue Jul 4 15:01:13 2017

Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "DGTP"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 13 seconds ago)
```

<!-- more -->
switchover测试
``` perl
# switchover到备库dgts
DGMGRL> switchover to dgts;
Performing switchover NOW, please wait...
Operation requires a connection to database "dgts"
Connecting ...
Connected to "DGTS"
Connected as SYSDBA.
New primary database "dgts" is opening...
Operation requires start up of instance "dgt" on database "dgtp"
Starting instance "dgt"...
ORACLE instance started.
Database mounted.
Database opened.
Connected to "DGTP"
Switchover succeeded, new primary is "dgts"
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgts - Primary database
    dgtp - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 71 seconds ago)

# switchover到原主库dgtp
DGMGRL> switchover to dgtp;
Performing switchover NOW, please wait...
Operation requires a connection to database "dgtp"
Connecting ...
Connected to "DGTP"
Connected as SYSDBA.
New primary database "dgtp" is opening...
Operation requires start up of instance "dgt" on database "dgts"
Starting instance "dgt"...
ORACLE instance started.
Database mounted.
Database opened.
Connected to "DGTS"
Switchover succeeded, new primary is "dgtp"
```

### 练习failover
``` perl
# 模拟主库宕机
sqlplus sys/Center08@dgtp as sysdba
shutdown abort

# 在备库上执行failover，failover只能在备库上运行
dgmgrl sys/Center08@dgts

DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    Error: ORA-1034: ORACLE not available

    dgts - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
ERROR   (status updated 0 seconds ago)

# 成功执行failover
DGMGRL> failover to dgts;
Performing failover NOW, please wait...
Failover succeeded, new primary is "dgts"

# 提示要原主库需要执行reinstated操作，才能变更为物理备库
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgts - Primary database
    dgtp - Physical standby database (disabled)
      ORA-16661: the standby database needs to be reinstated

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 16 seconds ago)


# 主库恢复后是无法直接打开
SQL> startup
ORACLE instance started.

Total System Global Area 3992977408 bytes
Fixed Size                  8800136 bytes
Variable Size            1056966776 bytes
Database Buffers         2919235584 bytes
Redo Buffers                7974912 bytes
Database mounted.
ORA-16649: possible failover to another database prevents this database from
being opened

SQL> select open_mode from v$database;

OPEN_MODE
--------------------
MOUNTED

# 在备库执行reinstate database，将原主库变为备库
DGMGRL> reinstate database dgtp;
Reinstating database "dgtp", please wait...
Reinstatement of database "dgtp" succeeded

DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgts - Primary database
    dgtp - Physical standby database 

Fast-Start Failover: DISABLED

Configuration Status:
SUCCESS   (status updated 44 seconds ago)

# 查看原主库状态
SQL> select open_mode,database_role from v$database;

OPEN_MODE            DATABASE_ROLE
-------------------- ----------------
READ ONLY WITH APPLY PHYSICAL STANDBY


```

## 练习自动化切换 Fast-Start Failover
### 配置开启Fast-Start Failover
``` perl
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - Physical standby database 

Fast-Start Failover: DISABLED   #  初始自动化切换是DISABLED

Configuration Status:
SUCCESS   (status updated 18 seconds ago)

# 1. 查看主/备库是否开启闪回数据库
## 备库开启闪回数据库用以下命令
recover managed standby database cancel;
alter database flashback on;
recover managed standby database disconnect from session;

# 2. LISTENER中要配置GLOBAL_DBNAME=db_unique_name_DGMGRL，如
   (SID_DESC =
      (GLOBAL_DBNAME = DGTP_DGMGRL)
      (ORACLE_HOME = /u01/app/oracle/product/12.2.0/db_1)
      (SID_NAME = dgt)
    )

# 3. 在DG BROKER中设置主备库的FastStartFailoverTarget
DGMGRL> edit database dgtp set property FastStartFailoverTarget='dgts';
Property "faststartfailovertarget" updated

DGMGRL> edit database dgts set property FastStartFailoverTarget='dgtp';
Property "faststartfailovertarget" updated

DGMGRL> show database verbose dgtp;
.....
LogXptMode                      = 'ASYNC'   # Protection Mode: MaxPerformance
.....
FastStartFailoverTarget         = 'dgts'
.....

# 4. 如果主备的设置为最大高可用保护模式，则需要设置LogXptMode为sync，最大性能保护模式，则需要设置LogXptMode为async

# 满足以上4个条件后，使用以下命令启用Fast-Start Failover
DGMGRL> enable fast_start failover;
Enabled.

DGMGRL> start observer;
[W000 07/04 16:13:04.06] FSFO target standby is dgts
[W000 07/04 16:13:06.22] Observer trace level is set to USER
[W000 07/04 16:13:06.22] Try to connect to the primary.
[W000 07/04 16:13:06.22] Try to connect to the primary dgtp.
[W000 07/04 16:13:06.23] The standby dgts is ready to be a FSFO target
[W000 07/04 16:13:08.23] Connection to the primary restored!
[W000 07/04 16:13:09.23] Disconnecting from database dgtp.

# 不要关闭上面的窗口，重新打开一个
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - (*) Physical standby database 

Fast-Start Failover: ENABLED    # 状态已经是ENABLED了

Configuration Status:
SUCCESS   (status updated 10 seconds ago)

```

### 模拟Fast-Start Failover
``` perl
# 主库shutdown abort
shutdown abort

# observer上
DGMGRL> start observer;
[W000 07/04 16:13:04.06] FSFO target standby is dgts
[W000 07/04 16:13:06.22] Observer trace level is set to USER
[W000 07/04 16:13:06.22] Try to connect to the primary.
[W000 07/04 16:13:06.22] Try to connect to the primary dgtp.
[W000 07/04 16:13:06.23] The standby dgts is ready to be a FSFO target
[W000 07/04 16:13:08.23] Connection to the primary restored!
[W000 07/04 16:13:09.23] Disconnecting from database dgtp.
[W000 07/04 16:16:36.48] Primary database cannot be reached.
[W000 07/04 16:16:36.48] Fast-Start Failover threshold has not exceeded. Retry for the next 30 seconds
[W000 07/04 16:16:37.48] Try to connect to the primary.
[W000 07/04 16:16:39.56] Primary database cannot be reached.
[W000 07/04 16:16:40.56] Try to connect to the primary.
[W000 07/04 16:17:04.15] Primary database cannot be reached.
[W000 07/04 16:17:04.15] Fast-Start Failover threshold has not exceeded. Retry for the next 2 seconds
[W000 07/04 16:17:05.15] Try to connect to the primary.
[W000 07/04 16:17:07.23] Primary database cannot be reached.
[W000 07/04 16:17:07.23] Fast-Start Failover threshold has expired.
[W000 07/04 16:17:07.23] Try to connect to the standby.
[W000 07/04 16:17:07.23] Making a last connection attempt to primary database before proceeding with Fast-Start Failover.
[W000 07/04 16:17:07.23] Check if the standby is ready for failover.
[S002 07/04 16:17:07.23] Fast-Start Failover started...

16:17:07.23  Tuesday, July 04, 2017
Initiating Fast-Start Failover to database "dgts"...
[S002 07/04 16:17:07.23] Initiating Fast-start Failover.
Performing failover NOW, please wait...
Failover succeeded, new primary is "dgts"
16:17:18.42  Tuesday, July 04, 2017
[S002 07/04 16:17:18.42] Fast-Start Failover finished...
[W000 07/04 16:17:18.42] Failover succeeded. Restart pinging.
[W000 07/04 16:17:18.43] Primary database has changed to dgts.
[W000 07/04 16:17:18.49] Try to connect to the primary.
[W000 07/04 16:17:18.49] Try to connect to the primary dgts.
[W000 07/04 16:17:19.59] Connection to the primary restored!
[W000 07/04 16:17:19.60] The standby dgtp needs to be reinstated
[W000 07/04 16:17:19.60] Try to connect to the new standby dgtp.
[W000 07/04 16:17:20.60] Disconnecting from database dgts.
[W000 07/04 16:17:22.60] Connection to the new standby restored!
[W000 07/04 16:17:22.61] Failed to ping the new standby.
.....

# 主库恢复后，自动Reinstate
SQL> startup
ORACLE instance started.

Total System Global Area 3992977408 bytes
Fixed Size                  8800136 bytes
Variable Size            1056966776 bytes
Database Buffers         2919235584 bytes
Redo Buffers                7974912 bytes
Database mounted.
ORA-16649: possible failover to another database prevents this database from
being opened


DGMGRL> show configuration;
ORA-16795: the standby database needs to be re-created

Configuration details cannot be determined by DGMGRL

# observer上
[W000 07/04 17:10:30.30] Try to connect to the primary dgts.
[W000 07/04 17:10:32.31] Connection to the primary restored!
[W000 07/04 17:10:33.31] Wait for new primary to be ready to reinstate.
[W000 07/04 17:10:34.31] New primary is now ready to reinstate.
[W000 07/04 17:10:34.31] Issuing REINSTATE command.      # 提示开始自动做REINSTATE

17:10:34.31  Tuesday, July 04, 2017
Initiating reinstatement for database "dgtp"...
Reinstating database "dgtp", please wait...
[W000 07/04 17:10:46.45] The standby dgtp is ready to be a FSFO target
Reinstatement of database "dgtp" succeeded
17:10:56.69  Tuesday, July 04, 2017
[W000 07/04 17:10:57.46] Successfully reinstated database dgtp.
[W000 07/04 17:10:58.46] The reinstatement of standby dgtp was just done
.....

[oracle@dgtp12 ~]$ sqlplus sys/Center08@dgtp as sysdba

SQL*Plus: Release 12.2.0.1.0 Production on Tue Jul 4 17:14:27 2017

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select open_mode,database_role from v$database;

OPEN_MODE            DATABASE_ROLE
-------------------- ----------------
READ ONLY WITH APPLY PHYSICAL STANDBY


[oracle@dgtp12 ~]$ dgmgrl sys/Center08@dgtp
DGMGRL for Linux: Release 12.2.0.1.0 - Production on Tue Jul 4 17:15:33 2017

Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "DGTP"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgts - Primary database
    dgtp - (*) Physical standby database 

Fast-Start Failover: ENABLED

Configuration Status:
SUCCESS   (status updated 42 seconds ago)
```

## 练习Active Data Guard连接会话保留的特性
``` perl
SQL> show parameter standby

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
enabled_PDBs_on_standby              string      *
standby_archive_dest                 string      ?#/dbs/arch
standby_db_preserve_states           string      NONE
standby_file_management              string      AUTO

# 主备库都执行
alter system set standby_db_preserve_states=all scope=spfile;

shutdown immediate
startup

# 在备库上
$ sqlplus system/Center08@dgts

SQL*Plus: Release 12.2.0.1.0 Production on Tue Jul 4 17:36:03 2017

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production

SQL> select count(*) from cat;

  COUNT(*)
----------
       153

SQL> set timing on
SQL> select count(*) from cat;

  COUNT(*)
----------
       153

Elapsed: 00:00:00.03

# 另开一个窗口做switchover
$ dgmgrl sys/Center08@dgts
DGMGRL for Linux: Release 12.2.0.1.0 - Production on Tue Jul 4 17:35:08 2017

Copyright (c) 1982, 2017, Oracle and/or its affiliates.  All rights reserved.

Welcome to DGMGRL, type "help" for information.
Connected to "DGTS"
Connected as SYSDBA.
DGMGRL> show configuration;

Configuration - dgt_dg

  Protection Mode: MaxPerformance
  Members:
  dgtp - Primary database
    dgts - (*) Physical standby database 

Fast-Start Failover: ENABLED

Configuration Status:
SUCCESS   (status updated 27 seconds ago)

# 回到上面的会话，在会话中执行查询
SQL> select count(*) from cat;

  COUNT(*)
----------
       153

Elapsed: 00:00:00.02
SQL> select count(*) from cat;

  COUNT(*)
----------
       153

Elapsed: 00:00:02.80    # 有2秒延迟，但会话不中断
SQL> select count(*) from cat;

  COUNT(*)
----------
       153

Elapsed: 00:00:00.07
```


## 其他
### DG BROKER中新增的validate命令
``` perl
DGMGRL> validate database verbose dgtp;

  Database Role:    Primary database

  Ready for Switchover:  Yes

  Flashback Database Status:
    dgtp:  On

  Capacity Information:
    Database  Instances        Threads        
    dgtp      1                1              

  Managed by Clusterware:
    dgtp:  NO             
    Warning: Ensure primary database s StaticConnectIdentifier property
    is configured properly so that the primary database can be restarted
    by DGMGRL after switchover

  Temporary Tablespace File Information:
    dgtp TEMP Files:  3

  Data file Online Move in Progress:
    dgtp:  No

  Transport-Related Information:
    Transport On:  Yes

  Log Files Cleared:
    dgtp Standby Redo Log Files:  Cleared

DGMGRL> validate database verbose dgts;

  Database Role:     Physical standby database
  Primary Database:  dgtp

  Ready for Switchover:  Yes
  Ready for Failover:    Yes (Primary Running)

  Flashback Database Status:
    dgtp:  On
    dgts:  Off

  Capacity Information:
    Database  Instances        Threads        
    dgtp      1                1              
    dgts      1                1              

  Managed by Clusterware:
    dgtp:  NO             
    dgts:  NO             
    Warning: Ensure primary database s StaticConnectIdentifier property
    is configured properly so that the primary database can be restarted
    by DGMGRL after switchover

  Temporary Tablespace File Information:
    dgtp TEMP Files:  3
    dgts TEMP Files:  3

  Data file Online Move in Progress:
    dgtp:  No
    dgts:  No

  Standby Apply-Related Information:
    Apply State:      Running
    Apply Lag:        0 seconds (computed 1 second ago)
    Apply Delay:      0 minutes

  Transport-Related Information:
    Transport On:      Yes
    Gap Status:        No Gap
    Transport Lag:     0 seconds (computed 1 second ago)
    Transport Status:  Success

  Log Files Cleared:
    dgtp Standby Redo Log Files:  Cleared
    dgts Online Redo Log Files:   Cleared
    dgts Standby Redo Log Files:  Available

  Current Log File Groups Configuration:
    Thread #  Online Redo Log Groups  Standby Redo Log Groups Status       
              (dgtp)                  (dgts)                               
    1         3                       2                       Insufficient SRLs

  Future Log File Groups Configuration:
    Thread #  Online Redo Log Groups  Standby Redo Log Groups Status       
              (dgts)                  (dgtp)                               
    1         3                       2                       Insufficient SRLs

  Current Configuration Log File Sizes:
    Thread #   Smallest Online Redo      Smallest Standby Redo    
               Log File Size             Log File Size            
               (dgtp)                    (dgts)                   
    1          200 MBytes                200 MBytes               

  Future Configuration Log File Sizes:
    Thread #   Smallest Online Redo      Smallest Standby Redo    
               Log File Size             Log File Size            
               (dgts)                    (dgtp)                   
    1          200 MBytes                200 MBytes               

  Apply-Related Property Settings:
    Property                        dgtp Value               dgts Value
    DelayMins                       0                        0
    ApplyParallel                   AUTO                     AUTO
    ApplyInstances                  0                        0

  Transport-Related Property Settings:
    Property                        dgtp Value               dgts Value
    LogXptMode                      ASYNC                    ASYNC
    Dependency                      <empty>                  <empty>
    DelayMins                       0                        0
    Binding                         optional                 optional
    MaxFailure                      0                        0
    MaxConnections                  1                        1
    ReopenSecs                      300                      300
    NetTimeout                      30                       30
    RedoCompression                 DISABLE                  DISABLE
    LogShipping                     ON                       ON

```

### 开启备库的闪回数据库
``` perl
SQL> select db_unique_name,open_mode,database_role,flashback_on from v$database;

DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE    FLASHBACK_ON
------------------------------ -------------------- ---------------- ------------------
DGTS                           READ ONLY WITH APPLY PHYSICAL STANDBY NO


SQL> alter database flashback on;
alter database flashback on
*
ERROR at line 1:
ORA-01153: an incompatible media recovery is active

SQL> recover managed standby database cancel;
Media recovery complete.

SQL> alter database flashback on;

Database altered.

SQL> recover managed standby database disconnect from session;
Media recovery complete.

DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE    FLASHBACK_ON
------------------------------ -------------------- ---------------- ------------------
DGTS                           READ ONLY WITH APPLY PHYSICAL STANDBY YES
```

