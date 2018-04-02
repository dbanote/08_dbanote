---
title: Oracle特殊恢复原理与实战_06 使用BBED手工修复block数据
date: 2018-03-28
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## Oracle 11g Data Block Layout
![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_01.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_02.png)

<!-- more -->

![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_03.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_04.png)

### KCBH
![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_05.png)
``` perl
SQL> conn lyj/lyj
SQL> select ROWID,ID,NAME,DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) FILE#,DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) BLOCK# from t1;

ROWID                      ID NAME            FILE#     BLOCK#
------------------ ---------- ---------- ---------- ----------
AAAW0oAAGAAAACDAAA          1 AAAAAA              6        131
AAAW0oAAGAAAACDAAB          2 BBBBB               6        131

# 进入BBED
BBED> set file 6 block 131
        FILE#           6
        BLOCK#          131

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 131                                   Dba:0x01800083
------------------------------------------------------------
 KTB Data Block (Table/Cluster)

 struct kcbh, 20 bytes                      @0       
    ub1 type_kcbh                           @0       
    ub1 frmt_kcbh                           @1       
    ub1 spare1_kcbh                         @2       
    ub1 spare2_kcbh                         @3       
    ub4 rdba_kcbh                           @4       
    ub4 bas_kcbh                            @8       
    ub2 wrp_kcbh                            @12      
    ub1 seq_kcbh                            @14      
    ub1 flg_kcbh                            @15      
    ub2 chkval_kcbh                         @16      
    ub2 spare3_kcbh                         @18

BBED> p kcbh
struct kcbh, 20 bytes                       @0       
   ub1 type_kcbh                            @0        0x06    # 06代表数据块
   ub1 frmt_kcbh                            @1        0xa2    # 0xa2代表块的大小是8k
   ub1 spare1_kcbh                          @2        0x00    # 保留值
   ub1 spare2_kcbh                          @3        0x00    # 保留值
   ub4 rdba_kcbh                            @4        0x01800083 # RDBA相对数据块地址 6号文件的131号块（转换方法在下面）
   ub4 bas_kcbh                             @8        0x0037066e # 块头的SCN(低32位) 
   ub2 wrp_kcbh                             @12       0x0000     # 块头的SCN(高16位)
   ub1 seq_kcbh                             @14       0x02       # seq最大值254
   ub1 flg_kcbh                             @15       0x06 (KCBHFDLC, KCBHFCKV)
   ub2 chkval_kcbh                          @16       0x9200     # db_block_checksum 
   ub2 spare3_kcbh                          @18       0x0000     # 保留值
```

#### rdba转换成文件号块号的方法
``` perl
rdba：0x01800083   
      0000 0001 1000 0000 0000 0000 1000 0011
前10位文件号：0000 0001 10  -  6号文件
后22位块号：  00 0000 0000 0000 1000 0011 - 131号块
```

#### RDBA转换函数
``` perl
# 下以函数用来从RDBA中转换file#和block#出来
CREATE OR REPLACE FUNCTION getbfno (p_dba IN VARCHAR2)
   RETURN VARCHAR2
IS
   l_str   VARCHAR2 (255) DEFAULT NULL;
   l_fno   VARCHAR2 (15);
   l_bno   VARCHAR2 (15);
BEGIN
   l_fno :=
      DBMS_UTILITY.data_block_address_file (TO_NUMBER (LTRIM (p_dba, '0x'),
                                                       'xxxxxxxx'
                                                      )
                                           );
   l_bno :=
      DBMS_UTILITY.data_block_address_block (TO_NUMBER (LTRIM (p_dba, '0x'),
                                                        'xxxxxxxx'
                                                       )
                                            );
   l_str :=
         'datafile# is:'
      || l_fno
      || CHR (10)
      || 'datablock is:'
      || l_bno
      || CHR (10)
      || 'dump command:alter system dump datafile '
      || l_fno
      || ' block '
      || l_bno
      || ';';
   RETURN l_str;
END;
/

SQL> select getbfno('0x01800083') BFNO from dual;

BFNO
--------------------------------------------------------------------------------
datafile# is:6
datablock is:131
dump command:alter system dump datafile 6 block 131;
```

#### scn wrap和scn
在Oracle内部，SCN分为两部分存储，分别称之为scn wrap和scn base。 
实际上SCN长度为48位，即它其实就是一个48位的整数。只不过可能是由于在早些年通常只能处理32位甚至是16位的数据，所以人为地分成了低32位（scnbase）和高16位（scn wrap）。
为什么不设计成64位，这个或许是觉得48位已经足够长了并且为了节省两个字节的空间：）。那么SCN这个48位长的整数，最大就是2^48（2的48次方, 281万亿，281474976710656），很大的一个数字了。 这里有一个重要的公式：`SCN= (SCN_WRP * 4294967296) + SCN_BAS`

### KTBBH: Transaction Layer
![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_06.png)
``` perl
# DUMP的结构
Block header dump:  0x01800083
 Object id on Block? Y
 seg/obj: 0x16d28  csc: 0x00.367d20  itc: 2  flg: E  typ: 1 - DATA
     brn: 0  bdba: 0x1800080 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0009.01a.00000954  0x00c06012.03cd.1d  --U-    1  fsc 0x0000.00367d21
0x02   0x0009.00c.00000965  0x00c06012.03d1.37  --U-    1  fsc 0x0000.0037066e
bdba: 0x01800083
data_block_dump,data header at 0x7f4671807a64

# BBED
BBED> p ktbbh
struct ktbbh, 72 bytes                      @20      
   ub1 ktbbhtyp                             @20       0x01 (KDDBTDATA)
   union ktbbhsid, 4 bytes                  @24      
      ub4 ktbbhsg1                          @24       0x00016d28
      ub4 ktbbhod1                          @24       0x00016d28
   struct ktbbhcsc, 8 bytes                 @28      
      ub4 kscnbas                           @28       0x00367d20
      ub2 kscnwrp                           @32       0x0000
   sb2 ktbbhict                             @36      -2046
   ub1 ktbbhflg                             @38       0x32 (NONE)
   ub1 ktbbhfsl                             @39       0x00
   ub4 ktbbhfnx                             @40       0x01800080
   struct ktbbhitl[0], 24 bytes             @44      
      struct ktbitxid, 8 bytes              @44      
         ub2 kxidusn                        @44       0x0009
         ub2 kxidslt                        @46       0x001a
         ub4 kxidsqn                        @48       0x00000954
      struct ktbituba, 8 bytes              @52      
         ub4 kubadba                        @52       0x00c06012
         ub2 kubaseq                        @56       0x03cd
         ub1 kubarec                        @58       0x1d
      ub2 ktbitflg                          @60       0x2001 (KTBFUPB)
      union _ktbitun, 2 bytes               @62      
         sb2 _ktbitfsc                      @62       0
         ub2 _ktbitwrp                      @62       0x0000
      ub4 ktbitbas                          @64       0x00367d21
   struct ktbbhitl[1], 24 bytes             @68      
      struct ktbitxid, 8 bytes              @68      
         ub2 kxidusn                        @68       0x0009
         ub2 kxidslt                        @70       0x000c
         ub4 kxidsqn                        @72       0x00000965
      struct ktbituba, 8 bytes              @76      
         ub4 kubadba                        @76       0x00c06012
         ub2 kubaseq                        @80       0x03d1
         ub1 kubarec                        @82       0x37
      ub2 ktbitflg                          @84       0x2001 (KTBFUPB)
      union _ktbitun, 2 bytes               @86      
         sb2 _ktbitfsc                      @86       0
         ub2 _ktbitwrp                      @86       0x0000
      ub4 ktbitbas                          @88       0x0037066e
```

### KDBH
![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_07.png)
``` perl
# dump
tsiz: 0x1f98
hsiz: 0x16
pbl: 0x7f4671807a64
     76543210
flag=--------
ntab=1
nrow=2
frre=-1
fsbo=0x16
fseo=0x1f7f
avsp=0x1f69
tosp=0x1f69
0xe:pti[0]      nrow=2  offs=0
0x12:pri[0]     offs=0x1f8b
0x14:pri[1]     offs=0x1f7f
block_row_dump:
tab 0, row 0, @0x1f8b
tl: 13 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 2]  c1 02
col  1: [ 6]  41 41 41 41 41 41
tab 0, row 1, @0x1f7f
tl: 12 fb: --H-FL-- lb: 0x2  cc: 2
col  0: [ 2]  c1 03
col  1: [ 5]  42 42 42 42 42
end_of_block_dump

# bbed
BBED> p kdbr
sb2 kdbr[0]                                 @118      8075
sb2 kdbr[1]                                 @120      8063

BBED> p *kdbr[0]
rowdata[12]
-----------
ub1 rowdata[12]                             @8175     0x2c

BBED> x/rncccccc
rowdata[12]                                 @8175    
-----------
flag@8175: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8176: 0x01
cols@8177:    2

col    0[2] @8178: 1 
col    1[6] @8181: AAAAAA


BBED>  p *kdbr[1]
rowdata[0]
----------
ub1 rowdata[0]                              @8163     0x2c

BBED> x/rncccccc
rowdata[0]                                  @8163    
----------
flag@8163: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8164: 0x02
cols@8165:    2

col    0[2] @8166: 2 
col    1[5] @8169: BBBBB

# 转换语句
SQL> select dump('AAAAAA',16) from dual;

DUMP('AAAAAA',16)
-------------------------------
Typ=96 Len=6: 41,41,41,41,41,41
```

## Oracle ROWID格式解析
![](http://p2c0rtsgc.bkt.clouddn.com/0327_oracle_dsi_08.png)
``` perl
SQL> select ROWID,ID,NAME,DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) FILE#,DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) BLOCK# from t1;

ROWID                      ID NAME            FILE#     BLOCK#
------------------ ---------- ---------- ---------- ----------
AAAW0oAAGAAAACDAAA          1 AAAAAA              6        131
AAAW0oAAGAAAACDAAB          2 BBBBB               6        131
```

## 数据行格式解析
![](http://p2c0rtsgc.bkt.clouddn.com/0329_oracle_dsi_01.png)
``` perl
# dump trace中数据行格式解析
tl: 13 fb: --H-FL-- lb: 0x1  cc: 2
col  0: [ 2]  c1 02
col  1: [ 6]  41 41 41 41 41 41
fb: --H-FL-- 即是开始F，又是结束L，但表只有一行记录
lb: 0x1 1号事务槽
cc: 2 有两列
col  0: [ 2]  c1 02: 第一列
col  1: [ 6]  41 41 41 41 41 41：第二列

# BBED中数据行解析
2c 010202c1 02064141 41414141
2c - 行的状态，行正常的话都是2c，如果是3c的话，表示该行被删除了
01 - lb，锁的字节标记，代表在哪个事务槽上面，01代表在第一个事务槽
02 - 代表有2列
02 - 第1个列有2个字节组成
c1 02 - 第1列的2个字节c1 02 代表第1列对应的数据为1
06 - 第2个列有6个字节组成
4141 41414141 - 第2个列6个字节4141 41414141 ，对应的数值为AAAAAA
```

## 实例
### insert一条记录，没有提交事务，会写入Datablock吗？
会写入，验证步骤如下：
``` perl
conn lyj/lyj
create table t2 (id number,name varchar2(10));
insert into t2 values (1, 'AAAAAA');
select ROWID,ID,NAME,DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) FILE#,DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) BLOCK# from t2;

ROWID                      ID NAME            FILE#     BLOCK#
------------------ ---------- ---------- ---------- ----------
AAAXCgAAGAAAACNAAA          1 AAAAAA              6        141

# 打开另一个会话
alter system flush buffer_cache;
alter system dump datafile 6 block 141;
select VALUE from v$diag_info where NAME='Default Trace File';

VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_23817.trc

# 打开trace文件
#--------------------------------------------------------------------------------
Block header dump:  0x0180008d
 Object id on Block? Y
 seg/obj: 0x170a0  csc: 0x00.3ddad2  itc: 2  flg: E  typ: 1 - DATA
     brn: 0  bdba: 0x1800088 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc，
0x01   0x0002.019.00000a13  0x00c011b6.034d.14  ----    1  fsc 0x0000.00000000  # 第一个事务槽，flag:---- 代表没有提交
0x02   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
bdba: 0x0180008d
data_block_dump,data header at 0x7f4560e76a64
.......
block_row_dump:
tab 0, row 0, @0x1f8b
tl: 13 fb: --H-FL-- lb: 0x1  cc: 2  # 0x1 查看上面第一个事务槽
col  0: [ 2]  c1 02                 # 1
col  1: [ 6]  41 41 41 41 41 41     # AAAAAA
end_of_block_dump
End dump data blocks tsn: 7 file#: 6 minblk 141 maxblk 141
#--------------------------------------------------------------------------------

FLAG: C=Committed; U=Commit Upper Bound; T=Active as CSC

# BBED
BBED> set file 6 block 141

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 141                                   Dba:0x0180008d
------------------------------------------------------------
 KTB Data Block (Table/Cluster)

 struct kcbh, 20 bytes                      @0       
    ub1 type_kcbh                           @0       
    ub1 frmt_kcbh                           @1       
    ub1 spare1_kcbh                         @2       
    ub1 spare2_kcbh                         @3       
    ub4 rdba_kcbh                           @4       
    ub4 bas_kcbh                            @8       
    ub2 wrp_kcbh                            @12      
    ub1 seq_kcbh                            @14      
    ub1 flg_kcbh                            @15      
    ub2 chkval_kcbh                         @16      
    ub2 spare3_kcbh                         @18      

 struct ktbbh, 72 bytes                     @20       # 事务   
    ub1 ktbbhtyp                            @20      
    union ktbbhsid, 4 bytes                 @24      
    struct ktbbhcsc, 8 bytes                @28      
    sb2 ktbbhict                            @36      
    ub1 ktbbhflg                            @38      
    ub1 ktbbhfsl                            @39      
    ub4 ktbbhfnx                            @40      
    struct ktbbhitl[2], 48 bytes            @44
.....

BBED> p ktbbh
struct ktbbh, 72 bytes                      @20      
   ub1 ktbbhtyp                             @20       0x01 (KDDBTDATA)
   union ktbbhsid, 4 bytes                  @24      
      ub4 ktbbhsg1                          @24       0x000170a0
      ub4 ktbbhod1                          @24       0x000170a0
   struct ktbbhcsc, 8 bytes                 @28      
      ub4 kscnbas                           @28       0x003ddad2
      ub2 kscnwrp                           @32       0x0000
   sb2 ktbbhict                             @36       2
   ub1 ktbbhflg                             @38       0x32 (NONE)
   ub1 ktbbhfsl                             @39       0x00
   ub4 ktbbhfnx                             @40       0x01800088
   struct ktbbhitl[0], 24 bytes             @44                    # 第一个事务槽
      struct ktbitxid, 8 bytes              @44      
         ub2 kxidusn                        @44       0x0002
         ub2 kxidslt                        @46       0x0019
         ub4 kxidsqn                        @48       0x00000a13
      struct ktbituba, 8 bytes              @52      
         ub4 kubadba                        @52       0x00c011b6
         ub2 kubaseq                        @56       0x034d
         ub1 kubarec                        @58       0x14
      ub2 ktbitflg                          @60       0x0001 (NONE)   # 00代表没提交，01代表在偏移量60的地方锁了1行记录
                                                                      # 前两位是80代表提交，20代表快速提交，
.....

BBED> set offset 0 count 8192
BBED> dump
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 141              Offsets:    0 to 8191           Dba:0x0180008d
------------------------------------------------------------------------
 06a20000 8d008001 d2da3d00 00000304 cc440000 01000000 a0700100 d2da3d00 
 00000000 02003200 88008001 02001900 130a0000 b611c000 4d031400 01000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00010100 ffff1400 8b1f771f 771f0000 01008b1f 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
......
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 0000002c 010202c1 02064141 41414141 0306d2da   # 2c 010202c1 02064141 41414141 这个就是没提交insert的值
```

### 使用BBED手工修复DELETE数据
#### 先看一下未删除前数据块内容
``` perl
# 提交上面insert的数据
commit;
alter system flush buffer_cache;

# 退出重新登陆BBED
BBED> set file 6 block 141

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 141                                   Dba:0x0180008d
------------------------------------------------------------
 KTB Data Block (Table/Cluster)

 struct kcbh, 20 bytes                      @0       
    ub1 type_kcbh                           @0       
    ub1 frmt_kcbh                           @1       
    ub1 spare1_kcbh                         @2       
    ub1 spare2_kcbh                         @3       
    ub4 rdba_kcbh                           @4       
    ub4 bas_kcbh                            @8       
    ub2 wrp_kcbh                            @12      
    ub1 seq_kcbh                            @14      
    ub1 flg_kcbh                            @15      
    ub2 chkval_kcbh                         @16      
    ub2 spare3_kcbh                         @18      

 struct ktbbh, 72 bytes                     @20      
    ub1 ktbbhtyp                            @20      
    union ktbbhsid, 4 bytes                 @24      
    struct ktbbhcsc, 8 bytes                @28      
    sb2 ktbbhict                            @36      
    ub1 ktbbhflg                            @38      
    ub1 ktbbhfsl                            @39      
    ub4 ktbbhfnx                            @40      
    struct ktbbhitl[2], 48 bytes            @44    #
.....

BBED> p ktbbhitl
struct ktbbhitl[0], 24 bytes                @44      
   struct ktbitxid, 8 bytes                 @44      
      ub2 kxidusn                           @44       0x0002
      ub2 kxidslt                           @46       0x0019
      ub4 kxidsqn                           @48       0x00000a13
   struct ktbituba, 8 bytes                 @52      
      ub4 kubadba                           @52       0x00c011b6
      ub2 kubaseq                           @56       0x034d
      ub1 kubarec                           @58       0x14
   ub2 ktbitflg                             @60       0x8000 (KTBFCOM)  # 80代表提交 00 没有锁行
   union _ktbitun, 2 bytes                  @62      
      sb2 _ktbitfsc                         @62       0
      ub2 _ktbitwrp                         @62       0x0000
   ub4 ktbitbas                             @64       0x003de705

BBED> p kdbr
sb2 kdbr[0]                                 @118      8075

BBED> p *kdbr[0]
rowdata[0]
----------
ub1 rowdata[0]                              @8175     0x2c   # 绝对位置：上面相对位置+100 (ASSM)
                                                             # ASSM +100   MSSM +92  详细算法见下面ASSM和MSSM下block结构的一点差异图

BBED> x/rnccccccc       # r-读 n-数值型 c-字符型  
rowdata[0]                                  @8175    
----------
flag@8175: 0x2c (KDRHFL, KDRHFF, KDRHFH)  # 2c 行的状态
lock@8176: 0x00
cols@8177:    2

col    0[2] @8178: 1 
col    1[6] @8181: AAAAAA

# dump trace
#--------------------------------------------------------------------------------
 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0002.019.00000a13  0x00c011b6.034d.14  C---    0  scn 0x0000.003de705
0x02   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
bdba: 0x0180008d
data_block_dump,data header at 0x7ff1f6bf9a64
......
block_row_dump:
tab 0, row 0, @0x1f8b
tl: 13 fb: --H-FL-- lb: 0x0  cc: 2
col  0: [ 2]  c1 02
col  1: [ 6]  41 41 41 41 41 41
#--------------------------------------------------------------------------------
```

#### ASSM和MSSM下block结构的一点差异
![](http://p2c0rtsgc.bkt.clouddn.com/0330_oracle_dsi_01.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0330_oracle_dsi_02.png)

#### 删除记录并使用BBED恢复
``` perl
# 删除记录，并确保写到文件中
delete from t2 where id=1;
commit;
alter system flush buffer_cache;

# 重新登陆bbed查看file 6 block 141
BBED> set file 6 block 141

BBED> p *kdbr[0]
rowdata[0]
----------
ub1 rowdata[0]                              @8175     0x3c

BBED> x/rnccccccc
rowdata[0]                                  @8175    
----------
flag@8175: 0x3c (KDRHFL, KDRHFF, KDRHFD, KDRHFH)    # 3c 行状态为删除
lock@8176: 0x02
cols@8177:    0

BBED> dump offset 8000 count 200
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 141              Offsets: 8000 to 8191           Dba:0x0180008d
------------------------------------------------------------------------
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 0000003c 020202c1 02064141 41414141 0106c071 

3c 020202c1 02064141 41414141
3c - 删除
02 - 第2个事务槽
02 - 代表有2列
02 - 第1个列有2个字节组成
c1 02 - 第1列的2个字节c1 02 代表第1列对应的数据为1
06 - 第2个列有6个字节组成
4141 41414141 - 第2个列6个字节4141 41414141 ，对应的数值为AAAAAA

# 使用BBED恢复
modify /x 2c offset 8175
modify /x 01 offset 8176
sum apply

# 在SQLPLUS中已经能查到刚才被删除的数据
alter system flush buffer_cache;  (可选)
select * from t2;

        ID NAME
---------- ----------
         1 AAAAAA
```

### 使用BBED手工修复UPDATE数据
``` perl
conn lyj/lyj
create table t3 (id number,name varchar(10));
insert into t3 values (1,'AAAAA');
insert into t3 values (2,'BBBBB');
commit;
select id,name,DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) FILE#,DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) BLOCK# from t3;

        ID NAME            FILE#     BLOCK#
---------- ---------- ---------- ----------
         1 AAAAA               6        148
         2 BBBBB               6        148
alter system flush buffer_cache;

# 使用BBED dump块
BBED> set file 6 block 148
        FILE#           6
        BLOCK#          148

BBED> dump/v offset 8000 count 200
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148     Offsets: 8000 to 8191  Dba:0x01800094
-------------------------------------------------------
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 2c010202 c1030542 42424242 l ....,......BBBBB
 2c010202 c1020541 41414141 01065e76 l ,......AAAAA..^v
 
2c010202 c1020541 41414141
2c - 正常状态
01 - 第1个事务槽
02 - 代表有2列
02 - 第1个列有2个字节组成
c1 02 - 第1列的2个字节c1 02 代表第1列对应的数据为1
05 - 第2个列有5个字节组成
41 41414141 - 第2个列5个字节41 41414141 ，对应的数值为AAAAA

2c010202 c1030542 42424242
2c - 正常状态
01 - 第1个事务槽
02 - 代表有2列
02 - 第1个列有2个字节组成
c103 - 第1列的2个字节c1 03 代表第1列对应的数据为2
05 - 第2个列有5个字节组成
42 42424242 - 第2个列5个字节42 42424242 ，对应的数值为BBBBB

# update字符等长时（直接替换）
update t3 set name='CCCCC' where name='BBBBB';
commit;
alter system flush buffer_cache;

# 重进BBED
BBED> set file 6 block 148

BBED> dump/v offset 8000 count 200
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148     Offsets: 8000 to 8191  Dba:0x01800094
-------------------------------------------------------
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 2c020202 c1030543 43434343 l ....,......CCCCC   # update前的事务槽是01，update后事务槽是02
 2c000202 c1020541 41414141 02068782 l ,......AAAAA....

# update字符超过原有长时
update t3 set name='DDDDDD' where name='CCCCC';
commit;
alter system flush buffer_cache;

# 重进BBED
BBED> set file 6 block 148

BBED> dump/v offset 8000 count 200
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148     Offsets: 8000 to 8191  Dba:0x01800094
-------------------------------------------------------
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 00000000 00000000 00000000 l ................
 00000000 0000002c 010202c1 03064444 l .......,......DD  # DDDDDD是新写到文件中，并不直接修改
 44444444 2c000202 c1030543 43434343 l DDDD,......CCCCC  # 原有的CCCCC还在文件中，状态也是c2
 2c000202 c1020541 41414141 02064883 l ,......AAAAA..H.
```

下面要使用BBED工具，手工将DDDDDD，改为CCCCC
``` perl
select * from t3;

        ID NAME
---------- ----------
         1 AAAAA
         2 DDDDDD

BBED> set file 6 block 148
        FILE#           6
        BLOCK#          148

BBED> p kdbr
sb2 kdbr[0]                                 @118      8076
sb2 kdbr[1]                                 @120      8051     # offset 120 是第二行记录

BBED> p *kdbr[1]
rowdata[0]
----------
ub1 rowdata[0]                              @8151     0x2c     # 8151-100=8051

BBED> x /rncccccc
rowdata[0]                                  @8151    
----------
flag@8151: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8152: 0x01
cols@8153:    2

col    0[2] @8154: 2 
col    1[6] @8157: DDDDDD


# 变更指向位置
BBED> dump /v offset 8164 count 100
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148     Offsets: 8164 to 8191  Dba:0x01800094
-------------------------------------------------------
 2c000202 c1030543 43434343 2c000202 l ,......CCCCC,...  
 c1020541 41414141 02064883          l ...AAAAA..H.

8164 - 100 = 8064 

select to_char(8064,'xxxxxxxxxx') c from dual;

C
-----------
       1f80         # 801f

BBED> dump /v offset 120 count 16
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148     Offsets:  120 to  135  Dba:0x01800094
-------------------------------------------------------
 731f0000 00000000 00000000 00000000 l s...............     # 1f73

select to_number('1f73','xxxxxxxxxx') n from dual;

         N
----------
      8051   # 8051+100  DDDDDD所在位置的绝对偏移量

BBED> modify /x 80 offset 120
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148              Offsets:  120 to  135           Dba:0x01800094
------------------------------------------------------------------------
 801f0000 00000000 00000000 00000000 

BBED> sum apply
Check value for File 6, Block 148:
current = 0xae1b, required = 0xae1b

# 8152事务槽设置为00
BBED> dump /v offset 8152
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148     Offsets: 8152 to 8167  Dba:0x01800094
-------------------------------------------------------
 010202c1 03064444 44444444 2c000202 l ......DDDDDD,...

BBED> modify /x 00
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148              Offsets: 8152 to 8167           Dba:0x01800094
------------------------------------------------------------------------
 000202c1 03064444 44444444 2c000202 

# 8165事务槽设置为02
BBED> dump /v offset 8165
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148     Offsets: 8165 to 8180  Dba:0x01800094
-------------------------------------------------------
 000202c1 03054343 4343432c 000202c1 l ......CCCCC,....

BBED> sum apply
Check value for File 6, Block 148:
current = 0xac1a, required = 0xac1a

# 有效空间设置
BBED> modify /x 5c offset 110
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148              Offsets:  110 to  125           Dba:0x01800094
------------------------------------------------------------------------
 5c1f691f 00000200 8c1f801f 00000000 

 <32 bytes per line>

BBED> modify /x 5c offset 112
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 148              Offsets:  112 to  127           Dba:0x01800094
------------------------------------------------------------------------
 5c1f0000 02008c1f 801f0000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 148:
current = 0xac1a, required = 0xac1a

# 校验
BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/skip_arch01.dbf
BLOCK = 148

Block Checking: DBA = 25165972, Block Type = KTB-managed data block
data header at 0x21c4a64
kdbchk: row locked by non-existent transaction
        table=0   slot=1
        lockid=2   ktbbhitc=2
Block 148 failed with check code 6101

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


# 验证
SQL> alter system flush buffer_cache;

System altered.

SQL> select * from t3;

        ID NAME
---------- ----------
         1 AAAAA
         2 CCCCC
```

### 使用BBED恢复UPDATE的数据2
根据以下景场操作，使用BBED恢复UPDATE的数据，把BBBBBB恢复成AAAAA（即把6个B恢复5个A）。
```perl
create table t1 (id int,name varchar2(10));
insert into t1 values(1,'AAAAA');
commit;
update t1 set name='BBBBBB' where name='AAAAA';    #误操作
commit;
```

恢复步骤
``` perl
alter system flush buffer_cache;
select id,name,DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) FILE#,DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) BLOCK# from t1;

        ID NAME            FILE#     BLOCK#
---------- ---------- ---------- ----------
         1 BBBBBB              6        135

# 查看dump trace (可选步骤)
alter system dump datafile 6 block 135;
select * from v$diag_info;
#-----------------------------------------------------------------------------
Block header dump:  0x01800087
 Object id on Block? Y
 seg/obj: 0x1723e  csc: 0x00.40b170  itc: 2  flg: E  typ: 1 - DATA
     brn: 0  bdba: 0x1800080 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x000a.000.0000097d  0x00c00866.0370.22  C---    0  scn 0x0000.0040b16f
0x02   0x0005.000.00000acd  0x00c00ee9.03f6.14  --U-    1  fsc 0x0000.0040b173
#-----------------------------------------------------------------------------

# 登入BBED
BBED> set file 6 block 135
        FILE#           6
        BLOCK#          131

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 135                                   Dba:0x01800087
------------------------------------------------------------
 KTB Data Block (Table/Cluster)

 struct kcbh, 20 bytes                      @0       
    ub1 type_kcbh                           @0       
    ub1 frmt_kcbh                           @1       
    ub1 spare1_kcbh                         @2       
    ub1 spare2_kcbh                         @3       
    ub4 rdba_kcbh                           @4       
    ub4 bas_kcbh                            @8       
    ub2 wrp_kcbh                            @12      
    ub1 seq_kcbh                            @14      
    ub1 flg_kcbh                            @15      
    ub2 chkval_kcbh                         @16      
    ub2 spare3_kcbh                         @18      

 struct ktbbh, 72 bytes                     @20      
    ub1 ktbbhtyp                            @20      
    union ktbbhsid, 4 bytes                 @24      
    struct ktbbhcsc, 8 bytes                @28      
    sb2 ktbbhict                            @36      
    ub1 ktbbhflg                            @38      
    ub1 ktbbhfsl                            @39      
    ub4 ktbbhfnx                            @40      
    struct ktbbhitl[2], 48 bytes            @44      

 struct kdbh, 14 bytes                      @100     
    ub1 kdbhflag                            @100     
    sb1 kdbhntab                            @101     
    sb2 kdbhnrow                            @102     
    sb2 kdbhfrre                            @104     
    sb2 kdbhfsbo                            @106     
    sb2 kdbhfseo                            @108     
    sb2 kdbhavsp                            @110     
    sb2 kdbhtosp                            @112     

 struct kdbt[1], 4 bytes                    @114     
    sb2 kdbtoffs                            @114     
    sb2 kdbtnrow                            @116     

 sb2 kdbr[1]                                @118     

 ub1 freespace[8043]                        @120     

 ub1 rowdata[25]                            @8163    

 ub4 tailchk                                @8188    

BBED> p kdbr
sb2 kdbr[0]                                 @118      8063

BBED> p *kdbr[0]
rowdata[0]
----------
ub1 rowdata[0]                              @8163     0x2c

BBED> x/rnccccccc
rowdata[0]                                  @8163    
----------
flag@8163: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@8164: 0x02
cols@8165:    2

col    0[2] @8166: 1 
col    1[6] @8169: BBBBBB

BBED> dump /v offset 8163
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 135     Offsets: 8163 to 8191  Dba:0x01800087
-------------------------------------------------------
 2c020202 c1020642 42424242 422c0002 l ,......BBBBBB,..
 02c10205 41414141 41020673 b1       l ....AAAAA..s

 <16 bytes per line>


BBED> dump /v offset 8176        # 8163+13
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 131     Offsets: 8176 to 8191  Dba:0x01800083
-------------------------------------------------------
 2c000202 c1020541 41414141 02065593 l ,......AAAAA..U.

 <16 bytes per line>

SQL> select to_char(8076,'xxxxxxxx') from dual;       # 8176-100

TO_CHAR(8
---------
     1f8c        # Little Endian：8c1f

BBED> dump offset 118 count 16
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 131              Offsets:  118 to  133           Dba:0x01800083
------------------------------------------------------------------------
 7f1f0000 00000000 00000000 00000000 

 <32 bytes per line>

# 修改
BBED> modify /x 8c offset 118
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 135              Offsets:  118 to  133           Dba:0x01800087
------------------------------------------------------------------------
 8c1f0000 00000000 00000000 00000000 

 <32 bytes per line>

# 修改对应的事务槽
BBED> modify /x 00 offset 8164
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 135              Offsets: 8164 to 8179           Dba:0x01800087
------------------------------------------------------------------------
 000202c1 02064242 42424242 2c000202 

 <32 bytes per line>

BBED> modify /x 02 offset 8177
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 135              Offsets: 8177 to 8191           Dba:0x01800087
------------------------------------------------------------------------
 020202c1 02054141 41414102 0673b1 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 135:
current = 0xd252, required = 0xd252

# 有效空间设置
## 检验
BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/skip_arch01.dbf
BLOCK = 135

Block Checking: DBA = 25165959, Block Type = KTB-managed data block
data header at 0x7fcea7922264
kdbchk: the amount of space used is not equal to block size # 使用的空间大小不等于块大小
        used=32 fsc=0 avsp=8055 dtl=8088   
        # 提示数据块的空间使用不正确（dtl-used=kdbhavsp=kdbhtosp）
        # 8088-32=8056 与 avsp=8055相差1,也就是说我要恢复到8056
Block 135 failed with check code 6110

DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 1
Total Blocks Failing   (Data) : 1    # 有错误
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 0
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED

BBED> p kdbh
struct kdbh, 14 bytes                       @100     
   ub1 kdbhflag                             @100      0x00 (NONE)
   sb1 kdbhntab                             @101      1
   sb2 kdbhnrow                             @102      1
   sb2 kdbhfrre                             @104     -1
   sb2 kdbhfsbo                             @106      20
   sb2 kdbhfseo                             @108      8063
   sb2 kdbhavsp                             @110      8055
   sb2 kdbhtosp                             @112      8055

# 修改空间，把空间修改为781f,以dtl-used=kdbhavsp为主，恢复到8088-32=8086 （算法见上面注释）
SQL> select to_char(8056,'xxxxxxxx') from dual;

TO_CHAR(8
---------
     1f78        # 781f

BBED> modify /x 78 offset 110
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 131              Offsets:  110 to  125           Dba:0x01800083
------------------------------------------------------------------------
 5c1f771f 00000100 8c1f0000 00000000 

 <32 bytes per line>

BBED> modify /x 78 offset 112
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 131              Offsets:  112 to  127           Dba:0x01800083
------------------------------------------------------------------------
 5c1f0000 01008c1f 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 135:
current = 0xd252, required = 0xd252

# 校验
BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/skip_arch01.dbf
BLOCK = 135

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

# 恢复成功
SQL> alter system flush buffer_cache;

System altered.

SQL> select * from t1;

        ID NAME
---------- ----------
         1 AAAAA
```


### 使用BBED手工提交delete操作的事务
根据以下景场操作，使用BBED手工提交delete操作的事务
``` perl
sqlplus lyj/lyj
drop table t2 purge;
create table t2 (id int,name varchar2(10));
insert into t2 values(1,'AAAAA');
commit;
alter system flush buffer_cache;

select id,name,DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) FILE#,DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) BLOCK# from t2;

        ID NAME            FILE#     BLOCK#
---------- ---------- ---------- ----------
         1 AAAAA               6        143

会话一：
delete from t2 where name='AAAAA';  #不允许用commit命令提交
alter system flush buffer_cache;

会话二：
select * from t2;                   #如果查不到name='AAAAA'这条记录，说明BBED手工提交事务成功！
```

操作步骤：
``` perl
# 可选查看dump数据块内容
alter system dump datafile 6 block 143;
select * from v$diag_info;
#-----------------------------------------------------------------------------
Block header dump:  0x0180008f
 Object id on Block? Y
 seg/obj: 0x17240  csc: 0x00.40bc99  itc: 2  flg: E  typ: 1 - DATA
     brn: 0  bdba: 0x1800088 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0002.003.00000a73  0x00c00201.037a.02  C---    0  scn 0x0000.0040bc86
0x02   0x0006.01c.00000ae5  0x00c00dee.040c.04  ----    1  fsc 0x000a.00000000
bdba: 0x0180008f
data_block_dump,data header at 0x7f7743224a64

tsiz: 0x1f98
hsiz: 0x14
pbl: 0x7f7743224a64
     76543210
flag=--------
ntab=1
nrow=1
frre=-1
fsbo=0x14
fseo=0x1f8c
avsp=0x1f78
tosp=0x1f84
0xe:pti[0]      nrow=1  offs=0
0x12:pri[0]     offs=0x1f8c
block_row_dump:
tab 0, row 0, @0x1f8c
tl: 2 fb: --HDFL-- lb: 0x2
end_of_block_dump
#-----------------------------------------------------------------------------

# 重新登陆BBED
BBED> set file 6 block 143
        FILE#           6
        BLOCK#          143

BBED> p kdbr
sb2 kdbr[0]                                 @118      8076

BBED> p *kdbr[0]
rowdata[0]
----------
ub1 rowdata[0]                              @8176     0x3c    # 3c-数据已是删除状态

BBED> x/rncccc
rowdata[0]                                  @8176    
----------
flag@8176: 0x3c (KDRHFL, KDRHFF, KDRHFD, KDRHFH)
lock@8177: 0x02
cols@8178:    0

# 更改更备槽状态为0
BBED> modify /x 00 offset 8177
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 143              Offsets: 8177 to 8191           Dba:0x0180008f
------------------------------------------------------------------------
 000202c1 02054141 41414101 0699bc 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 143:
current = 0x65d1, required = 0x65d1

# 更改提交状态
BBED> p ktbbh
struct ktbbh, 72 bytes                      @20      
   ub1 ktbbhtyp                             @20       0x01 (KDDBTDATA)
   union ktbbhsid, 4 bytes                  @24      
      ub4 ktbbhsg1                          @24       0x00017240
      ub4 ktbbhod1                          @24       0x00017240
   struct ktbbhcsc, 8 bytes                 @28      
      ub4 kscnbas                           @28       0x0040bc99
      ub2 kscnwrp                           @32       0x0000
   sb2 ktbbhict                             @36       2
   ub1 ktbbhflg                             @38       0x32 (NONE)
   ub1 ktbbhfsl                             @39       0x00
   ub4 ktbbhfnx                             @40       0x01800088
   struct ktbbhitl[0], 24 bytes             @44      
      struct ktbitxid, 8 bytes              @44      
         ub2 kxidusn                        @44       0x0002
         ub2 kxidslt                        @46       0x0003
         ub4 kxidsqn                        @48       0x00000a73
      struct ktbituba, 8 bytes              @52      
         ub4 kubadba                        @52       0x00c00201
         ub2 kubaseq                        @56       0x037a
         ub1 kubarec                        @58       0x02
      ub2 ktbitflg                          @60       0x8000 (KTBFCOM)
      union _ktbitun, 2 bytes               @62      
         sb2 _ktbitfsc                      @62       0
         ub2 _ktbitwrp                      @62       0x0000
      ub4 ktbitbas                          @64       0x0040bc86
   struct ktbbhitl[1], 24 bytes             @68      
      struct ktbitxid, 8 bytes              @68      
         ub2 kxidusn                        @68       0x0006
         ub2 kxidslt                        @70       0x001c
         ub4 kxidsqn                        @72       0x00000ae5
      struct ktbituba, 8 bytes              @76      
         ub4 kubadba                        @76       0x00c00dee
         ub2 kubaseq                        @80       0x040c
         ub1 kubarec                        @82       0x04
      ub2 ktbitflg                          @84       0x0001 (NONE)     # 状态是非提交，需要修改成8000 (0080)
      union _ktbitun, 2 bytes               @86      
         sb2 _ktbitfsc                      @86       10                # 需要将这里修改为0
         ub2 _ktbitwrp                      @86       0x000a
      ub4 ktbitbas                          @88       0x00000000


BBED> modify /x 0080 offset 84
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 143              Offsets:   84 to  595           Dba:0x0180008f
------------------------------------------------------------------------
 00800a00 00000000 00000000 00000000 00010100 ffff1400 8c1f781f 841f0000 
 01008c1f 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
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
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 00 offset 86
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 143              Offsets:   86 to  597           Dba:0x0180008f
------------------------------------------------------------------------
 00000000 00000000 00000000 00000001 0100ffff 14008c1f 781f841f 00000100 
 8c1f0000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
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
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 143:
current = 0x67db, required = 0x67db

# 校验
BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/skip_arch01.dbf
BLOCK = 143

Block Checking: DBA = 25165967, Block Type = KTB-managed data block
data header at 0x1ac7864
kdbchk: the amount of space used is not equal to block size
        used=22 fsc=0 avsp=8056 dtl=8088   # 8088-22=8066
Block 143 failed with check code 6110

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

BBED> p kdbh
struct kdbh, 14 bytes                       @100     
   ub1 kdbhflag                             @100      0x00 (NONE)
   sb1 kdbhntab                             @101      1
   sb2 kdbhnrow                             @102      1
   sb2 kdbhfrre                             @104     -1
   sb2 kdbhfsbo                             @106      20
   sb2 kdbhfseo                             @108      8076
   sb2 kdbhavsp                             @110      8056   # 需要修改
   sb2 kdbhtosp                             @112      8068   # 需要修改（这个其实证明不用修改）
 
BBED> dump /v offset 110 count 16
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 143     Offsets:  110 to  125  Dba:0x0180008f
-------------------------------------------------------
 781f841f 00000100 8c1f0000 00000000 l x...............

 <16 bytes per line>

BBED> dump /v offset 112 count 16
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 143     Offsets:  112 to  127  Dba:0x0180008f
-------------------------------------------------------
 841f0000 01008c1f 00000000 00000000 l ................

 <16 bytes per line>

SQL> select to_char(8066,'xxxxxxxx') from dual;

TO_CHAR(8
---------
     1f82     # 821f

BBED> modify /x 82 offset 110
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 143              Offsets:  110 to  125           Dba:0x0180008f
------------------------------------------------------------------------
 821f841f 00000100 8c1f0000 00000000 

 <32 bytes per line>

BBED> modify /x 82 offset 112
 File: /u01/app/oracle/oradata/orcl/skip_arch01.dbf (6)
 Block: 143              Offsets:  112 to  127           Dba:0x0180008f
------------------------------------------------------------------------
 821f0000 01008c1f 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 6, Block 143:
current = 0x652d, required = 0x652d


BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/skip_arch01.dbf
BLOCK = 143

Block Checking: DBA = 25165967, Block Type = KTB-managed data block
data header at 0x1dafa64
kdbchk: space available on commit is incorrect
        tosp=8066 fsc=0 stb=2 avsp=8066   # 这里提示stb=2，那说明这个avsp 实际上应该是8066+2=8068才对
Block 143 failed with check code 6111

DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 1
Total Blocks Failing   (Data) : 1    # 仍有问题
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 0
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED

SQL> select to_char(8068,'xxxxxxxx') from dual;

TO_CHAR(8
---------
     1f84

BBED> sum apply
Check value for File 6, Block 143:
current = 0x652b, required = 0x652b

# 检验 - 终于对了
BBED> verify
DBVERIFY - Verification starting
FILE = /u01/app/oracle/oradata/orcl/skip_arch01.dbf
BLOCK = 143


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


会话二：
SQL> select * from t2; 

        ID NAME
---------- ----------
         1 AAAAA

SQL> alter system flush buffer_cache;

System altered.

SQL> select * from t2; 

no rows selected
```

### 为什么ASSM要比MSSM多了8个byte？请给出实验步骤证明
因为在ASSM下，ORACLE改变了block内部table directory和row directory的位置，oracle把它们顺延了8个字节
``` perl
SQL> select * from v$type_size where component in ('KCB','KTB');

COMPONEN TYPE     DESCRIPTION                       TYPE_SIZE
-------- -------- -------------------------------- ----------
KCB      KCBH     BLOCK COMMON HEADER                      20
KTB      KTBIT    TRANSACTION VARIABLE HEADER              24
KTB      KTBBH    TRANSACTION FIXED HEADER                 48
KTB      KTBBH_BS TRANSACTION BLOCK BITMAP SEGMENT          8

# 创建MSSM表空间
create tablespace mssm 
  datafile '/u01/app/oracle/oradata/orcl/mssm01.dbf' size 10m
  autoextend on next 1m segment space management manual;

select TABLESPACE_NAME,SEGMENT_SPACE_MANAGEMENT from dba_tablespaces;

TABLESPACE_NAME                SEGMEN
------------------------------ ------
SYSTEM                         MANUAL
SYSAUX                         AUTO
UNDOTBS1                       MANUAL
TEMP                           MANUAL
USERS                          AUTO
DSI                            AUTO
SKIP_ARCH                      AUTO
TS_16                          AUTO
MSSM                           MANUAL

create table lyj.t4(id number,name varchar2(10)) tablespace mssm;
insert into lyj.t4 values(1,'TTTTT');
commit;

select id,name,DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) FILE#,DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) BLOCK# from lyj.t4;

        ID NAME            FILE#     BLOCK#
---------- ---------- ---------- ----------
         1 TTTTT               8        129


# 登入BBED
echo "8 /u01/app/oracle/oradata/orcl/mssm01.dbf 10485760" >> filelist.txt
bbed

BBED> info
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /u01/app/oracle/oradata/orcl/system01.dbf                        97281
     2  /u01/app/oracle/oradata/orcl/sysaux01.dbf                        96001
     3  /u01/app/oracle/oradata/orcl/undotbs01.dbf                       21121
     4  /u01/app/oracle/oradata/orcl/users01.dbf                           641
     5  /u01/app/oracle/oradata/orcl/dsi01.dbf                           64001
     6  /u01/app/oracle/oradata/orcl/skip_arch01.dbf                      6400
     7  /u01/app/oracle/oradata/orcl/ts16_01.dbf                          6400
     8  /u01/app/oracle/oradata/orcl/mssm01.dbf                           1280

BBED> set file 8 block 129
        FILE#           8
        BLOCK#          129

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/mssm01.dbf (8)
 Block: 129                                   Dba:0x02000081
------------------------------------------------------------
 KTB Data Block (Table/Cluster)

 struct kcbh, 20 bytes                      @0       
    ub1 type_kcbh                           @0       
    ub1 frmt_kcbh                           @1       
    ub1 spare1_kcbh                         @2       
    ub1 spare2_kcbh                         @3       
    ub4 rdba_kcbh                           @4       
    ub4 bas_kcbh                            @8       
    ub2 wrp_kcbh                            @12      
    ub1 seq_kcbh                            @14      
    ub1 flg_kcbh                            @15      
    ub2 chkval_kcbh                         @16      
    ub2 spare3_kcbh                         @18      

 struct ktbbh, 72 bytes                     @20      
    ub1 ktbbhtyp                            @20      
    union ktbbhsid, 4 bytes                 @24      
    struct ktbbhcsc, 8 bytes                @28      
    sb2 ktbbhict                            @36      
    ub1 ktbbhflg                            @38      
    ub1 ktbbhfsl                            @39      
    ub4 ktbbhfnx                            @40      
    struct ktbbhitl[2], 48 bytes            @44      

 struct kdbh, 14 bytes                      @92      # 从上面的例子可以看出，ASSM kdbh是100，比MSSM多了8个偏移量
    ub1 kdbhflag                            @92      
    sb1 kdbhntab                            @93      
    sb2 kdbhnrow                            @94      
    sb2 kdbhfrre                            @96      
    sb2 kdbhfsbo                            @98      
    sb2 kdbhfseo                            @100     
    sb2 kdbhavsp                            @102     
    sb2 kdbhtosp                            @104     

 struct kdbt[1], 4 bytes                    @106     
    sb2 kdbtoffs                            @106     
    sb2 kdbtnrow                            @108     

 sb2 kdbr[1]                                @110     

 ub1 freespace[8064]                        @112     

 ub1 rowdata[12]                            @8176    

 ub4 tailchk                                @8188
```


## 参考
* [Oracle数据块损坏知识](https://wenku.baidu.com/view/0624178076eeaeaad0f33028.html)
* [数据在内存中的存储方式(Big Endian和Little Endian的区别)](http://www.cnblogs.com/renyuan/archive/2013/05/26/3099766.html)
* [利用BBED恢复UPDATE修改前的值](https://blog.csdn.net/guoyjoe/article/details/30615151)
* [How to drop a Index with bbed?](http://www.killdb.com/2014/10/13/how-to-use-bbed-to-drop-a-index.html)
