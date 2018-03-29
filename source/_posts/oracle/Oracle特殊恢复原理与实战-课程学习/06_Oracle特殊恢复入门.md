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


# 下面为rdba换算成文件号块号的方法

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


## 实例
### insert一条记录，没有提交事务，会写入Datablock吗？

## 参考
[Oracle数据块损坏知识](https://wenku.baidu.com/view/0624178076eeaeaad0f33028.html)