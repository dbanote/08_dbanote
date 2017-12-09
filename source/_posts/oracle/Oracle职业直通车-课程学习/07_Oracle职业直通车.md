---
title: Oracle职业直通车-07 重做日志、日志挖掘和回滚段
date: 2015-04-08
tags:
- oracle
- logminer
categories:
- Oracle职业直通车
---

## Oracle的重做日志
### 为什么需要redo log
- 内存中数据修改后，不必立即更新到磁盘  ---效率
- 由日志完成数据的保护目的  ---效率
- 其它副产品
  - 数据恢复（备份集+归档日志）
  - 数据同步（DG ，streams，gg）
  - 日志挖掘

<!-- more -->
### REDO的机制
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_01.png)
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_02.png)
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_03.png)
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_04.png)
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_05.png)

### SCN--System Commit Number
SCN数据库中顺序增长的一个数字，用来精确的区别操作的先后顺序，比如 commit, rollback or checkpoint
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_06.png)

### 日志文件
#### 日志文件使用操作系统块大小
- 通常是 512 bytes
- 格式依赖于操作系统和Oracle版本

#### Redo日志组成
- 数据头
- Redo record
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_07.png)

#### Redo Record
- 一个Redo Record记录包括Redo记录头和一个或多个改变向量
- 每个Redo Record包含每个原子改变的undo和redo
- 某些改动不需要undo(临时表，直接加载...)

单行插入的redo record
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_08.png)

多行插入的redo record
![](http://oligvdnzp.bkt.clouddn.com/1101_oracle_redo_09.png)

临时表的数据块不产生REDO

#### Oracle Dump Redo Log File
``` perl
SQL> select FORCE_LOGGING,SUPPLEMENTAL_LOG_DATA_MIN from v$database;

FOR SUPPLEME
--- --------
YES  NO

SQL> alter database add supplemental log data;

Database altered.

SQL> select GROUP#,STATUS from v$log;

    GROUP# STATUS
---------- ----------------
         1 CURRENT
         2 INACTIVE
         3 INACTIVE
         4 INACTIVE

SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    6001185

# 新开一个窗口用lyj登陆
SQL> conn lyj/lyj

SQL> insert into test02 values('lyj',100,sysdate);

1 row created.

SQL> commit;

Commit complete.


# 回到一开始的窗口
SQL> select current_scn from v$database;

CURRENT_SCN
-----------
    6001195

SQL> select group#,member from v$logfile;

    GROUP# MEMBER
---------- --------------------------------------------------
         3 /u01/app/oracle/oradata/orcl/redo03.log
         2 /u01/app/oracle/oradata/orcl/redo02.log
         1 /u01/app/oracle/oradata/orcl/redo01.log
         4 /u01/app/oracle/oradata/orcl/redo04.log

SQL> alter system dump logfile '/u01/app/oracle/oradata/orcl/redo01.log' scn min 6001185 scn max 6001195;

System altered.

SQL> select VALUE from v$diag_info where name='Default Trace File';

VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_5838.trc     # 这个就是一会dump出来的logfile trace了

# 也可以通过下面方法找到对应的trace文件
oradebug setmypid
oradebug tracefile_name

# 基于记录块位置导出redo log trace
select distinct dbms_rowid.rowid_relative_fno(rowid) rel_fno,dbms_rowid.rowid_block_number(rowid)blockno from lyj.test02;

   REL_FNO    BLOCKNO
---------- ----------
        14        643
        14        644

SQL> col MEMBER for a50
SQL> select a.group#,a.status,b.member from v$log a,v$logfile b where a.group#=b.group#;

    GROUP# STATUS           MEMBER
---------- ---------------- --------------------------------------------------
         3 CURRENT          /u01/app/oracle/oradata/orcl/redo03.log
         2 INACTIVE         /u01/app/oracle/oradata/orcl/redo02.log
         1 INACTIVE         /u01/app/oracle/oradata/orcl/redo01.log
         4 INACTIVE         /u01/app/oracle/oradata/orcl/redo04.log

alter system dump logfile '/u01/app/oracle/oradata/orcl/redo01.log' dba min 14 643 dba max 14 644;

# chr()函数将ASCII码转换为字符，可用来验证dump出来的内容值
select chr(to_number('6c','xx')) from dual;
```

下面是dump出来redo log
![](http://oligvdnzp.bkt.clouddn.com/1102_oracle_redo_01.png)
![](http://oligvdnzp.bkt.clouddn.com/1102_oracle_redo_02.png)

#### 重做向量--redo vector
- redo vector redo log存放数据库修改数据的信息。
- redo vector 里面保存着每个数据块的修改，而不是数据块修改的SQL语句

>redo里面记录的信息是物理数据块改变的向量，不是SQL语句

#### 日志文件镜像
为了保证日志文件的安全，应该给每个日志文件增加镜像文件，LGWR将向每组多个日志文件同时写入数据
``` perl
alter database add logfile member '/u01/app/oracle/oradata/orcl/redo05.log' to group 1;
alter database add logfile member '/u01/app/oracle/oradata/orcl/redo06.log' to group 2;
alter database add logfile member '/u01/app/oracle/oradata/orcl/redo07.log' to group 3;
alter database add logfile member '/u01/app/oracle/oradata/orcl/redo08.log' to group 4;

SQL> select group#,member from v$logfile order by 1;

    GROUP# MEMBER
---------- --------------------------------------------------
         1 /u01/app/oracle/oradata/orcl/redo01.log
         1 /u01/app/oracle/oradata/orcl/redo05.log
         2 /u01/app/oracle/oradata/orcl/redo02.log
         2 /u01/app/oracle/oradata/orcl/redo06.log
         3 /u01/app/oracle/oradata/orcl/redo07.log
         3 /u01/app/oracle/oradata/orcl/redo03.log
         4 /u01/app/oracle/oradata/orcl/redo08.log
         4 /u01/app/oracle/oradata/orcl/redo04.log
```

## 日志挖掘-logminer

### 日志挖掘工具包dbms_logmnr
- 可以基于日志文件分析（一个或者多个）
- 可以基于时间段分析
- 可以基于SCN分析

### logminer日志挖掘
``` perl
col MEMBER for a50
select a.group#,a.status,b.member from v$log a,v$logfile b where a.group#=b.group# order by 1;

    GROUP# STATUS           MEMBER
---------- ---------------- --------------------------------------------------
         1 CURRENT          /u01/app/oracle/oradata/orcl/redo01.log
         1 CURRENT          /u01/app/oracle/oradata/orcl/redo05.log
         2 INACTIVE         /u01/app/oracle/oradata/orcl/redo02.log
         2 INACTIVE         /u01/app/oracle/oradata/orcl/redo06.log
         3 INACTIVE         /u01/app/oracle/oradata/orcl/redo07.log
         3 INACTIVE         /u01/app/oracle/oradata/orcl/redo03.log
         4 INACTIVE         /u01/app/oracle/oradata/orcl/redo08.log
         4 INACTIVE         /u01/app/oracle/oradata/orcl/redo04.log

select dbms_flashback.get_system_change_number from dual;

GET_SYSTEM_CHANGE_NUMBER
------------------------
                 6008670

insert into lyj.test02 values('lyj',101,sysdate);
commit;
select dbms_flashback.get_system_change_number from dual;

GET_SYSTEM_CHANGE_NUMBER
------------------------
                 6008691

# 第一个logfile
exec dbms_logmnr.add_logfile(logfilename=>'/u01/app/oracle/oradata/orcl/redo01.log', options=>dbms_logmnr.new);
# 追加logfile
exec dbms_logmnr.add_logfile(logfilename=>'/u01/app/oracle/oradata/orcl/redo02.log', options=>dbms_logmnr.addfile);

exec dbms_logmnr.start_logmnr(options=>dbms_logmnr.dict_from_online_catalog,startscn=>6008670,endscn=>6008691);
exec dbms_logmnr.start_logmnr(options=>dbms_logmnr.dict_from_online_catalog,startscn=>&startscn,endscn=>&endscn);


# v$logmnr_contents里的内容是实时从加载的logfile中抽取的
col OPERATION for a15
col SQL_REDO for a50
col SQL_UNDO for a50
select operation,sql_redo,sql_undo from v$logmnr_contents;

OPERATION       SQL_REDO                                           SQL_UNDO
--------------- -------------------------------------------------- --------------------------------------------------
START           set transaction read write;
INSERT          insert into "LYJ"."TEST02"("USERNAME","USER_ID","C delete from "LYJ"."TEST02" where "USERNAME" = 'lyj'
                REATED") values ('lyj','101',TO_DATE('2017-11-02 1  and "USER_ID" = '101' and "CREATED" = TO_DATE('2
                5:51:24', 'yyyy-mm-dd hh24:mi:ss'));               017-11-02 15:51:24', 'yyyy-mm-dd hh24:mi:ss') and
                                                                   ROWID = 'AAAWG0AAOAAAAKFAAA';

COMMIT          commit;

# 查看更详细的信息
select username,session_info,operation,sql_redo,sql_undo from v$logmnr_contents;

#关闭logmnr
exec dbms_logmnr.end_logmnr;

# 注意需要开启附加日志
alter database add supplemental log data;
```

## Oracle的回滚段
### undo和redo
- undo 用于撤销修改的操作（事务回滚）
- redo 用于讲数据的修改重演一遍（恢复）

### UNDO的目的
- 事务的回滚
- 实例的恢复
- 提供查询的一致性读

### 回滚段和事务的联系
``` perl
conn / as sysdba
grant select on v_$transaction to lyj;

conn lyj/lyj
insert into t values(3,'jz');

select xid,xidusn,xidslot,xidsqn,ubafil,ubablk,ubasqn from v$transaction;

XID                  XIDUSN    XIDSLOT     XIDSQN     UBAFIL     UBABLK     UBASQN
---------------- ---------- ---------- ---------- ---------- ---------- ----------
0E00160067080000         14         22       2151          7      20147        600

SQL> select to_number('0E','xxxx'),to_number('0016','xxxx'),to_number('0867','xxxxxx') from dual;

TO_NUMBER('0E','XXXX') TO_NUMBER('0016','XXXX') TO_NUMBER('0867','XXXXXX')
---------------------- ------------------------ --------------------------
                    14                       22                       2151
# XID 事务号，事务唯一标识
# XIDUSN  回滚段
# XIDSLOT 回滚段槽
# XIDSQN  回滚段序列号
# UBAFIL  回滚段所在的文件号
# UBABLK  回滚段所在的文件号的块
# UBASQN  回滚段所在的文件号的序列号
```

### 事务回滚机制
![](http://oligvdnzp.bkt.clouddn.com/1103_oracle_undo_01.png)

### 回滚段的逻辑结构
![](http://oligvdnzp.bkt.clouddn.com/1103_oracle_undo_02.png)

### 回滚段的空间使用机制
#### 增长
![](http://oligvdnzp.bkt.clouddn.com/1103_oracle_undo_03.png)

#### 回收
![](http://oligvdnzp.bkt.clouddn.com/1103_oracle_undo_04.png)

### 一致性读
![](http://oligvdnzp.bkt.clouddn.com/1103_oracle_undo_05.png)
![](http://oligvdnzp.bkt.clouddn.com/1103_oracle_undo_06.png)

### UNDO的相关参数
``` perl
SQL> show parameter undo

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_management                      string      AUTO
undo_retention                       integer     900
undo_tablespace                      string      UNDOTBS2

# 注意：undo_retention是一个动态调整的参数，同时，Oracle无法保证在这个保留时间内的UNO数据不被覆盖
# 当UNDO空间不足时，Oracle将覆盖即使未过保留期的数据以释放空间。

# 强制保留undo_retention时间内的数据
alter tablespace UNDOTBS2 retention guarantee;
alter tablespace UNDOTBS2 retention noguarantee;
```

### UNDO的相关视图
``` perl
# v$rollstat  v$rollname
set line 150
set pagesize 9999
select a.usn,b.name,extents,rssize,shrinks,wraps,extends,status
  from v$rollstat a,v$rollname b
  where a.usn=b.usn;

       USN NAME                              EXTENTS     RSSIZE    SHRINKS      WRAPS    EXTENDS STATUS
---------- ------------------------------ ---------- ---------- ---------- ---------- ---------- ---------------
         0 SYSTEM                                  6     385024          0          0          0 ONLINE
        11 _SYSSMU11_1119644918$                  16   14802944          6         30         16 ONLINE
        12 _SYSSMU12_2617804053$                  13   11657216          5         28         12 ONLINE
        13 _SYSSMU13_142672783$                   14   12705792          4         31         16 ONLINE
        14 _SYSSMU14_3304230682$                   9    7462912          9         47         40 ONLINE
        15 _SYSSMU15_1030784191$                  28   27385856          7         54         38 ONLINE
        16 _SYSSMU16_2346225559$                  29   28434432         14         51         35 ONLINE
        17 _SYSSMU17_4024467071$                  13   11657216         17         29         25 ONLINE
        18 _SYSSMU18_2055119062$                  12   10608640          6         26         12 ONLINE
        19 _SYSSMU19_3736672493$                  24   23191552          5         41         16 ONLINE
        20 _SYSSMU20_598797340$                   23   22142976          8         39         25 ONLINE

# v$undostat: 视图中保留4天的数据，每次快照10分钟，再早的数据保留在DBA_HIST_UNDOSTAT视图中
select to_char(begin_time,'hh24:mi'),to_char(end_time,'hh24:mi'),
       undoblks,txncount,maxquerylen,maxqueryid,maxconcurrency,
       ssolderrcnt,nospaceerrcnt,activeblks,tuned_undoretention
   from v$undostat;
TO_CH TO_CH   UNDOBLKS   TXNCOUNT MAXQUERYLEN MAXQUERYID    MAXCONCURRENCY SSOLDERRCNT NOSPACEERRCNT ACTIVEBLKS TUNED_UNDORETENTION
----- ----- ---------- ---------- ----------- ------------- -------------- ----------- ------------- ---------- -------------------
18:12 18:13          0          0         488 0rc4km05kgzb9              0           0             0        160              273926
18:02 18:12          8        250         488 0rc4km05kgzb9              1           0             0        160              273926
17:52 18:02         95         75        1091 0rc4km05kgzb9              3           0             0        288              273449
17:42 17:52          2         14         491 0rc4km05kgzb9              2           0             0        288              276548
17:32 17:42          2         65        1096 0rc4km05kgzb9              2           0             0        288              275981
17:22 17:32          0          4         495 0rc4km05kgzb9              1           0             0        288              275412
......
```

## 本课作业

### 用图示展示Oracle更新一条数据然后提交的整个经过（包括undo,redo,后台进程)，并辅助语言描述。
``` perl
# 示例语句
update emp set sal=2000 where id=1;
commit;
```

数据块(id=1)从磁盘中被读入内存（除非它已经在内存中），同时undo数据块也被读取到内存中（除非它已经在内存中），回滚段块保留了数据前映像(1000)，由于回滚段的数据被修改了，因此一条redo信息记录了undo数据块的修改操作
![](http://oligvdnzp.bkt.clouddn.com/1102_oracle_redo_03.png)

数据块中原数据被修改(2000)，由于数据发生改变，因此一条redo信息记录了数据块的修改操作
![](http://oligvdnzp.bkt.clouddn.com/1102_oracle_redo_04.png)

commit执行后，一个commit的标识被记录在redo日志中，LGWR进程将redo log buffer中的内容写到redo log file中
![](http://oligvdnzp.bkt.clouddn.com/1102_oracle_redo_05.png)


### 演示一个UPDATE操作，从相关视图中查询产生的UNDO和REDO大小，给出操作过程。
``` perl
grant select on v_$mystat to lyj;
grant select on v_$statname to lyj;

conn lyj/lyj

select b.name,a.value from v$mystat a,v$statname b 
  where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              1600
undo change vector size                                                 188

SQL> update test03 set id=1000 where id<8;

7 rows updated.

SQL> commit;

Commit complete.

select b.name,a.value from v$mystat a,v$statname b 
  where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              4000
undo change vector size                                                1044

产生的REDO大小为(4000-1600)  UNDO大小为(1044-188) 
```


### 比较操作相同的数据，INSERT,DELETE和UPDATE各自产生的UNDO和REDO的大小，给出一个对比的列表，并说明导致这种差异的原因。
分别执行insert，delete，update语句后，产生的undo和redo的大小如图所示
![](http://oligvdnzp.bkt.clouddn.com/1102_oracle_redo_07.png)
结论：
1. insert产生的redo大小是由数据插入的redo和undo(回滚)产生的redo两部分组成，回滚是delete操作，产生的undo最少，相应的undo产生的redo就也少
2. delete产生的redo的大小是由数据删除和数据回滚的redo两部分组成，删除是delete操作，产生的redo最小，回滚是insert操作，相对产生的undo会多一些
3. update产生的redo和undo都是对数据的修改，产生的redo(含undo产生的redo)相比insert和delete操作产生的redo就要多

``` perl
SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              8076
undo change vector size                                                1720

SQL> 
SQL> insert into t values (2);

1 row created.

SQL> commit;

Commit complete.

SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              8724
undo change vector size                                                1856

SQL> 
SQL> update t set id=6 where id=1;

1 row updated.

SQL> commit;

Commit complete.

SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              9456
undo change vector size                                                2040

SQL> 
SQL> delete from t where id=6;

1 row deleted.

SQL> commit;

Commit complete.

SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                             10172
undo change vector size                                                2236
```

### DDL操作是否产生UNDO和REDO，用示例说明，并说明原因。
DDL操作产生UNDO和REDO，原因是：
1. DDL操作产生的redo是因为DDL修改的字典表和一些段头信息产生的redo
2. DDL操作所产生的undo量视乎其所要维护数据字典的操作类型和操作量。DDL执行失败也产生少量undo，因为执行少量递归操作后，Oracle发现所要drop的对象并不存在，将会rollback之前的"部分"递归dml操作

``` perl
# 当t表存在时
SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                             10172
undo change vector size                                                2236

SQL> drop table t purge;

Table dropped.

SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                             20068
undo change vector size                                                5716

# 当T表不存在时
SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                             21232
undo change vector size                                                5956

SQL> drop table t purge;
drop table t purge
           *
ERROR at line 1:
ORA-00942: table or view does not exist


SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                             22396
undo change vector size                                                6196
```

### 通过一个示例，演示利用logminer，恢复delete误删除操作的数据。
``` perl
SQL> show user
USER is "LYJ"

SQL> select * from t;

        ID NAME
---------- ----------
         1 lyj
         2 zc
SQL> delete from t where id=2;

1 row deleted.

SQL> commit;

Commit complete.

SQL> conn / as sysdba
Connected.

# 记录下删除后大致的系统时间
SQL> select sysdate from dual;

SYSDATE
-------------------
2017-11-03 10:26:11

SQL> col member for a50
SQL> select b.group#,b.member,a.status from v$log a,v$logfile b where a.group#=b.group# order by 1;

    GROUP# MEMBER                                             STATUS
---------- -------------------------------------------------- ----------------
         1 /u01/app/oracle/oradata/orcl/redo01.log            INACTIVE
         1 /u01/app/oracle/oradata/orcl/redo05.log            INACTIVE
         2 /u01/app/oracle/oradata/orcl/redo02.log            CURRENT
         2 /u01/app/oracle/oradata/orcl/redo06.log            CURRENT
         3 /u01/app/oracle/oradata/orcl/redo07.log            INACTIVE
         3 /u01/app/oracle/oradata/orcl/redo03.log            INACTIVE
         4 /u01/app/oracle/oradata/orcl/redo08.log            INACTIVE
         4 /u01/app/oracle/oradata/orcl/redo04.log            INACTIVE

# 加载当前的日志组文件
exec dbms_logmnr.add_logfile(logfilename=>'/u01/app/oracle/oradata/orcl/redo02.log', options=>dbms_logmnr.new);

# 如果不确定日志组是否已切换可以添加其他日志组文件
exec dbms_logmnr.add_logfile(logfilename=>'/u01/app/oracle/oradata/orcl/redo02.log', options=>dbms_logmnr.addfile);

exec dbms_logmnr.start_logmnr(options=>dbms_logmnr.dict_from_online_catalog,starttime=>to_date('2017-11-03 10:24:00','yyyy-mm-dd hh24:mi:ss'),endtime=>to_date('2017-11-03 10:26:11','yyyy-mm-dd hh24:mi:ss')); 

# 查看logminer挖掘内容
set line 150
set pagesize 9999
col operation for a15
col sql_redo for a60
col sql_undo for a60
select operation,sql_redo,sql_undo from v$logmnr_contents where seg_owner='LYJ' and operation='DELETE';

OPERATION       SQL_REDO                                                     SQL_UNDO
--------------- ------------------------------------------------------------ ------------------------------------------------------------
DELETE          delete from "LYJ"."T" where "ID" = '2' and "NAME" = 'zc' and insert into "LYJ"."T"("ID","NAME") values ('2','zc');
                 ROWID = 'AAAWH7AAOAAAAKOAAB';

#关闭logmnr
exec dbms_logmnr.end_logmnr;

# SQL_UNDO里就是恢复delete误删除操作的语句
insert into "LYJ"."T"("ID","NAME") values ('2','zc');
commit;
```

### 示例演示回滚是否产生REDO日志。
回滚产生REDO日志，示例如下：
``` perl
SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              1604
undo change vector size                                                 188

SQL> update t set id=6 where id=2;

1 row updated.

SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              2172
undo change vector size                                                 372

SQL> rollback;

Rollback complete.

SQL> select b.name,a.value from v$mystat a,v$statname b 
  2    where a.statistic#=b.statistic# and (b.name='redo size' or b.name='undo change vector size');

NAME                                                                  VALUE
---------------------------------------------------------------- ----------
redo size                                                              2700
undo change vector size                                                 372
```

### 示例演示为数据库创建一个新的UNDO表空间。
``` perl
SQL> create undo tablespace undotablespace datafile '/u01/app/oracle/oradata/orcl/undotablespace.dbf' size 100m autoextend on next 100m;

Tablespace created.

SQL> alter system set undo_tablespace='undotablespace';

System altered.
```

### 示例分别说明什么是consistent read和current read?
从内存中读取数据块，包括`consistent read`和`current read`，其中`current read`又叫做`db block gets`，逻辑读 = `db block gets + consistent read`。
- `consistent gets`：查询最终出结果的数据是查询开始的时间点的，若存在block在查询开始后发生了变化的情况，则会从回滚段中读`before image`数据，这就是一致性读。<font color=red>一致性读并非总要去读取回滚段。</font>
- `db block gets`：current read, 不管这个块上的数据是否存在`before image`，也就是说不管是否存在回滚中数据，只看见当前最新块的数据。比如DML的时候不需要看见别人更改前的数据，而是看见正在更改的，同时操作相同数据会被lock住。`current read`一次操作（查询）中的数据可能不在同一个时间点上。

``` perl
SQL> set autot trace stat
SQL> select * from test01;

68316 rows selected.


Statistics
----------------------------------------------------------
         85  recursive calls
          0  db block gets
       5663  consistent gets
       1009  physical reads
          0  redo size
    7935177  bytes sent via SQL*Net to client
      50614  bytes received via SQL*Net from client
       4556  SQL*Net roundtrips to/from client
         12  sorts (memory)
          0  sorts (disk)
      68316  rows processed

SQL> update test01 set object_id=object_id+10 where object_id<500;

10 rows updated.


Statistics
----------------------------------------------------------
         89  recursive calls
         12  db block gets      # current read
       1155  consistent gets
         13  physical reads
       2980  redo size
        842  bytes sent via SQL*Net to client
        813  bytes received via SQL*Net from client
          3  SQL*Net roundtrips to/from client
         24  sorts (memory)
          0  sorts (disk)
         10  rows processed
```

### 示例演示一个导致ora-01555错误的场景。
``` perl
# 如果用undotest1表空间，使用以下语句删除
drop tablespace undotest1 including contents and datafiles;

# 创建一个新的undo表空间，undo文件为1M，并且不能自动扩展
create undo tablespace undotest1 datafile '/u01/app/oracle/oradata/orcl/undotest01.dbf' size 1m;

alter system set undo_tablespace='undotest1';
alter system set undo_retention=1;

show parameter undo

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_management                      string      AUTO
undo_retention                       integer     1
undo_tablespace                      string      undotest1

# 会话1，定义并打开一个游标
conn lyj/lyj
var c1 refcursor
begin
open :c1 for select * from t;
end;
/

# 会话2，做一个大量的update操作，将T表中的所有ID字段更新了10万次
conn lyj/lyj
begin
for i in 1..100000 loop
update t set id=i;
commit;
end loop;
end;
/

# 会话1，打开刚才定义的游标C1，会得到如下ORA-01555的错误
SQL> print :c1
ERROR:
ORA-01555: snapshot too old: rollback segment number 22 with name
"_SYSSMU22_656159337$" too small

```