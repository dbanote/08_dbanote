---
title: Oracle特殊恢复原理与实战_09 Undo深入内部解析
date: 2018-04-26
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## UNDO回滚段的作用
1. 事务回滚
2. 实例恢复（利用回滚来恢复未提交的数据）
3. 读一致性（构造CR）
4. 数据库闪回查询
5. 数据库闪回恢复逻辑错误

<!-- more -->

## 重现ORA-01555快照过旧，分析内部原因和解决办法
### 创建一个不能自动扩展的UNDO表空间
``` perl
show parameter undo

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
_in_memory_undo                      boolean     FALSE
undo_management                      string      AUTO
undo_retention                       integer     900
undo_tablespace                      string      UNDOTBS1

create undo tablespace undotbs2 datafile 
  '/u01/app/oracle/oradata/orcl/undotbs02.dbf' size 5m;

alter system set undo_tablespace=undotbs2;

select name,value/1024 as KB
  from (select b.name,a.value from v$mystat a,v$statname b 
        where a.STATISTIC#=b.statistic#) 
  where name='redo size' or name like 'undo change%';

NAME                                                                     KB
---------------------------------------------------------------- ----------
redo size                                                            123.75
undo change vector size                                           31.703125

col tablespace_name for a15
col file_name for a50
select file_id,file_name,tablespace_name,bytes/1024/1024 MB 
  from dba_data_files 
  where file_name like '%undotbs02%';

   FILE_ID FILE_NAME                                          TABLESPACE_NAME         MB
---------- -------------------------------------------------- --------------- ----------
         6 /u01/app/oracle/oradata/orcl/undotbs02.dbf         UNDOTBS2                 5
```

### 重现ORA-01555快照过旧
``` perl
# 会话1，定义并打开一个游标
conn lyj/lyj
drop table test1 purge;
create table test1 as select * from all_objects;

set time on
var c1 refcursor
begin
open :c1 for select * from test1;
end;
/

# 会话2，做一个大量的update操作，将T表中的所有ID字段更新了100次
conn lyj/lyj
set time on
begin
for i in 1..100 loop
update test1 set object_id=i;
commit;
end loop;
end;
/

# 会话1，打开刚才定义的游标C1，会得到如下ORA-01555的错误
SQL> print :c1
ERROR:
ORA-01555: snapshot too old: rollback segment number 20 with name 
'_SYSSMU20_2141891769$' too small

# alert log中也有如下报错信息：
Fri Apr 27 14:09:41 2018
ORA-01555 caused by SQL statement below (SQL ID: 0m0zj87wk6wru, Query Duration=107 sec, SCN: 0x0000.00129590):
SELECT * FROM TEST1
```

### 分析内部原因和解决办法
ORACLE官方解释如下：
``` perl
oerr ora 01555
#----------------------------------------------------------------------------------------
01555, 00000, "snapshot too old: rollback segment number %s with name \"%s\" too small"
// *Cause: rollback records needed by a reader for consistent read are
//         overwritten by other writers
// *Action: If in Automatic Undo Management mode, increase undo_retention
//          setting. Otherwise, use larger rollback segments
#----------------------------------------------------------------------------------------
```

原因分析：
ORACLE一致性读，查询的结果是发起时间(SCN)那一刻的结果集。当大查询没有结束，但其中内容已被更改时，ORACLE会从UNDO里根据发起时间SCN的值找到相应的修改前的值。但如果这时UNDO里的值已经被覆盖，找到不修改前的值了，就会报ORA-01555错误。

解决办法：
1. 加大UNDO表空间大小：undo datafile设置成自动扩展（单个文件最大32G），增加undo datafile的个数
2. 加大undo_retention，使undo可以保留更长时间不被覆盖
3. 优化查询SQL，使用SQL可以在较短的时间完成

## 用dump分析UNDO的一致性读
### 构建分析测试环境
``` perl
# 删除上面刚测试用的UNDO表空间
alter system set undo_tablespace=undotbs1;
startup force
drop tablespace undotbs2 including contents and datafiles;

# 构建分析测试环境（为批执行命令，有显示结果放在最后了）
conn lyj/lyj
drop table lyj purge;
create table lyj(id int,name char(2000));
insert into lyj values(1,'AAAAA');
commit;

var x refcursor;
select current_scn from v$database;
exec open :x for select * from lyj where id=1;

update lyj set name='BBBBB' where id=1;
commit;

update lyj set name='CCCCC' where id=1;
commit;

update lyj set name='DDDDD' where id=1;
commit;

update lyj set name='EEEEE' where id=1;

col name for a20
print :x;
alter system flush buffer_cache;

# 显示的结果
select current_scn from v$database;
 CURRENT_SCN
-----------
    1396033

print :x;
    ID NAME
------ --------------------
     1 AAAAA
```

### 根据未提交事务获取回滚段信息
``` perl
# 可以通过v$transaction视图来确认事务当前使用的undo segment信息，开一个新会话
SQL> select XIDUSN,XIDSLOT,XIDSQN,UBABLK,UBAFIL,UBAREC,START_TIME,START_SCNB,STATUS from v$transaction;

    XIDUSN    XIDSLOT     XIDSQN     UBABLK     UBAFIL     UBAREC START_TIME           START_SCNB STATUS
---------- ---------- ---------- ---------- ---------- ---------- -------------------- ---------- ----------------
         2         30        627       2162          3         19 04/28/18 14:22:24       1396039 ACTIVE

# xidusn：undo segment number
# xidslot：slot number
# xidsqn：sequence number
# ubafil：undo block address (uba) filenum
# ubablk：uba block number
# ubarec：UBA record number

# 当undo_management设置成AUTO时，使用UNDO tablespace来管理回滚段。 
# 多个undo segment，并且这些segment是存放在UNDO表空间里，这样DB的性能会提高。
SQL> select * from v$rollname;

       USN NAME
---------- ------------------------------
         0 SYSTEM
         1 _SYSSMU1_108211372$
         2 _SYSSMU2_2934779712$   # XIDUSN=2
         3 _SYSSMU3_1316151929$   
         4 _SYSSMU4_354880685$
         5 _SYSSMU5_2085108374$
         6 _SYSSMU6_1379281026$
         7 _SYSSMU7_2299785305$
         8 _SYSSMU8_548913123$
         9 _SYSSMU9_3484822342$
        10 _SYSSMU10_493803725$

SQL> select header_file,header_block from dba_segments where segment_name='_SYSSMU2_2934779712$';

HEADER_FILE HEADER_BLOCK
----------- ------------
          3          144

SQL> select EXTENT_ID,FILE_ID, BLOCK_ID,BYTES,BLOCKS,STATUS from dba_undo_extents where segment_name='_SYSSMU2_2934779712$';

 EXTENT_ID    FILE_ID   BLOCK_ID      BYTES     BLOCKS STATUS
---------- ---------- ---------- ---------- ---------- ---------
         0          3        144      65536          8 EXPIRED
         1          3        296      65536          8 EXPIRED
         2          3       2048    1048576        128 ACTIVE

# 相关视图
SQL> select * from X$KTUXE where KTUXESTA='ACTIVE';
SQL> select * from v$rollstat;
```

### dump undo header
``` perl
alter system dump undo header '_SYSSMU2_2934779712$'; 
# or
alter system dump datafile 3 block 144;
# ALTER SYSTEM DUMP UNDO BLOCK 'segment_name' XID xidusn xidslot xidsqn; 
# alter system dump undo block '_SYSSMU2_2934779712$' XID 2 30 627;

# 确定dump trace位置
oradebug setmypid
oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_4440.trc
# 查看trace
********************************************************************************
Undo Segment:  _SYSSMU2_2934779712$ (2)
********************************************************************************
  Extent Control Header
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 3      #blocks: 143   
                  last map  0x00000000  #maps: 0      offset: 4080  
      Highwater::  0x00c00873  ext#: 2      blk#: 115    ext size: 128   
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 0     
  mapblk  0x00000000  offset: 2     
                   Unlocked
     Map Header:: next  0x00000000  #extents: 3    obj#: 0      flag: 0x40000000
  Extent Map
  -----------------------------------------------------------------
   0x00c00091  length: 7     
   0x00c00128  length: 8     
   0x00c00800  length: 128   
  
 Retention Table 
  -----------------------------------------------------------
 Extent Number:0  Commit Time: 1524758471
 Extent Number:1  Commit Time: 1524758471
 Extent Number:2  Commit Time: 1524758471
  
  TRN CTL:: seq: 0x00f1 chd: 0x000a ctl: 0x0006 inc: 0x00000000 nfb: 0x0001
            mgc: 0xb000 xts: 0x0068 flg: 0x0001 opt: 2147483646 (0x7ffffffe)
            uba: 0x00c00873.00f1.01 scn: 0x0000.00154902
Version: 0x01
  FREE BLOCK POOL::
    uba: 0x00c00873.00f1.01 ext: 0x2  spc: 0x1f4c  
    uba: 0x00000000.00f1.1f ext: 0x2  spc: 0xbf0   
    uba: 0x00000000.00f1.01 ext: 0x2  spc: 0x1f84  
    uba: 0x00000000.00f1.01 ext: 0x2  spc: 0x1f84  
    uba: 0x00000000.00f1.01 ext: 0x2  spc: 0x1f84  
  TRN TBL::
 
  index  state cflags  wrap#    uel         scn            dba            parent-xid    nub     stmt_num    cmt
  ------------------------------------------------------------------------------------------------
   0x00    9    0x00  0x0273  0x000c  0x0000.00154c1c  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x01    9    0x00  0x0271  0x0004  0x0000.00154a94  0x00c00871  0x0000.000.00000000  0x00000002   0x00000000  1524895467
   0x02    9    0x00  0x0272  0x0018  0x0000.00154a6a  0x00c00870  0x0000.000.00000000  0x00000001   0x00000000  1524895467
   0x03    9    0x00  0x0272  0x000b  0x0000.001549aa  0x00c00868  0x0000.000.00000000  0x00000001   0x00000000  1524895466
   0x04    9    0x00  0x0270  0x0008  0x0000.00154ac7  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524895555
   0x05    9    0x00  0x0273  0x0016  0x0000.00154bdc  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x06    9    0x00  0x026f  0xffff  0x0000.00154d62  0x00c00873  0x0000.000.00000000  0x00000001   0x00000000  1524896621
   0x07    9    0x00  0x0274  0x0006  0x0000.00154d3f  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896543
   0x08    9    0x00  0x0272  0x0010  0x0000.00154ba2  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x09    9    0x00  0x0273  0x0013  0x0000.00154c6a  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x0a    9    0x00  0x0272  0x001b  0x0000.00154926  0x00c00868  0x0000.000.00000000  0x00000001   0x00000000  1524895465
   0x0b    9    0x00  0x0273  0x0012  0x0000.001549cc  0x00c00868  0x0000.000.00000000  0x00000001   0x00000000  1524895466
   0x0c    9    0x00  0x0272  0x000e  0x0000.00154c2a  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x0d    9    0x00  0x0270  0x001a  0x0000.001549ff  0x00c00869  0x0000.000.00000000  0x00000001   0x00000000  1524895466
   0x0e    9    0x00  0x0271  0x0009  0x0000.00154c48  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x0f    9    0x00  0x0272  0x0002  0x0000.00154a64  0x00c00870  0x0000.000.00000000  0x00000002   0x00000000  1524895467
   0x10    9    0x00  0x0271  0x0019  0x0000.00154bb4  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x11    9    0x00  0x0271  0x000f  0x0000.00154a5e  0x00c00869  0x0000.000.00000000  0x00000001   0x00000000  1524895467
   0x12    9    0x00  0x0271  0x000d  0x0000.001549e5  0x00c00869  0x0000.000.00000000  0x00000001   0x00000000  1524895466
   0x13    9    0x00  0x0273  0x0017  0x0000.00154c88  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x14    9    0x00  0x0272  0x0021  0x0000.00154965  0x00c00868  0x0000.000.00000000  0x00000001   0x00000000  1524895465
   0x15    9    0x00  0x0273  0x001d  0x0000.00154bfc  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x16    9    0x00  0x0270  0x0015  0x0000.00154bf0  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x17    9    0x00  0x0272  0x0007  0x0000.00154c8e  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x18    9    0x00  0x0270  0x0001  0x0000.00154a74  0x00c00870  0x0000.000.00000000  0x00000001   0x00000000  1524895467
   0x19    9    0x00  0x0271  0x001f  0x0000.00154bbe  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x1a    9    0x00  0x0270  0x0020  0x0000.00154a30  0x00c00869  0x0000.000.00000000  0x00000001   0x00000000  1524895466
   0x1b    9    0x00  0x0273  0x0014  0x0000.00154938  0x00c00868  0x0000.000.00000000  0x00000001   0x00000000  1524895465
   0x1c    9    0x00  0x0271  0x0003  0x0000.001549a2  0x00c00868  0x0000.000.00000000  0x00000001   0x00000000  1524895466
   0x1d    9    0x00  0x0273  0x0000  0x0000.00154c14  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x1e   10    0x80  0x0273  0x0002  0x0000.00154d47  0x00c00872  0x0000.000.00000000  0x00000001   0x00000000  0
   0x1f    9    0x00  0x0271  0x0005  0x0000.00154bca  0x00c00871  0x0000.000.00000000  0x00000001   0x00000000  1524896156
   0x20    9    0x00  0x0271  0x0011  0x0000.00154a4a  0x00c00869  0x0000.000.00000000  0x00000001   0x00000000  1524895466
   0x21    9    0x00  0x026e  0x001c  0x0000.00154985  0x00c00868  0x0000.000.00000000  0x00000001   0x00000000  1524895465
  EXT TRN CTL::
  usn: 2
  sp1:0x00000000 sp2:0x00000000 sp3:0x00000000 sp4:0x00000000
  sp5:0x00000000 sp6:0x00000000 sp7:0x100000000 sp8:0x00000000
  EXT TRN TBL::
  index  extflag    extHash    extSpare1   extSpare2 
  ---------------------------------------------------
   0x00  0x00000000 0x00000000 0x00000000  0x00000000
   0x01  0x00000000 0x00000000 0x00000000  0x00000000
   0x02  0x00000000 0x00000000 0x00000000  0x00000000
   0x03  0x00000000 0x00000000 0x00000000  0x00000000
   0x04  0x00000000 0x00000000 0x00000000  0x00000000
   0x05  0x00000000 0x00000000 0x00000000  0x00000000
   0x06  0x00000000 0x00000000 0x00000000  0x00000000
   0x07  0x00000000 0x00000000 0x00000000  0x00000000
   0x08  0x00000000 0x00000000 0x00000000  0x00000000
   0x09  0x00000000 0x00000000 0x00000000  0x00000000
   0x0a  0x00000000 0x00000000 0x00000000  0x00000000
   0x0b  0x00000000 0x00000000 0x00000000  0x00000000
   0x0c  0x00000000 0x00000000 0x00000000  0x00000000
   0x0d  0x00000000 0x00000000 0x00000000  0x00000000
   0x0e  0x00000000 0x00000000 0x00000000  0x00000000
   0x0f  0x00000000 0x00000000 0x00000000  0x00000000
   0x10  0x00000000 0x00000000 0x00000000  0x00000000
   0x11  0x00000000 0x00000000 0x00000000  0x00000000
   0x12  0x00000000 0x00000000 0x00000000  0x00000000
   0x13  0x00000000 0x00000000 0x00000000  0x00000000
   0x14  0x00000000 0x00000000 0x00000000  0x00000000
   0x15  0x00000000 0x00000000 0x00000000  0x00000000
   0x16  0x00000000 0x00000000 0x00000000  0x00000000
   0x17  0x00000000 0x00000000 0x00000000  0x00000000
   0x18  0x00000000 0x00000000 0x00000000  0x00000000
   0x19  0x00000000 0x00000000 0x00000000  0x00000000
   0x1a  0x00000000 0x00000000 0x00000000  0x00000000
   0x1b  0x00000000 0x00000000 0x00000000  0x00000000
   0x1c  0x00000000 0x00000000 0x00000000  0x00000000
   0x1d  0x00000000 0x00000000 0x00000000  0x00000000
   0x1e  0x00000000 0x00000000 0x00000000  0x00000000
   0x1f  0x00000000 0x00000000 0x00000000  0x00000000
   0x20  0x00000000 0x00000000 0x00000000  0x00000000
   0x21  0x00000000 0x00000000 0x00000000  0x00000000
.....
```

### dump data block
``` perl
SQL> col name for a20
SQL> select dbms_rowid.rowid_relative_fno(rowid),dbms_rowid.rowid_block_number(rowid),id,name from lyj.lyj;

DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID)         ID NAME
------------------------------------ ------------------------------------ ---------- --------------------
                                   5                                  527          1 DDDDD

alter system dump datafile 5 block 527;

oradebug setmypid
oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_4457.trc
#------------------------------------------------------------------------
Block header dump:  0x0140020f
 Object id on Block? Y
 seg/obj: 0x4079  csc: 0x00.15339a  itc: 2  flg: E  typ: 1 - DATA
     brn: 0  bdba: 0x1400208 ver: 0x01 opc: 0
     inc: 0  exflg: 0
 
 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0002.01e.00000273  0x00c00872.00f1.13  ----    1  fsc 0x0000.00000000     # 1  未提交
0x02   0x0001.005.00000272  0x00c00ce7.01a1.17  C---    0  scn 0x0000.00154d46     # 2
bdba: 0x0140020f
data_block_dump,data header at 0x7ffb27345a64
#===============
tsiz: 0x1f98
hsiz: 0x14
pbl: 0x7ffb27345a64
     76543210
flag=--------
ntab=1
nrow=1
frre=-1
fsbo=0x14
fseo=0x17bf
avsp=0x17ab
tosp=0x17ab
0xe:pti[0]      nrow=1  offs=0
0x12:pri[0]     offs=0x17bf
block_row_dump:
tab 0, row 0, @0x17bf
tl: 2009 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 2]  c1 02
col  1: [2000]
 45 45 45 45 45 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20   # EEEEE  
......
#------------------------------------------------------------------------

SQL> select to_number('00154d46','xxxxxxxxxxx') from dual;

TO_NUMBER('00154D46','XXXXXXXXXXX')
-----------------------------------
                            1396038       # 因 1396038 > 1396033，print :x 需要往上推UNDO回滚

SQL> select chr(to_number('45','xx')) from dual;

CH
--
E

Uba: 0x00c00872.00f1.13 ==> 3号文件2162号块 第13记录  # 1
# getbfno函数见之前笔记
SQL> select getbfno('0x00c00872') BFNO from dual;

BFNO
------------------------------------------------------------------------------------------------------------------------------------------------------
datafile# is:3
datablock is:2162
dump command:alter system dump datafile 3 block 2162;


Uba: 0x00c00ce7.01a1.17 ==> 3号文件3303号块 第17记录  # 2
SQL> select getbfno('0x00c00ce7') BFNO from dual;

BFNO
------------------------------------------------------------------------------------------------------------------------------------------------------
datafile# is:3
datablock is:3303
dump command:alter system dump datafile 3 block 3303;
```

### dump undo block #1
``` perl
# 新开会话
# Uba: 0x00c00872.00f1.13 ==> 3号文件2162号块 第13记录  # 1
alter system dump datafile 3 block 2162;
oradebug setmypid
oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_4074.trc
# dump undo block，定位到Rec #0x13
*-----------------------------
* Rec #0x13  slt: 0x1e  objn: 16511(0x0000407f)  objd: 16511  tblspc: 5(0x00000005)
*       Layer:  11 (Row)   opc: 1   rci 0x00   
Undo type:  Regular undo    Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
rdba: 0x00000000Ext idx: 0
flg2: 0
*-----------------------------
uba: 0x00c00872.00f1.12 ctl max scn: 0x0000.001548cc prv tx scn: 0x0000.001548e4
txn start scn: scn: 0x0000.00154d47 logon user: 34
 prev brb: 12585064 prev bcl: 0
KDO undo record:
KTB Redo 
op: 0x04  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: L  itl: xid:  0x0007.018.00000270 uba: 0x00c00be4.0172.11           # 3
                      flg: C---    lkc:  0     scn: 0x0000.00154d44     
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 0
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x0140020f  hdba: 0x0140020a
itli: 1  ispac: 0  maxfr: 4858
vect = 3
col  1: [2000]
 44 44 44 44 44 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20  # DDDDD
......

SQL> select to_number('00154d44','xxxxxxxxxxx') from dual;

TO_NUMBER('00154D44','XXXXXXXXXXX')
-----------------------------------
                            1396036       # 因 1396036 > 1396033，print :x 需要往上推UNDO回滚

SQL> select chr(to_number('44','xx')) from dual;

CH
--
D

uba: 0x00c00be4.0172.11  ==>  3号文件3044号块 第11记录      # 3
SQL> select getbfno('0x00c00be4') BFNO from dual;

BFNO
------------------------------------------------------------------------------------------------------------------------
datafile# is:3
datablock is:3044
dump command:alter system dump datafile 3 block 3044;
```

### dump undo block #2
``` perl
#Uba: 0x00c00ce7.01a1.17 ==> 3号文件3303号块 第17记录  # 2
alter system dump datafile 3 block 3303;
oradebug setmypid
oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_4091.trc
*-----------------------------
* Rec #0x17  slt: 0x05  objn: 16511(0x0000407f)  objd: 16511  tblspc: 5(0x00000005)
*       Layer:  11 (Row)   opc: 1   rci 0x00   
Undo type:  Regular undo    Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
rdba: 0x00000000Ext idx: 0
flg2: 0
*-----------------------------
uba: 0x00c00ce7.01a1.15 ctl max scn: 0x0000.001548a7 prv tx scn: 0x0000.001548bf
txn start scn: scn: 0x0000.00154d45 logon user: 34
 prev brb: 12586205 prev bcl: 0
KDO undo record:
KTB Redo 
op: 0x04  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: L  itl: xid:  0x000a.00a.0000027a uba: 0x00c005bb.0133.12          # 4
                      flg: C---    lkc:  0     scn: 0x0000.00154d42
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 0
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x0140020f  hdba: 0x0140020a
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [2000]
 43 43 43 43 43 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20  #  CCCCC
......

SQL> select to_number('00154d42','xxxxxxxxxxx') from dual;

TO_NUMBER('00154D42','XXXXXXXXXXX')
-----------------------------------
                            1396034       # 因 1396034 > 1396033，print :x 需要往上推UNDO回滚

SQL> select chr(to_number('43','xx')) from dual;

CH
--
C

uba: 0x00c005bb.0133.12  ==>  3号文件1467号块 第12记录      # 4
SQL> select getbfno('0x00c005bb') BFNO from dual;

BFNO
------------------------------------------------------------------------------------------------------------------------
datafile# is:3
datablock is:1467
dump command:alter system dump datafile 3 block 1467;
```

### dump undo block #3
``` perl
# uba: 0x00c00be4.0172.11  ==>  3号文件3044号块 第11记录      # 3
alter system dump datafile 3 block 3044;
oradebug setmypid
oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_4117.trc
*-----------------------------
* Rec #0x11  slt: 0x18  objn: 16511(0x0000407f)  objd: 16511  tblspc: 5(0x00000005)
*       Layer:  11 (Row)   opc: 1   rci 0x00   
Undo type:  Regular undo    Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
rdba: 0x00000000Ext idx: 0
flg2: 0
*-----------------------------
uba: 0x00c00be4.0172.0f ctl max scn: 0x0000.001548c3 prv tx scn: 0x0000.001548dc
txn start scn: scn: 0x0000.00154d43 logon user: 34
 prev brb: 12585954 prev bcl: 0
KDO undo record:
KTB Redo 
op: 0x04  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: L  itl: xid:  0x0002.007.00000274 uba: 0x00c00872.00f1.12         
                      flg: C---    lkc:  0     scn: 0x0000.00154d3f
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 0
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x0140020f  hdba: 0x0140020a
itli: 1  ispac: 0  maxfr: 4858
vect = 3
col  1: [2000]
 42 42 42 42 42 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20      # BBBBB

......

SQL> select to_number('00154d3f','xxxxxxxxxxx') from dual;

TO_NUMBER('00154D42','XXXXXXXXXXX')
-----------------------------------
                            1396031       # 因 1396031 < 1396033，print :x 取undo回滚链上一级的值，也就是#4对应的值

SQL> select chr(to_number('42','xx')) from dual;

CH
--
B

uba: 0x00c00872.00f1.12  ==>  3号文件2162号块 第12记录      
SQL> select getbfno('0x00c00872') BFNO from dual;

BFNO
------------------------------------------------------------------------------------------------------------------------
datafile# is:3
datablock is:2162
dump command:alter system dump datafile 3 block 2162;
```

### dump undo block #4
``` perl
# uba: 0x00c005bb.0133.12  ==>  3号文件1467号块 第12记录
alter system dump datafile 3 block 1467;
oradebug setmypid
oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_4544.trc
*-----------------------------
* Rec #0x12  slt: 0x0a  objn: 16511(0x0000407f)  objd: 16511  tblspc: 5(0x00000005)
*       Layer:  11 (Row)   opc: 1   rci 0x00   
Undo type:  Regular undo    Begin trans    Last buffer split:  No 
Temp Object:  No 
Tablespace Undo:  No 
rdba: 0x00000000Ext idx: 0
flg2: 0
*-----------------------------
uba: 0x00c005ba.0133.31 ctl max scn: 0x0000.0015490e prv tx scn: 0x0000.0015493a
txn start scn: scn: 0x0000.00154d41 logon user: 34
 prev brb: 12584354 prev bcl: 0
KDO undo record:
KTB Redo 
op: 0x03  ver: 0x01  
compat bit: 4 (post-11) padding: 1
op: Z
Array Update of 1 rows: 
tabn: 0 slot: 0(0x0) flag: 0x2c lock: 0 ckix: 0
ncol: 2 nnew: 1 size: 0
KDO Op code:  21 row dependencies Disabled
  xtype: XAxtype KDO_KDOM2 flags: 0x00000080  bdba: 0x0140020f  hdba: 0x0140020a
itli: 2  ispac: 0  maxfr: 4858
vect = 3
col  1: [2000]
 41 41 41 41 41 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20     # AAAAA  print :x 就打印这个值
```

### UNDO回滚链
以上可见，这#1-#4构成了UNDO的回滚链，通过SCN，步步递推，实现oracle的一致性读
``` perl
 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0002.01e.00000273  0x00c00872.00f1.13  ----    1  fsc 0x0000.00000000     # 1  未提交
0x02   0x0001.005.00000272  0x00c00ce7.01a1.17  C---    0  scn 0x0000.00154d46     # 2

op: L  itl: xid:  0x0007.018.00000270 uba: 0x00c00be4.0172.11                      # 3
                      flg: C---    lkc:  0     scn: 0x0000.00154d44     

op: L  itl: xid:  0x000a.00a.0000027a uba: 0x00c005bb.0133.12                      # 4
                      flg: C---    lkc:  0     scn: 0x0000.00154d42

op: L  itl: xid:  0x0002.007.00000274 uba: 0x00c00872.00f1.12                      
                      flg: C---    lkc:  0     scn: 0x0000.00154d3f
```

## UNDO段头块
Undo Segment Header Block
``` perl
# 查看UNDO的组成
SQL> select * from v$type_size where type like 'KTU%';

COMPONEN TYPE     DESCRIPTION                       TYPE_SIZE
-------- -------- -------------------------------- ----------
KTU      KTUBH    UNDO HEADER                              16
KTU      KTUXE    UNDO TRANSACTION ENTRY                   40
KTU      KTUXC    UNDO TRANSACTION CONTROL                104
```

链接：[RDBA/UBA转换文件号和块号的函数getbfno](/2018/03/28/oracle/Oracle%E7%89%B9%E6%AE%8A%E6%81%A2%E5%A4%8D%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%98-%E8%AF%BE%E7%A8%8B%E5%AD%A6%E4%B9%A0/06_Oracle%E7%89%B9%E6%AE%8A%E6%81%A2%E5%A4%8D%E5%85%A5%E9%97%A8/#RDBA转换函数)

参考：[探索undo一致性读](https://blog.csdn.net/gumengkai/article/details/53186456)
