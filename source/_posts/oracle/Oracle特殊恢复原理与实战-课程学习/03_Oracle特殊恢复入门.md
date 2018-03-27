---
title: Oracle特殊恢复原理与实战_03 Control file深入内部解析
date: 2018-03-10
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## Control file: dump
``` perl
alter session set events 'immediate trace name controlf level <n>';
如：
alter session set events 'immediate trace name controlf level 1';

# 通过以下命令查看dump trace文件的位置
select * from v$diag_info where NAME='Default Trace File'; 
```

>level <n>:
level 1: Generic File Header
Level 2: Level 1 + database information + database entry + check point
         progress records + Extended database entry
level 3 or Higher< 9: level 2 + reuse record section
level 10: Memory dump of all the control file logical blocks

<!-- more -->

## Control file: file header
``` perl
alter session set events 'immediate trace name controlf level 1';
select * from v$diag_info where NAME='Default Trace File'; 

dump trace:
DUMP OF CONTROL FILES, Seq # 28874 = 0x70ca
 V10 STYLE FILE HEADER:
        Compatibility Vsn = 186647552=0xb200400     # 数据库版本 11.2.0.4
        Db ID=1493347395=0x5902ac43, Db Name='ORCL'
        Activation ID=0=0x0
        Control Seq=28874=0x70ca, File size=614=0x266  # 控制文件大小
        File Number=0, Blksiz=16384, File Type=1 CONTROL

# Seq
SQL> select CONTROLFILE_SEQUENCE# from v$database;

CONTROLFILE_SEQUENCE#
---------------------
                28875
```


## Control file: DATABASE ENTRY
``` perl
alter session set events 'immediate trace name controlf level 2';
select * from v$diag_info where NAME='Default Trace File'; 

dump trace:
***************************************************************************
DATABASE ENTRY
***************************************************************************
 (size = 316, compat size = 316, section max = 1, section in-use = 1,
  last-recid= 0, old-recno = 0, last-recno = 0)
 (extent = 1, blkno = 1, numrecs = 1)
 02/26/2018 17:12:45
 DB Name "ORCL"
 Database flags = 0x00404001 0x00001200
 Controlfile Creation Timestamp  02/26/2018 17:12:46
 Incmplt recovery scn: 0x0000.00000000
 Resetlogs scn: 0x0000.002d8db8 Resetlogs Timestamp  02/26/2018 17:24:23
 Prior resetlogs scn: 0x0000.002d3346 Prior resetlogs Timestamp  02/26/2018 14:33:26
 Redo Version: compatible=0xb200400
 #Data files = 7, #Online files = 7
 Database checkpoint: Thread=1 scn: 0x0000.00370ab3
 Threads: #Enabled=1, #Open=1, Head=1, Tail=1
 enabled  threads:  01000000 00000000 00000000 00000000 00000000 00000000
...

#Database flags : 当前数据库的状态
    KCCDIMRE 0x00000001 whether media recovery enabled(that is: ARCHIVELOG mode)  #
    KCCDICKD 0x00000002 if dictionary must be checked with control file
    KCCDIRLR 0x00000004 DB OPEN RESETLOGS required
    KCCDIJNK 0x00000008 (junk value from beta)
    KCCDIMRC 0x00000010 was/is last mounted READ_COMPATIBLE
    KCCDICNV 0x00000020 controlfile was just created by convert from v6
    KCCDIIRA 0x00000040 Incomplete Recovery Allowed when resetting logs
    KCCDICCF 0x00000100 Controlfile was created with CREATE CONTROLFILE
    KCCDIINV 0x00000200 Invalid control file or database; still creating
    KCCDISBD 0x00000400 StandBy Database; control file for hot standby
    KCCDIORL 0x00000800 Opened ResetLogs; set until dictionary check
    KCCDICFC 0x00001000 valid ControlFile Checkpoint in backup cf
    KCCDISSN 0x00002000 SnapShot controlfile fileName pointer valid
    KCCDIUCD 0x00004000 lazy file header Update Checkpoint cycle Done  #
    KCCDICLO 0x00008000 clone database
    KCCDINDL 0x00010000 standby database No Data Loss
    KCCDISPK 0x00020000 Supplemental log primary keys
    KCCDISUI 0x00040000 Supplemental log unique indexes
    KCCDISFK 0x00080000 Supplemental log foreign keys
    KCCDIGDA 0x00100000 Database guard all
    KCCDIGDS 0x00200000 Database guard standby data
    KCCDIIMR 0x00400000 Group Membership Recovery is supported   #
    KCCDIEAR 0x00800000 End-of-redo Archival Received
    KCCDISTR 0x01000000 Standby Terminal Recovery
    KCCDILSB 0x02000000 Logical StandBy database

Database flags = 0x00404001 = KCCDIIMR + KCCDIUCD + KCCDIMRE

# 一般控制文件的检查点要比数据库的检查点大一些
SQL> select CONTROLFILE_CHANGE#, CHECKPOINT_CHANGE# from v$database;

CONTROLFILE_CHANGE# CHECKPOINT_CHANGE#
------------------- ------------------
            3616799            3607219
```

## Control file: PROGRESS RECORDS
``` perl
***************************************************************************
CHECKPOINT PROGRESS RECORDS
***************************************************************************
 (size = 8180, compat size = 8180, section max = 11, section in-use = 0,
  last-recid= 0, old-recno = 0, last-recno = 0)
 (extent = 1, blkno = 2, numrecs = 11)
THREAD #1 - status:0x2 flags:0x0 dirty:30
low cache rba:(0x36.42a6.0) on disk rba:(0x36.42d3.0)  # 实例恢复时，启动数据库从low cache rba，到on disk rba结束
on disk scn: 0x0000.00372e55 03/20/2018 15:46:19
resetlogs scn: 0x0000.002d8db8 02/26/2018 17:24:23
heartbeat: 971215409 mount id: 1498920122
```

## Control file: EXTENDED DATABASE ENTRY
``` perl
***************************************************************************
EXTENDED DATABASE ENTRY
***************************************************************************
 (size = 900, compat size = 900, section max = 1, section in-use = 1,
  last-recid= 0, old-recno = 0, last-recno = 0)
 (extent = 1, blkno = 150, numrecs = 1)
Control AutoBackup date(dd/mm/yyyy)=26/ 2/2018
Next AutoBackup sequence= 0
Database recovery target inc#:2, Last open inc#:2
flg:0x0, flag:0x0
Change tracking state=0, file index=0, checkpoint count=0scn: 0x0000.00000000
Flashback log count=0, block count=0
Desired flashback log size=0 blocks
Oldest guarantee restore point=0
Highest thread enable/disable scn: 0x0000.00000000
Number of Open thread with finite next SCN in last log: 0
Number of half-enabled redo threads: 0
Sum of absolute file numbers for files currently being moved online: 0
```

## 课后作业
1.数据文件5号文件头offset=1的a2代表什么意思？如何把5号文件的文件头offset=1的值a2变为c2(写出详细操作步骤，切不能用BBED修改)
``` perl
SQL> select * from v$dbfile where file#=5;

     FILE# NAME
---------- --------------------------------------------------
         5 /u01/app/oracle/oradata/orcl/dsi01.dbf

bbed

BBED> info
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /u01/app/oracle/oradata/orcl/system01.dbf                        97281
     2  /u01/app/oracle/oradata/orcl/sysaux01.dbf                        96001
     3  /u01/app/oracle/oradata/orcl/undotbs01.dbf                       21121
     4  /u01/app/oracle/oradata/orcl/users01.dbf                           641
     5  /u01/app/oracle/oradata/orcl/dsi01.dbf                           64001
     6  /u01/app/oracle/oradata/orcl/skip_arch01.dbf                      6400

BBED> set file 5 block 1
        FILE#           5
        BLOCK#          1

BBED> dump count 32
 File: /u01/app/oracle/oradata/orcl/dsi01.dbf (5)
 Block: 1                Offsets:    0 to   31           Dba:0x01400001
------------------------------------------------------------------------
 0ba20000 01004001 00000000 00000104 63cf0000 00000000 0004200b 43ac0259 


BBED> p kcvfhbfh
struct kcvfhbfh, 20 bytes                   @0       
   ub1 type_kcbh                            @0        0x0b
   ub1 frmt_kcbh                            @1        0xa2   # blocksize 8k
   ub1 spare1_kcbh                          @2        0x00
   ub1 spare2_kcbh                          @3        0x00
   ub4 rdba_kcbh                            @4        0x01400001
   ub4 bas_kcbh                             @8        0x00000000
   ub2 wrp_kcbh                             @12       0x0000
   ub1 seq_kcbh                             @14       0x01
   ub1 flg_kcbh                             @15       0x04 (KCBHFCKV)
   ub2 chkval_kcbh                          @16       0xcf63
   ub2 spare3_kcbh                          @18       0x0000

frmt_kcbh：块格式。不同版本，不同的文件，这个值也不一样
Oracle 6,7 : 0x01
8i~9i:all: 0x02
10g~12c:2k : 0x62
4k : 0x82
8k : 0xa2
16k : 0xc2  
Redo 6~12c : 0x22
```

正常是无法修改已创建datafile块的大小，只能创建一个新的表空间，指定块大小为16k，最后把8k表空间的内容MOVE到16k的表空间中。
``` perl
sqlplus / as sysdba

SQL> show parameter cache_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
client_result_cache_size             big integer 0
db_16k_cache_size                    big integer 0
db_2k_cache_size                     big integer 0
db_32k_cache_size                    big integer 0
db_4k_cache_size                     big integer 0
db_8k_cache_size                     big integer 0
db_cache_size                        big integer 0
db_flash_cache_size                  big integer 0
db_keep_cache_size                   big integer 0
db_recycle_cache_size                big integer 0
SQL> alter system set db_16k_cache_size = 100M;

System altered.

SQL> show parameter cache_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
client_result_cache_size             big integer 0
db_16k_cache_size                    big integer 112M
db_2k_cache_size                     big integer 0
db_32k_cache_size                    big integer 0
db_4k_cache_size                     big integer 0
db_8k_cache_size                     big integer 0
db_cache_size                        big integer 0
db_flash_cache_size                  big integer 0
db_keep_cache_size                   big integer 0
db_recycle_cache_size                big integer 0
SQL> create tablespace ts_16 blocksize 16k datafile '/u01/app/oracle/oradata/orcl/ts16_01.dbf' size 50m;

Tablespace created.

SQL> select file#||chr(9)||name||chr(9)||bytes from v$datafile;

FILE#||CHR(9)||NAME||CHR(9)||BYTES
--------------------------------------------------------------------------------
1       /u01/app/oracle/oradata/orcl/system01.dbf       796917760
2       /u01/app/oracle/oradata/orcl/sysaux01.dbf       786432000
3       /u01/app/oracle/oradata/orcl/undotbs01.dbf      209715200
4       /u01/app/oracle/oradata/orcl/users01.dbf        5242880
5       /u01/app/oracle/oradata/orcl/dsi01.dbf  524288000
6       /u01/app/oracle/oradata/orcl/skip_arch01.dbf    52428800
7       /u01/app/oracle/oradata/orcl/ts16_01.dbf        52428800

# 新建的7号文件文件头offset=1的位置就是c2了
echo "7 /u01/app/oracle/oradata/orcl/ts16_01.dbf 52428800" >> filelist.txt

bbed

BBED> info
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /u01/app/oracle/oradata/orcl/system01.dbf                        97281
     2  /u01/app/oracle/oradata/orcl/sysaux01.dbf                        96001
     3  /u01/app/oracle/oradata/orcl/undotbs01.dbf                       21121
     4  /u01/app/oracle/oradata/orcl/users01.dbf                           641
     5  /u01/app/oracle/oradata/orcl/dsi01.dbf                           64001
     6  /u01/app/oracle/oradata/orcl/skip_arch01.dbf                      6400
     7  /u01/app/oracle/oradata/orcl/ts16_01.dbf                          6400

BBED> set file 7 block 1
        FILE#           7
        BLOCK#          1

BBED> dump count 32
 File: /u01/app/oracle/oradata/orcl/ts16_01.dbf (7)
 Block: 1                Offsets:    0 to   31           Dba:0x01c00001
------------------------------------------------------------------------
 0bc20000 0100c001 00000000 00000104 7bd70000 00000000 0004200b 43ac0259 
```

2.Oracle实例恢复从low cache rba开始恢复，至少恢复到on disk rba，请用实验来证明？（给出详细操作步骤）
``` perl
SQL> shutdown abort;
ORACLE instance shut down.
SQL> startup mount
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             889193112 bytes
Database Buffers         3472883712 bytes
Redo Buffers               11661312 bytes
Database mounted.
SQL> alter session set events 'immediate trace name controlf level 2';

Session altered.

SQL> select * from v$diag_info where NAME='Default Trace File'; 

   INST_ID NAME
---------- ----------------------------------------------------------------
VALUE
--------------------------------------------------------------------------------
         1 Default Trace File
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_8852.trc


查看dump trace CHECKPOINT PROGRESS RECORDS部分信息：
***************************************************************************
CHECKPOINT PROGRESS RECORDS
***************************************************************************
 (size = 8180, compat size = 8180, section max = 11, section in-use = 0,
  last-recid= 0, old-recno = 0, last-recno = 0)
 (extent = 1, blkno = 2, numrecs = 11)
THREAD #1 - status:0x2 flags:0x0 dirty:21
low cache rba:(0x36.4da4.0) on disk rba:(0x36.4dbd.0)    # 从(0x36.4da4.0) 开始恢复到(0x36.4dbd.0)结束
on disk scn: 0x0000.003731ec 03/20/2018 16:15:06
resetlogs scn: 0x0000.002d8db8 02/26/2018 17:24:23
heartbeat: 971294256 mount id: 1499044775
```

解读：
``` perl
low cache rba:(0x36.4da4.0)
SQL> select to_number(36,'xxxxxx') from dual;

TO_NUMBER(36,'XXXXXX')
----------------------
                    54

SQL> select to_number('4da4','xxxxxx') from dual;

TO_NUMBER('4DA4','XXXXXX')
--------------------------
                     19876


on disk rba:(0x36.4dbd.0)
SQL> select to_number(36,'xxxxxx') from dual;

TO_NUMBER(36,'XXXXXX')
----------------------
                    54

SQL> select to_number('4dbd','xxxxxx') from dual;

TO_NUMBER('4DA4','XXXXXX')
--------------------------
                     19901

实例恢复将从第54号日志第19876块偏移量为0开始，到第54号日志第19901块偏移量为0结束
```

验证：
``` perl
# 启动数据库
alter database open;

# 查看alert日志
Beginning crash recovery of 1 threads
 parallel recovery started with 3 processes
Started redo scan
Completed redo scan
 read 12 KB redo, 21 data blocks need recovery
Started redo application at
 Thread 1: logseq 54, block 19876
Recovery of Online Redo Log: Thread 1 Group 3 Seq 54 Reading mem 0
  Mem# 0: /u01/app/oracle/oradata/orcl/redo03.log
Completed redo application of 0.01MB
Completed crash recovery at
 Thread 1: logseq 54, block 19901, scn 3637260
 21 data blocks read, 21 data blocks written, 12 redo k-bytes read

从alert日志中可以验证，实例从low cache rba开始恢复，至少恢复到on disk rba
```


3.误操作rm -rf control0*.ctl删除全部控制文件，通过文件描述符对控制文件进行恢复。（给出详细操作步骤）
``` perl
SQL> show parameter control_files

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string      /u01/app/oracle/oradata/orcl/c
                                                 ontrol01.ctl, /u01/app/oracle/
                                                 fast_recovery_area/orcl/contro
                                                 l02.ctl

exit

# 先复制一份，以防万一
cp /u01/app/oracle/oradata/orcl/control01.ctl /u01/app/oracle/oradata/orcl/control01.ctl.bak
cp /u01/app/oracle/fast_recovery_area/orcl/control02.ctl /u01/app/oracle/fast_recovery_area/orcl/control02.ctl.bak

# 删除控制文件
rm -rf /u01/app/oracle/oradata/orcl/control01.ctl /u01/app/oracle/fast_recovery_area/orcl/control02.ctl

# 查看检查点进程
ps -ef | grep ckpt | grep -v grep
oracle    8834     1  0 16:15 ?        00:00:00 ora_ckpt_orcl


cd /proc/8834/fd
total 0
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 0 -> /dev/null
l-wx------ 1 oracle oinstall 64 Mar 20 16:59 1 -> /dev/null
lrwx------ 1 oracle oinstall 64 Mar 20 16:59 10 -> /u01/app/oracle/11.2.0.4/db_1/dbs/lkORCL
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 11 -> /u01/app/oracle/11.2.0.4/db_1/rdbms/mesg/oraus.msb
l-wx------ 1 oracle oinstall 64 Mar 20 16:59 2 -> /dev/null
lrwx------ 1 oracle oinstall 64 Mar 20 16:59 256 -> /u01/app/oracle/oradata/orcl/control01.ctl (deleted)    ##
lrwx------ 1 oracle oinstall 64 Mar 20 16:59 257 -> /u01/app/oracle/fast_recovery_area/orcl/control02.ctl (deleted) ##
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 3 -> /dev/null
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 4 -> /dev/null
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 5 -> /dev/null
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 6 -> /u01/app/oracle/11.2.0.4/db_1/rdbms/mesg/oraus.msb
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 7 -> /proc/8834/fd
lr-x------ 1 oracle oinstall 64 Mar 20 16:59 8 -> /dev/zero
lrwx------ 1 oracle oinstall 64 Mar 20 16:59 9 -> /u01/app/oracle/11.2.0.4/db_1/dbs/hc_orcl.dat

cp 256 /u01/app/oracle/oradata/orcl/control01.ctl
cp 257 /u01/app/oracle/fast_recovery_area/orcl/control02.ctl

# 关库的时候报错，但可以正常启动
[oracle@orcl11g ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Tue Mar 20 17:00:38 2018

Copyright (c) 1982, 2013, Oracle.  All rights reserved.


Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options

SQL> shutdown immediate
Database closed.
ORA-03113: end-of-file on communication channel
Process ID: 9043
Session ID: 191 Serial number: 13


SQL> exit
Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning, OLAP, Data Mining and Real Application Testing options
[oracle@orcl11g ~]$ sqlplus / as sysdba

SQL*Plus: Release 11.2.0.4.0 Production on Tue Mar 20 17:00:56 2018

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to an idle instance.

SQL> startup
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             889193112 bytes
Database Buffers         3472883712 bytes
Redo Buffers               11661312 bytes
Database mounted.
Database opened.
```


## 参考文档
* [oracle控制文件转储说明](http://m.blog.itpub.net/28539951/viewspace-1994654/)