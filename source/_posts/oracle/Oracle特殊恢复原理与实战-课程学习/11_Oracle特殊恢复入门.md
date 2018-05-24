---
title: Oracle特殊恢复原理与实战_11 ORA-8102 Index Corruption修复
date: 2018-05-17
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## ORA-8102 Index Corruption修复
### ORA-8102解析
``` perl
oerr ora 8102
08102, 00000, "index key not found, obj# %s, file %s, block %s (%s)"
// *Cause:  Internal error: possible inconsistency in index
// *Action:  Send trace file to your customer support representative, along
//           with information on reproducing the error

# 常见于索引键值与表上存的值不一致，可能是ORACLE的bug，也可能是由于硬件I/0错误所引起！
```

<!-- more -->

### 重现ORA-8102错误环境准备
``` perl
conn lyj/lyj

create table lyj_1000(id int,name varchar2(100));

begin
  for i in 1 .. 5000 loop
  insert into lyj_1000 values(i,'lyj'||i);
  commit;
  end loop;
end;
/

select user_id,username from dba_users where username='LYJ';

   USER_ID USERNAME
---------- ------------------------------
        34 LYJ

alter table LYJ_1000 add primary key(id);

select CONSTRAINT_NAME from dba_constraints where table_name='LYJ_1000';

CONSTRAINT_NAME
------------------------------
SYS_C003989

conn / as sysdba
select dbms_rowid.rowid_relative_fno(rowid) file#,
       dbms_rowid.rowid_block_number(rowid) block#,
       dbms_rowid.rowid_row_number(rowid) row# 
from con$ 
where name='_NEXT_CONSTRAINT';

     FILE#     BLOCK#       ROW#
---------- ---------- ----------
         1        289         12

select /*+ FULL(t1) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3990

select /*+ index(t1 I_CON2) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3990

# 上面这两个值相等的时候，是正常的，创建新索引时，会分别更新上面的两个值
```

### 重现ORA-8102错误
``` perl
bbed

BBED> set file 1 block 289
        FILE#           1
        BLOCK#          289

# 找到12行（* 绝对值）
BBED> p *kdbr[12]
rowdata[0]
----------
ub1 rowdata[0]                              @1108     0x2c

# 看这行记录的信息
BBED> x /rccnn
rowdata[0]                                  @1108    
----------
flag@1108: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@1109: 0x02
cols@1110:    4

col    0[1] @1111: .
col   1[16] @1113: _NEXT_CONSTRAINT
col    2[3] @1130: 3990   # # 
col    3[1] @1134: 0 

# 查看offset 1130
BBED> dump /v offset 1130 count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 289     Offsets: 1130 to 1161  Dba:0x00400121
-------------------------------------------------------
 03c2285b 01802c00 04018010 5f4e4558 l ..([..,....._NEX   
 545f434f 4e535452 41494e54 02c22801 l T_CONSTRAINT..(.

 <16 bytes per line>

select dump(3990,16) from dual;

DUMP(3990,16)
---------------------
Typ=2 Len=3: c2,28,5b

select dump(3991,16) from dual;  # 改成大一些的值，重现故障

DUMP(3991,16)
---------------------
Typ=2 Len=3: c2,28,5c

BBED> modify /x 5c offset 1133
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 289              Offsets: 1133 to 1164           Dba:0x00400121
------------------------------------------------------------------------
 5c01802c 00040180 105f4e45 58545f43 4f4e5354 5241494e 5402c228 01802c00 

BBED> sum apply
Check value for File 1, Block 289:
current = 0x8edd, required = 0x8edd

# 查看修改后状态
BBED> dump /v offset 1130 count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 289     Offsets: 1130 to 1161  Dba:0x00400121
-------------------------------------------------------
 03c2285c 01802c00 04018010 5f4e4558 l ..(\..,....._NEX
 545f434f 4e535452 41494e54 02c22801 l T_CONSTRAINT..(.

BBED> p *kdbr[12]
rowdata[0]
----------
ub1 rowdata[0]                              @1108     0x2c

BBED> x /rccnn
rowdata[0]                                  @1108    
----------
flag@1108: 0x2c (KDRHFL, KDRHFF, KDRHFH)
lock@1109: 0x02
cols@1110:    4

col    0[1] @1111: .
col   1[16] @1113: _NEXT_CONSTRAINT
col    2[3] @1130: 3991 
col    3[1] @1134: 0

# 重启数据库
shutdown immediate
startup

# 重建索引时出现ORA-08102报错
conn lyj/lyj
alter table LYJ_1000 drop primary key;

alter table LYJ_1000 add primary key(id);
alter table LYJ_1000 add primary key(id)
*
ERROR at line 1:
ORA-00604: error occurred at recursive SQL level 1
ORA-08102: index key not found, obj# 52, file 1, block 15536 (2)
```

### 分析ORA-8102错误
``` perl
# 查看相关日志
vi alert.log
#----------------------------------------------------------------------------------------------------
Thu May 17 16:52:20 2018
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_24252.trc:
#----------------------------------------------------------------------------------------------------

vi /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_24252.trc
#----------------------------------------------------------------------------------------------------
oer 8102.2 - obj# 52, rdba: 0x00403cb0(afn 1, blk# 15536)       # obj# 52 -> bootstrap$
kdk key 8102.2:
  ncol: 1, len: 4
  key: (4):  03 c2 28 5c
  mask: (4096):
 61 01 00 00 00 18 87 97 65 01 00 00 00 00 00 00 00 00 00 00 00 60 19 12 0c
.....
#============
Plan Table
#============
--------------------------------------+-----------------------------------+
| Id  | Operation           | Name    | Rows  | Bytes | Cost  | Time      |
--------------------------------------+-----------------------------------+
| 0   | UPDATE STATEMENT    |         |       |       |     2 |           |
| 1   |  UPDATE             | CON$    |       |       |       |           |  # 更新CON$时发现不一致
| 2   |   INDEX UNIQUE SCAN | I_CON1  |     1 |    22 |     1 |  00:00:01 |
--------------------------------------+-----------------------------------+
#----------------------------------------------------------------------------------------------------

set line 100
select * from bootstrap$ where obj#=52;

     LINE#       OBJ#
---------- ----------
SQL_TEXT
----------------------------------------------------------------------------------------------------
        52         52
CREATE UNIQUE INDEX I_CON2 ON CON$(CON#) PCTFREE 10 INITRANS 2 MAXTRANS 255 STORAGE (  INITIAL 64K N
EXT 1024K MINEXTENTS 1 MAXEXTENTS 2147483645 PCTINCREASE 0 OBJNO 52 EXTENTS (FILE 1 BLOCK 464))

# 这两个值不一样了
select /*+ FULL(t1) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3991

select /*+ index(t1 I_CON2) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3990

# dump afn 1, blk# 15536(新开一个会话)
alter system dump datafile 1 block 15536;
oradebug setmypid
oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_24374.trc

Block header dump:  0x00403cb0
 Object id on Block? Y
 seg/obj: 0x34  csc: 0x00.2240b6  itc: 3  flg: O  typ: 2 - INDEX   # 索引块
     fsl: 0  fnx: 0x403cb1 ver: 0x01

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0001.00b.0000006d  0x00c02131.0014.01  CB--    0  scn 0x0000.0001b805
0x02   0x0008.009.00000403  0x00c01041.01ee.0c  C---    0  scn 0x0000.0022368f
0x03   0x0005.01f.000003f6  0x00c00cbd.02b6.0c  --U-    1  fsc 0x000e.002240b7
Leaf block dump
#===============
header address 140264942398068=0x7f91fa10ea74
kdxcolev 0  # 0 叶子块 根块>0
KDXCOLEV Flags = - - -
kdxcolok 0
kdxcoopc 0x80: opcode=0: iot flags=--- is converted=Y
kdxconco 1
kdxcosdc 1
kdxconro 498
kdxcofbo 1032=0x408
kdxcofeo 1184=0x4a0
kdxcoavs 1004
kdxlespl 0
kdxlende 1
kdxlenxt 0=0x0
kdxleprv 4194775=0x4001d7
kdxledsz 6
kdxlebksz 8008
row#0[7996] flag: ------, lock: 0, len=12, data:(6):  00 40 33 e5 00 6f
col 0; len 3; (3):  c2 23 17
row#1[7984] flag: ------, lock: 0, len=12, data:(6):  00 40 33 e5 00 70
col 0; len 3; (3):  c2 23 18
.....
row#497[1196] flag: ------, lock: 0, len=12, data:(6):  00 40 01 21 00 0c
col 0; len 3; (3):  c2 28 5b   # 这个是最后的行
----- end of leaf block dump -----
End dump data blocks tsn: 0 file#: 1 minblk 15536 maxblk 15536

select utl_raw.cast_to_number(replace('c2 28 5b',' ')) from dual;  

UTL_RAW.CAST_TO_NUMBER(REPLACE('C2285B',''))
--------------------------------------------
                                        3990                            
```

### 解决ORA-8102错误
``` perl
bbed

BBED> set file 1 block 15536
        FILE#           1
        BLOCK#          15536

# row#497[1196] flag: ------, lock: 0, len=12, data:(6):  00 40 01 21 00 0c
# 1196 是相对的偏移量，需要先转到绝对偏移量，ASSM数据块是+100，索引是+124，因为索引的IT槽是3个，计算绝对偏移量公式如下
SQL> select 1196+20+24+24*3+8 from dual;

1196+20+24+24*3+8
-----------------
             1320

BBED> dump /v offset 1320 count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 15536   Offsets: 1320 to 1351  Dba:0x00403cb0
-------------------------------------------------------
 03c2285b 00000040 33e7005e 03c22859 l ..([...@3..^..(Y    # col 0; len 3; (3):  c2 28 5b
 01000040 0121000c 03c2285a 00000040 l ...@.!....(Z...@


BBED> modify /x 5c offset 1323
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 15536            Offsets: 1323 to 1354           Dba:0x00403cb0
------------------------------------------------------------------------
 5c000000 4033e700 5e03c228 59010000 40012100 0c03c228 5a000000 4033e700 

 <32 bytes per line>

BBED> dump /v offset 1320 count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 15536   Offsets: 1320 to 1351  Dba:0x00403cb0
-------------------------------------------------------
 03c2285c 00000040 33e7005e 03c22859 l ..(\...@3..^..(Y
 01000040 0121000c 03c2285a 00000040 l ...@.!....(Z...@

 <16 bytes per line>

BBED> sum apply
Check value for File 1, Block 15536:
current = 0x5958, required = 0x5958

# 重启数据库
shutdown immediate
startup

select /*+ FULL(t1) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3991

select /*+ index(t1 I_CON2) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3991

# 重建索引时恢复正常
conn lyj/lyj
alter table LYJ_1000 add primary key(id);

conn /as sysdba
select /*+ FULL(t1) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3992

select /*+ index(t1 I_CON2) */ owner#,name,con# from con$ t1 WHERE NAME='_NEXT_CONSTRAINT';

    OWNER# NAME                                 CON#
---------- ------------------------------ ----------
         0 _NEXT_CONSTRAINT                     3992
```

## 索引块结构解析
### B树索引的结构
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_01.png)

### ROWID
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_02.png)

### 转储B树索引
``` perl
conn lyj/lyj
select uo.object_id idx_object_id, ui.index_name, ui.table_name
  from user_objects uo,user_indexes ui
  where uo.object_name=ui.index_name;

IDX_OBJECT_ID INDEX_NAME                     TABLE_NAME
------------- ------------------------------ ------------------------------
        18360 SYS_C003991                    LYJ_1000

alter session set events 'immediate trace name treedump level 18360';

select VALUE from v$diag_info where NAME='Default Trace File';

vi /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_27697.trc
#---------------------------------------------------------------------------------
----- begin tree dump
branch: 0x1400223 20972067 (0: nrow: 10, level: 1)
   leaf: 0x1400224 20972068 (-1: nrow: 520 rrow: 520)
   leaf: 0x1400225 20972069 (0: nrow: 513 rrow: 513)
   leaf: 0x1400226 20972070 (1: nrow: 513 rrow: 513)
   leaf: 0x1400227 20972071 (2: nrow: 513 rrow: 513)
   leaf: 0x1400228 20972072 (3: nrow: 513 rrow: 513)
   leaf: 0x1400229 20972073 (4: nrow: 513 rrow: 513)
   leaf: 0x140022a 20972074 (5: nrow: 513 rrow: 513)
   leaf: 0x140022b 20972075 (6: nrow: 513 rrow: 513)
   leaf: 0x140022c 20972076 (7: nrow: 513 rrow: 513)
   leaf: 0x140022d 20972077 (8: nrow: 376 rrow: 376)
----- end tree dump
#---------------------------------------------------------------------------------
```

### 索引块结构
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_03.png)

### 分支块的结构
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_04.png)

### 根块
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_05.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_06.png)

示例：
``` perl
select header_file,header_block from dba_segments where segment_name='SYS_C003991';

HEADER_FILE HEADER_BLOCK
----------- ------------
          5          546

# root:546+1=547

alter system dump datafile 5 block 547;
select VALUE from v$diag_info where NAME='Default Trace File';

vi /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_27766.trc
#---------------------------------------------------------------------------------
Block header dump:  0x01400223
 Object id on Block? Y
 seg/obj: 0x47b8  csc: 0x00.224881  itc: 1  flg: E  typ: 2 - INDEX
     brn: 0  bdba: 0x1400220 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.00224881
Branch block dump
#=================
header address 139748680186444=0x7f19c670ba4c
kdxcolev 1
KDXCOLEV Flags = - - -
kdxcolok 0
kdxcoopc 0x80: opcode=0: iot flags=--- is converted=Y
kdxconco 1
kdxcosdc 0
kdxconro 9
kdxcofbo 46=0x2e
kdxcofeo 7984=0x1f30
kdxcoavs 7938
kdxbrlmc 20972068=0x1400224   # lmc: left most children
kdxbrsno 0
kdxbrbksz 8056
kdxbr2urrc 5
row#0[8048] dba: 20972069=0x1400225
col 0; len 3; (3):  c2 06 16
row#1[8040] dba: 20972070=0x1400226
col 0; len 3; (3):  c2 0b 23
row#2[8032] dba: 20972071=0x1400227
col 0; len 3; (3):  c2 10 30
row#3[8024] dba: 20972072=0x1400228
col 0; len 3; (3):  c2 15 3d
row#4[8016] dba: 20972073=0x1400229
col 0; len 3; (3):  c2 1a 4a
row#5[8008] dba: 20972074=0x140022a
col 0; len 3; (3):  c2 1f 57
row#6[8000] dba: 20972075=0x140022b
col 0; len 3; (3):  c2 24 64
row#7[7992] dba: 20972076=0x140022c
col 0; len 3; (3):  c2 2a 0d
row#8[7984] dba: 20972077=0x140022d
col 0; len 3; (3):  c2 2f 1a
----- end of branch block dump -----
#---------------------------------------------------------------------------------
```

### 叶子块
叶子块的结构
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_07.png)

转储索引叶子块（非唯一）
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_08.png)

转储索引叶子块（唯一）
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_09.png)

唯一与非唯一叶子块结构对比
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_10.png)

使用[DBA转换函数getbfno](/2018/03/28/oracle/Oracle%E7%89%B9%E6%AE%8A%E6%81%A2%E5%A4%8D%E5%8E%9F%E7%90%86%E4%B8%8E%E5%AE%9E%E6%88%98-%E8%AF%BE%E7%A8%8B%E5%AD%A6%E4%B9%A0/06_Oracle%E7%89%B9%E6%AE%8A%E6%81%A2%E5%A4%8D%E5%85%A5%E9%97%A8/#RDBA转换函数)计算出文件号和块号，转储页对应的叶子块示例如下：
``` perl
# row#0[8048] dba: 20972069=0x1400225
select getbfno('0x1400225') BFNO from dual;

BFNO
----------------------------------------------------------------------------------------------------
datafile# is:5
datablock is:549
dump command:alter system dump datafile 5 block 549;

select VALUE from v$diag_info where NAME='Default Trace File';

vi /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_27850.trc
#---------------------------------------------------------------------------------
Block header dump:  0x01400225
 Object id on Block? Y
 seg/obj: 0x47b8  csc: 0x00.224881  itc: 2  flg: E  typ: 2 - INDEX
     brn: 0  bdba: 0x1400220 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
0x02   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.00224881
Leaf block dump
# ===============
header address 140111196736100=0x7f6e2e1c4a64
kdxcolev 0
KDXCOLEV Flags = - - -
kdxcolok 0
kdxcoopc 0x80: opcode=0: iot flags=--- is converted=Y
kdxconco 1
kdxcosdc 0
kdxconro 513
kdxcofbo 1062=0x426
kdxcofeo 1881=0x759
kdxcoavs 819
kdxlespl 0
kdxlende 0
kdxlenxt 20972070=0x1400226   # 下一个块
kdxleprv 20972068=0x1400224   # 上一个块
kdxledsz 6
kdxlebksz 8032
row#0[8020] flag: ------, lock: 0, len=12, data:(6):  01 40 02 14 00 35
col 0; len 3; (3):  c2 06 16
......
#---------------------------------------------------------------------------------
```

BBED查看索引跟块和叶子块
``` perl
# 根块
BBED> set file 5 block 547
        FILE#           5
        BLOCK#          547

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 547                                   Dba:0x01400223
------------------------------------------------------------
 KTB Data Block (Index Branch)

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

 struct ktbbh, 48 bytes                     @20      
    ub1 ktbbhtyp                            @20      
    union ktbbhsid, 4 bytes                 @24      
    struct ktbbhcsc, 8 bytes                @28      
    sb2 ktbbhict                            @36      
    ub1 ktbbhflg                            @38      
    ub1 ktbbhfsl                            @39      
    ub4 ktbbhfnx                            @40      
    struct ktbbhitl[1], 24 bytes            @44      

 struct kdxbr, 24 bytes                     @76      
    struct kdxbrxco, 16 bytes               @76      
    ub4 kdxbrlmc                            @92      
    sb2 kdxbrsno                            @96      

 sb2 kd_off[9]                              @100     

 ub1 freespace[7938]                        @118     

 ub1 rowdata[72]                            @8056    

 ub4 tailchk                                @8188    

# 叶子块
BBED> set file 5 block 549
        FILE#           5
        BLOCK#          549

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/lyj_01.dbf (5)
 Block: 549                                   Dba:0x01400225
------------------------------------------------------------
 KTB Data Block (Index Leaf)

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

 struct kdxle, 32 bytes                     @100     
    struct kdxlexco, 16 bytes               @100     
    sb2 kdxlespl                            @116     
    sb2 kdxlende                            @118     
    ub4 kdxlenxt                            @120     
    ub4 kdxleprv                            @124     
    ub1 kdxledsz                            @128     
    ub1 kdxleflg                            @129     

 sb2 kd_off[513]                            @132     

 ub1 freespace[819]                         @1158    

 ub1 rowdata[6151]                          @1977    

 ub4 tailchk                                @8188   
```

### 唯一索引与非唯一索引的本质区别实验
#### 环境准备
``` perl
conn lyj/lyj
drop table t1 purge;
drop table t2 purge;
create table t1 (id number, name varchar(20));
create table t2 (id number, name varchar(20));

begin
  for i in 1 .. 5000 loop
  insert into t1 values(i,'lyj'||i);
  commit;
  end loop;
end;
/

begin
  for i in 1 .. 5000 loop
  insert into t2 values(i,'lyj'||i);
  commit;
  end loop;
end;
/

create index t1_idx on t1(id);
create unique index t2_idx on t2(id);
alter system flush buffer_cache;

select object_id, object_name from user_objects where object_name in ('T1_IDX','T2_IDX');

 OBJECT_ID OBJECT_NAME
---------- -------------------------
     18457 T1_IDX
     18458 T2_IDX
```

#### 非唯一索引叶子块结构
``` perl
# 新开会话，转储非唯一索引
alter session set events 'immediate trace name treedump level 18457';
select VALUE from v$diag_info where NAME='Default Trace File';
#---------------------------------------------------------------------------------
----- begin tree dump
branch: 0x140024b 20972107 (0: nrow: 11, level: 1)
   leaf: 0x140024c 20972108 (-1: nrow: 485 rrow: 485)  # 导出此叶子节点
   leaf: 0x140024d 20972109 (0: nrow: 479 rrow: 479)
   leaf: 0x140024e 20972110 (1: nrow: 479 rrow: 479)
   leaf: 0x140024f 20972111 (2: nrow: 479 rrow: 479)
   leaf: 0x1400250 20972112 (3: nrow: 479 rrow: 479)
   leaf: 0x1400251 20972113 (4: nrow: 478 rrow: 478)
   leaf: 0x1400252 20972114 (5: nrow: 479 rrow: 479)
   leaf: 0x1400253 20972115 (6: nrow: 479 rrow: 479)
   leaf: 0x1400254 20972116 (7: nrow: 479 rrow: 479)
   leaf: 0x1400255 20972117 (8: nrow: 478 rrow: 478)
   leaf: 0x1400256 20972118 (9: nrow: 206 rrow: 206)
----- end tree dump
#---------------------------------------------------------------------------------

select getbfno('0x140024c') BFNO from dual;

BFNO
--------------------------------------------------------------------
datafile# is:5
datablock is:588
dump command:alter system dump datafile 5 block 588;

# 新开会话dump上面的块的内容如下
#---------------------------------------------------------------------------------
Block header dump:  0x0140024c
 Object id on Block? Y
 seg/obj: 0x4819  csc: 0x00.232721  itc: 2  flg: E  typ: 2 - INDEX
     brn: 0  bdba: 0x1400248 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
0x02   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.00232721
Leaf block dump
# ===============
header address 139863743519332=0x7f3490bfda64
kdxcolev 0
KDXCOLEV Flags = - - -
kdxcolok 0
kdxcoopc 0x80: opcode=0: iot flags=--- is converted=Y
kdxconco 2
kdxcosdc 0
kdxconro 485
kdxcofbo 1006=0x3ee
kdxcofeo 1830=0x726
kdxcoavs 824
kdxlespl 0
kdxlende 0
kdxlenxt 20972109=0x140024d
kdxleprv 0=0x0
kdxledsz 0                      # Bytes in ROWID data
kdxlebksz 8032
row#0[8020] flag: ------, lock: 0, len=12
col 0; len 2; (2):  c1 02
col 1; len 6; (6):  01 40 00 85 00 00     # ROWID
row#1[8008] flag: ------, lock: 0, len=12
col 0; len 2; (2):  c1 03
col 1; len 6; (6):  01 40 00 85 00 01     # ROWID
#---------------------------------------------------------------------------------

非唯一索引的叶子节点每行len=12，ROWID值在col 1; len 6; (6)
```

#### 唯一索引叶子块结构
``` perl
# 新开会话，转储唯一索引
alter session set events 'immediate trace name treedump level 18458';
select VALUE from v$diag_info where NAME='Default Trace File';
#---------------------------------------------------------------------------------
----- begin tree dump
branch: 0x140025b 20972123 (0: nrow: 10, level: 1)
   leaf: 0x140025c 20972124 (-1: nrow: 520 rrow: 520)  # 导出此叶子节点
   leaf: 0x140025d 20972125 (0: nrow: 513 rrow: 513)
   leaf: 0x140025e 20972126 (1: nrow: 513 rrow: 513)
   leaf: 0x140025f 20972127 (2: nrow: 513 rrow: 513)
   leaf: 0x1400260 20972128 (3: nrow: 513 rrow: 513)
   leaf: 0x1400261 20972129 (4: nrow: 513 rrow: 513)
   leaf: 0x1400262 20972130 (5: nrow: 513 rrow: 513)
   leaf: 0x1400263 20972131 (6: nrow: 513 rrow: 513)
   leaf: 0x1400264 20972132 (7: nrow: 513 rrow: 513)
   leaf: 0x1400265 20972133 (8: nrow: 376 rrow: 376)
----- end tree dump
#---------------------------------------------------------------------------------

select getbfno('0x140025c') BFNO from dual;

BFNO
--------------------------------------------------------------------
datafile# is:5
datablock is:604
dump command:alter system dump datafile 5 block 604;

# 新开会话dump上面的块的内容如下
#---------------------------------------------------------------------------------
Block header dump:  0x0140025c
 Object id on Block? Y
 seg/obj: 0x481a  csc: 0x00.23272d  itc: 2  flg: E  typ: 2 - INDEX
     brn: 0  bdba: 0x1400258 ver: 0x01 opc: 0
     inc: 0  exflg: 0

 Itl           Xid                  Uba         Flag  Lck        Scn/Fsc
0x01   0x0000.000.00000000  0x00000000.0000.00  ----    0  fsc 0x0000.00000000
0x02   0xffff.000.00000000  0x00000000.0000.00  C---    0  scn 0x0000.0023272d
Leaf block dump
#===============
header address 140346102651492=0x7fa4df981a64
kdxcolev 0
KDXCOLEV Flags = - - -
kdxcolok 0
kdxcoopc 0x80: opcode=0: iot flags=--- is converted=Y
kdxconco 1
kdxcosdc 0
kdxconro 520
kdxcofbo 1076=0x434
kdxcofeo 1896=0x768
kdxcoavs 820
kdxlespl 0
kdxlende 0
kdxlenxt 20972125=0x140025d
kdxleprv 0=0x0
kdxledsz 6              # Bytes in ROWID data
kdxlebksz 8032
row#0[8021] flag: ------, lock: 0, len=11, data:(6):  01 40 02 3d 00 00    <- ROWID DATA
col 0; len 2; (2):  c1 02
row#1[8010] flag: ------, lock: 0, len=11, data:(6):  01 40 02 3d 00 01    <- ROWID DATA
col 0; len 2; (2):  c1 03
#---------------------------------------------------------------------------------

非唯一索引的叶子节点每行len=11，ROWID值在data:(6)
```

可见唯一索引与非唯一索引的本质区别就是ROWID所在的位置，唯一索引比非唯一索引的结构少ROWID Length Byte

## 表空间管理中的ASSM与MSSM的区别
### MSSM—Manual Segment Space Management 手动段空间管理（手动设置对象的存储属性）
这种管理方式就是使用freelist管理数据块的分配和回收，是一种只针对数据块分配的管理方式，这种方式可以让DBA有更大的空间管理余地，更大自由发挥空间，在早期的Oracle上都是通过这种方式管理块分配的。

**场景：** 对于一些数据块操作非常敏感的场合相对适用
**参数：** MSSM-由你设置freelists、freelistgroups、pctused、pctfree、initrans等参数来控制如何分配、使用段中的空间

**缺点：**
1. 需要设置更多的参数，例如上面所写的参数，操作复杂度更强
2. 参数设置值比较难评估，需要大量的测试过程
3. 需要了解数据库体系结构的DBA设置

**注意：**
freelist空闲列表是放在段头里面的，如果有多个用户同时访问列表势必会引发段头争用，导致“buffer busy waits”等待事件发生，建议多设几个freelist，防止争用。


### ASSM—Automatic SegmentSpace Management 自动段空间管理（自动设置对象的存储属性）
这种管理方式就是使用“位图bitmap”管理数据块的分配和回收，1为占用块不可分配，0为空闲块可分配，由于计算机就是由二进制编码的，所以操作二进制代码是非常快捷的。现在Oracle 10g 11g 默认值都是ASSM段空间管理方式。

**场景：** 希望数据块由Oracle自动分配管理的场合，不需要DBA介入太多

**参数：** ASSM-你只需控制一个参数pctfree，其他参数由Oracle自动管理，如果强行设置也将被忽略。

**三层位图模式管理段空间：**
第一层BMB（bit map block）记录每个extent中数据块的存储信息，只管理当前的extent内块，放在extent头中，这是leaf节点
第二层BMB管理第一层BMB记录，这是branch节点
第三层BMB管理第二层BMB记录，放在段头中，这是root节点

ASSM段头包含了每个Extent存储信息，Extent区头包含BMB信息

**优点：**
1. 自动化管理段空间，无需手动设置大量参数，简化了操作
2. 增大并发度，由于ASSM没有freelist概念，也就没有freelist列表争用情况，也就没有段头争用的情况，提高资源利用率。

**缺点：**
1.全表扫描性能没有MSSM模式下好
2.大数据加载，会导致性能下降，因为要自动维护位图表，需要一定的开销
3.影响索引的集群因子（clusteringfactor）
