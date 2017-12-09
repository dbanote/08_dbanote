---
title: Oracle职业直通车-05 数据字典&数据库的备份和恢复
date: 2015-03-23
tags:
- oracle
categories:
- Oracle职业直通车
---

### 创建一张表，在表上创建一个索引，查询表，索引各自分配了多少个extents,多少个数据块以及共占空间大小(bytes)
``` perl
conn lyj/lyj
drop table t purge;
create table t as select * from all_objects;
create index ind_t_oid on t(object_id);

col segment_name for a20
select segment_name,extents,blocks,bytes/1024/1024 size_m
  from user_segments
  where segment_name='T';

SEGMENT_NAME            EXTENTS     BLOCKS     SIZE_M
-------------------- ---------- ---------- ----------
T                            23       1024          8

select segment_name,extents,blocks,bytes/1024/1024 size_m
  from user_segments
  where segment_name='IND_T_OID';

SEGMENT_NAME            EXTENTS     BLOCKS     SIZE_M
-------------------- ---------- ---------- ----------
IND_T_OID                    17        256          2
```

<!-- more -->
### 创建一个分区表T，创建2个分区P1,P2，并且把每个分区放在不同的表空间上，从视图中查到表和分区的信息，以及每个分区所在表空间的信息。注意观察当前表T所在的表空间是什么？给出原因。
``` perl
conn / as sysdba
create tablespace test1 datafile '/u01/app/oracle/oradata/orcl/test01.dbf' size 100m;
create tablespace test2 datafile '/u01/app/oracle/oradata/orcl/test02.dbf' size 100m;

conn lyj/lyj
drop table t purge;
create table t (object_id number,object_name varchar2(30))
partition by range(object_id)
(
  partition p1 values less than(50000) tablespace test1,
  partition p2 values less than(maxvalue) tablespace test2
);

insert into t select object_id,object_name from all_objects;
commit;

create index ind_t_id on t(object_id) local;

select TABLE_NAME,TABLESPACE_NAME,PARTITIONED from user_tables where TABLE_NAME='T';

TABLE_NAME                     TABLESPACE_NAME                PAR
------------------------------ ------------------------------ ---
T                                                             YES

select table_name,partition_name,tablespace_name
from user_tab_partitions
where table_name='T';

TABLE_NAME                     PARTITION_NAME                 TABLESPACE_NAME
------------------------------ ------------------------------ ------------------------------
T                              P1                             TEST1
T                              P2                             TEST2
```

### 查看当前用户下有哪些对象？有哪些表？有哪些索引？
``` perl
select object_name from user_objects;
select table_name from user_tables;
select index_name from user_indexes;
```

### 分别演示数据库打开的三个阶段(nomount,mount,open)，并分别从视图中查到它的状态。
``` perl
SQL> conn / as sysdba
Connected.
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup nomount
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             956301976 bytes
Database Buffers         3405774848 bytes
Redo Buffers               11661312 bytes
SQL> select status from v$instance;

STATUS
------------------------
STARTED

SQL> alter database mount;

Database altered.

SQL> select status from v$instance;

STATUS
------------
MOUNTED

SQL> alter database open;

Database altered.

SQL> select status from v$instance;

STATUS
------------
OPEN
```

### 发出一条sql语句，从视图中找到这条sql，同时找到这条SQL的执行时间。
``` perl
conn lyj/lyj
select count(*) from t;

# 新打开一个会话
conn / as sysdba
col sql_text for a40
select sql_text,
       to_char(cpu_time/1000000,'999,990.999999')  "CPU_TIME(S)",
       to_char(elapsed_time/1000000,'999,990.999999') "ELAPSED_TIME(S)"
from v$session n,v$sql l
where n.sql_id=l.sql_id and n.username='LYJ';

SQL_TEXT                                 CPU_TIME(S)     ELAPSED_TIME(S)
---------------------------------------- --------------- ---------------
select count(*) from t                          0.150000        0.152085

# CPU Time 指的是CPU在忙于执行当前任务的时间，并没有考虑等待时间，如IO等待，网络等待等，而Elapsed Time则是执行当前任务所花费的总时间，单位是微妙
```

### 演示用Rman对数据库做全备份和全库恢复的示例
``` perl
$ rman target / 

# 备份整个数据库
RMAN> backup database format '/oradata/rmanbackup/dbbak_%d_%T_%s_%U';

# 查看备份集
RMAN> list backup;
RMAN> list backupset;

# 恢复前可以在数据库中做一下DML操作，并摸拟数据文件损坏
conn lyj/lyj
create table test1 (id int);
insert into test1 select object_id from all_objects where rownum<10;
commit;
! rm -rf /u01/app/oracle/oradata/orcl/users01.dbf

# 出现故障
conn / as sysdba
alter system checkpoint;
alter system checkpoint
*
ERROR at line 1:
ORA-03113: end-of-file on communication channel
Process ID: 1600
Session ID: 355 Serial number: 11

# 在RMAN中恢复数据库
RMAN> startup mount
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open;

database opened
```

### 演示将某个表的数据闪回到历史某个时间点的示例，贴出整个操作过程（查询闪回）
``` perl
conn lyj/lyj
SQL> select sysdate,timestamp_to_scn(sysdate) scn from dual;

SYSDATE                    SCN
------------------- ----------
2016-10-20 11:06:32    5305382

SQL> select count(*) from flashback_test;

  COUNT(*)
----------
     68323

SQL> delete from flashback_test where OBJECT_ID<300;

4 rows deleted.

SQL> commit;

Commit complete.

SQL> select count(*) from flashback_test;

  COUNT(*)
----------
     68319

# 开启允许行迁移
SQL> alter table flashback_test enable row movement;

Table altered.

SQL> flashback table flashback_test to scn 5305382;

Flashback complete.

SQL> select count(*) from flashback_test;

  COUNT(*)
----------
     68323

SQL> select count(*) from flashback_test as of scn 5305382;

  COUNT(*)
----------
     68323

# 查询闪回
SQL> delete flashback_test where object_id<1000;

10 rows deleted.

SQL> commit;

Commit complete.

# 当前
SQL> select count(*) from flashback_test;

  COUNT(*)
----------
     68313

# 基于时间的闪回查询（5分钟前）
SQL> select count(*) from flashback_test as of timestamp sysdate-5/1440;

  COUNT(*)
----------
     68323

# 基于SCN号的闪回查询
SQL> select count(*) from flashback_test as of scn 5305382;

  COUNT(*)
----------
     68323
```

### 写出10条你认为DBA应该会使用的SQL语句
#### 查看隐藏参数
``` perl
set pagesize 9999
set line 180
col name for a50
col value for a10
col description for a70
select a.ksppinm name, b.ksppstvl value, a.ksppdesc description
  from x$ksppi a, x$ksppcv b
  where a.indx = b.indx and a.ksppinm like '_%&parameter%';
```

#### 查看表空间使用情况
``` perl
select
  total.tablespace_name,
  to_char(total.MB,'9,999,990.99') as Total_MB,
  to_char(total.MB-free.MB, '9,999,990.99') as Used_MB,
  to_char((1-free.MB/total.MB)*100, '990.00') ||'%' as "% Used"
from (select tablespace_name, sum(bytes)/1024/1024 as MB from dba_free_space group by tablespace_name) free,
     (select tablespace_name, sum(bytes)/1024/1024 as MB from dba_data_files group by tablespace_name) total
where free.tablespace_name=total.tablespace_name
union all
select tablespace_name,
  to_char(TABLESPACE_SIZE/1024/1024, '999,990.99') as Total_MB,
  to_char((TABLESPACE_SIZE-FREE_SPACE)/1024/1024, '999,990.99') as Used_MB,
  to_char(((TABLESPACE_SIZE-FREE_SPACE)/TABLESPACE_SIZE)*100,'990.00') ||'%' as "% Used"
from dba_temp_free_space order by 1;
```

#### 查看ASM使用情况
``` perl
select
  NAME,
  to_char(TOTAL_MB/1024,'999,990.99') TOTAL_GB,
  to_char((TOTAL_MB-USABLE_FILE_MB)/1024,'999,990.99') USED_GB,
  to_char(USABLE_FILE_MB/1024,'999,990.00') FREE_GB,
  to_char((TOTAL_MB-USABLE_FILE_MB)/TOTAL_MB*100,'990.00')||'%' "% USED"
from v$asm_diskgroup;
```

#### 查看数据库中的锁
``` perl
SELECT l.session_id sid,e.serial#,
       l.locked_mode,
       l.oracle_username,
       e.user#,
       l.os_user_name,
       e.machine,
       e.terminal,
       a.sql_text,
       a.action
  FROM v$sqlarea a, v$session e, v$locked_object l
WHERE l.session_id = e.sid
   AND e.prev_sql_addr = a.address
ORDER BY sid, e.serial#;
```

#### 查询正在进行表扫表的会话及执行的sql语句
``` perl
select vsl.sid,
       vsn.serial#,
       vsn.status,
       vsl.opname,
       vsl.target,
       ds.bytes/1024/1024 as "SEGMENT(MB)",
       dt.last_analyzed,
       vs.hash_value,
       vs.sql_text
from v$session_longops vsl, 
     v$session vsn, 
     dba_segments ds, 
     dba_tables dt,
     v$sql vs
where vsl.opname='Table Scan'
and vsl.sid=vsn.sid
and vsn.status='ACTIVE'
and vsl.target=ds.owner || '.' || ds.segment_name
and vsl.target=dt.owner || '.' || dt.table_name
and vsn.sql_hash_value=vs.hash_value
order by vsn.sid, vsn.serial#, vsl.target, vsn.status;
```

#### 查询系统内大于500M的表和索引的比例
``` perl
select dtb.owner,
       count(dtb.table_name) as tbs,
       count(didx.index_name) as idxs
from dba_tables dtb, 
     dba_indexes didx,
     dba_segments ds
where dtb.owner=didx.table_owner(+)
and dtb.table_name=didx.table_name(+)
and dtb.table_name=ds.segment_name
and ds.bytes/1024/1024>500
group by dtb.owner
order by tbs desc, idxs desc;
```

#### 查询系统内大于200M且没有索引的表
``` perl
select ds.owner,
       ds.segment_name,
       ds.bytes/1024/1024 as "SEGMENT(MB)",
       ds.segment_type
from  dba_segments ds
where ds.owner not in ('MDSYS','SYS','SYSTEM','OUTLN','WMSYS','XDB','SYSMAN','DBSNMP','EXFSYS')
and ds.segment_name not in (select didx.table_name from dba_indexes didx)
and ds.segment_type in ('TABLE','TABLE PARTITION')
and ds.bytes/1024/1024>200
order by ds.owner,ds.segment_name;
```

#### 查询系统非空闲等待事件
``` perl
select vsn.sid,
       vsn.serial#,
       vsw.event,
       vsw.wait_time
from v$session_wait vsw,
     v$session vsn
where vsw.sid=vsn.sid
and vsw.event not like 'SQL%'
and vsw.event not like 'rdbms%'
and vsw.event not like '%timer';
```

#### 定位系统内IO消耗最高的SQL语句
``` perl
select vs.hash_value,
       vs.sql_text,
       sum(vs.disk_reads) as R,
       sum(vs.direct_writes) AS W     
from v$sql vs
group by vs.hash_value, vs.sql_text
order by R desc,W desc
```

#### 定位系统内CPU消耗最高的SQL语句
``` perl
select vs.hash_value,
       vs.sql_text,
       sum(vs.cpu_time) as cpu
from v$sql vs
group by vs.hash_value, vs.sql_text
order by cpu desc
```

#### 查询硬解析
```
select d.plan_hash_value plan_hash_value ,
       d.execnt  execnt ,
       a.hash_value hash_value ,
       a.sql_text  sql_text
from v$sqltext a,
    (select plan_hash_value,
            hash_value,
            execnt 
     from (select c.plan_hash_value,
                  b.hash_value,
                  c.execnt,
                  rank() over(partition by c.plan_hash_value order by b.hash_value) as hashrank
           from v$sql b,
                (select count(*) as execnt,
                        plan_hash_value
                 from v$sql
                 where plan_hash_value <> 0
                 group by plan_hash_value
                 having count(*) > 10
                 order by count(*) desc) c
           where b.plan_hash_value = c.plan_hash_value
           group by c.plan_hash_value,b.hash_value,c.execnt)
     where hashrank<=3) d
where a.hash_value = d.hash_value
order by d.execnt desc,a.hash_value,a.piece
```

## 课堂笔记
``` perl
# 查看所有数据字典视图列表
select table_name from dict;
select table_name from dict where table_name like '%PRIVS%';

# 查看数据字典基表的方法-查看执行执行
set autot trace exp
select * from v$lock;
set autot off

# 常用的数据字典-静态视图
user_tables
user_tab_partitions
user_indexes
user_ind_partitions
user_segments
dba_data_files

# 常用的数据字典-动态视图
v$instance
v$database
v$log
v$logfile
v$session
v$sql
v$session_wait
v$lock
v$locked_object

create table t_part (id int) 
  partition by range(id)
  (
    partition p1 values less than(5),
    partition p2 values less than(10),
    partition pmax values less than(maxvalue)
  );

select * from t_part partition(pmax);
select table_name,partition_name,tablespace_name from user_tab_partitions;

alter table t_part move partition pmax tablespace test_ts;
create index ind_t_part on t_part(id) local tablespace users;

select INDEX_NAME,TABLESPACE_NAME from user_ind_partitions group by index_name,tablespace_name;

select i.table_name,p.index_name,p.partition_name,p.tablespace_name
from user_ind_partitions p,user_indexes i
where i.index_name=p.index_name;

select sum(size_m) all_size_m from
(select sum(bytes/1024/1024) size_m from dba_data_files
 union
 select sum(bytes/1024/1024) size_m from dba_temp_files);

grant select on v_$mystat to lyj;
conn lyj/lyj
select distinct sid from v$mystat;

select event,seconds_in_wait from v$session_wait where sid=355;

select sid,type,lmode,request,block from v$lock where type in ('TM','TX');
```

### 实例的恢复(crash recovery)
``` perl
# 实例恢复发生在那个阶段？
sql> startup nomount       # (读取spfle) ，没有实例恢复。
sql> mount database        # (读取控制文件)，没有实例恢复。
sql> alter database open   # (检查控制文件，数据文件头)，发生实例恢复。
# Oracle在打开数据库时(alter database open)，会检查每个文件头上的信息(SCN)，
# 并同控制文件中相应的信息(SCN)比较，如果不一致，则进行实例恢复。
```

#### 实例恢复的过程
1. 前滚 rolling forward
  读取状态为current和active状态的日志(redo log)，将发生crash时，没有来得及写到磁盘上的数据块，使用redo的信息来恢复。
2. 打开数据库(alter database open)
3. 回滚 rolling back
  将没有提交的事务进行回滚

### 介质恢复（Media recovery)
当发生以下情况时，实例恢复无效，需要进行介质恢复：
1. 数据文件丢失，损坏。
2. 在线日志文件(online redo)丢失，损坏。
3. 数据文件太旧 (比如从一个备份集中恢复过来的文件)
4. 文件太新（比如，其它所有的文件都是从备份中恢复过来的)

示例：
``` perl
SQL> select FILE_ID,FILE_NAME,TABLESPACE_NAME,STATUS from dba_data_files order by 1;

   FILE_ID FILE_NAME                                          TABLESPACE_NAME                STATUS
---------- -------------------------------------------------- ------------------------------ ---------
         1 /u01/app/oracle/oradata/orcl/system01.dbf          SYSTEM                         AVAILABLE
         2 /u01/app/oracle/oradata/orcl/sysaux01.dbf          SYSAUX                         AVAILABLE
         3 /u01/app/oracle/oradata/orcl/undotbs01.dbf         UNDOTBS1                       AVAILABLE
         4 /u01/app/oracle/oradata/orcl/users01.dbf           USERS                          AVAILABLE
         5 /u01/app/oracle/oradata/orcl/example01.dbf         EXAMPLE                        AVAILABLE
         6 /u01/app/oracle/oradata/orcl/users02.dbf           USERS                          AVAILABLE
         7 /u01/app/oracle/oradata/orcl/undotbs02.dbf         UNDOTBS2                       AVAILABLE
         8 /u01/app/oracle/oradata/orcl/test_ts01.dbf         TEST_TS                        AVAILABLE
         9 /u01/app/oracle/oradata/orcl/tbs_a.dbf             TBS_A                          AVAILABLE
        10 /u01/app/oracle/oradata/orcl/test01.dbf            TEST1                          AVAILABLE
        11 /u01/app/oracle/oradata/orcl/test02.dbf            TEST2                          AVAILABLE

SQL> alter database datafile 11 offline;

Database altered.

SQL> alter system checkpoint;

System altered.

SQL> alter database datafile 11 online;
alter database datafile 11 online
*
ERROR at line 1:
ORA-01113: file 11 needs media recovery
ORA-01110: data file 11: '/u01/app/oracle/oradata/orcl/test02.dbf'

SQL> recover datafile 11;
Media recovery complete.

SQL> alter database datafile 11 online;

Database altered.
```


## RAC
### RAC的目的
* 提供实例级别的冗余
* 提供更多的系统资源
* 增加更多的并行处理

### RAC的优点和缺点
#### 优点
* 提供系统冗余
* 更多的系统资源
* 业务分割处理

#### 缺点
* 内存共享与资源争用（Cache Fusion）
* 底层技术复杂，对DBA技术要求高

CRS（Cluster Ready Service）

## Data Guard
DG是一个容错方案，一般使用于数据量不是很大的数据库，对于OLTP非常适合。
OLAP数据量太大，一般选择关键数据创建DG，常规数据，选择其他方式备份。