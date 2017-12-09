---
title: flashback常用命令整理
date: 2017-10-19
tags:
- oracle
- flashback
categories:
- 常用命令与脚本
---

## flashback常用命令整理

### 打开数据库闪回
``` perl
SQL> select name,open_mode,log_mode,force_logging,flashback_on from v$database;

NAME      OPEN_MODE            LOG_MODE     FOR FLASHBACK_ON
--------- -------------------- ------------ --- ------------------
ORCL      READ WRITE           ARCHIVELOG   NO  NO

SQL> archive log list;
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            /u01/app/arch/
Oldest online log sequence     246
Next log sequence to archive   249
Current log sequence           249

SQL> show parameter recovery

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_recovery_file_dest                string
db_recovery_file_dest_size           big integer 0
recovery_parallelism                 integer     0

alter system set log_archive_dest='';
alter system set db_recovery_file_dest_size=10g;
alter system set db_recovery_file_dest='/u01/app/fra';

# 2880单位是分钟minutes
alter system set db_flashback_retention_target=2880;

alter database flashback on;
```

<!-- more -->
### 数据库的闪回
``` perl
# 基于SCN的闪回
shutdown immediate
startup
flashback database to scn 5303782;
alter database open resetlogs;

# 基于时间闪回
shutdown immediate
startup
flashback database to timestamp to_timestamp('2017-10-20 10:09:56','yyyy-mm-dd hh24:mi:ss');
alter database open resetlogs;
```

### 删除表的闪回
``` perl
conn lyj/lyj
drop table flashback_test;

show recyclebin;
ORIGINAL NAME    RECYCLEBIN NAME                OBJECT TYPE  DROP TIME
---------------- ------------------------------ ------------ -------------------
FLASHBACK_TEST   BIN$W/KZ5pudF7bgU24D9QoueQ==$0 TABLE        2017-10-20:10:55:11

flashback table FLASHBACK_TEST to before drop;
```

### DML操作的闪回
``` perl
# 根据时间确定SCN号
select timestamp_to_scn(to_date('2017-10-20 11:23:44','yyyy-mm-dd hh24:mi:ss')) scn from dual;
alter table flashback_test enable row movement;
flashback table flashback_test to scn 5305382;
```

### 查询的闪回
``` perl
# 基于时间的闪回查询（5分钟前）
select count(*) from flashback_test as of timestamp sysdate-5/1440;

# 基于SCN号的闪回查询
select count(*) from flashback_test as of scn 5305382;

# 闪回查询时间设置参数，单位是秒 10800s=3小时间
alter system set undo_retention=10800;
```


