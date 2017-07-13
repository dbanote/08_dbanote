---
title: Oracle 12c特性解读-容器数据库和灾备-06 PDB备份与恢复
date: 2017-06-10
tags:
- oracle
- 12c
---

## 模拟4种完全恢复场景
### 数据库open状态，普通表空间损坏
``` perl
# 备份整个数据库
rman target /
RMAN> backup database 
format '/u01/app/rmanbackup/bak_%d_%T_%s_%U';

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO

SQL> conn lyj/lyj@pdb1
SQL> select count(*) from test;

  COUNT(*)
----------
    272404

SQL> select FILE_NAME,TABLESPACE_NAME from dba_data_files;

FILE_NAME                                          TABLESPACE_NAME
-------------------------------------------------- ------------------------------
/u01/app/oracle/oradata/orcl1/pdb1/system01.dbf    SYSTEM
/u01/app/oracle/oradata/orcl1/pdb1/sysaux01.dbf    SYSAUX
/u01/app/oracle/oradata/orcl1/pdb1/undotbs01.dbf   UNDOTBS1
/u01/app/oracle/oradata/orcl1/pdb1/users01.dbf     USERS

# 模拟普通表空间损坏
SQL> !rm /u01/app/oracle/oradata/orcl1/pdb1/users01.dbf

# 可以创建表并插入数据都可以正常完成
create table test1 (id int, name varchar2(10));
insert into test1 values (1,'aaaa');
insert into test1 values (2,'bbbb');
commit;

SQL> col SEGMENT_NAME for a50
SQL> select SEGMENT_NAME,TABLESPACE_NAME from user_segments;

SEGMENT_NAME                                       TABLESPACE_NAME
-------------------------------------------------- ------------------------------
TEST                                               USERS
TEST1                                              USERS

# 再创建一个大表时报错
SQL> create table test2 as select * from all_objects;
create table test2 as select * from all_objects
                                    *
ERROR at line 1:
ORA-01110: data file 30: '/u01/app/oracle/oradata/orcl1/pdb1/users01.dbf'
ORA-01116: error in opening database file 30
ORA-01110: data file 30: '/u01/app/oracle/oradata/orcl1/pdb1/users01.dbf'
ORA-27041: unable to open file
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3

# 打开另一个session窗口
conn / as sysdba
SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO

alter pluggable database pdb1 close;

# 这时直接打开肯定也会报错
SQL> alter pluggable database pdb1 open;
alter pluggable database pdb1 open
*
ERROR at line 1:
ORA-01157: cannot identify/lock data file 30 - see DBWR trace file
ORA-01110: data file 30: '/u01/app/oracle/oradata/orcl1/pdb1/users01.dbf'

# 使用RMAN恢复PDB
rman target /
restore pluggable database pdb1;
recover pluggable database pdb1;

# 这时pdb1就可以OPEN了
alter pluggable database pdb1 open;

# 删除表空间文件后，创建的表也恢复成功
conn lyj/lyj@pdb1
select count(*) from test1;
  COUNT(*)
----------
         2
```

<!-- more -->

### 数据库关闭状态，系统表空间损坏
``` perl
alter pluggable database pdb1 close;
# 模拟系统表空间损坏
!rm /u01/app/oracle/oradata/orcl1/pdb1/system01.dbf

SQL> alter pluggable database pdb1 open;
alter pluggable database pdb1 open
*
ERROR at line 1:
ORA-01157: cannot identify/lock data file 27 - see DBWR trace file
ORA-01110: data file 27: '/u01/app/oracle/oradata/orcl1/pdb1/system01.dbf'

# 使用RMAN恢复系统表空间
rman target /
restore tablespace pdb1:system;
recover tablespace pdb1:system;

# 恢复后，PDB数据库正常OPEN
alter pluggable database pdb1 open;
```

### 数据库关闭状态，普通表空间损坏
``` perl
alter pluggable database pdb1 close;
!rm /u01/app/oracle/oradata/orcl1/pdb1/users01.dbf

SQL> alter pluggable database pdb1 open;
alter pluggable database pdb1 open
*
ERROR at line 1:
ORA-01157: cannot identify/lock data file 30 - see DBWR trace file
ORA-01110: data file 30: '/u01/app/oracle/oradata/orcl1/pdb1/users01.dbf'

# 使用RMAN恢复普通用户表空间
rman target /
restore tablespace pdb1:users;
recover tablespace pdb1:users;

# 恢复后，PDB数据库正常OPEN
SQL> alter pluggable database pdb1 open;

Pluggable database altered.

SQL> conn lyj/lyj@pdb1
Connected.
SQL> select count(*) from test1;

  COUNT(*)
----------
        10
```

### 数据库open状态，未备份的数据文件恢复
``` perl
conn / as sysdba
alter session set container=pdb1;
create tablespace bi datafile '/u01/app/oracle/oradata/orcl1/pdb1/bi01.dbf' size 100m;
create user bi identified by bi default tablespace bi;
grant dba to bi;
conn bi/bi@pdb1
create table test as select * from all_objects;

!rm /u01/app/oracle/oradata/orcl1/pdb1/bi01.dbf

SQL> create table test1 as select * from all_objects;
create table test1 as select * from all_objects
                                    *
ERROR at line 1:
ORA-01116: error in opening database file 31
ORA-01110: data file 31: '/u01/app/oracle/oradata/orcl1/pdb1/bi01.dbf'
ORA-27041: unable to open file
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3

conn / as sysdba
alter pluggable database pdb1 close;

# 使用RMAN恢复PDB数据库
rman target /
restore pluggable database pdb1;
recover pluggable database pdb1;

# 恢复成功后打开PDB数据库，并验证数据
SQL> alter pluggable database pdb1 open;

Pluggable database altered.

SQL> conn bi/bi@pdb1
Connected.
SQL> select count(*) from test;

  COUNT(*)
----------
     68103
```

## 模拟不完全恢复场景（drop user）
``` perl
select systimestamp from dual;

SYSTIMESTAMP
---------------------------------------------------------------------------
12-JUN-17 04.09.11.763875 PM +08:00

drop user bi cascade;

SQL> conn bi/bi@pdb1
ERROR:
ORA-01017: invalid username/password; logon denied


Warning: You are no longer connected to ORACLE.


alter pluggable database pdb1 close;

# 使用rman进行不完全恢复
rman target /

# 不加AUXILIARY DESTINATION '/tmp'报错
run 
{ 
  set until time "to_date('2017-06-12 16:09:11','YYYY-MM-DD HH24:MI:SS')"; 
  restore pluggable database pdb1; 
  recover pluggable database pdb1 AUXILIARY DESTINATION '/tmp';
}

conn / as sysdba
alter pluggable database pdb1 open resetlogs;

# 用户恢复成功
SQL> conn bi/bi@pdb1
Connected.
SQL> select count(*) from test;

  COUNT(*)
----------
     68103
```

## RMAN配置总结
``` perl
# RMAN常用配置命令
SHOW ALL;
CONFIGURE RETENTION POLICY TO REDUNDANCY 1;
CONFIGURE RETENTION POLICY TO recovery window of 3 days
CONFIGURE DEFAULT DEVICE TYPE TO DISK;
CONFIGURE CONTROLFILE AUTOBACKUP OFF|ON;

# 备份分片
CONFIGURE CHANNEL DEVICE TYPE DISK MAXPIECESIZE 50 M;
CONFIGURE CHANNEL 1 DEVICE TYPE DISK maxpiecesize 100M format '/u02/ora11g/flash_recovery_area/TEST/full_bak_%U';
backup filesperset = 5 as compressed backupset database format '/tmp/full_db_%U';
backup filesperset = 5 as compressed backupset pluggable database FF format '/tmp/ff_db_%U';

# 1个channel和多个channel
CONFIGURE CHANNEL 1 DEVICE TYPE DISK MAXPIECESIZE 50 M FORMAT '/u02/ora11g/flash_recovery_area/TEST/full_bak_%U';
CONFIGURE CHANNEL 2 DEVICE TYPE DISK MAXPIECESIZE 50 M FORMAT '/u02/ora11g/flash_recovery_area/TEST/full_bak_%U';
configure channel 2 device type disk clear;
show device type;
show default device type;
configure channel 1 device type disk format '/home/oracle/%U';
configure channel 1 device type disk clear;

# 显示和报告
list backup;
list expired backup;
list backupset summary;
list backup of spfile;
list backup of controlfile
report schema;
report need backup days 3;
report need backup;
report obsolete;

list backup of pluggable database tcymob1_new12c;
list backup of tablespace data;
list backup of tablespace tcymob1_new12c:system
list backup of datafile 1;

# 检测失效及删除
crosscheck copy;
crosscheck backup;
crosscheck archivelog all;
delete noprompt expired copy;
delete noprompt expired backup;
delete noprompt expired archivelog all;
delete noprompt obsolete;
backup database plus archivelog skip inaccessible delete input;

# 手工注册归档日志
catalog start with '/u01/app/oracle/archive';

# Block change track
alter database enable block change tracking using file '/u01/app/oracle/change_tracking';
alter database disable block change tracking;

# 增量备份
backup incremental level=0 database;
backup incremental level=1 database;
backup incremental level=2 database;

# 其他
backup as compressed backupset database format '/oradata/rmanbakcup/bak_%d_%T_%s_%U';
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/oradata/rmanbakcup/ctl_%F';
```

## 杂记
### 数据恢复的原理
* SCN 数据库对自身变化的一个标记
* 系统级（checkpoint_change# v$database）
* 数据文件级 (file#,checkpoint_change# v$datafile)
* 结束SCN(file#,last_change# v$datafile）
* 数据文件头部 (name,checkpoint_change# v$datafile_header)
* 12c中 v$database con_id=0

### RMAN的核心
* recover.bsq
* Dbms_rcvman
* Dbms_backup_restore

### 12c RMAN过期的命令
[http://docs.oracle.com/database/122/RCMRF/deprecated-rman-syntax.htm#RCMRF910 ](http://docs.oracle.com/database/122/RCMRF/deprecated-rman-syntax.htm#RCMRF910)

### RMAN还原恢复数据库
``` perl
# CDB级别的数据还原恢复
startup nomount
restore controlfile from ‘xxxx’;
alter database mount;
catalog start with ‘xxxxx’;
restore database;

# PDB级别的数据备份
backup pluggable database tcymob0;

# PDB级别的数据还原恢复
catalog start with ‘xxxxx’;
restore pluggable database xx;
recover tablespace pdb1:system;
```

### rman连接方式
``` perl
rman target /
rman target sys/oracle
rman target sys/oracle@orcl catalog rman/rman@emrep

connect target sys/oracle
connect target pdb_mgr/oracle@pdb
```

### 删除了所有的数据文件，日志文件，控制文件的恢复
``` perl
restore controlfile from autobackup;
restore controlfile from '/u02/oracle/flash_recovery_area/TEST10G/ctl_c-1135735312-20150802-0b';
run{
  restore database; 
  recover database; 
}

run{ 
  set until sequence 1; 
  recover database; 
}

alter database open resetlogs;
```

### 恢复相关的修改隐含参数
``` perl
_allow_resetlogs_corruption=true
_corrupted_rollback_segments=true
_offline_rollback_segments=true
```