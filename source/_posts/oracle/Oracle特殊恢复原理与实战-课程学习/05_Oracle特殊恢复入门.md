---
title: Oracle特殊恢复原理与实战_05 使用BBED跳过归档的恢复
date: 2018-03-26
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## 使用BBED跳过归档的恢复
### 模拟场景：在做恢复时发现丢失部分归档
开启归档
``` perl
SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     44
Next log sequence to archive   46
Current log sequence           46

SQL> show parameter recovery

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string      /u01/app/oracle/fast_recovery_
                                                 area
db_recovery_file_dest_size           big integer 10G
recovery_parallelism                 integer     0
```

<!-- more -->

创建表空间、用户、表并插入数据
``` perl
create tablespace skip_arch datafile '/u01/app/oracle/oradata/orcl/skip_arch01.dbf' size 50m;
create user lyj identified by lyj default tablespace skip_arch;
grant dba to lyj;
conn lyj/lyj
create table t1 (id int,name varchar2(10));
insert into t1 values (1,'AAAAAA');
commit;
```

对6号文件做备份
``` perl
conn / as sysdba
col FILE_NAME for a60
select FILE_ID,FILE_NAME from dba_data_files order by 1;

   FILE_ID FILE_NAME
---------- ------------------------------------------------------------
         1 /u01/app/oracle/oradata/orcl/system01.dbf
         2 /u01/app/oracle/oradata/orcl/sysaux01.dbf
         3 /u01/app/oracle/oradata/orcl/undotbs01.dbf
         4 /u01/app/oracle/oradata/orcl/users01.dbf
         5 /u01/app/oracle/oradata/orcl/dsi01.dbf
         6 /u01/app/oracle/oradata/orcl/skip_arch01.dbf

rman target /
backup datafile 6 format '/oradata/rmanbackup/datafile5_%U';
```

切换归档日志
``` perl
select sequence#,status from v$archived_log order by 1 desc;
 SEQUENCE# S
---------- -
        45 A   
        44 A
        43 A
        42 A
        41 A
        40 A

# 多次切换
alter system switch logfile;
/

select sequence#,status from v$archived_log order by 1 desc;

 SEQUENCE# S
---------- -
        51 A
        50 A
        49 A
        48 A
        47 A
        46 A

# 查看归档文件
cd /u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_03_19
ll
total 56544
-rw-r----- 1 oracle oinstall 36331008 Mar 19 00:01 o1_mf_1_45_fbx39wby_.arc
-rw-r----- 1 oracle oinstall 21547008 Mar 19 15:05 o1_mf_1_46_fbyrbkbz_.arc
-rw-r----- 1 oracle oinstall     2048 Mar 19 15:05 o1_mf_1_47_fbyrbmnn_.arc
-rw-r----- 1 oracle oinstall     1536 Mar 19 15:05 o1_mf_1_48_fbyrbqww_.arc  
-rw-r----- 1 oracle oinstall     1024 Mar 19 15:06 o1_mf_1_49_fbyrbs1o_.arc
-rw-r----- 1 oracle oinstall     1024 Mar 19 15:06 o1_mf_1_50_fbyrbt9w_.arc
-rw-r----- 1 oracle oinstall     1024 Mar 19 15:06 o1_mf_1_51_fbyrbv64_.arc

# 删除48、49号归档
rm o1_mf_1_48_fbyrbqww_.arc o1_mf_1_49_fbyrbs1o_.arc
ll
total 56536
-rw-r----- 1 oracle oinstall 36331008 Mar 19 00:01 o1_mf_1_45_fbx39wby_.arc
-rw-r----- 1 oracle oinstall 21547008 Mar 19 15:05 o1_mf_1_46_fbyrbkbz_.arc
-rw-r----- 1 oracle oinstall     2048 Mar 19 15:05 o1_mf_1_47_fbyrbmnn_.arc
-rw-r----- 1 oracle oinstall     1024 Mar 19 15:06 o1_mf_1_50_fbyrbt9w_.arc
-rw-r----- 1 oracle oinstall     1024 Mar 19 15:06 o1_mf_1_51_fbyrbv64_.arc
```

离线6号文件
``` perl
select FILE#, CREATION_CHANGE#,CHECKPOINT_CHANGE#,UNRECOVERABLE_CHANGE#,LAST_CHANGE#,OFFLINE_CHANGE#,STATUS 
  from v$datafile order by 1;

     FILE# CREATION_CHANGE# CHECKPOINT_CHANGE# UNRECOVERABLE_CHANGE# LAST_CHANGE# OFFLINE_CHANGE# STATUS
---------- ---------------- ------------------ --------------------- ------------ --------------- -------
         1                7            3571679                     0                      2985399 SYSTEM
         2             1834            3571679                     0                      2985399 ONLINE
         3           923328            3571679                     0                      2985399 ONLINE
         4            16143            3571679                     0                      2985399 ONLINE
         5          2959979            3571679                     0                      2985399 ONLINE
         6          3570623            3571679                     0                            0 ONLINE

SQL> alter database datafile 6 offline;

Database altered.

select FILE#, CREATION_CHANGE#,CHECKPOINT_CHANGE#,UNRECOVERABLE_CHANGE#,LAST_CHANGE#,OFFLINE_CHANGE#,STATUS 
  from v$datafile order by 1;

     FILE# CREATION_CHANGE# CHECKPOINT_CHANGE# UNRECOVERABLE_CHANGE# LAST_CHANGE# OFFLINE_CHANGE# STATUS
---------- ---------------- ------------------ --------------------- ------------ --------------- -------
         1                7            3571679                     0                      2985399 SYSTEM
         2             1834            3571679                     0                      2985399 ONLINE
         3           923328            3571679                     0                      2985399 ONLINE
         4            16143            3571679                     0                      2985399 ONLINE
         5          2959979            3571679                     0                      2985399 ONLINE
         6          3570623            3571679                     0      3572789               0 RECOVER
```

>archivelog模式下，当数据文件offline时，其对应的数据文件头stop scn会更新，同时controlfile中该datafile的stop scn信息也会更新，此时也会更新offline scn，并且offline scn等于stop scn。

对6号文件进行还原
``` perl
rman target /
RMAN> restore datafile 6;

Starting restore at 2018-03-19 15:49:54
using channel ORA_DISK_1

channel ORA_DISK_1: starting datafile backup set restore
channel ORA_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_DISK_1: restoring datafile 00006 to /u01/app/oracle/oradata/orcl/skip_arch01.dbf
channel ORA_DISK_1: reading from backup piece /oradata/rmanbackup/datafile5_05su6blm_1_1
channel ORA_DISK_1: piece handle=/oradata/rmanbackup/datafile5_05su6blm_1_1 tag=TAG20180319T145901
channel ORA_DISK_1: restored backup piece 1
channel ORA_DISK_1: restore complete, elapsed time: 00:00:03
Finished restore at 2018-03-19 15:49:57
```

6号数据文件无法被online
``` perl
SQL> alter database datafile 6 online;
alter database datafile 6 online
*
ERROR at line 1:
ORA-01113: file 6 needs media recovery
ORA-01110: data file 6: '/u01/app/oracle/oradata/orcl/skip_arch01.dbf'
```

对6号文件进行恢复时因归档丢失报错
``` perl
RMAN> recover datafile 6;

Starting recover at 2018-03-19 15:53:03
using channel ORA_DISK_1

starting media recovery

archived log for thread 1 with sequence 46 is already on disk as file /u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_03_19/o1_mf_1_46_fbyrbkbz_.arc
archived log for thread 1 with sequence 47 is already on disk as file /u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_03_19/o1_mf_1_47_fbyrbmnn_.arc
archived log for thread 1 with sequence 50 is already on disk as file /u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_03_19/o1_mf_1_50_fbyrbt9w_.arc
archived log for thread 1 with sequence 51 is already on disk as file /u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_03_19/o1_mf_1_51_fbyrbv64_.arc
RMAN-00571: ===========================================================
RMAN-00569: =============== ERROR MESSAGE STACK FOLLOWS ===============
RMAN-00571: ===========================================================
RMAN-03002: failure of recover command at 03/19/2018 15:53:03
RMAN-06053: unable to perform media recovery because of missing log
RMAN-06025: no backup of archived log for thread 1 with sequence 49 and starting SCN of 3571670 found to restore
RMAN-06025: no backup of archived log for thread 1 with sequence 48 and starting SCN of 3571667 found to restore
```

### 跳归档恢复
``` perl
# 归档序列48、49已丢失，从50开始恢复
select to_char(SEQUENCE#,'xxxxxxxxxxx') seq，
       to_char(FIRST_CHANGE#,'xxxxxxxxxxx') scn
  from v$archived_log where SEQUENCE#=50;

SEQ          SCN
------------ ------------
          32       367fd9     # d97f3600

# 把上面的值更新到6号文件头
BBED> info
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /u01/app/oracle/oradata/orcl/system01.dbf                        97281
     2  /u01/app/oracle/oradata/orcl/sysaux01.dbf                        96001
     3  /u01/app/oracle/oradata/orcl/undotbs01.dbf                       21121
     4  /u01/app/oracle/oradata/orcl/users01.dbf                           641
     5  /u01/app/oracle/oradata/orcl/dsi01.dbf                           64001
     6  /u01/app/oracle/oradata/orcl/skip_arch01.dbf                      6400

BBED> set file 6 block 1
        FILE#           6
        BLOCK#          1

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1                                     Dba:0x01800001
------------------------------------------------------------
 Data File Header

 struct kcvfh, 860 bytes                    @0       
    struct kcvfhbfh, 20 bytes               @0       
    struct kcvfhhdr, 76 bytes               @20      
    ub4 kcvfhrdb                            @96      
    struct kcvfhcrs, 8 bytes                @100     
    ub4 kcvfhcrt                            @108     
    ub4 kcvfhrlc                            @112     
    struct kcvfhrls, 8 bytes                @116     
    ub4 kcvfhbti                            @124     
    struct kcvfhbsc, 8 bytes                @128     
    ub2 kcvfhbth                            @136     
    ub2 kcvfhsta                            @138     
    struct kcvfhckp, 36 bytes               @484      ##


BBED> p kcvfhckp
struct kcvfhckp, 36 bytes                   @484     
   struct kcvcpscn, 8 bytes                 @484      ## SCN
      ub4 kscnbas                           @484      0x00367eb2
      ub2 kscnwrp                           @488      0x0000
   ub4 kcvcptim                             @492      0x39e32eb6
   ub2 kcvcpthr                             @496      0x0001
   union u, 12 bytes                        @500     
      struct kcvcprba, 12 bytes             @500     
         ub4 kcrbaseq                       @500      0x0000002e   ## SEQ
         ub4 kcrbabno                       @504      0x00009623
         ub2 kcrbabof                       @508      0x0010
   ub1 kcvcpetb[0]                          @512      0x02
   ub1 kcvcpetb[1]                          @513      0x00
   ub1 kcvcpetb[2]                          @514      0x00
   ub1 kcvcpetb[3]                          @515      0x00
   ub1 kcvcpetb[4]                          @516      0x00
   ub1 kcvcpetb[5]                          @517      0x00
   ub1 kcvcpetb[6]                          @518      0x00
   ub1 kcvcpetb[7]                          @519      0x00

# 修改SCN
BBED> dump /v offset 484 count 32
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1       Offsets:  484 to  515  Dba:0x01800001
-------------------------------------------------------
 b27e3600 00000000 b62ee339 01000000 l .~6........9....
 2e000000 23960000 1000774a 02000000 l ....#.....wJ....

 <16 bytes per line>

BBED> modify /x d9 offset 484
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1                Offsets:  484 to  515           Dba:0x01800001
------------------------------------------------------------------------
 d97e3600 00000000 b62ee339 01000000 2e000000 23960000 1000774a 02000000 

 <32 bytes per line>

BBED> modify /x 7f3600 offset 485
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1                Offsets:  485 to  516           Dba:0x01800001
------------------------------------------------------------------------
 7f360000 000000b6 2ee33901 0000002e 00000023 96000010 00774a02 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 1:
current = 0xc392, required = 0xc392

# 修改SEQ
SQL> select to_number('2e','xxxxxxxxxxxxxxxxx') from dual;

TO_NUMBER('2E','XXXXXXXXXXXXXXXXX')
-----------------------------------
                                 46

SQL> select to_char(50,'xxxxxxxxx') from dual;

TO_CHAR(50
----------
        32

BBED> dump /v offset 500
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1       Offsets:  500 to  531  Dba:0x01800001
-------------------------------------------------------
 2e000000 23960000 1000774a 02000000 l ....#.....wJ....
 00000000 00000000 00000000 00000000 l ................

 <16 bytes per line>

BBED> modify /x 32 offset 500
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1                Offsets:  500 to  531           Dba:0x01800001
------------------------------------------------------------------------
 32000000 23960000 1000774a 02000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>


# 修改块号
BBED> dump /v offset 504
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1       Offsets:  504 to  535  Dba:0x01800001
-------------------------------------------------------
 23960000 1000774a 02000000 00000000 l #.....wJ........
 00000000 00000000 00000000 00000000 l ................

 <16 bytes per line>

BBED> modify /x 0100 offset 504
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 1                Offsets:  504 to  535           Dba:0x01800001
------------------------------------------------------------------------
 01000000 1000774a 02000000 00000000 00000000 00000000 00000000 00000000 

BBED> sum apply
Check value for File 6, Block 1:
current = 0x55ac, required = 0x55ac

# 之后6号数据文件在recover后就可以online
SQL> recover datafile 6;
Media recovery complete.
SQL> alter database datafile 6 online;

Database altered.
```

## datafile的status有哪些
datafile的status：
```
KCCFEFDB 0x0001 /* file read-only, plugged from foreign DB */
KCCFEONL 0x0002 /* file is ONLine */
KCCFERDE 0x0004 /* ReaDing is Enabled */
KCCFECGE 0x0008 /* ChanGing is Enabled */
KCCFEMRR 0x0010 /* Media Recovery Required */
KCCFEGEM 0x0020 /* Generate End hot backup Marker at next open */
KCCFECKD 0x0040 /* File record generated by check dictionary */
KCCFESOR 0x0080 /* Save Offline scn Range at next checkpoint */
KCCFERMF 0x0100 /* Renamed Missing File */
KCCFEGOI 0x0200 /* Generate Off-line Immediate marker */
KCCFECUV 0x0400 /* Checkpoint by instance where UnVerified */
KCCFEDRP 0x0800 /* offline to be DRoPped */
KCCFEODC 0x2000 /* Online at Dictionary Check if read/only tblspc */
KCCFEDBR 0x4000 /* entry created by DBMS_BACKUP_RESTORE */
KCCFETRO 0x8000 /* Transition Read Only */define KCCFEWCC 0x1000 /*
```

示例
``` perl
DATA FILE #1: 
  name #8: /u01/app/oracle/oradata/orcl/system01.dbf
creation size=0 block size=8192 status=0xe head=8 tail=8 dup=1

0xe = 0x8 + 0x4 + 0x2
Change/Write + Read + Online => Normal status of open files
```

## 详解检查点的结构
通过Data File Header Dump，可以从dump出的trace文件看到检查点最核心的结构，
``` perl
Checkpointed at scn:  0x0000.0036b8dd 03/19/2018 22:00:37
 thread:1 rba:(0x35.2.10)
 enabled  threads:  01000000 00000000 00000000 00000000 00000000 00000000

SCN：0x0000.0036b8dd 03/19/2018 22:00:37 时间 
RBA：日志的地址（log sequence number + block number + 偏移量）
THREAD：线程，1单实例
```

## 其他内容
Data File Header Dump
``` perl
alter session set events 'immediate trace name file_hdrs level <n>';
如：
alter session set events 'immediate trace name file_hdrs level 10';
```

level
```
level 1: control file's data file entry
level 2 & 4 : level 1 + generic file header
level 3 or higher: level 2 + data file header
level 10: Most commonly used, It provides the same output as level 3.
```

File Type
``` perl
KCCTYPCF 1 /* control file */
KCCTYPRL 2 /* redo log file */
KCCTYPDF 3 /* vanilla db file */
KCCTYPBC 4 /* backup control file */
KCCTYPBP 5 /* backup piece */
KCCTYPTF 6 /* temporary db file */
KCCTYPCT 7 /* change tracking file */
KCCTYPFL 8 /* flashback database log file */
KCCTYPAL 9 /* archivelog file */
KCCTYPDC 10 /* datafile copy file */
KCCTYPIR 11 /* incompletely restored db file */
KCCTYPEL 12 /* foreign archivelog file */
KCCTYPLB 13 /* LOB */
```