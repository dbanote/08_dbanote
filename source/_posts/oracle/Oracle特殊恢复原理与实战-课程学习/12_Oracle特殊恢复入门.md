---
title: Oracle特殊恢复原理与实战_12 Oracle坏块处理
date: 2018-05-22
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## 数据坏块的类型 
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_01.png)

<!-- more -->

## 物理坏块的模拟
### Bad Header
the beginning of the block(cache header) is corrupt with invalid values，故障模拟：
``` perl
conn lyj/lyj
create table lyj_001(id int,name varchar2(10));
insert into lyj_001 values (1,'AAAAAA');
commit;

select dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) block#
from lyj_001;

     FILE#     BLOCK#
---------- ----------
         5        623

alter system flush buffer_cache;

# 打开BBED
BBED> set file 5 block 623
        FILE#           5
        BLOCK#          623

BBED> p kcbh
struct kcbh, 20 bytes                       @0       
   ub1 type_kcbh                            @0        0x06
   ub1 frmt_kcbh                            @1        0xa2
   ub1 spare1_kcbh                          @2        0x00
   ub1 spare2_kcbh                          @3        0x00
   ub4 rdba_kcbh                            @4        0x0140026f
   ub4 bas_kcbh                             @8        0x0026cb11
   ub2 wrp_kcbh                             @12       0x0000
   ub1 seq_kcbh                             @14       0x01        # 取值范围0-254，下面修改成255，模拟物理块坏
   ub1 flg_kcbh                             @15       0x06 (KCBHFDLC, KCBHFCKV)
   ub2 chkval_kcbh                          @16       0x940f
   ub2 spare3_kcbh                          @18       0x0000

BBED> modify /x ff offset 14
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 623              Offsets:   14 to  525           Dba:0x0140026f
------------------------------------------------------------------------
 ff060f94 00000100 0000ac49 000010cb 26000000 00000200 32006802 40010400 
 08008504 0000bf0f c000f902 11000120 000011cb 26000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000001 0100ffff 14008b1f 
 771f771f 00000100 8b1f0000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 5, Block 623:
current = 0x94f1, required = 0x94f1

# 重启数据库
SQL> conn / as sysdba
Connected.
SQL> startup force
ORACLE instance started.

Total System Global Area 4392697856 bytes
Fixed Size                  2260368 bytes
Variable Size             855638640 bytes
Database Buffers         3523215360 bytes
Redo Buffers               11583488 bytes
Database mounted.
Database opened.

# 查看lyj_001报块损坏
SQL> select * from lyj.lyj_001;
select * from lyj.lyj_001
                  *
ERROR at line 1:
ORA-01578: ORACLE data block corrupted (file # 5, block # 623)
ORA-01110: data file 5: '/u01/app/oracle/oradata/orcl/lyj_01.dbf'

# alert log中错误信息
Hex dump of (file 5, block 623) in trace file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_13642.trc
Corrupt block relative dba: 0x0140026f (file 5, block 623)
Fractured block found during buffer read     # 断裂块
Data in bad block:
 type: 6 format: 2 rdba: 0x0140026f
 last change scn: 0x0000.0026cb11 seq: 0xff flg: 0x06
 spare1: 0x0 spare2: 0x0 spare3: 0x0
 consistency value in tail: 0xcb110601
 check value in block header: 0x94f1
 computed block checksum: 0x0
Reading datafile '/u01/app/oracle/oradata/orcl/lyj_01.dbf' for corruption at rdba: 0x0140026f (file 5, block 623)
Reread (file 5, block 623) found same corrupt data (no logical check)
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_13642.trc  (incident=22944):
ORA-01578: ORACLE data block corrupted (file # 5, block # 623)
ORA-01110: data file 5: '/u01/app/oracle/oradata/orcl/lyj_01.dbf'
Incident details in: /u01/app/oracle/diag/rdbms/orcl/orcl/incident/incdir_22944/orcl_ora_13642_i22944.trc
Tue May 22 14:13:07 2018
Corrupt Block Found
         TSN = 5, TSNAME = LYJ_TS
         RFN = 5, BLK = 623, RDBA = 20972143
         OBJN = 18860, OBJD = 18860, OBJECT = LYJ_001, SUBOBJECT = 
         SEGMENT OWNER = LYJ, SEGMENT TYPE = Table Segment
Tue May 22 14:13:09 2018
Dumping diagnostic data in directory=[cdmp_20180522141309], requested by (instance=1, osid=13642), summary=[incident=22944].
Tue May 22 14:13:11 2018
Sweep [inc][22944]: completed
Sweep [inc2][22944]: completed

```

### The block is Fractured/Incomplete
header and footer of the block do not match，故障模拟：
``` perl
conn lyj/lyj
create table lyj_002(id int,name varchar2(10));
insert into lyj_002 values (1,'AAAAAA');
commit;

select dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) block#
from lyj_002;

     FILE#     BLOCK#
---------- ----------
         5        629

alter system flush buffer_cache;

# 打开BBED
BBED> set file 5 block 629
        FILE#           5
        BLOCK#          623

BBED> p kcbh
struct kcbh, 20 bytes                       @0       
   ub1 type_kcbh                            @0        0x06          # type: 06
   ub1 frmt_kcbh                            @1        0xa2
   ub1 spare1_kcbh                          @2        0x00
   ub1 spare2_kcbh                          @3        0x00
   ub4 rdba_kcbh                            @4        0x01400275
   ub4 bas_kcbh                             @8        0x00271b7c    # 2 lower bytes of scn base : 1b7c
   ub2 wrp_kcbh                             @12       0x0000
   ub1 seq_kcbh                             @14       0x01          # seq: 01
   ub1 flg_kcbh                             @15       0x06 (KCBHFDLC, KCBHFCKV)
   ub2 chkval_kcbh                          @16       0x990a
   ub2 spare3_kcbh                          @18       0x0000

tail = 2 lower bytes of SCN Base + type + seq  = 1b7c0601

BBED> dump /v count 32
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 629     Offsets:    0 to   31  Dba:0x01400275
-------------------------------------------------------
 06a20000 75024001 7c1b2700 00000106 l ....u.@.|.'.....     # -> scn base: 00271b7c
 0a990000 01000000 b1490000 7b1b2700 l .........I..{.'.

 <16 bytes per line>

BBED> dump /v offset 8188
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 629     Offsets: 8188 to 8191  Dba:0x01400275
-------------------------------------------------------
 01067c1b                            l ..|.                 # -> tail: 1b7c0601

 <16 bytes per line>

# 修改tail
BBED> modify /x 01067c1c offset 8188
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 629              Offsets: 8188 to 8191           Dba:0x01400275
------------------------------------------------------------------------
 01067c1c 

 <32 bytes per line>

BBED> sum apply
Check value for File 5, Block 629:
current = 0x9e0a, required = 0x9e0a

# 重启数据库
SQL> conn / as sysdba
Connected.
SQL> startup force
ORACLE instance started.

Total System Global Area 4392697856 bytes
Fixed Size                  2260368 bytes
Variable Size             855638640 bytes
Database Buffers         3523215360 bytes
Redo Buffers               11583488 bytes
Database mounted.
Database opened.

# 查看lyj_002报块损坏错误
SQL> select * from lyj.lyj_002;
select * from lyj.lyj_002
                  *
ERROR at line 1:
ORA-01578: ORACLE data block corrupted (file # 5, block # 629)
ORA-01110: data file 5: '/u01/app/oracle/oradata/orcl/lyj_01.dbf'

# alert log报错信息
Hex dump of (file 5, block 629) in trace file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_13840.trc
Corrupt block relative dba: 0x01400275 (file 5, block 629)
Fractured block found during multiblock buffer read
Data in bad block:
 type: 6 format: 2 rdba: 0x01400275
 last change scn: 0x0000.00271b7c seq: 0x1 flg: 0x06
 spare1: 0x0 spare2: 0x0 spare3: 0x0
 consistency value in tail: 0x1c7c0601
 check value in block header: 0x9e0a
 computed block checksum: 0x0
Reading datafile '/u01/app/oracle/oradata/orcl/lyj_01.dbf' for corruption at rdba: 0x01400275 (file 5, block 629)
Reread (file 5, block 629) found same corrupt data (no logical check)
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_13840.trc  (incident=24147):
ORA-01578: ORACLE data block corrupted (file # 5, block # 629)
ORA-01110: data file 5: '/u01/app/oracle/oradata/orcl/lyj_01.dbf'
Tue May 22 14:40:05 2018
Corrupt Block Found
         TSN = 5, TSNAME = LYJ_TS
         RFN = 5, BLK = 629, RDBA = 20972149
         OBJN = 18865, OBJD = 18865, OBJECT = LYJ_002, SUBOBJECT = 
         SEGMENT OWNER = LYJ, SEGMENT TYPE = Table Segment
Incident details in: /u01/app/oracle/diag/rdbms/orcl/orcl/incident/incdir_24147/orcl_ora_13840_i24147.trc
Corrupt Block Found
         TSN = 5, TSNAME = LYJ_TS
         RFN = 5, BLK = 629, RDBA = 20972149
         OBJN = 18865, OBJD = 18865, OBJECT = LYJ_002, SUBOBJECT = 
         SEGMENT OWNER = LYJ, SEGMENT TYPE = Table Segment
Tue May 22 14:40:07 2018
Dumping diagnostic data in directory=[cdmp_20180522144007], requested by (instance=1, osid=13840), summary=[incident=24147].
```

### The block checksum is invalid
Checksum是oracle写入后，其他外部因素导致块checksum改变的情况
Checksum只有DBWR进程写入，或者直接从磁盘读取
>**怎么计算出正确的checksum值：**
>- oracle在alert log里记录的computed block checksum值实际上并不是这个block被oracle计算出来的正确的checksum值，而是这个block被oracle计算出来的正确的checksum值与block的第16、17位记录的checksum值做异或操作后的值
>- 需要把block 16、17位清零，这样从alert log就可以看到oracle计算出来的checksum值了，而且这个时候alert log里的checksum值就是这个block被oracle计算出来的正确的checksum
>- 当把一个block的16、17位清零后，因为0与一个值做异或操作后还是等于那个值，所以在将16、17位清零后，这个时候alert log里的checksum值就是这个block被oracle计算出来的正确的checksum值了

故障模拟：
``` perl
conn lyj/lyj
create table lyj_003(id int,name varchar2(10));
insert into lyj_003 values (1,'AAAAAA');
commit;

select dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) block#
from lyj_003;

     FILE#     BLOCK#
---------- ----------
         5        637

alter system flush buffer_cache;

# 登陆BBED
BBED> set file 5 block 637
        FILE#           5
        BLOCK#          637

BBED> p chkval_kcbh
ub2 chkval_kcbh                             @16       0x9d9b       # checksum

# 找到第一行记录
BBED> p *kdbr[0]
rowdata[0]
----------
ub1 rowdata[0]                              @8175     0x2c

BBED>  x /rccnn
rowdata[0]                                  @8175    
----------
flag@8175: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8176: 0x01
cols@8177:    2

col    0[2] @8178: ..     # 第一行记录
col    1[6] @8181: AAAAAA

BBED> dump /v offset 16 count 16
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 637     Offsets:   16 to   31  Dba:0x0140027d
-------------------------------------------------------
 9b9d0000 01000000 bb490000 377a2700 l .........I..7z        # 16、17 ： 9b 9d

BBED> dump /v offset 8178 count 16
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 637     Offsets: 8178 to 8191  Dba:0x0140027d
-------------------------------------------------------    # col0 2个字节：c102 -> 1
 02c10206 41414141 41410106 387a     l ....AAAAAA..8z      # col1 6个字节：414141414141 -> AAAAAA

# 计算检验值
BBED> sum
Check value for File 5, Block 637:
current = 0x9d9b, required = 0x9d9b     # 与上面一致说明没有问题

# shutdown数据库
conn / as sysdba
shutdown immediate

# 用dd把5号文件637号块拷出来
dd if=/u01/app/oracle/oradata/orcl/lyj_01.dbf of=/tmp/disdb01.dd count=1 skip=637 bs=8192

# 传到本机上
cd /tmp/
sz disdb01.dd
```

用UE工具打开disdb01.dd，从下图可以看出位置16、17号是9B 9D
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_07.png)

定位以8175，将ID为1的记录的ID值由1改为2，后保存
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_04.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_03.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_05.png)
``` perl
SQL> select dump(1,16) from dual;

DUMP(1,16)
-----------------
Typ=2 Len=2: c1,2

SQL> select utl_raw.cast_to_number(replace('c1 03',' ')) from dual;  

UTL_RAW.CAST_TO_NUMBER(REPLACE('C103',''))
------------------------------------------
                                         2
```

改完后，把上述修改过的block给拷回去
``` perl
dd if=/tmp/disdb01.dd of=/u01/app/oracle/oradata/orcl/lyj_01.dbf bs=8192 seek=637 count=1 conv=notrunc
```

重启数据库，查看表报错
``` perl
startup

SQL> select * from lyj.lyj_003;
select * from lyj.lyj_003
                  *
ERROR at line 1:
ORA-01578: ORACLE data block corrupted (file # 5, block # 637)
ORA-01110: data file 5: '/u01/app/oracle/oradata/orcl/lyj_01.dbf'

# alert log
Corrupt block relative dba: 0x0140027d (file 5, block 637)
Bad check value found during validation
Data in bad block:
 type: 6 format: 2 rdba: 0x0140027d
 last change scn: 0x0000.00277a38 seq: 0x1 flg: 0x06
 spare1: 0x0 spare2: 0x0 spare3: 0x0
 consistency value in tail: 0x7a380601
 check value in block header: 0x9d9b
 computed block checksum: 0x1
Reread of blocknum=637, file=/u01/app/oracle/oradata/orcl/lyj_01.dbf. found same corrupt data
Reread of blocknum=637, file=/u01/app/oracle/oradata/orcl/lyj_01.dbf. found same corrupt data
```

从上面的alert log里可以看出以下两点：
1. 块头记录的checksum值是 9d9b, oracle这里做异或操作后的checksum的值是01
2. oracle当发现checksum值不对的时候会尝试再读一下该block

根据9d 9b和00 01来计算上面block在修改后正确的checksum值
9d9b = 1001 1101 1001 1011
0001 = 0000 0000 0000 0001
异或： 1001 1101 1001 1010  ->  9d9a

使用bbed改成新的checksum
``` perl
BBED> set file 5 block 637
        FILE#           5
        BLOCK#          637

BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/lyj_01.dbf
BLOCK = 637

Block 637 is corrupt    # 检验有块坏
Corrupt block relative dba: 0x0140027d (file 0, block 637)
Bad check value found during verification
Data in bad block:
 type: 6 format: 2 rdba: 0x0140027d
 last change scn: 0x0000.00277a38 seq: 0x1 flg: 0x06
 spare1: 0x0 spare2: 0x0 spare3: 0x0
 consistency value in tail: 0x7a380601
 check value in block header: 0x9d9b
 computed block checksum: 0x1


DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 0
Total Blocks Failing   (Data) : 0
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 1
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED

BBED> sum
Check value for File 5, Block 637:
current = 0x9d9b, required = 0x9d9a

BBED> map /v

BBED> p kcbh

BBED> p chkval_kcbh
ub2 chkval_kcbh                             @16       0x9d9b

BBED> dump /v offset 16 count 16
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 637     Offsets:   16 to   47  Dba:0x0140027d
-------------------------------------------------------
 9b9d0000 01000000 bb490000 377a2700 l .........I..7z.
 00000000 02003200 78024001 05001c00 l ......2.x.@.....

 <16 bytes per line>

BBED> modify /x 9a9d offset 16
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 637              Offsets:   16 to   47           Dba:0x0140027d
------------------------------------------------------------------------
 9a9d0000 01000000 bb490000 377a2700 00000000 02003200 78024001 05001c00 

BBED> sum apply
Check value for File 5, Block 637:
current = 0x9d9a, required = 0x9d9a

BBED> sum
Check value for File 5, Block 637:
current = 0x9d9a, required = 0x9d9a

BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/lyj_01.dbf
BLOCK = 637


DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 1
Total Blocks Failing   (Data) : 0
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 0
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED

# 再次查询时可以查出结果了，并且id值由1改成2了
SQL> select * from lyj.lyj_003;

        ID NAME
---------- ----------
         2 AAAAAA
```

### The block is misplaced
行锁错位的故障模拟：
``` perl
BBED> set file 5 block 637
        FILE#           5
        BLOCK#          637

BBED> p *kdbr
rowdata[0]
----------
ub1 rowdata[0]                              @8175     0x2c

BBED> x/ rnccc
rowdata[0]                                  @8175    
----------
flag@8175: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8176: 0x01       # 行锁
cols@8177:    2

col    0[2] @8178: 2 
col    1[6] @8181: AAAAAA

BBED> modify /x 00 offset 8176
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 637              Offsets: 8176 to 8191           Dba:0x0140027d
------------------------------------------------------------------------
 000202c1 03064141 41414141 0106387a 

BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/lyj_01.dbf
BLOCK = 637

Block Checking: DBA = 20972157, Block Type = KTB-managed data block
data header at 0x7f3984a36264
kdbchk: xaction header lock count mismatch     # 报行锁错位
        trans=1 ilk=1 nlo=0
Block 637 failed with check code 6108

DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 1
Total Blocks Failing   (Data) : 1
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 0
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED

# 经测试，虽然块出现行锁错位，但表还是能打开的和操作的
SQL> conn lyj/lyj
Connected.
SQL> select * from lyj_003;

        ID NAME
---------- ----------
         2 AAAAAA

SQL> insert into lyj_003 values (3,'BBBBBB');

1 row created.

SQL> commit;

Commit complete.
```

### Zeroed out blocks - ORA-8103
ORA-8103故障模拟：
``` perl
conn / as sysdba
drop user lyj cascade;
create tablespace disdb datafile '/u01/app/oracle/oradata/orcl/disdb01.dbf' size 50M;
create user lyj identified by lyj default tablespace disdb;
grant dba to lyj;

conn lyj/lyj
create table lyj_001(id int,name varchar2(100)) tablespace disdb;  
begin
  for i in 1 .. 5000 loop
  insert into lyj_001 values(i,'lyj'||i);
  commit;
  end loop;
end;
/

col NAME for a20
col SEGMENT_NAME for a20
SELECT owner, segment_name, EXTENT_ID, FILE_ID, BLOCK_ID, BLOCKS 
 FROM dba_extents 
 WHERE segment_name='LYJ_001' AND owner='LYJ';

OWNER                          SEGMENT_NAME          EXTENT_ID    FILE_ID   BLOCK_ID     BLOCKS
------------------------------ -------------------- ---------- ---------- ---------- ----------
LYJ                            LYJ_001                       0          6        128          8
LYJ                            LYJ_001                       1          6        136          8
  
SELECT DISTINCT dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) blk#
FROM LYJ_001  ORDER BY 1,2;

     FILE#       BLK#
---------- ----------
         6        131
         6        132
         6        133
         6        134
         6        135
         6        137
         6        138
         6        139
         6        140
         6        141
         6        142
         6        143

# 关闭数据库   
conn / as sysdba
shutdown immediate;

# 模拟块损坏
dd if=/dev/zero of=/u01/app/oracle/oradata/orcl/disdb01.dbf bs=8192 seek=140 count=1 conv=notrunc

# 启动数据库
startup

# 查看表时报块损坏错误
conn lyj/lyj
select count(*) from lyj_001;
select count(*) from lyj_001
                     *
ERROR at line 1:
ORA-01578: ORACLE data block corrupted (file # 6, block # 140)
ORA-01110: data file 6: '/u01/app/oracle/oradata/orcl/disdb01.dbf'
# 注意：在oracle 11g中使用该方法来模拟ORA-08103，直接提示坏块，不会出现ORA-08103
```

## 物理坏块的检测
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_13.png)

### dbv检测物理坏块
dbv全称是DBVERIFY，是oracle自带的一个工具
``` perl
$ dbv
#-------------------------------------------------------------------------------------
DBVERIFY: Release 11.2.0.4.0 - Production on Wed May 23 11:35:57 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

Keyword     Description                    (Default)
----------------------------------------------------
FILE        File to Verify                 (NONE)
START       Start Block                    (First Block of File)
END         End Block                      (Last Block of File)
BLOCKSIZE   Logical Block Size             (8192)
LOGFILE     Output Log                     (NONE)
FEEDBACK    Display Progress               (0)
PARFILE     Parameter File                 (NONE)
USERID      Username/Password              (NONE)
SEGMENT_ID  Segment ID (tsn.relfile.block) (NONE)
HIGH_SCN    Highest Block SCN To Verify    (NONE)
            (scn_wrap.scn_base OR scn) 
#-------------------------------------------------------------------------------------

$ dbv file=/u01/app/oracle/oradata/orcl/disdb01.dbf
#-------------------------------------------------------------------------------------
DBVERIFY: Release 11.2.0.4.0 - Production on Wed May 23 11:44:18 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/orcl/disdb01.dbf
Page 140 is marked corrupt       # 坏块标记
Corrupt block relative dba: 0x0180008c (file 6, block 140)
Completely zero block found during dbv: 


DBVERIFY - Verification complete

Total Pages Examined         : 6400
Total Pages Processed (Data) : 12
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 0
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 130
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 6257
Total Pages Marked Corrupt   : 1        # 坏块标记，这是物理坏块
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 2652450 (0.2652450)
#-------------------------------------------------------------------------------------
```

### RMAN检测物理坏块
``` perl
rman target /

RMAN> backup validate datafile 6;

Starting backup at 2018-05-23 11:48:20
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=68 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00006 name=/u01/app/oracle/oradata/orcl/disdb01.dbf
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:02
List of Datafiles
#=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
6    FAILED 0              6257         6400            2652450       # Status FAILED 说明6号文件有坏块
  File Name: /u01/app/oracle/oradata/orcl/disdb01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              12              
  Index      0              0               
  Other      1              131                # Blocks Failing

validate found one or more corrupt blocks
See trace file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_20217.trc for details
Finished backup at 2018-05-23 11:48:23

# trace中错误信息
Hex dump of (file 6, block 140)
Dump of memory from 0x00007FBB3C201000 to 0x00007FBB3C203000
7FBB3C201000 0000A200 0180008C 00000000 05010000  [................]
7FBB3C201010 0000A60C 00000000 00000000 00000000  [................]
7FBB3C201020 00000000 00000000 00000000 00000000  [................]
        Repeat 508 times
7FBB3C202FF0 00000000 00000000 00000000 00000001  [................]
Corrupt block relative dba: 0x0180008c (file 6, block 140)

# 如果有RMAN备份，使用以下命令恢复
# 恢复全部损坏的块
RMAN> recover corruption list;

# 恢复单个块
RMAN> recover datafile <fileno> block <block number> to <block number>;
RMAN> recover datafile 6 block 140;

# 根据dba地址恢复
select dbms_utility.make_data_block_address(6,140) from dual;
RMAN> recover tablespace <name> dba <integer value>;
```

### SQL检测物理坏块
``` perl
SQL> select * from v$database_block_corruption;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO
---------- ---------- ---------- ------------------ ---------
         5        623          1                  0 FRACTURED
         5        629          1                  0 FRACTURED
         5        637          1                  0 CHECKSUM
         6        140          1                  0 ALL ZERO   # ALL ZERO表示块内是空的

# 如果字段CORRUPTION_CHANGE#为0，表示是物理坏块，如果是非0值，表示逻辑坏块
```

CORRUPTION_TYPE：
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_18.png)

### 使用dbms_repair包检测物理坏块
``` perl
# 创建repair table
conn / as sysdba
# drop table REPAIR_TABLE purge;
BEGIN
   DBMS_REPAIR.ADMIN_TABLES (
   TABLE_NAME => 'REPAIR_TABLE',
   TABLE_TYPE => dbms_repair.repair_table,
   ACTION => dbms_repair.create_action,
   TABLESPACE => 'DISDB');
END;
/

# 检查对象上是否存在坏块
set serveroutput on
DECLARE num_corrupt INT;
BEGIN
num_corrupt := 0;
DBMS_REPAIR.CHECK_OBJECT (
SCHEMA_NAME  => 'LYJ',
OBJECT_NAME  => 'LYJ_001',
REPAIR_TABLE_NAME  => 'REPAIR_TABLE',
corrupt_count  => num_corrupt);
DBMS_OUTPUT.PUT_LINE('number corrupt: ' || TO_CHAR (num_corrupt));
END;
/

number corrupt: 1

PL/SQL procedure successfully completed.

# 查看REPAIR_TABLE
col CORRUPT_DESCRIPTION for a100
col OBJECT_NAME for a15
select OBJECT_ID,RELATIVE_FILE_ID,BLOCK_ID,CORRUPT_TYPE,OBJECT_NAME,CORRUPT_DESCRIPTION from repair_table;

 OBJECT_ID RELATIVE_FILE_ID   BLOCK_ID CORRUPT_TYPE OBJECT_NAME     CORRUPT_DESCRIPTION
---------- ---------------- ---------- ------------ --------------- ---------------------------------
     18975                6        140         6148 LYJ_001

# 标记坏块
declare       
  fix_count int;
  begin
  fix_count := 0;
  dbms_repair.fix_corrupt_blocks (
  schema_name  => 'LYJ',
  object_name  => 'LYJ_001',
  object_type  => dbms_repair.table_object,
 repair_table_name => 'REPAIR_TABLE',
 fix_count  => fix_count);
 dbms_output.put_line('fix count: ' || to_char(fix_count));
 end;
/

# 设置全表扫描跳过标记的坏块
begin
  dbms_repair.skip_corrupt_blocks (
  schema_name  => 'LYJ',
  object_name  => 'LYJ_001',
 object_type  => dbms_repair.table_object,
 flags =>  dbms_repair.skip_flag);
 end;
 /
```

## 逻辑坏块的模拟
### 逻辑块坏
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_14.png)

### 逻辑数据坏块的场景
1. oracle bug可能导致逻辑坏块的产生，特别是parallel dml，例如：
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_15.png)
2. 多数情况下逻辑坏块可能都是软件问题导致的，当然数据库异常可能导致，比如突然断电的情况下，就可能导致块内数据不一致

### 逻辑坏块的分类
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_20.png)

### 模拟逻辑坏块
很多情况下逻辑坏块都发生在索引上，下面以模拟索引逻辑坏块为例：
``` perl
conn lyj/lyj
create table tab_logical_block_corruption as select owner,object_id,object_name from dba_objects where rownum < 200;
create index index_id on tab_logical_block_corruption(object_id);
select DISTINCT dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) blk# 
  from tab_logical_block_corruption;

     FILE#       BLK#
---------- ----------
         6        147

select owner,object_id from dba_objects where object_name=upper('index_id');

OWNER                           OBJECT_ID
------------------------------ ----------
LYJ                                 18981

# dump这个索引对象
alter session set events 'immediate trace name treedump level 18981';

select value from v$diag_info where name='Default Trace File';

VALUE
---------------------------------------------------------------
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_20528.trc

# 查看这个dump出来的trace文件
#---------------------------------------------------------------
----- begin tree dump
leaf: 0x180009b 25165979 (0: nrow: 199 rrow: 199) # 叶子所在的地址
----- end tree dump
#---------------------------------------------------------------

# 通过索引叶子所在地址，以下两种方法都可以获得该索引所在文件号和块号
select dbms_utility.data_block_address_file(TO_NUMBER('180009b', 'XXXXXXXX')) file_id,
dbms_utility.data_block_address_block(TO_NUMBER('180009b', 'XXXXXXXX')) block_id from dual;

   FILE_ID   BLOCK_ID
---------- ----------
         6        155

SQL> select getbfno('180009b') from dual;

GETBFNO('180009B')
----------------------------------------------------------------------------------------------------
datafile# is:6
datablock is:155
dump command:alter system dump datafile 6 block 155;

# 进入BBED
BBED> set file 6 block 155
        FILE#           6
        BLOCK#          155

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (6)
 Block: 155                                   Dba:0x0180009b
------------------------------------------------------------
 KTB Data Block (Index Leaf)          # 索引叶子
......

BBED> p kdxle    # 叶子块的结构
struct kdxle, 32 bytes                      @100     
   struct kdxlexco, 16 bytes                @100     
      ub1 kdxcolev                          @100      0x00
      ub1 kdxcolok                          @101      0x00
      ub1 kdxcoopc                          @102      0x80
      ub1 kdxconco                          @103      0x02
      ub4 kdxcosdc                          @104      0x00000000
      sb2 kdxconro                          @108      199        # 记录数，共199行，下面改成198
      sb2 kdxcofbo                          @110      434
      sb2 kdxcofeo                          @112      5545
      sb2 kdxcoavs                          @114      5111
   sb2 kdxlespl                             @116      0
   sb2 kdxlende                             @118      0
   ub4 kdxlenxt                             @120      0x00000000
   ub4 kdxleprv                             @124      0x00000000
   ub1 kdxledsz                             @128      0x00
   ub1 kdxleflg                             @129      0x00 (NONE)

BBED> dump /v offset 108 count 16
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (6)
 Block: 155     Offsets:  108 to  123  Dba:0x0180009b
-------------------------------------------------------
 c700b201 a915f713 00000000 00000000 l ................       # c7

SQL> select to_char(199,'xxxxx') from dual;

TO_CHA
------
    c7

SQL> select to_char(198,'xxxxx') from dual;

TO_CHA
------
    c6

BBED> modify /x c6 offset 108
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (6)
 Block: 155              Offsets:  108 to  123           Dba:0x0180009b
------------------------------------------------------------------------
 c600b201 a915f713 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 155:
current = 0x3146, required = 0x3146

# 重启数据库
conn / as sysdba
shutdown immediate
startup
conn lyj/lyj
```

## 逻辑坏块的检测
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_16.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_19.png)

>备注：
* RMAN备份会忽略Soft Corruption
* Soft Corrupt的块不计入MAXCORRUPT
* Media Recovery 会忽略Soft Corrupt
* Rman validate命令不会在alert中记录Soft Corrupt的信息，但是会在`v$database_block_corruption`中记录
* DBV可以检测Soft Corruption
* 如果不设置event 10231或者类似事件，软损坏的块再次访问时报ORA-1578

### 使用analyze检查
``` perl
SQL> analyze table tab_logical_block_corruption validate structure cascade online;
analyze table tab_logical_block_corruption validate structure cascade online
*
ERROR at line 1:
ORA-01499: table/index cross reference failure - see trace file
```

### 使用rman检查
``` perl
rman target /

RMAN> backup validate check logical database;
#---------------------------------------------------------------------------------------
Starting backup at 2018-05-23 13:57:26
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=67 device type=DISK
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
input datafile file number=00001 name=/u01/app/oracle/oradata/orcl/system01.dbf
input datafile file number=00002 name=/u01/app/oracle/oradata/orcl/sysaux01.dbf
input datafile file number=00003 name=/u01/app/oracle/oradata/orcl/undotbs01.dbf
input datafile file number=00005 name=/u01/app/oracle/oradata/orcl/lyj_01.dbf
input datafile file number=00006 name=/u01/app/oracle/oradata/orcl/disdb01.dbf
input datafile file number=00004 name=/u01/app/oracle/oradata/orcl/users01.dbf
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:07
List of Datafiles
#=================
......
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
6    FAILED 0              6249         6400            2658373   
  File Name: /u01/app/oracle/oradata/orcl/disdb01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              13              
  Index      1              1              # 6号文件索引有块块 
  Other      1              137             

validate found one or more corrupt blocks
See trace file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_20658.trc for details
channel ORA_DISK_1: starting full datafile backup set
channel ORA_DISK_1: specifying datafile(s) in backup set
including current control file in backup set
including current SPFILE in backup set
channel ORA_DISK_1: backup set complete, elapsed time: 00:00:01
List of Control File and SPFILE
#===============================
File Type    Status Blocks Failing Blocks Examined
------------ ------ -------------- ---------------
SPFILE       OK     0              2               
Control File OK     0              594             
Finished backup at 2018-05-23 13:57:35
#---------------------------------------------------------------------------------------
```

### 查看视图
``` perl
SQL> select * from v$database_block_corruption;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO
---------- ---------- ---------- ------------------ ---------
         5        623          1                  0 FRACTURED
         5        629          1                  0 FRACTURED
         5        637          1                  0 CHECKSUM
         6        140          1                  0 ALL ZERO
         6        155          1            2658375 CORRUPT      # CORRUPTION_CHANGE#=2658375大于0，表示是逻辑坏块

# 通过上面的文件号和块号，可以找到坏块所在的对象
set autot off
set lines 150
col segment_name for a15
col owner for a20
SELECT tablespace_name, segment_type, owner, segment_name
FROM dba_extents
WHERE file_id = 6
and 155 between block_id AND block_id + blocks - 1;

TABLESPACE_NAME                SEGMENT_TYPE       OWNER                SEGMENT_NAME
------------------------------ ------------------ -------------------- ---------------
DISDB                          INDEX              LYJ                  INDEX_ID
```

### DBV检测逻辑坏块
``` perl
$ dbv file=/u01/app/oracle/oradata/orcl/disdb01.dbf
#---------------------------------------------------------------------------------------
DBVERIFY: Release 11.2.0.4.0 - Production on Wed May 23 14:16:07 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/orcl/disdb01.dbf
Page 140 is marked corrupt
Corrupt block relative dba: 0x0180008c (file 6, block 140)
Completely zero block found during dbv: 

Block Checking: DBA = 25165979, Block Type = KTB-managed data block
**** kdxcofbo = 434 != 428        # 逻辑坏块 位置不对
---- end index block validation    
Page 155 failed with check code 6401


DBVERIFY - Verification complete

Total Pages Examined         : 6400
Total Pages Processed (Data) : 13
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 1      # 判断出是index block
Total Pages Failing   (Index): 1      # index block Failing
Total Pages Processed (Other): 136  
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 6249
Total Pages Marked Corrupt   : 1
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 2658375 (0.2658375)
#---------------------------------------------------------------------------------------
```

### 使用dbms_repair包检测
``` perl
# 创建repair table
conn / as sysdba
drop table REPAIR_TABLE purge;
BEGIN
   DBMS_REPAIR.ADMIN_TABLES (
   TABLE_NAME => 'REPAIR_TABLE',
   TABLE_TYPE => dbms_repair.repair_table,
   ACTION => dbms_repair.create_action,
   TABLESPACE => 'DISDB');
END;
/

# 检查对象上是否存在坏块
set serveroutput on
DECLARE num_corrupt INT;
BEGIN
num_corrupt := 0;
DBMS_REPAIR.CHECK_OBJECT (
SCHEMA_NAME  => 'LYJ',
OBJECT_NAME  => 'TAB_LOGICAL_BLOCK_CORRUPTION',
REPAIR_TABLE_NAME  => 'REPAIR_TABLE',
corrupt_count  => num_corrupt);
DBMS_OUTPUT.PUT_LINE('number corrupt: ' || TO_CHAR (num_corrupt));
END;
/

number corrupt: 0

PL/SQL procedure successfully completed.
# 这里没有检测出逻辑坏块
```



## 数据坏块相关参数
### db_block_checksum
``` perl
SQL> show parameter db_block_checksum

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_block_checksum                    string      TYPICAL
```

![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_08.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_09.png)

>db_block_checksum该参数注意是控制dbwn进程在将block写入到磁盘时，是否根据存储在block的byte大小进行估算一个checksum值，并将其写入到data block的cache header中。该参数可以在很大程度上避免坏块的产生，但是会额外资源消耗。
- 当设置`typical`时，会增加`1~2%`的资源消耗
- 当设置为`full` 时，会增加`4~5%`的资源消耗

### db_block_checking
``` perl
SQL> show parameter db_block_checking

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_block_checking                    string      FALSE
SQL> 
```

![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_10.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_11.png)

>db_block_checking参数默认关闭，该参数的设置，可能增加`1~10%`的资源消耗

### db_block_checksum和db_block_checking的性能影响
![](http://p2c0rtsgc.bkt.clouddn.com/0522_oracle_dsi_12.png)

## 重现数据块内空间计算错误
``` perl
conn lyj/lyj
create table lyj_004(id int,name varchar2(10));
insert into lyj_004 values (1,'AAAAAA');
commit;

select dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) block#
from lyj_004;

     FILE#     BLOCK#
---------- ----------
         5        157

alter system flush buffer_cache;

BBED> set file 5 block 157
        FILE#           5
        BLOCK#          157

BBED> p kdbh
struct kdbh, 14 bytes                       @100     
   ub1 kdbhflag                             @100      0x00 (NONE)
   sb1 kdbhntab                             @101      1
   sb2 kdbhnrow                             @102      1
   sb2 kdbhfrre                             @104     -1
   sb2 kdbhfsbo                             @106      20
   sb2 kdbhfseo                             @108      8075
   sb2 kdbhavsp                             @110      8055    # 改此值
   sb2 kdbhtosp                             @112      8055    # 改此值


BBED> dump /v offset 110
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (5)
 Block: 157     Offsets:  110 to  125  Dba:0x0140009d
-------------------------------------------------------
 771f771f 00000100 8b1f0000 00000000 l w.w.............

 <16 bytes per line>

SQL> select to_char(8055,'xxxxx') from dual;

TO_CHA
------
  1f77

SQL> select to_char(8054,'xxxxx') from dual;

TO_CHA
------
  1f76      ->  761f

BBED> modify /x 761f offset 110
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (5)
 Block: 157              Offsets:  110 to  125           Dba:0x0140009d
------------------------------------------------------------------------
 761f781f 00000100 8b1f0000 00000000 

 <32 bytes per line>

BBED> modify /x 761f offset 112
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (5)
 Block: 157              Offsets:  112 to  127           Dba:0x0140009d
------------------------------------------------------------------------
 761f0000 01008b1f 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 5, Block 157:
current = 0x90ad, required = 0x90ad

BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/disdb01.dbf
BLOCK = 157

Block Checking: DBA = 20971677, Block Type = KTB-managed data block
data header at 0x169ba64
kdbchk: the amount of space used is not equal to block size
        used=33 fsc=0 avsp=8054 dtl=8088      # 记录  dtl=8088  used=33
Block 157 failed with check code 6110

DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 1
Total Blocks Failing   (Data) : 1
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 0
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED

dbv file=/u01/app/oracle/oradata/orcl/disdb01.dbf
#---------------------------------------------------------------------------------------
DBVERIFY: Release 11.2.0.4.0 - Production on Wed May 23 17:35:26 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/orcl/disdb01.dbf
Block Checking: DBA = 20971677, Block Type = KTB-managed data block
data header at 0x7ff0dcbfb064
kdbchk: the amount of space used is not equal to block size
        used=33 fsc=0 avsp=8054 dtl=8088
Page 157 failed with check code 6110


DBVERIFY - Verification complete

Total Pages Examined         : 6400
Total Pages Processed (Data) : 23
Total Pages Failing   (Data) : 1
Total Pages Processed (Index): 0
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 136
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 6241
Total Pages Marked Corrupt   : 0
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 2707283 (0.2707283)
#---------------------------------------------------------------------------------------
```

解决错误
``` perl
SQL> select 8088-33 from dual;

   8088-33
----------
      8055

SQL> select to_char(8055,'xxxxxx') from dual;

TO_CHAR
-------
   1f77   -> 771f

BBED> p kdbh
struct kdbh, 14 bytes                       @100     
   ub1 kdbhflag                             @100      0x00 (NONE)
   sb1 kdbhntab                             @101      1
   sb2 kdbhnrow                             @102      1
   sb2 kdbhfrre                             @104     -1
   sb2 kdbhfsbo                             @106      20
   sb2 kdbhfseo                             @108      8075
   sb2 kdbhavsp                             @110      8054
   sb2 kdbhtosp                             @112      8054

BBED> modify /x 771f offset 110
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (5)
 Block: 157              Offsets:  110 to  125           Dba:0x0140009d
------------------------------------------------------------------------
 771f761f 00000100 8b1f0000 00000000 

 <32 bytes per line>

BBED> modify /x 771f offset 112
 File: /u01/app/oracle/oradata/orcl/disdb01.dbf (5)
 Block: 157              Offsets:  112 to  127           Dba:0x0140009d
------------------------------------------------------------------------
 771f0000 01008b1f 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 5, Block 157:
current = 0x90ad, required = 0x90ad

BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/disdb01.dbf
BLOCK = 157


DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 1
Total Blocks Failing   (Data) : 0
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 0
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED
```