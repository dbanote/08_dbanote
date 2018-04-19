---
title: Oracle特殊恢复原理与实战_07 归档模式下缺失Redo Log后的恢复
date: 2018-04-03
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## Inactive redo log丢失或损坏的恢复
将数据库置于非归档模式
``` perl
SQL> archive log list;
Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     90
Current log sequence           91
```

<!-- more -->
INACTIVE STATUS：redo log里面的日志所对应的buffer cache里的脏块都写到datafile中了
``` perl
# 模拟出丢失或损坏的INACTIVE redo log，如第2组
select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC STATUS           FIRST_CHANGE# FIRST_TIME          NEXT_CHANGE# NEXT_TIME
---------- ---------- ---------- ---------- ---------- ---------- --- ---------------- ------------- ------------------- ------------ -------------------
         1          1         91   52428800        512          1 NO  CURRENT                4256107 2018-04-02 22:06:01   2.8147E+14
         2          1         89   52428800        512          1 YES INACTIVE               4227420 2018-04-02 07:00:17      4254495 2018-04-02 22:00:17
         3          1         90   52428800        512          1 YES INACTIVE               4254495 2018-04-02 22:00:17      4256107 2018-04-02 22:06:01

col MEMBER for a50
select * from v$logfile;

    GROUP# STATUS  TYPE    MEMBER                                             IS_
---------- ------- ------- -------------------------------------------------- ---
         3         ONLINE  /u01/app/oracle/oradata/orcl/redo03.log            NO
         2         ONLINE  /u01/app/oracle/oradata/orcl/redo02.log            NO     # 下面模拟2损坏
         1         ONLINE  /u01/app/oracle/oradata/orcl/redo01.log            NO


dd if=/dev/null of=/u01/app/oracle/oradata/orcl/redo02.log bs=512 count=20

# 改后可以直接正常关闭数据库
shutdown immediate

# 但打开数据库时报错
startup
#-----------------------------------------------------------------------------------
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             889193112 bytes
Database Buffers         3472883712 bytes
Redo Buffers               11661312 bytes
Database mounted.
ORA-03113: end-of-file on communication channel
Process ID: 18001
Session ID: 191 Serial number: 3
#-----------------------------------------------------------------------------------

# alert日志中有如下错误信息
#-----------------------------------------------------------------------------------
Tue Apr 03 16:08:12 2018
ARC0 started with pid=20, OS id=18003
ARC0: Archival started
LGWR: STARTING ARCH PROCESSES COMPLETE
ARC0: STARTING ARCH PROCESSES
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_lgwr_17981.trc:
ORA-00313: open failed for members of log group 2 of thread 1
ORA-00312: online log 2 thread 1: '/u01/app/oracle/oradata/orcl/redo02.log'     # 提示打开 log group 2 失败
ORA-27047: unable to read the header block of file
Linux-x86_64 Error: 25: Inappropriate ioctl for device
Additional information: 1
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_lgwr_17981.trc:
ORA-00313: open failed for members of log group 2 of thread 1
ORA-00312: online log 2 thread 1: '/u01/app/oracle/oradata/orcl/redo02.log'
ORA-27047: unable to read the header block of file
Linux-x86_64 Error: 25: Inappropriate ioctl for device
Additional information: 1
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_18001.trc:
ORA-00313: open failed for members of log group 1 of thread
ORA-00312: online log 2 thread 1: '/u01/app/oracle/oradata/orcl/redo02.log'
#-----------------------------------------------------------------------------------

# 这时，先正常打开数据库到mount状态
startup mount

# 查看log group 2状态
col MEMBER for a50
select l.STATUS,f.member,BYTES/1024/1024 size_m from v$log l, v$logfile f where l.GROUP#=2 and l.GROUP#=f.GROUP#;

STATUS           MEMBER                                                 SIZE_M
---------------- -------------------------------------------------- ----------
INACTIVE         /u01/app/oracle/oradata/orcl/redo02.log                    50

# 在控制文件中清除logfile group 2
alter database clear logfile group 2;
alter database drop logfile group 2;

# 删除logfile group 2对应的文件
! rm -rf /u01/app/oracle/oradata/orcl/redo02.log

# 增加logfile group 2
alter database add logfile group 2 ('/u01/app/oracle/oradata/orcl/redo02.log') size 50m;

# 能正常打开数据库了
SQL> alter database open;

Database altered.
```

## Active redo log丢失或损坏的恢复
ACTIVE STATUS：redo log里面的日志所对应的buffer cache里的脏块还没有刷新到datafile中
``` perl
# 准备环境
create tablespace lyj datafile '/u01/app/oracle/oradata/orcl/lyj_01.dbf' size 1g autoextend on next 500m maxsize unlimited;
drop user lyj cascade;
create user lyj identified by lyj default tablespace lyj;
grant dba to lyj;

# 增加并扩大logfile group
alter database add logfile group 4 ('/u01/app/oracle/oradata/orcl/redo04.log') size 200m;
alter database add logfile group 5 ('/u01/app/oracle/oradata/orcl/redo05.log') size 200m;
alter database add logfile group 6 ('/u01/app/oracle/oradata/orcl/redo06.log') size 200m;

select group#,status from v$log; 

    GROUP# STATUS
---------- ----------------
         1 CURRENT
         2 INACTIVE
         3 INACTIVE
         4 UNUSED
         5 UNUSED
         6 UNUSED

alter system switch logfile;
alter system checkpoint;

select group#,status from v$log; 

    GROUP# STATUS
---------- ----------------
         1 INACTIVE
         2 INACTIVE
         3 INACTIVE
         4 INACTIVE
         5 CURRENT
         6 INACTIVE

alter database drop logfile group 1;
alter database drop logfile group 2;
alter database drop logfile group 3;
! rm -rf /u01/app/oracle/oradata/orcl/redo01.log
! rm -rf /u01/app/oracle/oradata/orcl/redo02.log
! rm -rf /u01/app/oracle/oradata/orcl/redo03.log

alter database add logfile group 1 ('/u01/app/oracle/oradata/orcl/redo01.log') size 200m;
alter database add logfile group 2 ('/u01/app/oracle/oradata/orcl/redo02.log') size 200m;
alter database add logfile group 3 ('/u01/app/oracle/oradata/orcl/redo03.log') size 200m;

# 模拟一个大事务
conn lyj/lyj
drop table t1 purge;
create table t1(id int,name varchar(100));
begin
  for i in 1 .. 5000000
  loop
    insert into t1 values(i,'AAAAAA');
  end loop;
  commit;
end;
/

# 新打开一个窗口
set line 150
col member for a50
select l.group#,l.STATUS,f.member,BYTES/1024/1024 size_m 
  from v$log l, v$logfile f 
  where l.GROUP#=f.GROUP# order by 1;

    GROUP# STATUS           MEMBER                                                 SIZE_M
---------- ---------------- -------------------------------------------------- ----------
         1 CURRENT          /u01/app/oracle/oradata/orcl/redo01.log                   200
         2 INACTIVE         /u01/app/oracle/oradata/orcl/redo02.log                   200
         3 INACTIVE         /u01/app/oracle/oradata/orcl/redo03.log                   200
         4 INACTIVE         /u01/app/oracle/oradata/orcl/redo04.log                   200
         5 ACTIVE           /u01/app/oracle/oradata/orcl/redo05.log                   200 
         6 INACTIVE         /u01/app/oracle/oradata/orcl/redo06.log                   200

# 为了保障状态是active，直接shutdown abort数据库
SQL> shutdown abort

# 启动到mount状态后再查看日志组状态
startup mount
set line 150
col member for a50
select l.group#,l.STATUS,f.member,BYTES/1024/1024 size_m 
  from v$log l, v$logfile f 
  where l.GROUP#=f.GROUP# order by 1;

    GROUP# STATUS           MEMBER                                                 SIZE_M
---------- ---------------- -------------------------------------------------- ----------
         1 CURRENT          /u01/app/oracle/oradata/orcl/redo01.log                   200
         2 INACTIVE         /u01/app/oracle/oradata/orcl/redo02.log                   200
         3 INACTIVE         /u01/app/oracle/oradata/orcl/redo03.log                   200
         4 INACTIVE         /u01/app/oracle/oradata/orcl/redo04.log                   200
         5 ACTIVE           /u01/app/oracle/oradata/orcl/redo05.log                   200
         6 INACTIVE         /u01/app/oracle/oradata/orcl/redo06.log                   200

# 模拟状态为ACTIVE的log group 5损坏
dd if=/dev/null of=/u01/app/oracle/oradata/orcl/redo05.log bs=512 count=20


# 正常打开数据库时报以下错误
SQL> alter database open;
#-----------------------------------------------------------------------------------
alter database open
*
ERROR at line 1:
ORA-00313: open failed for members of log group 5 of thread 1
ORA-00312: online log 5 thread 1: '/u01/app/oracle/oradata/orcl/redo05.log'
ORA-27047: unable to read the header block of file
Linux-x86_64 Error: 25: Inappropriate ioctl for device
Additional information: 1
#-----------------------------------------------------------------------------------

# alert日志中有如下错误信息
#-----------------------------------------------------------------------------------
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_22346.trc:
ORA-00313: open failed for members of log group 5 of thread 1
ORA-00312: online log 5 thread 1: '/u01/app/oracle/oradata/orcl/redo05.log'
ORA-27047: unable to read the header block of file
#-----------------------------------------------------------------------------------

# 因为logfile group 5状态是active，clear和drop都不行
SQL> alter database clear logfile group 5;
alter database clear logfile group 5
*
ERROR at line 1:
ORA-01624: log 5 needed for crash recovery of instance orcl (thread 1)
ORA-00312: online log 5 thread 1: '/u01/app/oracle/oradata/orcl/redo05.log'


SQL> alter database drop logfile group 5;
alter database drop logfile group 5
*
ERROR at line 1:
ORA-01624: log 5 needed for crash recovery of instance orcl (thread 1)
ORA-00312: online log 5 thread 1: '/u01/app/oracle/oradata/orcl/redo05.log'


# 使用resetlog打开数据库时报以下错误
SQL> alter database open resetlogs;
alter database open resetlogs
*
ERROR at line 1:
ORA-01139: RESETLOGS option only valid after an incomplete database recovery

# 介质恢复
SQL> recover database until cancel;
ORA-00279: change 4326696 generated at 04/04/2018 13:58:56 needed for thread 1
ORA-00289: suggestion :
/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_04_04/o1_mf_1_162_%u_.arc
ORA-00280: change 4326696 for thread 1 is in sequence #162


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}

ORA-00308: cannot open archived log
'/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_04_04/o1_mf_1_162_%u_.arc'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3

ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/u01/app/oracle/oradata/orcl/system01.dbf'

# 再次resetlogs打开数据库
SQL> alter database open resetlogs;
alter database open resetlogs
*
ERROR at line 1:
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/u01/app/oracle/oradata/orcl/system01.dbf'

# 修改数据库pfile添加隐含参数
create pfile from spfile;

echo "*._allow_resetlogs_corruption=true" >> $ORACLE_HOME/dbs/init"$ORACLE_SID".ora
echo "*._allow_error_simulation=true" >> $ORACLE_HOME/dbs/init"$ORACLE_SID".ora

# 数据库使用pfile mount
shutdown abort
startup pfile="/u01/app/oracle/11.2.0.4/db_1/dbs/initorcl.ora" mount

# 使用resetlogs方式可以打开数据库
alter database open resetlogs;

Database altered.

# 查看redo log group状态，SEQUENCE#已从0开始
set line 150
col member for a50
select l.group#,l.STATUS,f.member,BYTES/1024/1024 size_m,sequence#
  from v$log l, v$logfile f 
  where l.GROUP#=f.GROUP# order by 1;

    GROUP# STATUS           MEMBER                                                 SIZE_M  SEQUENCE#
---------- ---------------- -------------------------------------------------- ---------- ----------
         1 CURRENT          /u01/app/oracle/oradata/orcl/redo01.log                   200          1
         2 UNUSED           /u01/app/oracle/oradata/orcl/redo02.log                   200          0
         3 UNUSED           /u01/app/oracle/oradata/orcl/redo03.log                   200          0
         4 UNUSED           /u01/app/oracle/oradata/orcl/redo04.log                   200          0
         5 UNUSED           /u01/app/oracle/oradata/orcl/redo05.log                   200          0
         6 UNUSED           /u01/app/oracle/oradata/orcl/redo06.log                   200          0

# 切换日志，在alert日志中查看是否有报错。
alter system switch logfile;

注意：这种修改隐含参数后使用resetlogs方式打开的数据库，生产环境中有可能的话（数量不太大），尽量将数据导出后重建库恢复。
```

## Current redo log丢失或损坏的恢复
CURRENT STATUS：实例正在使用的日志文件，并且redo log里面的日志可能还没有刷新到datafile中
``` perl
# 模拟一个大事务
conn lyj/lyj
drop table t1 purge;
create table t1(id int,name varchar(100));
begin
  for i in 1 .. 5000000
  loop
    insert into t1 values(i,'AAAAAA');
    commit;
  end loop;
end;
/

# 新打开一个窗口
set line 150
col member for a50
select l.group#,l.STATUS,f.member,BYTES/1024/1024 size_m 
  from v$log l, v$logfile f 
  where l.GROUP#=f.GROUP# order by 1;

    GROUP# STATUS           MEMBER                                                 SIZE_M
---------- ---------------- -------------------------------------------------- ----------
         1 ACTIVE           /u01/app/oracle/oradata/orcl/redo01.log                   200
         2 ACTIVE           /u01/app/oracle/oradata/orcl/redo02.log                   200
         3 ACTIVE           /u01/app/oracle/oradata/orcl/redo03.log                   200
         4 ACTIVE           /u01/app/oracle/oradata/orcl/redo04.log                   200
         5 CURRENT          /u01/app/oracle/oradata/orcl/redo05.log                   200
         6 INACTIVE         /u01/app/oracle/oradata/orcl/redo06.log                   200

# 新开一个窗口，将状态为CURRENT的log group 5损坏
dd if=/dev/null of=/u01/app/oracle/oradata/orcl/redo05.log bs=512 count=20

# 稍过一会，执行大事务的会话中断，实例shutdown
ERROR at line 1:
ORA-03113: end-of-file on communication channel
Process ID: 25677
Session ID: 122 Serial number: 5

# 启动数据库实例报错
SQL> startup
ORACLE instance started.

Total System Global Area 4359294976 bytes
Fixed Size                  2260288 bytes
Variable Size             855638720 bytes
Database Buffers         3489660928 bytes
Redo Buffers               11735040 bytes
Database mounted.
ORA-00313: open failed for members of log group 5 of thread 1
ORA-00312: online log 5 thread 1: '/u01/app/oracle/oradata/orcl/redo05.log'
ORA-27048: skgfifi: file header information is invalid
Additional information: 13

# 尝试open resetlogs报需要恢复
SQL> alter database open resetlogs;
alter database open resetlogs
*
ERROR at line 1:
ORA-01139: RESETLOGS option only valid after an incomplete database recovery

# 介质恢复

SQL> recover database until cancel;
ORA-00279: change 208852 generated at 04/04/2018 21:26:30 needed for thread 1
ORA-00289: suggestion :
/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_04_04/o1_mf_1_23_%u_.arc
ORA-00280: change 208852 for thread 1 is in sequence #23


Specify log: {<RET>=suggested | filename | AUTO | CANCEL}

ORA-00308: cannot open archived log
'/u01/app/oracle/fast_recovery_area/ORCL/archivelog/2018_04_04/o1_mf_1_23_%u_.arc'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3


ORA-01547: warning: RECOVER succeeded but OPEN RESETLOGS would get error below
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/u01/app/oracle/oradata/orcl/system01.dbf'

# 再次open resetlogs
SQL> alter database open resetlogs;
alter database open resetlogs
*
ERROR at line 1:
ORA-01194: file 1 needs more recovery to be consistent
ORA-01110: data file 1: '/u01/app/oracle/oradata/orcl/system01.dbf'

# 修改数据库pfile添加隐含参数
create pfile from spfile;

echo "*._allow_resetlogs_corruption=true" >> $ORACLE_HOME/dbs/init"$ORACLE_SID".ora
echo "*._allow_error_simulation=true" >> $ORACLE_HOME/dbs/init"$ORACLE_SID".ora

# 使用pfile open resetlog 
SQL> shutdown abort
ORACLE instance shut down.
SQL> startup pfile="/u01/app/oracle/11.2.0.4/db_1/dbs/initorcl.ora" mount
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             889193112 bytes
Database Buffers         3472883712 bytes
Redo Buffers               11661312 bytes
Database mounted.

SQL> alter database open resetlogs;
alter database open resetlogs
*
ERROR at line 1:
ORA-01092: ORACLE instance terminated. Disconnection forced
ORA-00600: internal error code, arguments: [2663], [0], [210131], [0],
[217516], [], [], [], [], [], [], []
Process ID: 25824
Session ID: 122 Serial number: 3
```

>使用隐含参数_ALLOW_RESETLOGS_CORRUPTION后resetlogs打开数据库后,我们说很多时候你会遇到ORA-00600 2662号错误，这个错误的含义是:
A data block SCN is ahead of the current SCN.
The ORA-600 [2662] occurs when an SCN is compared to the dependent SCN
stored in a UGA variable.
If the SCN is less than the dependent SCN then we signal the ORA-600 [2662] internal error.
ORA-600 [2662] [a] [b] [c] [d] [e]
  Arg [a] Current SCN WRAP
  Arg [b] Current SCN BASE
  Arg [c] dependent SCN WRAP
  Arg [d] dependent SCN BASE
  Arg [e] Where present this is the DBA where the dependent SCN came from.
算法计算规则如下：Arg [c]*4得出一个数值，假设为V_Wrap,
如果Arg [d]=0，则V_Wrap值为需要的level
Arg [d] < 1073741824，V_Wrap+1为需要的level
Arg [d] < 2147483648，V_Wrap+2为需要的level
Arg [d] < 3221225472，V_Wrap+3为需要的level

使用10015 event手工推进scn
``` perl
startup pfile="/u01/app/oracle/11.2.0.4/db_1/dbs/initorcl.ora" mount
alter session set events '10015 trace name adjust_scn level 1';

SQL> alter database open;

Database altered.

这时数据库虽然能open，但alert中会有类似以下的报错，强烈建议将数据导出重建一个库
ORA-01595: error freeing extent (2) of rollback segment (2))
ORA-00600: internal error code, arguments: [4193], [31], [1], [], [], [], [], [], [], [], [], []
Dumping diagnostic data in directory=[cdmp_20180404214920], requested by (instance=1, osid=26128 (SMON)), summary=[incident=21732].
```

## 增进SCN有两种常用方法
1.通过immediate trace name方式(在数据库Open状态下)
``` perl 
alter session set events 'IMMEDIATE trace name ADJUST_SCN level x';
```

2.通过10015事件(在数据库无法打开，mount状态下)
``` perl
alter session set events '10015 trace name adjust_scn level x';
```
注:level 1为增进SCN 10亿(1 billion) (1024*1024*1024),通常Level 1已经足够。也可以根据实际情况适当调整。

## Dump logfile解析一个事务的日志格式
``` perl
conn lyj/lyj
create table t2 (id int, name varchar2(10));
insert into t2 values (1,'AAAAA');
commit;
alter system switch logfile;
update t2 set name='BBBBB' where name='AAAAA';
commit;

set line 150
col member for a50
select l.group#,l.STATUS,f.member,BYTES/1024/1024 size_m,sequence#
  from v$log l, v$logfile f 
  where l.GROUP#=f.GROUP# order by 1;

    GROUP# STATUS           MEMBER                                                 SIZE_M  SEQUENCE#
---------- ---------------- -------------------------------------------------- ---------- ----------
         1 ACTIVE           /u01/app/oracle/oradata/orcl/redo01.log                   200          1
         2 CURRENT          /u01/app/oracle/oradata/orcl/redo02.log                   200          2
         3 UNUSED           /u01/app/oracle/oradata/orcl/redo03.log                   200          0
         4 UNUSED           /u01/app/oracle/oradata/orcl/redo04.log                   200          0
         5 UNUSED           /u01/app/oracle/oradata/orcl/redo05.log                   200          0
         6 UNUSED           /u01/app/oracle/oradata/orcl/redo06.log                   200          0

# 导出当前的log file group
alter system dump logfile '/u01/app/oracle/oradata/orcl/redo02.log';

col VALUE for a80
select VALUE from v$diag_info where NAME='Default Trace File';

VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_22978.trc
```

dump的trace信息如下：
``` perl
......
REDO RECORD - Thread:1 RBA: 0x000002.00000004.0010 LEN: 0x024c VLD: 0x0d
SCN: 0x0000.00420e9e SUBSCN:  1 04/04/2018 15:10:43
(LWN RBA: 0x000002.00000004.0010 LEN: 0002 NST: 0001 SCN: 0x0000.00420e9d)
CHANGE #1 TYP:2 CLS:1 AFN:9 DBA:0x02400e87 OBJ:94895 SCN:0x0000.00420e97 SEQ:1 OP:11.19 ENC:0 RBL:0
KTB Redo
op: 0x11  ver: 0x01
compat bit: 4 (post-11) padding: 1
op: F  xid:  0x0002.017.00000a72    uba: 0x00c0011a.03e7.29
Block cleanout record, scn:  0x0000.00420e9d ver: 0x01 opt: 0x02, entries follow...
  itli: 1  flg: 2  scn: 0x0000.00420e97
Array Update of 1 rows:
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 2 ckix: 204
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x02400e87  hdba: 0x02400e82
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  42 42 42 42 42
CHANGE #2 TYP:0 CLS:19 AFN:3 DBA:0x00c00090 OBJ:4294967295 SCN:0x0000.00420e51 SEQ:2 OP:5.2 ENC:0 RBL:0
ktudh redo: slt: 0x0017 sqn: 0x00000a72 flg: 0x0012 siz: 168 fbi: 0
            uba: 0x00c0011a.03e7.29    pxid:  0x0000.000.00000000
CHANGE #3 TYP:0 CLS:19 AFN:3 DBA:0x00c00090 OBJ:4294967295 SCN:0x0000.00420e9e SEQ:1 OP:5.4 ENC:0 RBL:0
ktucm redo: slt: 0x0017 sqn: 0x00000a72 srt: 0 sta: 9 flg: 0x2 ktucf redo: uba: 0x00c0011a.03e7.29 ext: 1 spc: 2252 fbi: 0
CHANGE #4 TYP:0 CLS:20 AFN:3 DBA:0x00c0011a OBJ:4294967295 SCN:0x0000.00420e51 SEQ:3 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 168 spc: 2422 flg: 0x0012 seq: 0x03e7 rec: 0x29
            xid:  0x0002.017.00000a72
ktubl redo: slt: 23 rci: 0 opc: 11.1 [objn: 94895 objd: 94895 tsn: 10]
Undo type:  Regular undo        Begin trans    Last buffer split:  No
Temp Object:  No
Tablespace Undo:  No
             0x00000000  prev ctl uba: 0x00c0011a.03e7.26
prev ctl max cmt scn:  0x0000.0042094a  prev tx cmt scn:  0x0000.0042095c
txn start scn:  0x0000.00420e9c  logon user: 85  prev brb: 12583193  prev bcl: 0 BuExt idx: 0 flg2: 0
KDO undo record:
KTB Redo
op: 0x03  ver: 0x01
compat bit: 4 (post-11) padding: 1
op: Z
Array Update of 1 rows:
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 204
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x02400e87  hdba: 0x02400e82
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  41 41 41 41 41
......
```

RBA: 0x000002.00000004.0010  - redo log地址：sequence#是2的redolog file的第4个块的偏移量第10个节字开始
LEN: 0x024c - 长度
VLD: 0x0d - 类型
SCN: 0x0000.00420e9e - 对应日志所产生的SCN
SUBSCN:  1 04/04/2018 15:10:43 - 同一时间产生的日志量大时，通过SUBSCN来区别

OP:11.19 - OP是操作代码 11.19代表update
  OP:5.2 - 启动一个事务
  OP:5.4 - 提交
  OP:5.1 - 修改前的值

CHANGE #1 - TYP:2 CLS:1 AFN:9 DBA:0x02400e87 OBJ:94895 SCN:0x0000.00420e97 SEQ:1 OP:11.19 ENC:0 RBL:0
  OP:11.19-update修改的值  TYP:2-类型2是普通文件  CLS:1-数据类型  AFN:9-绝对文件号  DBA:0x02400e87-数据文件地址 OBJ:94895-对象编号
CHANGE #2 TYP:0 CLS:19 AFN:3 DBA:0x00c00090 OBJ:4294967295 SCN:0x0000.00420e51 SEQ:2 OP:5.2 ENC:0 RBL:0
  OP:5.2-启动一个事务(undo) AFN:3-绝对文件号(这个文件对应的是undo file)
CHANGE #3 TYP:0 CLS:19 AFN:3 DBA:0x00c00090 OBJ:4294967295 SCN:0x0000.00420e9e SEQ:1 OP:5.4 ENC:0 RBL:0
  OP:5.4-提交  AFN:3-绝对文件号(这个文件对应的是undo file)
CHANGE #4 TYP:0 CLS:20 AFN:3 DBA:0x00c0011a OBJ:4294967295 SCN:0x0000.00420e51 SEQ:3 OP:5.1 ENC:0 RBL:0
  OP:5.1-update修改前的值  AFN:3-绝对文件号(这个文件对应的是undo file)

## 查询隐含参数脚本
``` perl
vi $ORACLE_HOME/rdbms/admin/show_para.sql
#-----------------------------------------------------------------------------------
col p_name for a48
col p_DESCRIPTION for a62
col p_value for a10
set linesize 150
set pagesize 9999
SELECT  i.ksppinm p_name,
    i.ksppdesc p_description,
    CV.ksppstvl p_VALUE,
    CV.ksppstdf isdefault,
    DECODE (BITAND (CV.ksppstvf, 7),1, 'MODIFIED',4, 'SYSTEM_MOD', 'FALSE')
    ismodified,
    DECODE (BITAND (CV.ksppstvf, 2), 2, 'TRUE', 'FALSE') isadjusted
    FROM sys.x$ksppi i, sys.x$ksppcv CV
    WHERE i.inst_id = USERENV ('Instance')
    AND CV.inst_id = USERENV ('Instance')
    AND i.indx = CV.indx
    AND upper(i.ksppinm) LIKE upper('%&p%')
    ORDER BY REPLACE (i.ksppinm, '_', '');
#-----------------------------------------------------------------------------------

# 在sqlplus中执行
@?/rdbms/admin/show_para
```

