---
title: Oracle特殊恢复原理与实战_08 Redo Architecture and Configuration
date: 2018-04-18
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## IMU的好处
IMU(In Memory Undo)顾名思义就是在内存中的undo，现在每次更改data block，Oracle不用去更改这个undo block（也不会生成相应的redo了），而是把undo信息缓存到IMU里去了，只有最后commit或者flush IMU时，这些undo 信息才会批量更新到undo block,并生成redo。可以避免Undo信息以前在Buffer Cache中的读写操作，从而可以进一步的减少Redo生成，同时可以大大减少以前的UNDO SEGMENT的操作。IMU中数据通过暂存、整理与收缩之后也可以写出到回滚段，这样的写出提供了有序、批量写的性能提升。

<!-- more -->

## IMU主要作用
1. 减少CR块-->在构造CR block时，不用像以前那样从`undo block`中获取`undo record`了，而是用共享池私有IMU区域里的信息来构造`cr block`，减少了`BUFFER CACEH`中`CBC LATCH`竞争。
2. 减少REDO日志条目数-->不再是每条DML语句一个`redo records`,而是每个事务一个`redo records--REDO RECORD`的产生会传到`LOG BUFFER`，会申请LATCH。
3. 减少LATCH-->首先因为减少`REDO RECORD`数目；其次用一个`IMU latch`代替`redo allocation latch`和`redo copy latch`这两个,也减少了LATCH争用。

## 在哪些场景下不会使用IMU特性
* 在RAC环境中不支持IMU
* 开启FLASHBACK DATABASE时会开启打开辅助日志，此时不能用IMU
* 事务过大--据说每个IMU Buffer的Private redo strand area大小大概是64KB（64位的Oracle版本是128KB），大事务不能用。比如一个事务，先有一条UPDATE，此时将REDO私有区域使用完了，此事务的其它DML语句，将自动使用非IMU模式
* 共享池太小时，ORACLE会自动不使用IMU
* 无法获取IMU LATCH时，将自动使用非IMU模式

## IMU下的redo产生过程
![](http://p2c0rtsgc.bkt.clouddn.com/0418_oracle_dsi_02.png)
``` perl
# 查看隐含参数_in_memory_undo的状态，默认开启IMU开启TRUE
SQL> @show_para
Enter value for p: in_memory_undo
old  12:     AND upper(i.ksppinm) LIKE upper('%&p%')
new  12:     AND upper(i.ksppinm) LIKE upper('%in_memory_undo%')

P_NAME                                           P_DESCRIPTION                                                  P_VALUE    ISDEFAULT ISMODIFIED ISADJ
------------------------------------------------ -------------------------------------------------------------- ---------- --------- ---------- -----
_in_memory_undo                                  Make in memory undo for top level transactions                 TRUE       TRUE      FALSE      FALSE

# 切换日志组，使CURRENT日志组为新的
SQL> alter system switch logfile;
SQL> col member for a50
SQL> select l.GROUP#,l.STATUS,f.MEMBER from v$log l,v$logfile f
      where l.GROUP#=f.GROUP#;

    GROUP# STATUS           MEMBER
---------- ---------------- --------------------------------------------------
         1 INACTIVE         /u01/app/oracle/oradata/orcl/redo01.log
         2 INACTIVE         /u01/app/oracle/oradata/orcl/redo02.log
         3 CURRENT          /u01/app/oracle/oradata/orcl/redo03.log

# 执行测试语句
create tablespace lyj_ts datafile '/u01/app/oracle/oradata/orcl/lyj_01.dbf' size 100m autoextend on next 50m maxsize unlimited;
create user lyj identified by lyj default tablespace lyj_ts;
grant dba to lyj;
conn lyj/lyj
drop table t1 purge;
create table t1 (id int,name varchar2(10));
insert into t1 values(1,'AAAAA');
insert into t1 values(2,'BBBBB');
commit;
update t1 set name='aaaaa' where name='AAAAA';
update t1 set name='bbbbb' where name='BBBBB';
commit;

select object_id from user_objects where object_name='T1';

 OBJECT_ID
----------
     15682

# 导出当前redo log
alter system dump logfile '/u01/app/oracle/oradata/orcl/redo03.log';

SQL> select * from v$diag_info where name='Default Trace File';

   INST_ID NAME
---------- ----------------------------------------------------------------
VALUE
------------------------------------------------------------------------------------------------------------------------------------------------------
         1 Default Trace File
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_26888.trc
```

查看分析dump的trace日志，一共4条REDO RECORD
第一个REDO RECORD, REDO RECORD头+CHANGE VECTOR组成（一个CV就是一个操作），由三个CV组成
``` perl
REDO RECORD - Thread:1 RBA: 0x00003f.00000039.0010 LEN: 0x0170 VLD: 0x01
SCN: 0x0000.000c6433 SUBSCN:  3 04/19/2018 10:59:35
# --------------------------------------------------------->REDO RECORD头
RBA: 0x00003f.00000039.0010 ==（由三部分组成：序列号，块号，偏移量或着叫第几个字节）
LEN: 0x01e8：一条日志的长度
VLD: 0x01 ：日志类型

# CHANGE VECTOR 1
CHANGE #1 TYP:0 CLS:17 AFN:3 DBA:0x00c00080 OBJ:4294967295 SCN:0x0000.000c6425 SEQ:1 OP:5.2 ENC:0 RBL:0
ktudh redo: slt: 0x0003 sqn: 0x000001db flg: 0x0012 siz: 112 fbi: 0
            uba: 0x00c0032d.0103.17    pxid:  0x0000.000.00000000
# -------------------------------------------------------->UNDO段头事务表
OP:5.2==>OPRATION CODE 向UNDO段的段头的事务表写事务信息，开始事务          
TYP:0（普通块）
CLS:17 （CLASS）超过16表示undo
AFN:3  (绝对文件号)
DBA:0x00c00080   数据块的地址
OBJ:4294967295   FFFFFFFF  表示undo
SCN:0x0000.000c6425  产生事务的时间

# CHANGE VECTOR 2
CHANGE #2 TYP:0 CLS:18 AFN:3 DBA:0x00c0032d OBJ:4294967295 SCN:0x0000.000c6424 SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 112 spc: 3810 flg: 0x0012 seq: 0x0103 rec: 0x17
            xid:  0x0001.003.000001db  
ktubl redo: slt: 3 rci: 0 opc: 11.1 [objn: 15682 objd: 15682 tsn: 5]
Undo type:  Regular undo        Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
             0x00000000  prev ctl uba: 0x00c0032d.0103.15 
prev ctl max cmt scn:  0x0000.000c56d6  prev tx cmt scn:  0x0000.000c56d7 
txn start scn:  0xffff.ffffffff  logon user: 34  prev brb: 12583727  prev bcl: 0 BuExt idx: 0 flg2: 0
KDO undo record:
KTB Redo 
op: 0x03  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: Z
KDO Op code: DRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400087  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 0(0x0)
# ―――――――――――――――――――――――――――――――――――――――――――――――――――――――>undo数据块头
OP:5.1 undo segment header

# CHANGE VECTOR 3
CHANGE #3 TYP:0 CLS:1 AFN:5 DBA:0x01400087 OBJ:15682 SCN:0x0000.000c6433 SEQ:2 OP:11.2 ENC:0 RBL:0
KTB Redo 
op: 0x01  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: F  xid:  0x0001.003.000001db    uba: 0x00c0032d.0103.17
KDO Op code: IRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400087  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 0(0x0) size/delt: 12
fb: --H-FL-- lb: 0x1  cc: 2
null: --
col  0: [ 2]  c1 02
col  1: [ 5]  41 41 41 41 41
# ----------------------------------------------------------->插入AAAAA
OP:11.2:Insert Row Piece 
```

第二个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x00003f.00000039.0180 LEN: 0x0100 VLD: 0x01
SCN: 0x0000.000c6433 SUBSCN:  4 04/19/2018 10:59:35
# --------------------------------------------------------->REDO RECORD头

CHANGE #1 TYP:0 CLS:18 AFN:3 DBA:0x00c0032d OBJ:4294967295 SCN:0x0000.000c6433 SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 68 spc: 3696 flg: 0x0022 seq: 0x0103 rec: 0x18
            xid:  0x0001.003.000001db  
ktubu redo: slt: 3 rci: 23 opc: 11.1 objn: 15682 objd: 15682 tsn: 5
Undo type:  Regular undo       Undo type:  Last buffer split:  No 
Tablespace Undo:  No 
             0x00000000
KDO undo record:
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c0032d.0103.17
KDO Op code: DRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400087  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 1(0x1)
# ―――――――――――――――――――――――――――――――――――――――――――――――――――――――>undo数据块头


CHANGE #2 TYP:0 CLS:1 AFN:5 DBA:0x01400087 OBJ:15682 SCN:0x0000.000c6433 SEQ:3 OP:11.2 ENC:0 RBL:0
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c0032d.0103.18
KDO Op code: IRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400087  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 1(0x1) size/delt: 12
fb: --H-FL-- lb: 0x1  cc: 2
null: --
col  0: [ 2]  c1 03
col  1: [ 5]  42 42 42 42 42
# ----------------------------------------------------------->插入BBBBB
```

第三个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x00003f.0000003a.0090 LEN: 0x0060 VLD: 0x01
SCN: 0x0000.000c6434 SUBSCN:  1 04/19/2018 10:59:35

CHANGE #1 TYP:0 CLS:17 AFN:3 DBA:0x00c00080 OBJ:4294967295 SCN:0x0000.000c6433 SEQ:1 OP:5.4 ENC:0 RBL:0
ktucm redo: slt: 0x0003 sqn: 0x000001db srt: 0 sta: 9 flg: 0x2 ktucf redo: uba: 0x00c0032d.0103.18 ext: 2 spc: 3626 fbi: 0
# ----------------------------------------------------------->提交
OP:5.4 提交作为单独的一条RECORD
```

第四个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x00003f.0000003b.0010 LEN: 0x0374 VLD: 0x0d
SCN: 0x0000.000c6436 SUBSCN:  1 04/19/2018 10:59:35
(LWN RBA: 0x00003f.0000003b.0010 LEN: 0002 NST: 0001 SCN: 0x0000.000c6435)
# --------------------------------------------------------->REDO RECORD头

CHANGE #1 TYP:2 CLS:1 AFN:5 DBA:0x01400087 OBJ:15682 SCN:0x0000.000c6434 SEQ:1 OP:11.19 ENC:0 RBL:0
KTB Redo 
op: 0x11  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: F  xid:  0x0008.011.000001d7    uba: 0x00c00bd1.00dd.22
Block cleanout record, scn:  0x0000.000c6435 ver: 0x01 opt: 0x02, entries follow...
  itli: 1  flg: 2  scn: 0x0000.000c6434
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 2 ckix: 216
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400087  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  61 61 61 61 61
# -------------------------------------------------------->数据修改成aaaaa
OP:11.19 数据修改操作

CHANGE #2 TYP:0 CLS:31 AFN:3 DBA:0x00c000f0 OBJ:4294967295 SCN:0x0000.000c6404 SEQ:1 OP:5.2 ENC:0 RBL:0
ktudh redo: slt: 0x0011 sqn: 0x000001d7 flg: 0x0012 siz: 168 fbi: 0
            uba: 0x00c00bd1.00dd.22    pxid:  0x0000.000.00000000
# -------------------------------------------------------->UNDO段头事务表
OP:5.2 事务开始

CHANGE #3 TYP:0 CLS:1 AFN:5 DBA:0x01400087 OBJ:15682 SCN:0x0000.000c6436 SEQ:1 OP:11.19 ENC:0 RBL:0
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c00bd1.00dd.23
Array Update of 1 rows: 
tabn: 0 slot: 1(0x1) flag: 0x2c lock: 2 ckix: 216
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400087  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  62 62 62 62 62
# -------------------------------------------------------->数据修改成bbbbb

CHANGE #4 TYP:0 CLS:31 AFN:3 DBA:0x00c000f0 OBJ:4294967295 SCN:0x0000.000c6436 SEQ:1 OP:5.4 ENC:0 RBL:0
ktucm redo: slt: 0x0011 sqn: 0x000001d7 srt: 0 sta: 9 flg: 0x2 ktucf redo: uba: 0x00c00bd1.00dd.23 ext: 2 spc: 3340 fbi: 0 
# -------------------------------------------------------->提交

CHANGE #5 TYP:0 CLS:32 AFN:3 DBA:0x00c00bd1 OBJ:4294967295 SCN:0x0000.000c6403 SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 168 spc: 3636 flg: 0x0012 seq: 0x00dd rec: 0x22
            xid:  0x0008.011.000001d7  
ktubl redo: slt: 17 rci: 0 opc: 11.1 [objn: 15682 objd: 15682 tsn: 5]
Undo type:  Regular undo        Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
             0x00000000  prev ctl uba: 0x00c00bd1.00dd.21 
prev ctl max cmt scn:  0x0000.000c52b6  prev tx cmt scn:  0x0000.000c52c7 
txn start scn:  0x0000.000c6434  logon user: 34  prev brb: 12585939  prev bcl: 0 BuExt idx: 0 flg2: 0
KDO undo record:
KTB Redo 
op: 0x03  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: Z
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 216
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400087  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  41 41 41 41 41
# --------------------------------------------------------> Undo block 把数据修改前值放到UNDO


CHANGE #6 TYP:0 CLS:32 AFN:3 DBA:0x00c00bd1 OBJ:4294967295 SCN:0x0000.000c6436 SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 124 spc: 3466 flg: 0x0022 seq: 0x00dd rec: 0x23
            xid:  0x0008.011.000001d7  
ktubu redo: slt: 17 rci: 34 opc: 11.1 objn: 15682 objd: 15682 tsn: 5
Undo type:  Regular undo       Undo type:  Last buffer split:  No 
Tablespace Undo:  No 
             0x00000000
KDO undo record:
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c00bd1.00dd.22
Array Update of 1 rows: 
tabn: 0 slot: 1(0x1) flag: 0x2c lock: 0 ckix: 216
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400087  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  42 42 42 42 42
# --------------------------------------------------------> Undo block 把数据修改前值放到UNDO
```

四条日志：
 第一条日志：
   日志头
   OP=5.2 ==> 事务开始
   OP=5.1 ==> undo segment header
   OP=11.2 ==> 插入AAAAA
   
 第二条日志：
    日志头
    OP=5.1 ==> undo segment header
    OP=11.2 ==> 插入BBBBB
  
 第三条日志：
    日志头
    OP:5.4 ==> 提交（事务结束）

 第四条日志：
    日志头
    OP=11.19 ==> 数据修改成aaaaa
    OP=5.2 ==> 事务开始
    OP=11.19 ==> 数据修改成bbbbb
    OP:5.4 ==> 提交（事务结束）
    OP=5.1 ==> 把数据修改前值AAAAA放到UNDO
    OP=5.1 ==> 把数据修改前值BBBBB放到UNDO

## 非IMU下的redo产生过程
![](http://p2c0rtsgc.bkt.clouddn.com/0418_oracle_dsi_01.png)
实验：
``` perl
# 查看隐含参数_in_memory_undo的状态，默认开启IMU开启TRUE
SQL> @show_para
Enter value for p: in_memory_undo
old  12:     AND upper(i.ksppinm) LIKE upper('%&p%')
new  12:     AND upper(i.ksppinm) LIKE upper('%in_memory_undo%')

P_NAME                                           P_DESCRIPTION                                                  P_VALUE    ISDEFAULT ISMODIFIED ISADJ
------------------------------------------------ -------------------------------------------------------------- ---------- --------- ---------- -----
_in_memory_undo                                  Make in memory undo for top level transactions                 TRUE       TRUE      FALSE      FALSE

# 修改_in_memory_undo为false，并重启数据库
alter system set "_in_memory_undo"=false;
shutdown immediate
startup

SQL> @show_para
Enter value for p: in_memory_undo
old  12:     AND upper(i.ksppinm) LIKE upper('%&p%')
new  12:     AND upper(i.ksppinm) LIKE upper('%in_memory_undo%')

P_NAME                                           P_DESCRIPTION                                                  P_VALUE    ISDEFAULT ISMODIFIED ISADJ
------------------------------------------------ -------------------------------------------------------------- ---------- --------- ---------- -----
_in_memory_undo                                  Make in memory undo for top level transactions                 FALSE      FALSE     FALSE      FALSE

# 切换日志组，使CURRENT日志组为新的
SQL> alter system switch logfile;
SQL> col member for a50
SQL> select l.GROUP#,l.STATUS,f.MEMBER from v$log l,v$logfile f
      where l.GROUP#=f.GROUP#;

    GROUP# STATUS           MEMBER
---------- ---------------- --------------------------------------------------
         1 INACTIVE         /u01/app/oracle/oradata/orcl/redo01.log
         2 INACTIVE         /u01/app/oracle/oradata/orcl/redo02.log
         3 CURRENT          /u01/app/oracle/oradata/orcl/redo03.log

# 执行测试语句
conn lyj/lyj
drop table t1 purge;
create table t1 (id int,name varchar2(10));
insert into t1 values(1,'AAAAA');
insert into t1 values(2,'BBBBB');
commit;
update t1 set name='aaaaa' where name='AAAAA';
update t1 set name='bbbbb' where name='BBBBB';
commit;

select object_id from user_objects where object_name='T1';

 OBJECT_ID
----------
     15685

# 导出当前redo log
alter system dump logfile '/u01/app/oracle/oradata/orcl/redo03.log';

select * from v$diag_info where name='Default Trace File';

   INST_ID NAME
---------- ----------------------------------------------------------------
VALUE
------------------------------------------------------------------------------------------------------------------------------------------------------
         1 Default Trace File
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_26888.trc
```

查看分析dump的trace日志，一共6条REDO RECORD
第一个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x000042.00000039.0010 LEN: 0x0170 VLD: 0x01
SCN: 0x0000.000c749d SUBSCN:  3 04/19/2018 13:29:30
# --------------------------------------------------------> 日志头

CHANGE #1 TYP:0 CLS:31 AFN:3 DBA:0x00c000f0 OBJ:4294967295 SCN:0x0000.000c7436 SEQ:1 OP:5.2 ENC:0 RBL:0
ktudh redo: slt: 0x0004 sqn: 0x000001dc flg: 0x0012 siz: 112 fbi: 0
            uba: 0x00c00bd8.00dd.2f    pxid:  0x0000.000.00000000
# --------------------------------------------------------> 事务开始

CHANGE #2 TYP:0 CLS:32 AFN:3 DBA:0x00c00bd8 OBJ:4294967295 SCN:0x0000.000c7435 SEQ:2 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 112 spc: 1710 flg: 0x0012 seq: 0x00dd rec: 0x2f
            xid:  0x0008.004.000001dc  
ktubl redo: slt: 4 rci: 0 opc: 11.1 [objn: 15685 objd: 15685 tsn: 5]
Undo type:  Regular undo        Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
             0x00000000  prev ctl uba: 0x00c00bd8.00dd.2d 
prev ctl max cmt scn:  0x0000.000c6607  prev tx cmt scn:  0x0000.000c6621 
txn start scn:  0xffff.ffffffff  logon user: 34  prev brb: 12585942  prev bcl: 0 BuExt idx: 0 flg2: 0
KDO undo record:
KTB Redo 
op: 0x03  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: Z
KDO Op code: DRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400085  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 0(0x0)
# ―――――――――――――――――――――――――――――――――――――――――――――――――――――――> undo segment header事务表

CHANGE #3 TYP:0 CLS:1 AFN:5 DBA:0x01400085 OBJ:15685 SCN:0x0000.000c749d SEQ:2 OP:11.2 ENC:0 RBL:0
KTB Redo 
op: 0x01  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: F  xid:  0x0008.004.000001dc    uba: 0x00c00bd8.00dd.2f
KDO Op code: IRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400085  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 0(0x0) size/delt: 12
fb: --H-FL-- lb: 0x1  cc: 2
null: --
col  0: [ 2]  c1 02
col  1: [ 5]  41 41 41 41 41
# --------------------------------------------------------> 插入AAAAA
```

第二个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x000042.00000039.0180 LEN: 0x0100 VLD: 0x01
SCN: 0x0000.000c749d SUBSCN:  4 04/19/2018 13:29:30
# --------------------------------------------------------> 日志头

CHANGE #1 TYP:0 CLS:32 AFN:3 DBA:0x00c00bd8 OBJ:4294967295 SCN:0x0000.000c749d SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 68 spc: 1596 flg: 0x0022 seq: 0x00dd rec: 0x30
            xid:  0x0008.004.000001dc  
ktubu redo: slt: 4 rci: 47 opc: 11.1 objn: 15685 objd: 15685 tsn: 5
Undo type:  Regular undo       Undo type:  Last buffer split:  No 
Tablespace Undo:  No 
             0x00000000
KDO undo record:
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c00bd8.00dd.2f
KDO Op code: DRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400085  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 1(0x1)
# ―――――――――――――――――――――――――――――――――――――――――――――――――――――――> undo segment header事务表

CHANGE #2 TYP:0 CLS:1 AFN:5 DBA:0x01400085 OBJ:15685 SCN:0x0000.000c749d SEQ:3 OP:11.2 ENC:0 RBL:0
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c00bd8.00dd.30
KDO Op code: IRP row dependencies Disabled
  xtype: XA flags: 0x00000000  bdba: 0x01400085  hdba: 0x01400082
itli: 1  ispac: 0  maxfr: 4858
tabn: 0 slot: 1(0x1) size/delt: 12
fb: --H-FL-- lb: 0x1  cc: 2
null: --
col  0: [ 2]  c1 03
col  1: [ 5]  42 42 42 42 42
# --------------------------------------------------------> 插入BBBBB

```

第三个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x000042.0000003a.0090 LEN: 0x0060 VLD: 0x01
SCN: 0x0000.000c749e SUBSCN:  1 04/19/2018 13:29:30
# --------------------------------------------------------> 日志头

CHANGE #1 TYP:0 CLS:31 AFN:3 DBA:0x00c000f0 OBJ:4294967295 SCN:0x0000.000c749d SEQ:1 OP:5.4 ENC:0 RBL:0
ktucm redo: slt: 0x0004 sqn: 0x000001dc srt: 0 sta: 9 flg: 0x2 ktucf redo: uba: 0x00c00bd8.00dd.30 ext: 2 spc: 1526 fbi: 0
# --------------------------------------------------------> 提交(事务结束)
```

第四个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x000042.0000003b.0010 LEN: 0x0204 VLD: 0x05
SCN: 0x0000.000c749f SUBSCN:  1 04/19/2018 13:29:30
(LWN RBA: 0x000042.0000003b.0010 LEN: 0002 NST: 0001 SCN: 0x0000.000c749f)
# --------------------------------------------------------> 日志头

CHANGE #1 TYP:0 CLS:29 AFN:3 DBA:0x00c000e0 OBJ:4294967295 SCN:0x0000.000c7432 SEQ:1 OP:5.2 ENC:0 RBL:0
ktudh redo: slt: 0x0013 sqn: 0x000001d5 flg: 0x0012 siz: 168 fbi: 0
            uba: 0x00c008e0.012f.05    pxid:  0x0000.000.00000000
# --------------------------------------------------------> 事务开始

CHANGE #2 TYP:0 CLS:30 AFN:3 DBA:0x00c008e0 OBJ:4294967295 SCN:0x0000.000c742e SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 168 spc: 7682 flg: 0x0012 seq: 0x012f rec: 0x05
            xid:  0x0007.013.000001d5  
ktubl redo: slt: 19 rci: 0 opc: 11.1 [objn: 15685 objd: 15685 tsn: 5]
Undo type:  Regular undo        Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
             0x00000000  prev ctl uba: 0x00c008e0.012f.04 
prev ctl max cmt scn:  0x0000.000c6af1  prev tx cmt scn:  0x0000.000c6af2 
txn start scn:  0xffff.ffffffff  logon user: 34  prev brb: 12585145  prev bcl: 0 BuExt idx: 0 flg2: 0
KDO undo record:
KTB Redo 
op: 0x03  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: Z
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 27
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400085  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  41 41 41 41 41
# --------------------------------------------------------> 把数据修改前值AAAAA放到UNDO

CHANGE #3 TYP:2 CLS:1 AFN:5 DBA:0x01400085 OBJ:15685 SCN:0x0000.000c749e SEQ:1 OP:11.19 ENC:0 RBL:0
KTB Redo 
op: 0x11  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: F  xid:  0x0007.013.000001d5    uba: 0x00c008e0.012f.05
Block cleanout record, scn:  0x0000.000c749f ver: 0x01 opt: 0x02, entries follow...
  itli: 1  flg: 2  scn: 0x0000.000c749e
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 2 ckix: 27
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400085  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  61 61 61 61 61
# --------------------------------------------------------> 数据修改成aaaaa
```

第五个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x000042.0000003c.0024 LEN: 0x0140 VLD: 0x01
SCN: 0x0000.000c749f SUBSCN:  2 04/19/2018 13:29:30
# --------------------------------------------------------> 日志头

CHANGE #1 TYP:0 CLS:30 AFN:3 DBA:0x00c008e0 OBJ:4294967295 SCN:0x0000.000c749f SEQ:1 OP:5.1 ENC:0 RBL:0
ktudb redo: siz: 124 spc: 7512 flg: 0x0022 seq: 0x012f rec: 0x06
            xid:  0x0007.013.000001d5  
ktubu redo: slt: 19 rci: 5 opc: 11.1 objn: 15685 objd: 15685 tsn: 5
Undo type:  Regular undo       Undo type:  Last buffer split:  No 
Tablespace Undo:  No 
             0x00000000
KDO undo record:
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c008e0.012f.05
Array Update of 1 rows: 
tabn: 0 slot: 1(0x1) flag: 0x2c lock: 0 ckix: 27
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400085  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  42 42 42 42 42
# --------------------------------------------------------> 把数据修改前值BBBBB放到UNDO

CHANGE #2 TYP:0 CLS:1 AFN:5 DBA:0x01400085 OBJ:15685 SCN:0x0000.000c749f SEQ:1 OP:11.19 ENC:0 RBL:0
KTB Redo 
op: 0x02  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: C  uba: 0x00c008e0.012f.06
Array Update of 1 rows: 
tabn: 0 slot: 1(0x1) flag: 0x2c lock: 2 ckix: 27
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x01400085  hdba: 0x01400082
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [ 5]  62 62 62 62 62
# --------------------------------------------------------> 数据修改成bbbbb
```

第六个REDO RECORD
``` perl
REDO RECORD - Thread:1 RBA: 0x000042.0000003c.0164 LEN: 0x0060 VLD: 0x01
SCN: 0x0000.000c74a0 SUBSCN:  1 04/19/2018 13:29:30
CHANGE #1 TYP:0 CLS:29 AFN:3 DBA:0x00c000e0 OBJ:4294967295 SCN:0x0000.000c749f SEQ:1 OP:5.4 ENC:0 RBL:0
ktucm redo: slt: 0x0013 sqn: 0x000001d5 srt: 0 sta: 9 flg: 0x2 ktucf redo: uba: 0x00c008e0.012f.06 ext: 2 spc: 7386 fbi: 0 
# --------------------------------------------------------> 提交(事务结束)
```

六条日志：
 第一条日志：
   日志头
   OP=5.2 ==> 事务开始
   OP=5.1 ==> undo segment header事务表
   OP=11.2 ==> 插入AAAAA
   
 第二条日志：
    日志头
    OP=5.1 ==> undo segment header事务表
    OP=11.2 ==> 插入BBBBB
  
 第三条日志：
    日志头
    OP:5.4 ==> 提交（事务结束）

 第四条日志：
    日志头
    OP=5.2 ==> 事务开始
    OP=5.1 ==> 把数据修改前值AAAAA放到UNDO
    OP=11.19 ==> 数据修改成aaaaa

 第五条日志：
    日志头
    OP=5.1 ==> 把数据修改前值BBBBB放到UNDO
    OP=11.19 ==> 数据修改成bbbbb
  
 第六条日志：
    日志头
    OP:5.4 ==> 提交（事务结束）

## 简要解析BBED LOGFILE
``` perl
$ bbed

BBED: Release 2.0.0.0.0 - Limited Production on Thu Apr 19 15:21:28 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

************* !!! For Oracle Internal Use only !!! *************

BBED> set filename '/u01/app/oracle/oradata/orcl/redo03.log'
        FILENAME        /u01/app/oracle/oradata/orcl/redo03.log

BBED> dump
 File: /u01/app/oracle/oradata/orcl/redo03.log (0)
 Block: 1                Offsets:    0 to  511           Dba:0x00000000
------------------------------------------------------------------------
 01220000 01000000 42000000 00800b51 00000000 0004200b acfd6d59 4f52434c 
 00000000 2f1c0000 00900100 00020000 03000200 ac546d59 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 54687265 
 61642030 3030312c 20536571 23203030 30303030 30303636 2c205343 4e203078 
 30303030 30303063 37343736 2d307866 66666666 66666666 66666600 ffffffff 
 2cb2f839 01000000 00000000 01000000 01000000 76740c00 00000000 1df80b3a 
 ffffffff ffff0000 00000000 01000002 01000000 00000000 30b2f839 76740c00 
 00000000 1df80b3a 00000000 00008000 00000000 00000000 00000000 02000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 aa00d250 06c69aa1 6693626c 2ef450a8 114e1e31 0fceb064 d6bab350 5a915c29 
 05000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

# little Endian 注意颠倒
offset 0-1: 2201 表示这是redo logfile block，每个redo block头都是这值(跟平台有关系)
offset 4-5: 0001 表示redo logfile block number号
offset 8-9: 0042 表示redo sequence number号
offset 12-13: 8000 表示offset值(byte number with block)
```

## redo latch锁
>非IMU产生一条REDO RECORD的步骤
1、获取redo copy Latch
2、获取redo allocate Latch
3、申请到LOG BUFFER空间
4、释放redo allocate Latch
5、生产日志（从buffer cache copy 修改前的值。从PGA copy 修改后的值），产生日志头
6、释放redo copy Latch
>IMU--->把多条日志条目合并成一条，减少LATCH，但会多出一个IMU LATCH

``` perl
# 查看redo相关latch锁
select latch#,name from v$latch where name like '%redo%';

    LATCH# NAME
---------- ----------------------------------------------------------------
       168 redo on-disk SCN
       169 ping redo on-disk SCN
       207 redo writing
       208 redo copy
       209 redo allocation
       210 real redo SCN
       212 readredo stats and histogram
       238 readable standby metadata redo cache
       544 KFR redo allocation latch

select latch#,name from v$latch_children where name like '%redo%';


```

## OP CODE(部分)
``` perl
格式：layer: opcode
       LAYER的含义：
               4 ― Block Cleanout
               5 ― Transaction Management
               10 ― 索引操作
               11 ― 行操作
               13 ― 段[url=]管理[/url]
               14 ― Extent 管理
               17 ― 表空间管理
               18 ― Block Image (Hot Backups)
               19 ― Direct Loader
               20 ― Compatibility segment
               22 ― 本地管理表空间
               23 ― Block Writes
               24 ― DDL语句
Layer 1 : Transaction Control - KCOCOTCT     
   Opcode 1 : KTZFMT 
   Opcode 2 : KTZRDH 
   Opcode 3 : KTZARC
   Opcode 4 : KTZREP
Layer 2 : Transaction Read -  KCOCOTRD     
Layer 3 : Transaction Update -  KCOCOTUP     
Layer 4 : Transaction Block -  KCOCOTBK     [ktbcts.h]
         Opcode 1 : Block Cleanout 
         Opcode 2 : Physical Cleanout 
         Opcode 3 : Single Array Change
         Opcode 4 : Multiple Changes to an Array
         Opcode 5 : Format Block
Layer 5 : Transaction Undo -  KCOCOTUN     [ktucts.h]
         Opcode 1 : Undo block or undo segment header - KTURDB
         Opcode 2 : Update rollback segment header - KTURDH
         Opcode 3 : Rollout a transaction begin 
         Opcode 4 : Commit transaction (transaction table update) 
- no undo record 
         Opcode 5 : Create rollback segment (format) - no undo record 
         Opcode 6 : Rollback record index in an undo block - KTUIRB
         Opcode 7 : Begin transaction (transaction table update) 
         Opcode 8 : Mark transaction as dead 
         Opcode 9 : Undo routine to rollback the extend of a rollback segment 
         Opcode 10 :Redo to perform the rollback of extend of rollback segment 
                    to the segment header.  
         Opcode 11 :Rollback DBA in transaction table entry - KTUBRB 
         Opcode 12 :Change transaction state (in transaction table entry) 
         Opcode 13 :Convert rollback segment format (V6 -> V7) 
         Opcode 14 :Change extent allocation parameters in a rollback segment 
         Opcode 15 :
         Opcode 16 :
         Opcode 17 :
         Opcode 18 :
         Opcode 19 : Transaction start audit log record
         Opcode 20 : Transaction continue audit log record     
         Opcode 24 : Kernel Transaction Undo Relog CHanGe - KTURLGU
Layer 6 : Control File -  KCOCODCF     [tbs.h]
Layer 10 : INDEX -  KCOCODIX     [kdi.h]
         Opcode 1 : load index block (Loader with direct mode) 
         Opcode 2 : Insert leaf row 
         Opcode 3 : Purge leaf row 
         Opcode 4 : Mark leaf row deleted 
         Opcode 5 : Restore leaf row (clear leaf delete flags) 
         Opcode 6 : Lock index block 
         Opcode 7 : Unlock index block 
         Opcode 8 : Initialize new leaf block 
         Opcode 9 : Apply Itl Redo 
         Opcode 10 :Set leaf block next link 
         Opcode 11 :Set leaf block previous link 
         Opcode 12 :Init root block after split 
         Opcode 13 :Make leaf block empty 
         Opcode 14 :Restore block before image 
         Opcode 15 :Branch block row insert 
         Opcode 16 :Branch block row purge 
         Opcode 17 :Initialize new branch block 
         Opcode 18 :Update keydata in row 
         Opcode 19 :Clear row’s split flag 
         Opcode 20 :Set row’s split flag 
         Opcode 21 :General undo above the cache (undo) 
         Opcode 22 :Undo operation on leaf key above the cache (undo) 
         Opcode 23 :Restore block to b-tree 
         Opcode 24 :Shrink ITL (transaction entries) 
         Opcode 25 :Format root block redo 
         Opcode 26 :Undo of format root block (undo) 
         Opcode 27 :Redo for undo of format root block 
         Opcode 28 :Undo for migrating block
         Opcode 29 :Redo for migrating block
         Opcode 30 :IOT leaf block nonkey update
         Opcode 31 :Cirect load root redo
         Opcode 32 :Combine operation for insert and restore rows 
Layer 11 : Row Access -  KCOCODRW     [kdocts.h]
         Opcode 1 : Interpret Undo Record (Undo) 
         Opcode 2 : Insert Row Piece 
         Opcode 3 : Drop Row Piece 
         Opcode 4 : Lock Row Piece 
         Opcode 5 : Update Row Piece 
         Opcode 6 : Overwrite Row Piece 
         Opcode 7 : Manipulate First Column (add or delete the 1rst column) 
         Opcode 8 : Change Forwarding address 
         Opcode 9 : Change the Cluster Key Index 
         Opcode 10 :Set Key Links (change the forward & backward key links 
                    on a cluster key) 
         Opcode 11 :Quick Multi-Insert (ex: insert as select …) 
         Opcode 12 :Quick Multi-Delete 
         Opcode 13 :Toggle Block Header flags 
Layer 12 : Cluster -  KCOCODCL     [?]
Layer 13 : Transaction Segment -  KCOCOTSG     [ktscts.h]
         Opcode 1 : Data segment format 
         Opcode 2 : Merge 
         Opcode 3 : Set link in block 
         Opcode 4 : Not used 
         Opcode 5 : New block (affects segment header) 
         Opcode 6 : Format block (affects data block) 
         Opcode 7 : Record link 
         Opcode 8 : Undo free list (undo) 
         Opcode 9 : Redo free list head (called as part of undo) 
         Opcode 9 : Format free list block (freelist group) 
         Opcode 11 :Format new blocks in free list 
         Opcode 12 :free list clear 
         Opcode 13 :free list restore (back) (undo of opcode 12) 
Layer 14 : Transaction Extent -  KCOCOTEX     [kte.h]
         Opcode 1 : Add extent to segment 
         Opcode 2 : Unlock Segment Header 
         Opcode 3 : Extent DEaLlocation (DEL) 
         Opcode 4 : Undo to Add extent operation (see opcode 1) 
         Opcode 5 : Extent Incarnation number increment 
         Opcode 6 : Lock segment Header 
         Opcode 7 : Undo to rollback extent deallocation (see opcode 3) 
         Opcode 8 : Apply Position Update (truncate) 
         Opcode 9 : Link blocks to Freelist 
         Opcode 10 :Unlink blocks from Freelist 
         Opcode 11 :Undo to Apply Position Update (see opcode 8) 
         Opcode 12 :Convert segment header to 6.2.x type 
Layer 15 : Table Space -  KCOCOTTS     [ktt.h]
        Opcode 1 : Format deferred rollback segment header 
        Opcode 2 : Add deferred rollback record 
        Opcode 3 : Move to next block 
        Opcode 4 : Point to next deferred rollback record 
Layer 16 : Row Cache -  KCOCOQRC     
Layer 17 : Recovery (REDO) -  KCOCORCV     [kcv.h]
         Opcode 1 : End Hot Backup : This operation clears the hot backup 
                    in-progress flags in the indicated list of files 
         Opcode 2 : Enable Thread : This operation creates a redo record 
                    signalling that a thread has been enabled 
         Opcode 3 : Crash Recovery Marker 
         Opcode 4 : Resizeable datafiles
         Opcode 5 : Tablespace ONline
         Opcode 6 : Tablespace OFFline
         Opcode 7 : Tablespace ReaD Write
         Opcode 8 : Tablespace ReaD Only
         Opcode 9 : ADDing datafiles to database
         Opcode 10 : Tablespace DRoP
         Opcode 11 : Tablespace PitR     
Layer 18 : Hot Backup Log Blocks -  KCOCOHLB     [kcb.h]
         Opcode 1 : Log block image 
         Opcode 2 : Recovery testing 
Layer 19 : Direct Loader Log Blocks - KCOCODLB     [kcbl.h]
         Opcode 1 : Direct block logging 
         Opcode 2 : Invalidate range 
         Opcode 3 : Direct block relogging
         Opcode 4 : Invalidate range relogging     
Layer 20 : Compatibility Segment operations - KCOCOKCK  [kck.h]
         Opcode 1 : Format compatibility segment -  KCKFCS
         Opcode 2 : Update compatibility segment - KCKUCS
Layer 21 : LOB segment operations - KCOCOLFS     [kdl2.h]
         Opcode 1 : Write data into ILOB data block - KDLOPWRI
Layer 22 : Tablespace bitmapped file operations -  KCOCOTBF [ktfb.h]
Opcode 1 : format space header - KTFBHFO
Opcode 2 : space header generic redo - KTFBHREDO
Opcode 3 : space header undo - KTFBHUNDO
Opcode 4 : space bitmap block format - KTFBBFO
Opcode 5 : bitmap block generic redo - KTFBBREDO 
Layer 23 : write behind logging of blocks - KCOCOLWR [kcbb.h]
Opcode 1 : Dummy block written callback - KCBBLWR
Layer 24 : Logminer related (DDL or OBJV# redo) - KCOCOKRV [krv.h]
Opcode : common portion of the ddl - KRVDDL
Opcode : direct load redo - KRVDLR 
Opcode : lob related info - KRVLOB
Opcode : misc info - KRVMISC 
Opcode : user info - KRVUSER

还有begin backup时是18.1
end backup 是17.1
nologging操作是19.2 并注明是invld
```

## 参考
* [Oracle IMU模式下REDO格式详解](https://blog.csdn.net/haibusuanyun/article/details/18037635)




