---
title: Oracle特殊恢复原理与实战_04 SYSTEM文件头损坏的恢复
date: 2018-03-16
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## 10046跟踪数据库OPEN的过程
``` perl
shutdown immediate
startup mount
alter session set events '10046 trace name context forever,level 8';
alter database open;
col FILE_NAME for a50
select FILE_ID,FILE_NAME from dba_data_files;

   FILE_ID FILE_NAME
---------- --------------------------------------------------
         1 /u01/app/oracle/oradata/orcl/system01.dbf
         2 /u01/app/oracle/oradata/orcl/sysaux01.dbf
         3 /u01/app/oracle/oradata/orcl/undotbs01.dbf
         4 /u01/app/oracle/oradata/orcl/users01.dbf

select value from v$diag_info where name='Default Trace File';

VALUE
--------------------------------------------------------------------------------
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_10932.trc


从trace中能看到和分析数据库OPEN读取数据文件头的过程
#--------------------------------------------------------------------------------------------------------------------------
WAIT #139880206935832: nam='control file sequential read' ela= 11 file#=0 block#=23 blocks=1 obj#=-1 tim=1523176482644848
WAIT #139880206935832: nam='db file sequential read' ela= 20 file#=1 block#=1 blocks=1 obj#=-1 tim=1523176482644960
WAIT #139880206935832: nam='control file sequential read' ela= 12 file#=0 block#=179 blocks=1 obj#=-1 tim=1523176482645079
WAIT #139880206935832: nam='db file sequential read' ela= 11 file#=2 block#=1 blocks=1 obj#=-1 tim=1523176482645169
WAIT #139880206935832: nam='db file sequential read' ela= 10 file#=3 block#=1 blocks=1 obj#=-1 tim=1523176482645258
WAIT #139880206935832: nam='db file sequential read' ela= 26 file#=4 block#=1 blocks=1 obj#=-1 tim=1523176482645328
#--------------------------------------------------------------------------------------------------------------------------
```

<!-- more -->

## 模拟system文件头损坏的恢复
### 模拟破坏system文件头
``` perl
# 登陆BBED
BBED> info 
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /u01/app/oracle/oradata/orcl/system01.dbf                        89600
     2  /u01/app/oracle/oradata/orcl/sysaux01.dbf                        76800
     3  /u01/app/oracle/oradata/orcl/undotbs01.dbf                       25600
     4  /u01/app/oracle/oradata/orcl/users01.dbf                           640

# 模拟破坏system文件头
BBED> copy file 4 block 10 to file 1 block 1
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:    0 to  511           Dba:0x00400001
------------------------------------------------------------------------
 1ea20000 0a000001 6d3a0000 00000104 a6010000 04000000 80403600 00000000 
 00000000 00f80000 00000000 00000000 00000000 00000000 00000000 00000000 
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
Check value for File 1, Block 1:
current = 0x01a6, required = 0x01a6

# 此时，数据库已无法正常关闭和启动
SQL> shutdown immediate
ORA-01122: database file 1 failed verification check
ORA-01110: data file 1: '/u01/app/oracle/oradata/orcl/system01.dbf'
ORA-01210: data file header is media corrupt
SQL> shutdown abort
ORACLE instance shut down.
SQL> startup
ORACLE instance started.

Total System Global Area 4392697856 bytes
Fixed Size                  2260368 bytes
Variable Size             855638640 bytes
Database Buffers         3523215360 bytes
Redo Buffers               11583488 bytes
Database mounted.
ORA-01122: database file 1 failed verification check
ORA-01110: data file 1: '/u01/app/oracle/oradata/orcl/system01.dbf'
ORA-01210: data file header is media corrupt
```

### 构建SYSTEM文件头结构
``` perl
# BBED查看system文件头，报无效的块类型
BBED> set file 1 block
        FILE#           1
        BLOCK#          1

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                                     Dba:0x00400001
------------------------------------------------------------
BBED-00400: invalid blocktype (30)

# 找一个好的文件头，如2号文件
BBED> set file 2 block 1
        FILE#           2
        BLOCK#          1

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/sysaux01.dbf (2)
 Block: 1                                     Dba:0x00800001
------------------------------------------------------------
 Data File Header

 struct kcvfh, 860 bytes                    @0       
    struct kcvfhbfh, 20 bytes               @0       
    struct kcvfhhdr, 76 bytes               @20      
    ub4 kcvfhrdb                            @96      
    struct kcvfhcrs, 8 bytes                @100     
    ub4 kcvfhcrt                            @108     
    ub4 kcvfhrlc                            @112     
    struct kcvfhrls, 8 bytes                @116     
    ub4 kcvfhbti                            @124     
    struct kcvfhbsc, 8 bytes                @128     
    ub2 kcvfhbth                            @136     
    ub2 kcvfhsta                            @138     
    struct kcvfhckp, 36 bytes               @484     
    ub4 kcvfhcpc                            @140     
    ub4 kcvfhrts                            @144     
    ub4 kcvfhccc                            @148     
    struct kcvfhbcp, 36 bytes               @152     
    ub4 kcvfhbhz                            @312     
    struct kcvfhxcd, 16 bytes               @316     
    sword kcvfhtsn                          @332     
    ub2 kcvfhtln                            @336     
    text kcvfhtnm[30]                       @338     
    ub4 kcvfhrfn                            @368     
    struct kcvfhrfs, 8 bytes                @372     
    ub4 kcvfhrft                            @380     
    struct kcvfhafs, 8 bytes                @384     
    ub4 kcvfhbbc                            @392     
    ub4 kcvfhncb                            @396     
    ub4 kcvfhmcb                            @400     
    ub4 kcvfhlcb                            @404     
    ub4 kcvfhbcs                            @408     
    ub2 kcvfhofb                            @412     
    ub2 kcvfhnfb                            @414     
    ub4 kcvfhprc                            @416     
    struct kcvfhprs, 8 bytes                @420     
    struct kcvfhprfs, 8 bytes               @428     
    ub4 kcvfhtrt                            @444     

 ub4 tailchk                                @8188    

# 将2号文件的1号块拷贝到1号文件(system)的1号块上
BBED> copy file 2 block 1 to file 1 block 1
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:    0 to  511           Dba:0x00400001
------------------------------------------------------------------------
 0ba20000 01008000 00000000 00000104 608f0000 00000000 0004200b acfd6d59 
 4f52434c 00000000 00080000 002c0100 00200000 02000300 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 b9060000 00000000 3fb2f839 2cb2f839 01000000 00000000 00000000 
 00000000 00000000 00000400 21000000 00000000 20000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 01000000 06005359 53415558 00000000 00000000 
 00000000 00000000 00000000 00000000 02000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 e8b00500 00000000 22a3fd39 01000000 1c000000 1b990000 10005166 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0x8f60, required = 0x8f60
```

### BBED修复文件头
需要修改的项如下：
![](http://p2c0rtsgc.bkt.clouddn.com/0408_oracle_dsi_01.png)

#### BBED修改文件头block的rdba地址
``` perl
BBED> p kcvfhbfh
struct kcvfhbfh, 20 bytes                   @0       
   ub1 type_kcbh                            @0        0x0b
   ub1 frmt_kcbh                            @1        0xa2
   ub1 spare1_kcbh                          @2        0x00
   ub1 spare2_kcbh                          @3        0x00
   ub4 rdba_kcbh                            @4        0x00800001     # 要修改，0x00800001代表2号文件1号块，这个要改成0x00400001(01004000)
   ub4 bas_kcbh                             @8        0x00000000
   ub2 wrp_kcbh                             @12       0x0000
   ub1 seq_kcbh                             @14       0x01
   ub1 flg_kcbh                             @15       0x04 (KCBHFCKV)
   ub2 chkval_kcbh                          @16       0x8f60
   ub2 spare3_kcbh                          @18       0x0000

# rdba转换成文件号块号的方法
rdba：0x00800001
      0000 0000 1000 0000 0000 0000 0000 0001
前10位文件号：0000 0000 10  -  2号文件
后22位块号：  00 0000 0000 0000 0000 0001 - 1号块

1号文件1号块：
前10位文件号：0000 0000 01 - 1号文件
后22位块号：  00 0000 0000 0000 0000 0001 - 1号块

rdba: 0000 0000 0100 0000 0000 0000 0000 0001
      0x00400001

# 也可以在一个open数据库中执行以下命令计算rdba对应的文件号和块坏
select dbms_utility.data_block_address_file(to_number('800001','xxxxxxxx')) file_id,
       dbms_utility.data_block_address_block(to_number('800001','xxxxxxxx')) block_id 
from dual;

   FILE_ID   BLOCK_ID
---------- ----------
         2          1

# BBED
BBED> set file 1 block 1 offset 4 count 32
        FILE#           1
        BLOCK#          1
        OFFSET          4
        COUNT           32

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:    4 to   35           Dba:0x00400001
------------------------------------------------------------------------
 01008000 00000000 00000104 608f0000 00000000 0004200b acfd6d59 4f52434c 

 <32 bytes per line>

BBED> modify /x 01004000 offset 4
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:    4 to   35           Dba:0x00400001
------------------------------------------------------------------------
 01004000 00000000 00000104 608f0000 00000000 0004200b acfd6d59 4f52434c 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0x8fa0, required = 0x8fa0

BBED> p kcvfhbfh
struct kcvfhbfh, 20 bytes                   @0       
   ub1 type_kcbh                            @0        0x0b
   ub1 frmt_kcbh                            @1        0xa2
   ub1 spare1_kcbh                          @2        0x00
   ub1 spare2_kcbh                          @3        0x00
   ub4 rdba_kcbh                            @4        0x00400001
   ub4 bas_kcbh                             @8        0x00000000
   ub2 wrp_kcbh                             @12       0x0000
   ub1 seq_kcbh                             @14       0x01
   ub1 flg_kcbh                             @15       0x04 (KCBHFCKV)
   ub2 chkval_kcbh                          @16       0x8fa0
   ub2 spare3_kcbh                          @18       0x0000
```

#### BBED修改文件头的文件大小
``` perl
BBED> p kcvfhhdr
struct kcvfhhdr, 76 bytes                   @20      
   ub4 kccfhswv                             @20       0x00000000
   ub4 kccfhcvn                             @24       0x0b200400
   ub4 kccfhdbi                             @28       0x596dfdac
   text kccfhdbn[0]                         @32      O
   text kccfhdbn[1]                         @33      R
   text kccfhdbn[2]                         @34      C
   text kccfhdbn[3]                         @35      L
   text kccfhdbn[4]                         @36       
   text kccfhdbn[5]                         @37       
   text kccfhdbn[6]                         @38       
   text kccfhdbn[7]                         @39       
   ub4 kccfhcsq                             @40       0x00000800
   ub4 kccfhfsz                             @44       0x00012c00   # 需要修改的项
.....

# 查看system01.dbf文件大小
ll /u01/app/oracle/oradata/orcl/system01.dbf
-rw-r----- 1 oracle oinstall 734011392 Apr  8 17:57 /u01/app/oracle/oradata/orcl/system01.dbf

# 在一个open数据库中执行以下命令计算需要修改的文件大小（所有文件前面都有一个OS头块-0号块，计算时要减去）
SQL> select to_char((734011392-8192)/8192,'xxxxxxxx') kccfhfsz from dual;

KCCFHFSZ
---------
    15e00      # 换算Little Endian: 005e0100

# BBED修改文件头的文件大小
BBED> set file 1 block 1 offset 44
        FILE#           1
        BLOCK#          1
        OFFSET          44

BBED> modify /x 005e0100
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:   44 to   75           Dba:0x00400001
------------------------------------------------------------------------
 005e0100 00200000 02000300 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
Check value for File 1, Block 1:
current = 0xfda0, required = 0xfda0
```

#### BBED修改文件头的文件号
``` perl
BBED> p kcvfhhdr
struct kcvfhhdr, 76 bytes                   @20      
   ub4 kccfhswv                             @20       0x00000000
   ub4 kccfhcvn                             @24       0x0b200400
   ub4 kccfhdbi                             @28       0x596dfdac
   text kccfhdbn[0]                         @32      O
   text kccfhdbn[1]                         @33      R
   text kccfhdbn[2]                         @34      C
   text kccfhdbn[3]                         @35      L
   text kccfhdbn[4]                         @36       
   text kccfhdbn[5]                         @37       
   text kccfhdbn[6]                         @38       
   text kccfhdbn[7]                         @39       
   ub4 kccfhcsq                             @40       0x00000800
   ub4 kccfhfsz                             @44       0x00015e00
   s_blkz kccfhbsz                          @48       0x00
   ub2 kccfhfno                             @52       0x0002   # 需要修改的项

BBED> set file 1 block 1 offset 52
        FILE#           1
        BLOCK#          1
        OFFSET          52

BBED> dump count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:   52 to   83           Dba:0x00400001
------------------------------------------------------------------------
 02000300 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 01 offset 52
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:   52 to   83           Dba:0x00400001
------------------------------------------------------------------------
 01000300 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0xfda3, required = 0xfda3
```

#### BBED修改文件头的root数据块号
``` perl
# 只有system才有root数据块地址，放到1号文件520号块上(11g)
# 可以在一个open的同版本数据库中用以下命令确认root数据块号
SQL> select fhfno,to_char(fhrdb,'xxxxxxxx') fhrdb from x$kcvfh order by 1;

     FHFNO FHRDB
---------- ---------
         1    400208
         2         0
         3         0
         4         0

select dbms_utility.data_block_address_file(to_number('400208','xxxxxxxx')) file_id,
       dbms_utility.data_block_address_block(to_number('400208','xxxxxxxx')) block_id 
from dual;

   FILE_ID   BLOCK_ID
---------- ----------
         1        520

# 520号块转换成16进制
SQL> select to_char(520,'xxxxxxxx') from dual;

TO_CHAR(5
---------
      208   # ->00400208 (1号文件，520号块)  -> 换算Little Endian: 08024000

# BBED修改文件头的root数据块号
BBED> set file 1 block 1 offset 96 count 32
        FILE#           1
        BLOCK#          1
        OFFSET          96
        COUNT           32

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:   96 to  127           Dba:0x00400001
------------------------------------------------------------------------
 00000000 b9060000 00000000 3fb2f839 2cb2f839 01000000 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 08024000
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:   96 to  127           Dba:0x00400001
------------------------------------------------------------------------
 08024000 b9060000 00000000 3fb2f839 2cb2f839 01000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0xffeb, required = 0xffeb

# 如果1号文件的520号块损坏，可以找一个同版本数据库system文件，用它520号块进行覆盖即可
```

#### BBED修改文件头的文件创建SCN
``` perl
# 从控件文件中查看文件创建SCN(数据库在mount状态)
SQL> select file#,creation_change# from v$datafile;

     FILE# CREATION_CHANGE#
---------- ----------------
         1                7
         2             1721
         3             2686
         4            14940

SQL> select to_char(7,'xxxxxxx') from dual;

TO_CHAR(
--------
       7  # -> 换算Little Endian: 0700

SQL> select to_char(1721,'xxxxxxx') from dual;

TO_CHAR(
--------
     6b9  # -> 换算Little Endian: b906

# BBED修改文件头的文件创建SCN
BBED> p kcvfhcrs
struct kcvfhcrs, 8 bytes                    @100     
   ub4 kscnbas                              @100      0x000006b9
   ub2 kscnwrp                              @104      0x0000

BBED> set file 1 block 1 offset 100
        FILE#           1
        BLOCK#          1
        OFFSET          100

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  100 to  131           Dba:0x00400001
------------------------------------------------------------------------
 b9060000 00000000 3fb2f839 2cb2f839 01000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 0700
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  100 to  131           Dba:0x00400001
------------------------------------------------------------------------
 07000000 00000000 3fb2f839 2cb2f839 01000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0xf955, required = 0xf955                           @104      0x0000
```

#### BBED修改文件头的文件创建时间
``` perl
# 从控制文件中查看文件创建时间(数据库在mount状态)
select file#,to_char(creation_time,'yyyy-mm-dd hh24:mi:ss') creation_time_file,
(to_char(creation_time,'yyyy')-1988)*12*31*24*3600
+(to_char(creation_time,'mm')-1)*31*24*3600
+(to_char(creation_time,'dd')-1)*24*3600
+to_char(creation_time,'hh24')*3600
+to_char(creation_time,'mi')*60
+to_char(creation_time,'ss') creation_name_scn
from v$datafile order by 1;

     FILE# CREATION_TIME_FILE  CREATION_NAME_SCN
---------- ------------------- -----------------
         1 2018-04-04 22:37:40         972599860
         2 2018-04-04 22:37:51         972599871
         3 2018-04-04 22:37:57         972599877
         4 2018-04-04 22:38:08         972599888

# 说明：ORACLE 数据库是从1988/01/01 00:00:00 开始记录SCN,也就是说我们的数据库的使用最早时间只能是从1988 年元旦凌晨开始，
# 那么也就是说数据库记录的创建时间可以采用这个时间点为起点,然后每增加一秒,数据库的kcvfhcrt 就增加1，
# ORACLE 为了计算简便,每个月按照31 天计算

select to_char(972599860,'xxxxxxxx') from dual;

TO_CHAR(9
---------
 39f8b234   # -> 换算Little Endian: 34b2f839

select to_char(972599871,'xxxxxxxx') from dual;

TO_CHAR(9
---------
 39f8b23f   # -> 换算Little Endian: 3fb2f839

# BBED修改文件头的文件创建时间
BBED> p kcvfhcrt
ub4 kcvfhcrt                                @108      0x39f8b23f

BBED> set file 1 block 1 offset 108 count 32
        FILE#           1
        BLOCK#          1
        OFFSET          108
        COUNT           32

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  108 to  139           Dba:0x00400001
------------------------------------------------------------------------
 3fb2f839 2cb2f839 01000000 00000000 00000000 00000000 00000000 00000400 

 <32 bytes per line>

BBED> modify /x 34
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  108 to  139           Dba:0x00400001
------------------------------------------------------------------------
 34b2f839 2cb2f839 01000000 00000000 00000000 00000000 00000000 00000400 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0xf95e, required = 0xf95e
```

#### BBED修改文件头的文件状态
``` perl
BBED> p kcvfhsta
ub2 kcvfhsta                                @138      0x0004 (KCVFHOFZ)

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  138 to  169           Dba:0x00400001
------------------------------------------------------------------------
 04002100 00000000 00002000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

# 这里不需要修改
# 文件头的文件状态定义 -> from kcv3.h
KCVFHHBP 0x01 /* hotbackup-in-progress on file (fuzzy file) */
KCVFHOFZ 0x04 /* Online FuZzy because it was online and db open */
KCVFHMFZ 0x10 /* Media recovery FuZzy - file in media recovery */
KCVFHAFZ 0x40 /* Absolutely FuZzy - fuzzyness from file scan */
KCVFH_FUZZY (KCVFHHBP|KCVFHOFZ|KCVFHMFZ|KCVFHAFZ|KCVFHPCP|KCVFHSBY)
KCVFH_OOFUZZY (KCVFHHBP|KCVFHMFZ|KCVFHAFZ|KCVFHPCP|KCVFHSBY)
KCVFH_NMFUZZY (KCVFH_FUZZY^KCVFHMFZ^KCVFHSBY) /* non-MR fuzzy */
KCVFHBCP 0x100 /* Bad Checkpoint - no enabled thread bitvec */
KCVFHFMH 0x200 /* Freshly Munged Header. resetlogs not finished */
KCVFHXCH 0x400 /* eXternally CacHed by operating system */
KCVFHZBA 0x800 /* Zeroed Blocks Allowed */
KCVFHPCP 0x1000 /* Proxy Copy in Progress */
KCVFHRBS 0x2000 /* does kcvfhrdb point to bootstrap$ ? */
KCVFHSBY 0x4000 /* media rcv fuzzy due to standby apply */
KCVFHL0C 0x8000 /* Incremental level 0 copy */
```

#### BBED修改文件头的表空间号
``` perl
# 从控制文件中查看文件表空间号(数据库在mount状态)
SQL> select file#,ts# from v$datafile;

     FILE#        TS#
---------- ----------
         1          0
         2          1
         3          2
         4          4

# BBED修改文件头的表空间号
BBED> p kcvfhtsn
sword kcvfhtsn                              @332      1

BBED> set file 1 block 1 offset 332 count 32
        FILE#           1
        BLOCK#          1
        OFFSET          332
        COUNT           32

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  332 to  363           Dba:0x00400001
------------------------------------------------------------------------
 01000000 06005359 53415558 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 00
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  332 to  363           Dba:0x00400001
------------------------------------------------------------------------
 00000000 06005359 53415558 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0xf95f, required = 0xf95f
```

#### BBED修改文件头的表空间名长度
``` perl
BBED> p kcvfhtln
ub2 kcvfhtln                                @336      0x0006
# SYSTEM也是6个字符，这里不用改了
```

#### BBED修改文件头的表空间名称
``` perl
BBED> p kcvfhtnm
text kcvfhtnm[0]                            @338     S
text kcvfhtnm[1]                            @339     Y
text kcvfhtnm[2]                            @340     S
text kcvfhtnm[3]                            @341     A
text kcvfhtnm[4]                            @342     U
text kcvfhtnm[5]                            @343     X
......

BBED> set file 1 block 1 offset 338 count 32
        FILE#           1
        BLOCK#          1
        OFFSET          338
        COUNT           32

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  338 to  369           Dba:0x00400001
------------------------------------------------------------------------
 53595341 55580000 00000000 00000000 00000000 00000000 00000000 00000200 

 <32 bytes per line>

# SYSAUX转换成16进制
SQL> select dump('SYSAUX',16) from dual;

DUMP('SYSAUX',16)
-------------------------------
Typ=96 Len=6: 53,59,53,41,55,58

# SYSTEM转换成16进制
SQL> select dump('SYSTEM',16) from dual;

DUMP('SYSTEM',16)
-------------------------------
Typ=96 Len=6: 53,59,53,54,45,4d  # 字符型的不需要颠倒

# BBED修改文件头的表空间名称
BBED> set file 1 block 1 offset 338 count 32
        FILE#           1
        BLOCK#          1
        OFFSET          338
        COUNT           32

BBED> modify /x 53595354
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  338 to  369           Dba:0x00400001
------------------------------------------------------------------------
 53595354 55580000 00000000 00000000 00000000 00000000 00000000 00000200 

 <32 bytes per line>

# 338+4 = 342
BBED> dump offset 342 count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  342 to  373           Dba:0x00400001
------------------------------------------------------------------------
 55580000 00000000 00000000 00000000 00000000 00000000 00000200 00000000 

BBED> modify /x 454d offset 342
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  342 to  373           Dba:0x00400001
------------------------------------------------------------------------
 454d0000 00000000 00000000 00000000 00000000 00000000 00000200 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0xf94f, required = 0xf94f

BBED> p kcvfhtnm
text kcvfhtnm[0]                            @338     S
text kcvfhtnm[1]                            @339     Y
text kcvfhtnm[2]                            @340     S
text kcvfhtnm[3]                            @341     T
text kcvfhtnm[4]                            @342     E
text kcvfhtnm[5]                            @343     M
......
```

#### BBED修改文件头的相对文件号
``` perl
# 能从控制文件中查看文件头的相对文件号(数据库在mount状态)
SQL> select file#,rfile# from v$datafile;

     FILE#     RFILE#
---------- ----------
         1          1
         2          2
         3          3
         4          4

# BBED修改文件头的相对文件号
BBED> p kcvfhrfn
ub4 kcvfhrfn                                @368      0x00000002

BBED> set file 1 block 1 offset 368 count 32
        FILE#           1
        BLOCK#          1
        OFFSET          368
        COUNT           32

BBED> dump
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  368 to  399           Dba:0x00400001
------------------------------------------------------------------------
 02000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 01
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  368 to  399           Dba:0x00400001
------------------------------------------------------------------------
 01000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 1:
current = 0xf94c, required = 0xf94c
```

#### BBED修改文件头的检查点SCN
``` perl
# 能从控制文件中查看文件头的检查点SCN(数据库在mount状态)
select FILE#,CREATION_CHANGE#,CHECKPOINT_CHANGE#,UNRECOVERABLE_CHANGE#,LAST_CHANGE#,OFFLINE_CHANGE# 
from v$datafile order by 1,2;

     FILE# CREATION_CHANGE# CHECKPOINT_CHANGE# UNRECOVERABLE_CHANGE# LAST_CHANGE# OFFLINE_CHANGE#
---------- ---------------- ------------------ --------------------- ------------ ---------------
         1                7             372968                     0                            0
         2             1721             372968                     0                            0
         3             2686             372968                     0                            0
         4            14940             372968                     0                            0

select to_char(372968,'xxxxxxxx') from dual;

TO_CHAR(3
---------
    5b0e8   # -> 换算Little Endian: e8b005

# BBED修改文件头的检查点SCN不是一定要做的，但打开数据库时可能会提示需要介质恢复
BBED> p kcvfhckp
struct kcvfhckp, 36 bytes                   @484     
   struct kcvcpscn, 8 bytes                 @484     
      ub4 kscnbas                           @484      0x0005b0e8   # 与控制文件中一致，就不需要修改了
      ub2 kscnwrp                           @488      0x0000
   ub4 kcvcptim                             @492      0x39fda322
   ub2 kcvcpthr                             @496      0x0001
   union u, 12 bytes                        @500     
      struct kcvcprba, 12 bytes             @500     
         ub4 kcrbaseq                       @500      0x0000001c
         ub4 kcrbabno                       @504      0x0000991b
         ub2 kcrbabof                       @508      0x0010
   ub1 kcvcpetb[0]                          @512      0x02
   ub1 kcvcpetb[1]                          @513      0x00
   ub1 kcvcpetb[2]                          @514      0x00
   ub1 kcvcpetb[3]                          @515      0x00
   ub1 kcvcpetb[4]                          @516      0x00
   ub1 kcvcpetb[5]                          @517      0x00
   ub1 kcvcpetb[6]                          @518      0x00
   ub1 kcvcpetb[7]                          @519      0x00

BBED> dump offset 484 count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  484 to  515           Dba:0x00400001
------------------------------------------------------------------------
 e8b00500 00000000 22a3fd39 01000000 1c000000 1b990000 10005166 02000000 

 <32 bytes per line>
```

#### BBED修改文件头的检查点的时间
``` perl
# 能从控制文件中查看文件头的检查点时间(数据库在mount状态)
select file#,to_char(checkpoint_time,'yyyy-mm-dd hh24:mi:ss') checkpoint_time_file,
(to_char(checkpoint_time,'yyyy')-1988)*12*31*24*3600+
(to_char(checkpoint_time,'mm')-1)*31*24*3600
+(to_char(checkpoint_time,'dd')-1)*24*3600
+to_char(checkpoint_time,'hh24')*3600
+to_char(checkpoint_time,'mi')*60
+to_char(checkpoint_time,'ss') checkpoint_time_scn
from v$datafile order by 1;  

     FILE# CHECKPOINT_TIME_FIL CHECKPOINT_TIME_SCN
---------- ------------------- -------------------
         1 2018-04-08 16:34:42           972923682
         2 2018-04-08 16:34:42           972923682
         3 2018-04-08 16:34:42           972923682
         4 2018-04-08 16:34:42           972923682

select to_char(972923682,'xxxxxxxx') from dual;

TO_CHAR(9
---------
 39fda322   # -> 换算Little Endian: 22a3fd39

# BBED修改文件头的检查点的时间
BBED> p kcvfhckp
struct kcvfhckp, 36 bytes                   @484     
   struct kcvcpscn, 8 bytes                 @484     
      ub4 kscnbas                           @484      0x0005b0e8
      ub2 kscnwrp                           @488      0x0000
   ub4 kcvcptim                             @492      0x39fda322   # 文件头的检查点的时间偏移量是492
   ub2 kcvcpthr                             @496      0x0001
   union u, 12 bytes                        @500     
      struct kcvcprba, 12 bytes             @500     
         ub4 kcrbaseq                       @500      0x0000001c
         ub4 kcrbabno                       @504      0x0000991b
         ub2 kcrbabof                       @508      0x0010
   ub1 kcvcpetb[0]                          @512      0x02
   ub1 kcvcpetb[1]                          @513      0x00
   ub1 kcvcpetb[2]                          @514      0x00
   ub1 kcvcpetb[3]                          @515      0x00
   ub1 kcvcpetb[4]                          @516      0x00
   ub1 kcvcpetb[5]                          @517      0x00
   ub1 kcvcpetb[6]                          @518      0x00
   ub1 kcvcpetb[7]                          @519      0x00

BBED> dump offset 492 count 32
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 1                Offsets:  492 to  523           Dba:0x00400001
------------------------------------------------------------------------
 22a3fd39 01000000 1c000000 1b990000 10005166 02000000 00000000 00000000   # 与控制文件中一致，不需要修复

 <32 bytes per line>
```

### 通过dbv检查文件头是否正确
``` perl
dbv file=/u01/app/oracle/oradata/orcl/system01.dbf start=1 end=2

DBVERIFY: Release 11.2.0.4.0 - Production on Mon Apr 9 11:58:27 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/orcl/system01.dbf


DBVERIFY - Verification complete

Total Pages Examined         : 2
Total Pages Processed (Data) : 0
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 0
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 2
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 0
Total Pages Marked Corrupt   : 0
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 361334 (0.361334)
```

### 数据库正常打开
``` perl
SQL> alter database open;

Database altered.

# 如果打开库报错，可检查修改checkpoint count，或者重建控制文件
kcvfhcpc (offset 140) - Datafile checkpoint count      # 数据文件检查点的计数器
kcffhccc (offset 148) - Controlfile checkpoint count   # 控制文件检查点的计数器

# 一般kcvfhcpc比kcvfhccc大1
BBED> p kcvfhcpc
ub4 kcvfhcpc                                @140      0x00000024

BBED> p kcvfhccc
ub4 kcvfhccc                                @148      0x00000023
```

## 参考
* [Oracle RMAN与Fuzzy模糊位](http://www.askmaclean.com/archives/oracle-rman%E4%B8%8Efuzzy%E6%A8%A1%E7%B3%8A%E4%BD%8D.html)