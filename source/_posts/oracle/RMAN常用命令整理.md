---
title: RMAN (Recovery Manager) 常用命令整理
date: 2017-10-19
tags:
- oracle
- rman
categories:
- 常用命令与脚本
---

## RMAN核心文件
RMAN的核心文件：recover.bsq (dbms_rcvman dbms_backup_restore) 
所在位置：$ORACLE_HOME/rdbms/admin/recover.bsq

<!-- more -->

## RMAN常用命令
``` perl
rman target /
show all;
configure retention policy to redundancy 1;
configure retention policy to recovery window of 3 days;
configure retention policy clear;
configure controlfile autobackup on;
configure default device type to disk;
configure channel device type disk maxpiecesize 50M;
configure channel 1 device type disk maxpiecesize 100M format '/oradata/rmanbackup/full_bak_%U';
configure channel 2 device type disk maxpiecesize 100M format '/oradata/rmanbackup/full_bak_%U';
configure channel 3 device type disk maxpiecesize 100M format '/oradata/rmanbackup/full_bak_%U';
configure channel 4 device type disk maxpiecesize 100M format '/oradata/rmanbackup/full_bak_%U';
configure device type disk parallelism 4 backup type to compressed backupset;

show device type;
show default device type;

list backup;
list expired backup;
list backup of spfile;
list backup of controlfile;
list backup of tablespace users;
list backup of datafile 1;
list backup of archivelog all;

crosscheck copy;
crosscheck backup;
crosscheck archivelog all;
delete noprompt expired copy;
delete noprompt expired backup;
delete noprompt expired archivelog all;
report obsolete;
delete noprompt obsolete;


report schema;
report need backup days 3;
report need backup;
report obsolete;

backup database;
backup database plus archivelog skip inaccessible delete input;
backup incremental level=0 database;
backup incremental level=1 database;
backup incremental level=2 database;

delete backupset 1,2,3,4,5;

restore database;
restore tablespace users;
restore datafile 1;
restore controlfile from autobackup;

recover database;
recover tablespace users;
recover datafile 1;
recover database until sequence 8;
recover database until scn 1812584;

run{
    set until time "to_date('2016-08-29 11:15:22','YYYY-MM-DD HH24:MI:SS')";
    restore database;
    recover database;
}

run{
    set until sequence 9;
    recover database;
}

# 示例
startup nomount
restore controlfile from 'xxxx';
alter database mount;
catalog start with 'xxxxx';
restore database;

run {
allocate channel c1 type disk;
allocate channel c2 type disk;
allocate channel c3 type disk;
allocate channel c4 type disk;
backup as compressed backupset
    skip inaccessible
    tag db_bak
    filesperset 4
    format '/oradata/rmanbackup/bak_%d_%T_%s_%U'
    database;
    sql 'alter system archive log current';
release channel c1;
release channel c2;
release channel c3;
release channel c4;
}

run {
allocate channel c1 type disk;
allocate channel c2 type disk;
allocate channel c3 type disk;
allocate channel c4 type disk;
backup as compressed backupset
    tag arc_bak
    format '/oradata/rmanbackup/arc_%d_%T_%s_%U'
    archivelog all delete input;
release channel c1;
release channel c2;
release channel c3;
release channel c4;
}
```

## RMAN相关概念与脚本
### SCN：System Change Number 数据库对自身变化的一个标记
- 系统级 (v$database checkpoint_change#)
- 数据文件级/结束SCN (v$datafile file#, checkpoint_change#, last_change#)
- 数据文件头部 (v$datafile_header file#, checkpoint_change#)
``` perl
select checkpoint_change# from v$database;
select file#, checkpoint_change#, last_change# from v$datafile;
select file#, checkpoint_change# from v$datafile_header;
alter system checkpoint;
```

### 查看autobackup controlfile延迟时间隐藏参数脚本
正常情况下，建议RMAN中设置`CONFIGURE CONTROLFILE AUTOBACKUP ON`，这样在增加删除表空间等数据库维护操作时，做备份时，会自动建立控制文件与spfile文件的备份。在维护数据库增加数据文件时，并不会马上做控制文件的自动备份，实际上有一些小小延迟，延迟时间受参数`_controlfile_autobackup_delay`的控制，这样的目地在于如果增加很多数据文件，频繁备份并不是非常必要，适当延迟可以避免频繁备份控制文件。
``` perl
set pagesize 9999
set line 180
col name for a45
col value for a50
col description for a70
select a.ksppinm name, b.ksppstvl value, a.ksppdesc description
    from x$ksppi a, x$ksppcv b
    where a.indx = b.indx and a.ksppinm like '%control%delay';
```

### 开启归档模式
``` perl
archive log list
select log_mode from v$database;
shutdown immediate
startup mount
alter database archivelog;
alter database noarchivelog;
alter database open;
show parameter archive
show parameter recovery
alter system set log_archive_dest='/oradata/archivelog';
alter system set log_archive_format='arch_%t_%s_%r.arc' scope=spfile;
select group#,sequence#,members,status from v$log;
alter system switch logfile;
alter system checkpoint;
```

### 控制文件和参数文件的备份
``` perl
alter database backup controlfile to '/tmp/control_bak.ctl';
alter database backup controlfile to trace;
# 通过以下命令能查看trace位置
select * from v$diag_info where NAME='Default Trace File';

# 通过trace里脚本重建的控制文件，还需要手工修改temp表空间(表空间存在，数据文件实际存在，但没有被映射，以下脚本是进行复用)
select NAME from v$tempfile;
col FILE_NAME for a50
col TABLESPACE_NAME for a10
select FILE_NAME,TABLESPACE_NAME from dba_temp_files;
alter tablespace temp add tempfile '/u01/app/oracle/oradata/orcl/temp01.dbf' size 20M reuse;
show parameter control

create pfile='/tmp/pfile.ora' from spfile;
```

### Block change track
`Block change track`是Oracle块修改跟踪，会记录`data file`里每个block的update信息，这些tracking信息保存在tracking文件里。 
当启动`block change tracking`后，RMAN使用`trackingfile`里的信息，只读取改变的block信息，而不用在对整个data file进行扫描，从而提高了RMAN 备份的性能。启用/禁用的方法如下：
``` perl
alter database enable block change tracking using file '/u01/app/oracle/change_tracking';
alter database disable block change tracking;
```

###  修改隐含参数，极端情况下恢复数据
``` perl
alter system set "_allow_resetlogs_corruption"=true scope=spfile;
alter system set "_corrupted_rollback_segments"=true scope=spfile;
alter system set "_offline_rollback_segments"=true scope=spfile;

alter system set "_allow_resetlogs_corruption"=false scope=spfile;
alter system set "_corrupted_rollback_segments"=false scope=spfile;
alter system set "_offline_rollback_segments"=false scope=spfile;

# 设置"_allow_resetlogs_corruption"=true，在数据库Open过程中，Oracle会跳过某些一致性检查，从而使数据库可能跳过不一致状态
# 设置"_corrupted_rollback_segments"=true，以使数据库在启动的时候忽略损坏的回滚段，使数据库正常启动
# 设置"_offline_rollback_segments"=true，可以让指定的回滚段处于OFFLINE状态，从而达到和设置"_corrupted_rollback_segments"=true 一样的效果，两个参数可以配合使用.
```

### 热备 
- 存在Split blocks 分离数据块的问题
- 启动热备保护，解决Split blocks问题，方法如下
``` perl
alter tablespace xxxxx begin backup;
执行热备 ( cp scp ftp -> datafile )
alter tablespace xxxxx end backup;

# 生成热备所需批量脚本
select 'alter tablespace ' || tablespace_name || ' begin backup;' from dba_tablespaces where CONTENTS <> 'TEMPORARY';
select 'alter tablespace ' || tablespace_name || ' end backup;' from dba_tablespaces where CONTENTS <> 'TEMPORARY';
```


### 查看数据中所有文件和总大小脚本
``` perl
select name from v$datafile
union
select name from v$controlfile
union
select name from v$tempfile
union
select member from v$logfile;


select sum(sum_bytes)/1024/1024 m_bytes
from (
    select sum(bytes) sum_bytes from v$datafile
    union
    select sum(bytes) sum_bytes from v$tempfile
    union
    select (sum(bytes) * members) sum_bytes from v$log
    group by members);
```
