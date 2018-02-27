---
title: Oracle特殊恢复原理与实战_02 Control file丢失的恢复
date: 2018-02-26
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## 控制文件没有备份全部丢失恢复实战
### 准备测试环境
``` perl
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     110
Next log sequence to archive   112
Current log sequence           112

SQL> select open_mode from v$database;

OPEN_MODE
--------------------
READ WRITE

set line 150
set pagesize 9999
col name for a60
select * from v$controlfile;

STATUS  NAME                                                         IS_ BLOCK_SIZE FILE_SIZE_BLKS
------- ------------------------------------------------------------ --- ---------- --------------
        /u01/app/oracle/oradata/orcl/control01.ctl                   NO       16384            594
        /u01/app/oracle/fast_recovery_area/orcl/control02.ctl        NO       16384            594

# 为方便后续测试，先生成一个创建控制文件的脚本(也可以不生成，创建脚本可以oracle官方文档中找到)
alter database backup controlfile to trace as '/tmp/control_bak.sql';
```

<!-- more -->

### 摸拟控制文件全部丢失
``` perl
rm -rf /u01/app/oracle/oradata/orcl/control01.ctl
rm -rf /u01/app/oracle/fast_recovery_area/orcl/control02.ctl

# 丢失控制文件后，数据库仍能正常进行相关操作
SQL> alter system switch logfile;

System altered.

SQL> /

System altered.

SQL> alter system checkpoint;

System altered.
```

### 关闭和打开数据库报错
``` perl

SQL> shutdown immediate
ORA-00210: cannot open the specified control file
ORA-00202: control file: '/u01/app/oracle/oradata/orcl/control01.ctl'
ORA-27041: unable to open file
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3

# 强制关闭
SQL> shutdown abort
ORACLE instance shut down.

SQL> startup
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             905970328 bytes
Database Buffers         3456106496 bytes
Redo Buffers               11661312 bytes
ORA-00205: error in identifying control file, check alert log for more info

# alert日志中报错信息
Mon Feb 26 14:16:27 2018
ALTER DATABASE   MOUNT
ORA-00210: cannot open the specified control file
ORA-00202: control file: '/u01/app/oracle/fast_recovery_area/orcl/control02.ctl'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
ORA-00210: cannot open the specified control file
ORA-00202: control file: '/u01/app/oracle/oradata/orcl/control01.ctl'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
ORA-205 signalled during: ALTER DATABASE   MOUNT...
Mon Feb 26 14:16:27 2018
Checker run found 2 new persistent data failures
```

### 数据库无法mount情况下获得数据库基本信息
#### 获得DB_NAME
``` perl
SQL> select status from v$instance;

STATUS
------------------------
STARTED

SQL> show parameter db_name

NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
db_name                              string                 orcl
```

也可以使用[BBED解析DB-NAME](/2018/02/25/oracle/Oracle%E7%89%B9%E6%AE%8A%E6%81%A2%E5%A4%8D%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%98-%E8%AF%BE%E7%A8%8B%E5%AD%A6%E4%B9%A0/01_Oracle%E7%89%B9%E6%AE%8A%E6%81%A2%E5%A4%8D%E5%85%A5%E9%97%A8/#BBED解析DB-NAME)

#### 获得字符集
``` perl
SQL> select * from v$version;

BANNER
--------------------------------------------------------------------------------
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
PL/SQL Release 11.2.0.4.0 - Production
CORE    11.2.0.4.0      Production
TNS for Linux: Version 11.2.0.4.0 - Production
NLSRTL Version 11.2.0.4.0 - Production

# 在同版本可正常打开的数据库中执行
select distinct dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) block#
  from props$;

     FILE#     BLOCK#
---------- ----------
         1        801

# 使用dd工具dump
dd if=/u01/app/oracle/oradata/orcl/system01.dbf of=/tmp/props bs=8192 skip=801 count=1

strings /tmp/props
#------------------------------
......
NLS_CHARACTERSET
ZHS16GBK
Character set,
......
#------------------------------
```

#### 确认数据库是否归档
查看归档目录是否有文件，本测试是在归档环境

### 恢复控制文件
结合上面获得的信息，构建创建controlfile的脚本恢复控制文件，根据REDO LOG日志是否有丢失，有以下两种

#### NORESETLOGS
REDO LOG日志没有丢失时使用NORESETLOGS脚本创建控制文件
``` perl
STARTUP NOMOUNT
CREATE CONTROLFILE REUSE DATABASE "ORCL" NORESETLOGS  ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 100
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/u01/app/oracle/oradata/orcl/redo01.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 2 '/u01/app/oracle/oradata/orcl/redo02.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 3 '/u01/app/oracle/oradata/orcl/redo03.log'  SIZE 50M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '/u01/app/oracle/oradata/orcl/system01.dbf',
  '/u01/app/oracle/oradata/orcl/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl/users01.dbf',
  '/u01/app/oracle/oradata/orcl/dsi01.dbf'
CHARACTER SET ZHS16GBK
;
```

控制文件创建成功后，数据库打开到mount状态下
``` perl
# 可以看到，两个controlfile都已被创建成功
ll /u01/app/oracle/oradata/orcl/control01.ctl
-rw-r----- 1 oracle oinstall 10076160 Feb 26 15:58 /u01/app/oracle/oradata/orcl/control01.ctl

ll /u01/app/oracle/fast_recovery_area/orcl/control02.ctl
-rw-r----- 1 oracle oinstall 10076160 Feb 26 15:59 /u01/app/oracle/fast_recovery_area/orcl/control02.ctl

# 数据库打开到mount状态下
SQL> select status from v$instance;

STATUS
------------
MOUNTED
```

手工注册归档文件并打开数据库
``` perl
select SEQUENCE#,RESETLOGS_ID from v$archived_log;
no rows selected

alter database register physical logfile '/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_02_26/o1_mf_1_1_f97c8j9n_.arc';
alter database register physical logfile '/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_02_26/o1_mf_1_2_f97c8k76_.arc';
alter database register physical logfile '/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_02_26/o1_mf_1_3_f97c90sy_.arc';
alter database register physical logfile '/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_02_26/o1_mf_1_4_f97c92v1_.arc';

# 如果归档很多，使用rman catalog方式注册归档，命令如下
rman target /
catalog start with '/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_02_26/*.dbf';

SQL> select SEQUENCE#,RESETLOGS_ID from v$archived_log;

 SEQUENCE# RESETLOGS_ID
---------- ------------
         1    969114806
         2    969114806
         3    969114806
         4    969114806
# 恢复数据库
SQL> recover database;

# 所有redo日志归档（可选）
SQL> alter system archive log all;

# 打开数据库
SQL> alter database open;

# 创建临时表空间
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl/temp01.dbf'
     SIZE 500m  REUSE AUTOEXTEND ON NEXT 50m  MAXSIZE 32767M;
```

#### RESETLOGS
重新准备环境
``` perl
# 正常关闭数据库
shutdown immediate

# 删除所有控制文件和redo log文件
rm -rf /u01/app/oracle/oradata/orcl/control01.ctl
rm -rf /u01/app/oracle/fast_recovery_area/orcl/control02.ctl
rm -rf /u01/app/oracle/oradata/orcl/redo*.log

ll /u01/app/oracle/oradata/orcl/
total 2237496
-rw-r----- 1 oracle oinstall 524296192 Feb 26 17:07 dsi01.dbf
-rw-r----- 1 oracle oinstall 786440192 Feb 26 17:07 sysaux01.dbf
-rw-r----- 1 oracle oinstall 796925952 Feb 26 17:07 system01.dbf
-rw-r----- 1 oracle oinstall 524296192 Feb 26 16:34 temp01.dbf
-rw-r----- 1 oracle oinstall 173023232 Feb 26 17:07 undotbs01.dbf
-rw-r----- 1 oracle oinstall   5251072 Feb 26 17:07 users01.dbf
```

REDO LOG日志有丢失时需要重建时用RESETLOGS脚本创建控制文件
``` perl
STARTUP NOMOUNT
CREATE CONTROLFILE REUSE DATABASE "ORCL" RESETLOGS  ARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 100
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/u01/app/oracle/oradata/orcl/redo01.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 2 '/u01/app/oracle/oradata/orcl/redo02.log'  SIZE 50M BLOCKSIZE 512,
  GROUP 3 '/u01/app/oracle/oradata/orcl/redo03.log'  SIZE 50M BLOCKSIZE 512
-- STANDBY LOGFILE
DATAFILE
  '/u01/app/oracle/oradata/orcl/system01.dbf',
  '/u01/app/oracle/oradata/orcl/sysaux01.dbf',
  '/u01/app/oracle/oradata/orcl/undotbs01.dbf',
  '/u01/app/oracle/oradata/orcl/users01.dbf',
  '/u01/app/oracle/oradata/orcl/dsi01.dbf'
CHARACTER SET ZHS16GBK
;
# 控制文件创建成功后，数据库打开到mount状态下

# 直接recover database报错
SQL> recover database;
ORA-00283: recovery session canceled due to errors
ORA-01610: recovery using the BACKUP CONTROLFILE option must be done

SQL> recover database using backup controlfile until cancel;
#---------------------------------------------------------------------------------------------
ORA-00279: change 2985399 generated at 02/26/2018 17:07:08 needed for thread 1
ORA-00289: suggestion :
/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_02_26/o1_mf_1_10_%u_.arc
ORA-00280: change 2985399 for thread 1 is in sequence #10


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
CANCEL
Media recovery cancelled.
#---------------------------------------------------------------------------------------------

# resetlogs打开数据库
SQL> alter database open resetlogs;
Database altered.

# 创建临时表空间
ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/app/oracle/oradata/orcl/temp01.dbf'
     SIZE 500m  REUSE AUTOEXTEND ON NEXT 50m  MAXSIZE 32767M;
```

## RESETLOGS解析
![](http://p2c0rtsgc.bkt.clouddn.com/0226_oracle_dsi_01.png)

## 相关命令
``` perl
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/oradata/rmanbackup/ctrl_bak_%F';
backup database format '/oradata/rmanbackup/full_bak_%d_%T_%s_%U.bak';
restore controlfile from autobackup;

restore controlfile to '/u01/app/oracle/oradata/orcl/control01.ctl' from '/oradata/rmanbackup/ctrl_bak_c-1493347395-20180226-02';

list incarnation;

SQL> select * from v$database_incarnation;

INCARNATION# RESETLOGS_CHANGE# RESETLOGS_TIME      PRIOR_RESETLOGS_CHANGE# PRIOR_RESETLOGS_TIM STATUS  RESETLOGS_ID PRIOR_INCARNATION# FLASHBACK_DATABASE_ALLOWED
------------ ----------------- ------------------- ----------------------- ------------------- ------- ------------ ------------------ --------------------------
           1           2962246 2018-02-26 14:33:26                  925702 2018-01-16 12:57:41 PARENT     969114806                  0 NO
           2           2985400 2018-02-26 17:24:23                 2962246 2018-02-26 14:33:26 CURRENT    969125063                  1 NO
```

## 课后作业
### 哪些场景下需要用alter database open resetlogs打开库？
1. 在不完全恢复（介质恢复）时
2. 使用备份的控制文件进行数据库恢复时
3. 丢失redo log，并使用RESETLOGS方式手工创建控制文件进行数据库恢复时

### 在删除所有controlfile和redolog日志的情况下shutdown abort异常关库，能用resetlogs打开库吗？为什么？
不能。 
实例崩溃(shutdown abort)后启动数据库时需要恢复所需要的日志文件。丢失/损坏redolog日志的情况下，只能尝试使用隐含参数_allow_resetlogs_corruption，强制进行不完全恢复。

### 用dd命令损坏其中一个控制文件的文件头（1号块）,然后尝试用startup mount;命令挂载数据库报错,请用最快的恢复方式恢复控制文件，给出详细操作步骤？
``` perl
ll
#---------------------------------------------------------------
total 2400952
-rw-r----- 1 oracle oinstall  10076160 Feb 26 18:13 control01.ctl
-rw-r----- 1 oracle oinstall 524296192 Feb 26 18:06 dsi01.dbf
-rw-r----- 1 oracle oinstall  52429312 Feb 26 17:32 redo01.log
-rw-r----- 1 oracle oinstall  52429312 Feb 26 17:32 redo02.log
-rw-r----- 1 oracle oinstall  52429312 Feb 26 18:12 redo03.log
-rw-r----- 1 oracle oinstall 786440192 Feb 26 18:06 sysaux01.dbf
-rw-r----- 1 oracle oinstall 796925952 Feb 26 18:06 system01.dbf
-rw-r----- 1 oracle oinstall 524296192 Feb 26 17:25 temp01.dbf
-rw-r----- 1 oracle oinstall 173023232 Feb 26 18:06 undotbs01.dbf
-rw-r----- 1 oracle oinstall   5251072 Feb 26 18:06 users01.dbf
#---------------------------------------------------------------

echo "database_name:orcl" > db

dd if=db of=control01.ctl bs=16834 seek=1 count=1 conv=notrunc
# seek=BLOCKS     skip BLOCKS obs-sized blocks at start of output
#                 跳过一段以后才输出，是在备份时对of后面的部分也就是目标文件跳过多少块再开始写。
# conv=CONVS      convert the file as per the comma separated symbol list

sqlplus / as sysdba

SQL> shutdown immediate
ORA-00227: corrupt block detected in control file: (block 1, # blocks 1)
ORA-00202: control file: '/u01/app/oracle/oradata/orcl/control01.ctl'

SQL> shutdown abort;
ORACLE instance shut down.

SQL> startup mount;
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             905970328 bytes
Database Buffers         3456106496 bytes
Redo Buffers               11661312 bytes
ORA-00227: corrupt block detected in control file: (block 0, # blocks )


$ bbed

BBED: Release 2.0.0.0.0 - Limited Production on Mon Feb 26 18:22:19 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

************* !!! For Oracle Internal Use only !!! ***************

BBED> set filename '/u01/app/oracle/oradata/orcl/control01.ctl'
        FILENAME        /u01/app/oracle/oradata/orcl/control01.ctl

BBED> set block 1
        BLOCK#          1

BBED> dump
 File: /u01/app/oracle/oradata/orcl/control01.ctl (0)
 Block: 1                Offsets:    0 to  511           Dba:0x00000000
------------------------------------------------------------------------
 15c20000 01000000 00000000 00000104 e9bd0000 00000000 0004200b 43ac0259 
 4f52434c 00000000 6d560000 66020000 00400000 00000100 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 8d4f3859 0eaac339 c5942d00 00000000 c0b6c339 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 08000000 08000000 08000000 00000000 00000000 00000000 
 01000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00006461 74616261 73655f6e 616d653a 6f72636c 0a000000 00000000 00000000
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>
```

最快的恢复方式恢复控制文件，就是用第二个控制文件来恢复第一个 
``` perl
cp /u01/app/oracle/fast_recovery_area/orcl/control02.ctl /u01/app/oracle/oradata/orcl/control01.ctl

SQL> alter database mount;

Database altered.

SQL> alter database open;

Database altered.

```