---
title: Oracle职业直通车-06 Oracle的内存结构与后台进程
date: 2015-03-30
tags:
- oracle
categories:
- Oracle职业直通车
---

## Oracle实例
### SGA(System Global Area)
- SGA区包括Oracle实例需要的一系列内存组件，用于存放共享的数据信息和数据控制信息
  - Database Buffer Cache
  - Redo Log Buffer
  - Shared Pool  Library Cache、Data dictionary Cache
  - Large Pool
  - Streams Pool
  - Fixed SGA
- 这些内存信息被所有进程所共享(server process,background process)

``` perl
SQL> show sga

Total System Global Area 4375998464 bytes
Fixed Size- - - 2260328 bytes
Variable Size- -  838861464 bytes    # shared pool
Database Buffers-    3523215360 bytes
Redo Buffers- -    11661312 bytes
```

<!-- more -->
### Database Buffer Cache
Buffer cache里面存放这从磁盘上读到内存中的数据块，这些数据块可以被所有的会话访问，是全局共享的。
- Default pool 正常情况下，数据块存放的内存区域。
- Keep pool 这个区域（池）用于将一些数据始终固定在内存中，Default pool会根据一个过期算法（LRU）将过期的脏数据写到磁盘上。
- Recycle pool 存放一些不经常使用的数据块，避免这些数据块在Default pool池中占据空间。
- 2k，4k，16k 用于存放不是标准大小（8k）表空间的数据块信息
``` perl
# 创建一个16k块大小的表空间示例
show parameter cache_size

NAME- - - - -    TYPE-   VALUE
------------------------------------ ----------- ------------------------------
client_result_cache_size- -  big integer 0
db_16k_cache_size- - -   big integer 0
db_2k_cache_size- - -    big integer 0
db_32k_cache_size- - -   big integer 0
db_4k_cache_size- - -    big integer 0
db_8k_cache_size- - -    big integer 0
db_cache_size- - - - big integer 0
db_flash_cache_size- - - big integer 0
db_keep_cache_size- - -  big integer 0
db_recycle_cache_size- -     big integer 0

alter system set db_16k_cache_size=10M;
show parameter db_16k_cache_size
NAME- - - - -    TYPE-   VALUE
------------------------------------ ----------- ------------------------------
db_16k_cache_size- - -   big integer 16M

create tablespace ts_16 datafile '/u01/app/oracle/oradata/orcl/ts16k_01.dbf' size 100m blocksize 16k;
```

### buffer的概念
在database buffer cache当中，一个buffer指的是用来从磁盘中读入的数据块在内存中的存放位置，可以理解为`1 buffer=1 block`。

#### buffer（内存数据块）的3种状态
- unused
- clean
- dirty

#### Buffer 的2种模式
- Current mode     update/delete/insert
- Consistent mode  select

### Shared Pool - Server Result Cache
这部分内存中保留了SQL查询的结果集，这样对于后续的相同查询，Oracle无需重新加载数据块进行计算，直接使用现有的数据集。
- 由参数`RESULT_CACHE_MODE`设定
- 默认值`Manual`，需要使用`RESULT_CACHE hint`来启用。
- `FORCE`对所有`SELECT`操作生效（会消耗更多shared_pool空间）
``` perl
SQL> show parameter result_cache_mode

NAME- - - - -    TYPE-   VALUE
------------------------------------ ----------- ------------------------------
result_cache_mode- - -   string- MANUAL
```


### PGA(Program Global Area)
- 不同于SGA，PGA属于独占式内存区，它的数据和控制信息为某个会话所独有，当一个会话产生时，Oracle会为这个会话分配一个PGA内存区域。
- PGA属于单个的服务端进程或者后台进程，而实例级别说的PGA，通常指的是所有这些会话占用的PGA的总和，也就是由参数pga_aggregate_target设定的值。

### PGA & UGA
- PGA是一个进程占用的内存区域，可以理解为操作系统在一个进程启动时，为它分配的内存空间，是一个操作系统含义上的内存区。
- UGA(User global area)是一个会话含义的内存区，它保存着和会话相关的信息，比如会话登录信息，绑定变量的值，Pl/SQL包的参数信息等。
- UGA必须保证会话能够访问到UGA当中的数据，因此UGA的位置会随数据库连接方式的不同而不同。
  - 当连接使用的是专用模式(dedicated)时，因为会话是专有的，所以UGA属于PGA的一部分。
  - 当连接模式为MTS(Shared)时，由于会话可能会使用任意一个共享进程，因此这些进程必须能够同时访问UGA中的数据，此时Oracle会把UGA放在SGA区中（共享内存区域）。

## Oracle的进程
- 用户进程 user process
- 服务器进程 server process
- 实例后台进程 background process

### Oracle实例的后台进程--SMON
SMON 的主要工作：
- 数据库启动时的实例恢复，在RAC环境下，一个节点的SMON可以对另外一个节点做实例恢复
- 清理和释放临时段上的数据（排序，临时表...)
- 对于DMT(字典管理表空间）,SMON可以合并连续空闲的extent
- 维护回滚段的online,offline以及空间的回收

### Oracle实例的后台进程--PMON
#### Pmon后台进程负责的工作
- 进程异常终止
- 会话被杀掉
- 事务超过空闲时间
- 网络连接超时
- 将实例信息注册到监听器上
  - 手工注册 `alter system register;`

#### Pmon进程的清理工作
- 回滚未提交的事务，释放事务相关的资源
  - 重置undo数据块上的事务表的状态为inactive。
  - 释放事务产生的锁。
  - 从v$session中清除异常终止的会话ID。

### Oracle实例的后台进程--DBWn
- 负责将buffer cache中脏数据（修改过的数据）块写到磁盘上，由于数据块在磁盘上的位置不连续，这个过程会比LGWR比较耗时。
- DBWn触发条件
  - 当server process无法在buffer cache中无法找到可用的buffer时。
  - DBWn接到checkpoint的指令，将脏数据写到磁盘上。
- 可以通过设置多个DBWn进程加快脏数据写入磁盘的速度，参数是`DB_WRITER_PROCESSES`

### Oracle实例的后台进程--CKPT
#### checkpoint的目的和作用
- 减少数据库实例恢复的时间
- 让内存中的脏数据及时的写到磁盘上，CKPT进程通知DBWn进程开始将内存(buffer cache)中的脏数据写到磁盘的文件上
- 在安全关闭数据库时，保证所有提交的数据被写到磁盘上 
- CKPT负责更新文件头和控制文件的信息

#### checkpoint的触发
- database checkpoint
  - Consistent database shutdown
  - ALTER SYSTEM CHECKPOINT statement
  - Online redo log switch
  - ALTER DATABASE BEGIN BACKUP statement
- Tablespace and data file checkpoints
  - tablespace read only
  - tablespace offline normal
  - shrinking a data file
  - alter tablespace begin backup
- Incremental checkpoints

### Oracle实例的后台进程--LGWR
LGWR负责将log buffer中的数据顺序的写到磁盘上的online redo file,由于是顺序的写入，效率要比DBWn高很多。LGWR触发条件：
- 用户提交事务(commit)
- 日志切换
- 最后一次提交经过了3秒。
- redo log buffer容量达到1/3或者达到1M的redo数据。
- DBWn进程在把脏数据写入磁盘之前，必须保证这些脏数据对应的日志信息已经被写入磁盘，如果发现脏数据的日志信息没有写入磁盘，DBWn通知LGWR进程写日志信息，完成后继续将
脏数据写入磁盘。


### checkpoint和commit
commit 提交的事务修改的数据产生的日志，必须立即写到磁盘上。
- 刷新到磁盘上的日志中，包含未提交的日志。

checkpoint 将脏数据写到磁盘上，加快数据的恢复时间，为data buffer提供空闲空间
- 写到磁盘上的数据，并不一定是提交的。
- 数据写到磁盘之前，这部分数据对应的日志信息，必须提前写到磁盘上。
- 如果是alter system checkpoint,将只把commited的数据块写到磁盘。
- 安全关闭数据库，所有的commited的数据会写到磁盘上。

### Oracle实例的后台进程--ARCn
- 归档进程，当数据库处于归档模式时(archive mode)，ARCn负责将online redo file归档到目标存储位置，用于数据库的恢复，当在线日志切换时，会触发ARCn进程将在线日志文件归档。
- ARCn进程在data guard下，负责将日志向standby 服务器发送。

### Oracle实例的后台进程--其它
- ASMB--ASM实例的核心后台进程，负责管理ASM的存储。
- CJQ0--job任务协调进程，负责数据库中JOB的自动执行。
- Jnnn--job具体执行进程，接受CJQ0分发的job任务。
- MMAN--内存管理进程，负责内存的动态管理，分配和收回。
- Pnnn--并行执行子进程，接受并行协调进程分配的任务并执行。
- RBAL--ASM的rebalance进程，负责ASM数据的rebalance操作。
- RECO--分布式事务的恢复进程。
- Snnn--MTS下共享进程。


## 作业
### 用SQL计算某个表空间的大小及所包含对象的大小，给出SQL语句和结果。
``` perl
create tablespace ts_lyj datafile 
 '/u01/app/oracle/oradata/orcl/ts_lyj01.dbf' size 100m,
 '/u01/app/oracle/oradata/orcl/ts_lyj02.dbf' size 100m;

drop user lyj cascade;
create user lyj identified by lyj default tablespace ts_lyj;
grant connect,resource to lyj;

conn lyj/lyj
create table test01 as select * from all_objects;
create table test02 as select * from all_users;

conn / as sysdba
select TABLESPACE_NAME, sum(bytes)/1024/1024 tablespace_size_m from dba_data_files 
  where TABLESPACE_NAME='TS_LYJ' group by tablespace_name;

TABLESPACE_NAME                TABLESPACE_SIZE_M
------------------------------ -----------------
TS_LYJ                                       200


col OWNER for a10
col SEGMENT_NAME for a10
select TABLESPACE_NAME,sum(bytes)/1024/1024 object_size_m from dba_segments 
  where TABLESPACE_NAME='TS_LYJ' group by TABLESPACE_NAME;

TABLESPACE_NAME                OBJECT_SIZE_M
------------------------------ -------------
TS_LYJ                                8.0625

# 把以上两条语句拼成一条
select
  t.tablespace_name,
  t.tablespace_size_m,
  o.object_size_m
from (select TABLESPACE_NAME,sum(bytes)/1024/1024 tablespace_size_m 
         from dba_data_files where TABLESPACE_NAME='TS_LYJ' group by tablespace_name) t,
     (select TABLESPACE_NAME,sum(bytes)/1024/1024 object_size_m 
         from dba_segments where TABLESPACE_NAME='TS_LYJ' group by tablespace_name) o
where t.tablespace_name=o.tablespace_name;

TABLESPACE_NAME                TABLESPACE_SIZE_M OBJECT_SIZE_M
------------------------------ ----------------- -------------
TS_LYJ                                       200        8.0625
```

### 在告警日志中找到一条错误信息，并贴出来（如果没有，自己造出一条错误信息）。
``` perl
SQL> !rm /u01/app/oracle/oradata/orcl/ts_lyj01.dbf

# 重启数据库报错
SQL> startup force
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size- - - 2260328 bytes
Variable Size- -  939524760 bytes
Database Buffers-    3422552064 bytes
Redo Buffers- -    11661312 bytes
Database mounted.
ORA-01157: cannot identify/lock data file 13 - see DBWR trace file
ORA-01110: data file 13: '/u01/app/oracle/oradata/orcl/ts_lyj01.dbf'

# 找到alert日志所在目录
col VALUE for a80
select value from v$diag_info where NAME='Default Trace File';

VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_25472.trc   # 这个目录的地址就是alert日志所在的目录

# 查看alert日志
vi /u01/app/oracle/diag/rdbms/orcl/orcl/trace/alert_orcl.log 
#==============================================================================
ALTER DATABASE OPEN
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_dbw0_29603.trc:
ORA-01157: cannot identify/lock data file 13 - see DBWR trace file
ORA-01110: data file 13: '/u01/app/oracle/oradata/orcl/ts_lyj01.dbf'
ORA-27037: unable to obtain file status
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_29627.trc:
ORA-01157: cannot identify/lock data file 13 - see DBWR trace file
ORA-01110: data file 13: '/u01/app/oracle/oradata/orcl/ts_lyj01.dbf'
ORA-1157 signalled during: ALTER DATABASE OPEN...
#==============================================================================
```

### 学会使用官方文档，在网站`https://docs.oracle.com`上查找V$session的描述信息，查出`dbms_stats`包的信息，并截图贴出来。
![](http://oligvdnzp.bkt.clouddn.com/1031_oracle_01.png)
![](http://oligvdnzp.bkt.clouddn.com/1031_oracle_02.png)

### 画一个Oracle内存结构示意图，并给出文字描述。
![](http://oligvdnzp.bkt.clouddn.com/1031_oracle_01.gif)

System Global Area (SGA) | 文字描述
---------------|----------
Shared Pool | -  包括library cache，存储最近使用的SQL和pl/sql语句的信息<br/>- 通过“最近最少使用”`least recently used(LRU)`算法来管理<br/>- 还包括data dictionary cache在解析过程中会频繁的访问数据字典<br/> - 包括数据库文件、表、索引、列、用户、权限、和其它对象的信息<br/> - 在解析阶段，服务器进程寻找数据字典信息来解析对象的名字和访问权限<br/>- 把数据字典的信息加载到shared pool里面，来提高查询和DML(Data Manipulation Language)数据操纵语言语句的反应时间<br/>- 包括Server Result cache,sql 查询的结果集 
Database Buffer Cache  | - 数据库缓冲区快速缓存<br/> - 存储从数据文件提取出来的数据块的一个拷贝<br/> - 当提取数据或者修改数据的时候，能很大的提高性能<br/> - 通过LRU(Least Recently Used )算法来管理<br/> - DB_BLOCK_SIZE决定了主数据块的大小<br/> - Redo Log Buffer 重做日志缓冲区<br/> - 记录了对数据块的所有改变<br/> - 主要的目的是为了恢复数据库
Java Pool | - Java池<br/>- 用于解析java命令<br/>- 在安装和使用java的时候需要使用
Large Pool | - 大池<br/>- SGA中可选的一块内存区域<br/>- 减少了共享池（Shared Pool）的负担

System Global Area (SGA): 在实例启动的时候分配, 是数据库实例的基本组成部分
Program Global Area (PGA): 当服务器进程启动的时候分配
- 每个用户连接到Oracle数据库的内存区域；
- 当进程建立的时候单独分配；
- 当进程终止的时候释放；
- 只能用于一个进程,是私有的，不能够共享

### 画一个CKPT进程工作机制示意图，并给出文字描述。
>The checkpoint process (CKPT) updates the control file and data file headers with checkpoint information and signals DBWn to write blocks to disk. Checkpoint information includes the checkpoint position, SCN, location in online redo log to begin recovery, and so on. CKPT does not write data blocks to data files or redo blocks to online redo log files.

![](http://oligvdnzp.bkt.clouddn.com/1031_oracle_02.gif)
CKPT进程通知DBWn进程开始将内存(buffer cache)中的脏数据写到磁盘的文件上，并负责更新文件头和控制文件的信息。

### 列出你所用数据库运行的后台进程，并对每个进程做文字描述。
``` perl
ps -ef | grep ora_ | grep -v grep

oracle    7873     1  0 May27 ?        00:44:29 ora_pmon_bi
oracle    7876     1  0 May27 ?        00:33:30 ora_psp0_bi
oracle    7878     1  0 May27 ?        1-02:00:41 ora_vktm_bi
oracle    7882     1  0 May27 ?        00:06:12 ora_gen0_bi
oracle    7884     1  0 May27 ?        00:08:46 ora_diag_bi
oracle    7886     1  0 May27 ?        00:08:20 ora_dbrm_bi
oracle    7888     1  0 May27 ?        02:31:19 ora_dia0_bi
oracle    7890     1  0 May27 ?        00:07:32 ora_mman_bi
oracle    7892     1  0 May27 ?        01:20:11 ora_dbw0_bi
oracle    7894     1  0 May27 ?        01:18:40 ora_dbw1_bi
oracle    7896     1  0 May27 ?        01:21:50 ora_dbw2_bi
oracle    7898     1  0 May27 ?        02:14:10 ora_lgwr_bi
oracle    7900     1  0 May27 ?        01:05:55 ora_ckpt_bi
oracle    7902     1  0 May27 ?        00:44:00 ora_smon_bi
oracle    7904     1  0 May27 ?        00:02:26 ora_reco_bi
oracle    7906     1  0 May27 ?        00:07:04 ora_rbal_bi
oracle    7908     1  0 May27 ?        00:05:21 ora_asmb_bi
oracle    7910     1  0 May27 ?        02:00:37 ora_mmon_bi
oracle    7914     1  0 May27 ?        01:57:32 ora_mmnl_bi
oracle    7916     1  0 May27 ?        00:03:00 ora_d000_bi
oracle    7918     1  0 May27 ?        00:02:52 ora_s000_bi
oracle    7921     1  0 May27 ?        00:09:49 ora_mark_bi
oracle    8042     1  0 May27 ?        00:24:13 ora_arc0_bi
oracle    8061     1  0 May27 ?        00:06:31 ora_arc1_bi
oracle    8064     1  0 May27 ?        00:24:10 ora_arc2_bi
oracle    8066     1  0 May27 ?        00:24:05 ora_arc3_bi
oracle    8097     1  0 May27 ?        00:03:21 ora_qmnc_bi
oracle    8132     1  0 May27 ?        00:02:53 ora_emnc_bi
oracle    8248     1  0 May27 ?        00:02:32 ora_q001_bi
oracle    8270     1  0 May27 ?        00:03:10 ora_e000_bi
oracle    8272     1  0 May27 ?        00:03:14 ora_e001_bi
oracle    8276     1  0 May27 ?        00:03:13 ora_e002_bi
oracle    8282     1  0 May27 ?        00:03:13 ora_e003_bi
oracle    8284     1  0 May27 ?        00:31:24 ora_e004_bi
oracle    8307     1  0 May27 ?        02:40:31 ora_cjq0_bi
oracle    8789     1  0 May27 ?        00:09:11 ora_smco_bi
oracle   15439     1  0 Oct26 ?        00:00:12 ora_q002_bi
oracle   16793     1  0 04:37 ?        00:00:43 ora_o000_bi
oracle   24737     1  4 16:33 ?        00:01:46 ora_j001_bi
oracle   27582     1  0 16:59 ?        00:00:00 ora_w000_bi
oracle   28020     1  0 17:03 ?        00:00:01 ora_j000_bi
oracle   28135     1  0 17:05 ?        00:00:00 ora_w001_bi
```

Name | Description
---- | -----------
ARCn | Archiver Process, Copies the redo log files to archival storage when they are full or an online redo log switch occurs
ASMB | ASM Background Process, Communicates with the ASM instance, managing storage and providing statistics
CJQ0 | Job Queue Coordinator Process, Selects jobs that need to be run from the data dictionary and spawns job queue slave processes (Jnnn) to run the jobs
CKPT | Checkpoint Process, Signals DBWn at checkpoints and updates all the data files and control files of the database to indicate the most recent checkpoint
Dnnn | Dispatcher Process, Performs network communication in the shared server architecture
DBRM | Database Resource Manager Process, Sets resource plans and performs other tasks related to the Database Resource Manager
DBWn | Database Writer Process, Writes modified blocks from the database buffer cache to the data files
DIA0 | Diagnostic Process, Detects and resolves hangs and deadlocks
DIAG | Diagnostic Capture Process, Performs diagnostic dumps
EMNC | EMON Coordinator Process, Coordinates database event management and notifications
Ennn | EMON Slave Process, Performs database event management and notifications
GEN0 | General Task Execution Process, Performs required tasks including SQL and DML
Jnnn | Job Queue Slave Process, Executes jobs assigned by the job coordinator
LGWR | Log Writer Process, Writes redo entries to the online redo log
MARK | Mark AU for Resynchronization Coordinator Process, Marks ASM allocation units as stale following a missed write to an offline disk
MMAN | Memory Manager Process, Serves as the instance memory manager
MMNL | Manageability Monitor Lite Process, Performs tasks relating to manageability, including active session history sampling and metrics computation
MMON | Manageability Monitor Process, Performs or schedules many manageability tasks
Onnn | ASM Connection Pool Process, Maintains a connection to the ASM instance for metadata operations
PMON | Process Monitor, Monitors the other background processes and performs process recovery when a server or dispatcher process terminates abnormally
PSP0 | Process Spawner Process, Spawns Oracle background processes after initial instance startup
QMNC | AQ Coordinator Process, Monitors AQ
Qnnn | AQ Server Class Process, Performs various AQ-related background task for QMNC
RBAL | ASM Rebalance Master Process, Coordinates rebalance activity
RECO | Recoverer Process, Resolves distributed transactions that are pending because of a network or system failure in a distributed database
SMCO | Space Management Coordinator Process, Coordinates the execution of various space management tasks
SMON | System Monitor Process, Performs critical tasks such as instance recovery and dead transaction recovery, and maintenance tasks such as temporary space reclamation, data dictionary cleanup, and undo tablespace management
Snnn | Shared Server Process, Handles client requests in the shared server architecture
VKTM | Virtual Keeper of Time Process, Provides a wall clock time and reference time for time interval measurements
Wnnn | Space Management Slave Process, Performs various background space management tasks, including proactive space allocation and space reclamation

官方文档中全部的[Background Processes](https://docs.oracle.com/cd/E11882_01/server.112/e40402/bgprocesses.htm#REFRN104)说明

### 演示Oracle实例自动注册到监听器的例子，贴出输出或截图。
``` perl
[grid@bi ~]$ lsnrctl status

LSNRCTL for Linux: Version 11.2.0.3.0 - Production on 31-OCT-2017 18:05:11

Copyright (c) 1991, 2011, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 11.2.0.3.0 - Production
Start Date                23-OCT-2015 10:55:47
Uptime                    242 days 4 hr. 41 min. 31 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/11.2.0/grid/network/admin/listener.ora
Listener Log File         /u01/app/grid/diag/tnslsnr/bi/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=bi.localdomain)(PORT=1521)))
Services Summary...
Service "+ASM" has 1 instance(s).
  Instance "+ASM", status READY, has 1 handler(s) for this service...
Service "bi" has 1 instance(s).
  Instance "bi", status READY, has 1 handler(s) for this service...
Service "biXDB" has 1 instance(s).
  Instance "bi", status READY, has 1 handler(s) for this service...
The command completed successfully
[grid@bi ~]$ vi /u01/app/11.2.0/grid/network/admin/listener.ora
# listener.ora Network Configuration File: /u01/app/11.2.0/grid/network/admin/listener.ora
# Generated by Oracle configuration tools.

LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
      (ADDRESS = (PROTOCOL = TCP)(HOST = bi.localdomain)(PORT = 1521))
    )
  )

ADR_BASE_LISTENER = /u01/app/grid

ENABLE_GLOBAL_DYNAMIC_ENDPOINT_LISTENER=ON              # line added by Agent

#----ADDED BY TNSLSNR 23-OCT-2015 10:29:01---
SAVE_CONFIG_ON_STOP_LISTENER = ON
INBOUND_CONNECT_TIMEOUT_LISTENER = 0
#--------------------------------------------
```

### 将数据库设置为归档模式，手工产生一个归档日志，观察归档日志的产生，以及alert_xxx.ora日志中的信息，贴出演示过程和结果。
``` perl
SQL> archive log list;
Database log mode              No Archive Mode
Automatic archival             Disabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     40
Current log sequence           43
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> startup mount
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             939524760 bytes
Database Buffers         3422552064 bytes
Redo Buffers               11661312 bytes
Database mounted.
SQL> alter database archivelog;

Database altered.

SQL> alter database open;

Database altered.

SQL> alter system archive log current; 

System altered.

# alter日志中信息
#==============================================================================
Thread 1 advanced to log sequence 45 (LGWR switch)
  Current log# 1 seq# 45 mem# 0: /u01/app/oracle/oradata/orcl/redo01.log
Archived Log entry 297 added for thread 1 sequence 44 ID 0x588d1e5b dest 1:
#==============================================================================
```