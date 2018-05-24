---
title: Oracle特殊恢复原理与实战_10 恢复ora-600[4193][4194]的错误
date: 2018-05-10
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

## 恢复ora-600[4193]的错误
### ora-600[4193]解析
ORA-600[41XX] 这种错误基本都于UNDO有关系
ora-600[4193]：表示undo和redo不一致（Arg [a] Undo record seq number，Arg [b] Redo record seq number ）
数据库在启动时需要进行一个前滚的操作，在前滚时会应用redo 到undo block上，操作时会检查undo record里的seq#和redo record里的seq#。正常情况下，这2者的seq# 应该是一致的，在一致的情况下，我们才应用redo record 到undo record。如果不一致就会出现ORA-600[4193][a][b]的错误。其中a是undo里的seq#记录，b是redo里的seq#值。 这里的值都是十六进程，可以通过to_number() 函数来转换。

<!-- more -->

### 人为构造ORA-600[4193]错误
``` perl
SQL> select header_file,header_block from dba_segments where segment_name='SYSTEM';

HEADER_FILE HEADER_BLOCK
----------- ------------
          1          128

SQL> alter system dump datafile 1 block 128;

System altered.

SQL> oradebug setmypid
Statement processed.
SQL> oradebug tracefile_name
/u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_7485.trc

# 查看trace
#------------------------------------------------------------------------------------------------
  -----------------------------------------------------------------
  Extent Header:: spare1: 0      spare2: 0      #extents: 6      #blocks: 47    
                  last map  0x00000000  #maps: 0      offset: 4128  
      Highwater::  0x00400212  ext#: 2      blk#: 2      ext size: 8     
  #blocks in seg. hdr's freelists: 0     
  #blocks below: 0     
  mapblk  0x00000000  offset: 2     
                   Unlocked
     Map Header:: next  0x00000000  #extents: 6    obj#: 0      flag: 0x40000000
  Extent Map
  -----------------------------------------------------------------
   0x00400081  length: 7     
   0x00400088  length: 8     
   0x00400210  length: 8     
   0x00400218  length: 8     
   0x00400220  length: 8     
   0x00400228  length: 8     
  
  TRN CTL:: seq: 0x001a chd: 0x0002 ctl: 0x0003 inc: 0x00000000 nfb: 0x0001
            mgc: 0x8002 xts: 0x0068 flg: 0x0001 opt: 2147483646 (0x7ffffffe)
            uba: 0x00400212.001a.07 scn: 0x0000.0012926d     # uba: 0x00400212.001a.07
Version: 0x01
  FREE BLOCK POOL::   # 可用的UNDO块
    uba: 0x00400212.001a.08 ext: 0x2  spc: 0x18be  
    uba: 0x00000000.0019.21 ext: 0x1  spc: 0x11c4  
    uba: 0x00000000.0018.02 ext: 0x0  spc: 0x1e62  
    uba: 0x00000000.0000.00 ext: 0x0  spc: 0x0     
    uba: 0x00000000.0000.00 ext: 0x0  spc: 0x0     
  TRN TBL::
 
  index  state cflags  wrap#    uel         scn            dba            parent-xid    nub     stmt_num
  ------------------------------------------------------------------------------------------------
   0x00    9    0x00  0x0013  0x004a  0x0000.00129275  0x0040008e  0x0000.000.00000000  0x00000001   0x00000000
   0x01    9    0x00  0x0013  0x000a  0x0000.00129279  0x0040008e  0x0000.000.00000000  0x00000001   0x00000000
   0x02    9    0x00  0x0013  0x0007  0x0000.0012926f  0x0040008d  0x0000.000.00000000  0x00000001   0x00000000
.....
#------------------------------------------------------------------------------------------------

# BBED
bbed
BBED> set file 1 block 128
        FILE#           1
        BLOCK#          128

BBED> p ktuxc
struct ktuxc, 104 bytes                     @4148    
   struct ktuxcscn, 8 bytes                 @4148    
      ub4 kscnbas                           @4148     0x0012926d
      ub2 kscnwrp                           @4152     0x0000
   struct ktuxcuba, 8 bytes                 @4156    
      ub4 kubadba                           @4156     0x00400212
      ub2 kubaseq                           @4160     0x001a
      ub1 kubarec                           @4162     0x07        # uba: 0x00400212.001a.07  跟上面的dump对应
   sb2 ktuxcflg                             @4164     1 (KTUXCFSK)
   ub2 ktuxcseq                             @4166     0x001a
   sb2 ktuxcnfb                             @4168     1
   ub4 ktuxcinc                             @4172     0x00000000
   sb2 ktuxcchd                             @4176     2
   sb2 ktuxcctl                             @4178     3
   ub2 ktuxcmgc                             @4180     0x8002
   ub4 ktuxcopt                             @4188     0x7ffffffe
   struct ktuxcfbp[0], 12 bytes             @4192    
      struct ktufbuba, 8 bytes              @4192    
         ub4 kubadba                        @4192     0x00400212   
         ub2 kubaseq                        @4196     0x001a   # undo空闲块的sequence
         ub1 kubarec                        @4198     0x08   
.....


# 查看
BBED> set dba 0x00400212
        DBA             0x00400212 (4194834 1,530)

BBED> map /v
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 530                                   Dba:0x00400212
------------------------------------------------------------
 Undo Data

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

 struct ktubh, 32 bytes                     @20      
    struct ktubhxid, 8 bytes                @20      
    ub2 ktubhseq                            @28      
    ub1 ktubhcnt                            @30      
    ub1 ktubhirb                            @31      
    ub1 ktubhicl                            @32      
    ub1 ktubhflg                            @33      
    ub2 ktubhidx[9]                         @34      

 ub1 freespace[6336]                        @52      

 ub1 undodata[1800]                         @6388    

 ub4 tailchk                                @8188    

BBED> p ktubh
struct ktubh, 32 bytes                      @20      
   struct ktubhxid, 8 bytes                 @20      
      ub2 kxidusn                           @20       0x0000
      ub2 kxidslt                           @22       0x0003
      ub4 kxidsqn                           @24       0x00000014
   ub2 ktubhseq                             @28       0x001a    # 修改这个值
   ub1 ktubhcnt                             @30       0x08
   ub1 ktubhirb                             @31       0x08
   ub1 ktubhicl                             @32       0x00
   ub1 ktubhflg                             @33       0x00
   ub2 ktubhidx[0]                          @34       0x1fe8
   ub2 ktubhidx[1]                          @36       0x1f00
   ub2 ktubhidx[2]                          @38       0x1e24
   ub2 ktubhidx[3]                          @40       0x1d3c
   ub2 ktubhidx[4]                          @42       0x1c64
   ub2 ktubhidx[5]                          @44       0x1b7c
   ub2 ktubhidx[6]                          @46       0x1aa4
   ub2 ktubhidx[7]                          @48       0x19bc
   ub2 ktubhidx[8]                          @50       0x18e0

# 修改前先将数据库关了
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.

# 在BBED中修改
BBED> modify /x 1b offset 28
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 530              Offsets:   28 to  539           Dba:0x00400212
------------------------------------------------------------------------
 1b000808 0000e81f 001f241e 3c1d641c 7c1ba41a bc19e018 00000000 00000000 
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
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 530:
current = 0xb524, required = 0xb524

SQL> startup mount
ORACLE instance started.

Total System Global Area 4392697856 bytes
Fixed Size                  2260368 bytes
Variable Size             855638640 bytes
Database Buffers         3523215360 bytes
Redo Buffers               11583488 bytes
Database mounted.

SQL> alter database open;
alter database open
*
ERROR at line 1:
ORA-01092: ORACLE instance terminated. Disconnection forced
ORA-00600: internal error code, arguments: [4193], [], [], [], [], [], [], [], [], [], [], []
Process ID: 7732
Session ID: 63 Serial number: 5
```

看alert日志错误信息
``` perl
Block recovery completed at rba 19.68.16, scn 0.309423
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_smon_26350.trc:
ORA-01595: error freeing extent (2) of rollback segment (2))
ORA-00600: internal error code, arguments: [4193], [31], [1^A^A^A^@], [], [], [], [], [], [], [], [], []
Create Relation INCIDENT
Wed Apr 04 22:09:11 2018
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_p000_26578.trc  (incident=24182):
ORA-00600: internal error code, arguments: [2023], [0], [0], [], [], [], [], [], [], [], [], []
Incident details in: /u01/app/oracle/diag/rdbms/orcl/orcl/incident/incdir_24182/orcl_p000_26578_i24182.trc
Use ADRCI or Support Workbench to package the incident.
See Note 411.1 at My Oracle Support for error and packaging details.
Wed Apr 04 22:09:12 2018
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_smon_26350.trc  (incident=24145):
"alert.log" 3759L, 169144C                                                                    1,1           Top
Block recovery from logseq 298, block 40704 to scn 2058829
Recovery of Online Redo Log: Thread 1 Group 1 Seq 298 Reading mem 0
  Mem# 0: /u01/app/oracle/oradata/orcl/redo01.log
Block recovery stopped at EOT rba 298.40706.16
Block recovery completed at rba 298.40706.16, scn 0.2058827
Block recovery from logseq 298, block 40704 to scn 2058826
Recovery of Online Redo Log: Thread 1 Group 1 Seq 298 Reading mem 0
  Mem# 0: /u01/app/oracle/oradata/orcl/redo01.log
Sun May 13 22:31:58 2018
Dumping diagnostic data in directory=[cdmp_20180513223158], requested by (instance=1, osid=7732), summary=[incident=12142].
Block recovery completed at rba 298.40706.16, scn 0.2058827
Undo initialization errored: err:600 serial:0 start:1632220572 end:1632223302 diff:2730 (27 seconds)
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_7732.trc:
ORA-00600: internal error code, arguments: [4193], [], [], [], [], [], [], [], [], [], [], []
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_7732.trc:
ORA-00600: internal error code, arguments: [4193], [], [], [], [], [], [], [], [], [], [], []
Error 600 happened during db open, shutting down database
USER (ospid: 7732): terminating the instance due to error 600
Instance terminated by USER, pid = 7732
ORA-1092 signalled during: alter database open...
opiodr aborting process unknown ospid (7732) as a result of ORA-1092
Sun May 13 22:31:59 2018
ORA-1092 : opitsk aborting process
```

### 解决[4193]错误
尝试设置free block pool，如果不知道，就假设free block pool无可用的undo block，方法如下：
* ktuxcnfb设置为0x00
* kubadba 设置为0x00000000
``` perl
BBED> set file 1 block 128
        FILE#           1
        BLOCK#          128

BBED> p ktuxc
struct ktuxc, 104 bytes                     @4148    
   struct ktuxcscn, 8 bytes                 @4148    
      ub4 kscnbas                           @4148     0x0012926f
      ub2 kscnwrp                           @4152     0x0000
   struct ktuxcuba, 8 bytes                 @4156    
      ub4 kubadba                           @4156     0x00400212
      ub2 kubaseq                           @4160     0x001a
      ub1 kubarec                           @4162     0x09
   sb2 ktuxcflg                             @4164     1 (KTUXCFSK)
   ub2 ktuxcseq                             @4166     0x001a
   sb2 ktuxcnfb                             @4168     1          # 修改这项
   ub4 ktuxcinc                             @4172     0x00000000
   sb2 ktuxcchd                             @4176     7
   sb2 ktuxcctl                             @4178     2
   ub2 ktuxcmgc                             @4180     0x8002
   ub4 ktuxcopt                             @4188     0x7ffffffe
   struct ktuxcfbp[0], 12 bytes             @4192    
      struct ktufbuba, 8 bytes              @4192    
         ub4 kubadba                        @4192     0x00400212  # 修改这项
         ub2 kubaseq                        @4196     0x001a
         ub1 kubarec                        @4198     0x12
      sb2 ktufbext                          @4200     2
      sb2 ktufbspc                          @4202     4246
   struct ktuxcfbp[1], 12 bytes             @4204    
      struct ktufbuba, 8 bytes              @4204    
         ub4 kubadba                        @4204     0x00000000
         ub2 kubaseq                        @4208     0x0019
         ub1 kubarec                        @4210     0x21
      sb2 ktufbext                          @4212     1
      sb2 ktufbspc                          @4214     4548
   struct ktuxcfbp[2], 12 bytes             @4216    
      struct ktufbuba, 8 bytes              @4216    
         ub4 kubadba                        @4216     0x00000000
         ub2 kubaseq                        @4220     0x0018
         ub1 kubarec                        @4222     0x02
      sb2 ktufbext                          @4224     0
      sb2 ktufbspc                          @4226     7778
   struct ktuxcfbp[3], 12 bytes             @4228    
      struct ktufbuba, 8 bytes              @4228    
         ub4 kubadba                        @4228     0x00000000
         ub2 kubaseq                        @4232     0x0000
         ub1 kubarec                        @4234     0x00
      sb2 ktufbext                          @4236     0
      sb2 ktufbspc                          @4238     0
   struct ktuxcfbp[4], 12 bytes             @4240    
      struct ktufbuba, 8 bytes              @4240    
         ub4 kubadba                        @4240     0x00000000
         ub2 kubaseq                        @4244     0x0000
         ub1 kubarec                        @4246     0x00
      sb2 ktufbext                          @4248     0
      sb2 ktufbspc                          @4250     0

BBED> modify /x 00 offset 4168
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 128              Offsets: 4168 to 4679           Dba:0x00400080
------------------------------------------------------------------------
 00000000 00000000 07000200 02800100 68000000 feffff7f 12024000 1a001200 
 02009610 00000000 19002100 0100c411 00000000 18000200 0000621e 00000000 
 00000000 00000000 00000000 00000000 00000000 13000000 8e004000 75921200 
 00000000 09004a00 00000000 00000000 00000000 01000000 00000000 13000000 
 8e004000 79921200 00000000 09000a00 00000000 00000000 00000000 01000000 
 00000000 14000000 12024000 416a1f00 00000000 0900ffff 00000000 00000000 
 00000000 01000000 00000000 14000000 12024000 b42d1500 00000000 09000200 
 00000000 00000000 00000000 01000000 00000000 14000000 12024000 ae2d1500 
 00000000 09005100 00000000 00000000 00000000 01000000 00000000 14000000 
 11024000 a82d1500 00000000 09005e00 00000000 00000000 00000000 01000000 
 00000000 13000000 8f004000 95921200 00000000 09001700 00000000 00000000 
 00000000 01000000 00000000 13000000 8d004000 71921200 00000000 09006000 
 00000000 00000000 00000000 01000000 00000000 13000000 8f004000 89921200 
 00000000 09000e00 00000000 00000000 00000000 01000000 00000000 14000000 
 12024000 b22d1500 00000000 09000300 00000000 00000000 00000000 01000000 
 00000000 13000000 8e004000 7b921200 00000000 09001300 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 00000000 4192
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 128              Offsets: 4192 to 4703           Dba:0x00400080
------------------------------------------------------------------------
 00000000 1a001200 02009610 00000000 19002100 0100c411 00000000 18000200 
 0000621e 00000000 00000000 00000000 00000000 00000000 00000000 13000000 
 8e004000 75921200 00000000 09004a00 00000000 00000000 00000000 01000000 
 00000000 13000000 8e004000 79921200 00000000 09000a00 00000000 00000000 
 00000000 01000000 00000000 14000000 12024000 416a1f00 00000000 0900ffff 
 00000000 00000000 00000000 01000000 00000000 14000000 12024000 b42d1500 
 00000000 09000200 00000000 00000000 00000000 01000000 00000000 14000000 
 12024000 ae2d1500 00000000 09005100 00000000 00000000 00000000 01000000 
 00000000 14000000 11024000 a82d1500 00000000 09005e00 00000000 00000000 
 00000000 01000000 00000000 13000000 8f004000 95921200 00000000 09001700 
 00000000 00000000 00000000 01000000 00000000 13000000 8d004000 71921200 
 00000000 09006000 00000000 00000000 00000000 01000000 00000000 13000000 
 8f004000 89921200 00000000 09000e00 00000000 00000000 00000000 01000000 
 00000000 14000000 12024000 b22d1500 00000000 09000300 00000000 00000000 
 00000000 01000000 00000000 13000000 8e004000 7b921200 00000000 09001300 
 00000000 00000000 00000000 01000000 00000000 13000000 8e004000 7f921200 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 128:
current = 0x3c13, required = 0x3c13

SQL> startup mount
ORACLE instance started.

Total System Global Area 4392697856 bytes
Fixed Size                  2260368 bytes
Variable Size             855638640 bytes
Database Buffers         3523215360 bytes
Redo Buffers               11583488 bytes
Database mounted.
SQL> alter database open;

Database altered.
# 数据库正常打开，日志也没有报错
```

## 恢复ora-600[4194]的错误
### ora-600[4194]解析
ORA-600[4194]内部错误一般由重做记录与回滚记录不匹配引发。

Oracle在验证Undo record number时，会对比redo change和回滚段中的undo record number，若发现2者存在差异则报该4194错误。
其错误argument[a][b]，a代表回滚块中的最大undo record number，b代表重做日志中记录的undo record number。这个错误可能由回滚段或者redo log日志文件讹误引起。
ORA-00600[4194]错误的根本原因是 redo记录与回滚段(rollback/undo)记录之间的不一致。当ORACLE在验证undo记录时相对应的变化需要应用到undo数据块的最大undo记录上，此时若检验出错则会报ORA-00600[4194]

此错误不像ORA-600[2662]或ORA-600[4000]错误那样必然导致数据库无法打开，因为它很少出现在前滚阶段；当数据库被打开，smon开始执行事务恢复或一些回滚段的管理工作时则很有可能触发该错误。

ORA-600[4194]的2个的含义：
Arg [a] Maximum Undo record number in Undo block
Arg [b] Undo record number from Redo block

这个ORA-600[4194] 报错属于ORACLE内核从cache层到事务undo处理，可能的影响是进程失败或者可能的回滚段坏块。

### 重现ORA-600[4194]错误
``` perl
# 关库
shutdown immediate

# BBED修改UODO回滚段段头
BBED> set file 1 block 128
        FILE#           1
        BLOCK#          128

BBED> p ktuxc
struct ktuxc, 104 bytes                     @4148    
   struct ktuxcscn, 8 bytes                 @4148    
      ub4 kscnbas                           @4148     0x00152d1b
      ub2 kscnwrp                           @4152     0x0000
   struct ktuxcuba, 8 bytes                 @4156    
      ub4 kubadba                           @4156     0x00400215
      ub2 kubaseq                           @4160     0x001a
      ub1 kubarec                           @4162     0x13
   sb2 ktuxcflg                             @4164     1 (KTUXCFSK)
   ub2 ktuxcseq                             @4166     0x001a
   sb2 ktuxcnfb                             @4168     1
   ub4 ktuxcinc                             @4172     0x00000000
   sb2 ktuxcchd                             @4176     76
   sb2 ktuxcctl                             @4178     72
   ub2 ktuxcmgc                             @4180     0x8002
   ub4 ktuxcopt                             @4188     0x7ffffffe
   struct ktuxcfbp[0], 12 bytes             @4192    
      struct ktufbuba, 8 bytes              @4192    
         ub4 kubadba                        @4192     0x00400215
         ub2 kubaseq                        @4196     0x001a
         ub1 kubarec                        @4198     0x1c    # 第一个UNDO BLOCK, 修改UNDO RECORD
      sb2 ktufbext                          @4200     2
      sb2 ktufbspc                          @4202     1202        
      sb2 ktufbext                          @4200     2
      sb2 ktufbspc                          @4202     1202
   struct ktuxcfbp[1], 12 bytes             @4204    
      struct ktufbuba, 8 bytes              @4204    
         ub4 kubadba                        @4204     0x00000000  # 全0表示没有使用
         ub2 kubaseq                        @4208     0x0019
         ub1 kubarec                        @4210     0x21
      sb2 ktufbext                          @4212     1
      sb2 ktufbspc                          @4214     4548
......

BBED> modify /x 2e offset 4198
Warning: contents of previous BIFILE will be lost. Proceed? (Y/N) y
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 128              Offsets: 4198 to 4709           Dba:0x00400080
------------------------------------------------------------------------
 2e000200 b2040000 00001900 21000100 c4110000 00001800 02000000 621e0000 
 00000000 00000000 00000000 00000000 00000000 00001400 00001302 400075b8 
 1f000000 00000900 4a000000 00000000 00000000 00000100 00000000 00001400 
 00001302 400078b8 1f000000 00000900 0a000000 00000000 00000000 00000100 
 00000000 00001400 00001202 4000416a 1f000000 00000900 07000000 00000000 
 00000000 00000100 00000000 00001400 00001202 4000b42d 15000000 00000900 
 02000000 00000000 00000000 00000100 00000000 00001400 00001202 4000ae2d 
 15000000 00000900 51000000 00000000 00000000 00000100 00000000 00001400 
 00001102 4000a82d 15000000 00000900 5e000000 00000000 00000000 00000100 
 00000000 00001400 00001302 400090b8 1f000000 00000900 17000000 00000000 
 00000000 00000100 00000000 00001400 00001302 400071b8 1f000000 00000900 
 60000000 00000000 00000000 00000100 00000000 00001400 00001302 400084b8 
 1f000000 00000900 0e000000 00000000 00000000 00000100 00000000 00001400 
 00001202 4000b22d 15000000 00000900 03000000 00000000 00000000 00000100 
 00000000 00001400 00001302 400079b8 1f000000 00000900 13000000 00000000 
 00000000 00000100 00000000 00001400 00001302 40007bb8 1f000000 00000900 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 128:
current = 0x70ca, required = 0x70ca

# 启动数据库
SQL> startup
ORACLE instance started.

Total System Global Area 4392697856 bytes
Fixed Size                  2260368 bytes
Variable Size             855638640 bytes
Database Buffers         3523215360 bytes
Redo Buffers               11583488 bytes
Database mounted.
ORA-01092: ORACLE instance terminated. Disconnection forced
ORA-00600: internal error code, arguments: [4194], [], [], [], [], [], [], [], [], [], [], []
Process ID: 29129
Session ID: 63 Serial number: 5

# alert日志中的错误信息
SMON: enabling cache recovery
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_29344.trc  (incident=20542):
ORA-00600: internal error code, arguments: [4194], [], [], [], [], [], [], [], [], [], [], []
Incident details in: /u01/app/oracle/diag/rdbms/orcl/orcl/incident/incdir_20542/orcl_ora_29344_i20542.trc
Use ADRCI or Support Workbench to package the incident.
See Note 411.1 at My Oracle Support for error and packaging details.
Block recovery from logseq 314, block 3 to scn 2351103
Recovery of Online Redo Log: Thread 1 Group 2 Seq 314 Reading mem 0
  Mem# 0: /u01/app/oracle/oradata/orcl/redo02.log
Block recovery stopped at EOT rba 314.5.16
Block recovery completed at rba 314.5.16, scn 0.2351101
Block recovery from logseq 314, block 3 to scn 2351100
Recovery of Online Redo Log: Thread 1 Group 2 Seq 314 Reading mem 0
  Mem# 0: /u01/app/oracle/oradata/orcl/redo02.log
Block recovery completed at rba 314.5.16, scn 0.2351101
Undo initialization errored: err:600 serial:0 start:2050437032 end:2050438832 diff:1800 (18 seconds)
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_29344.trc:
ORA-00600: internal error code, arguments: [4194], [], [], [], [], [], [], [], [], [], [], []
Errors in file /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_29344.trc:
ORA-00600: internal error code, arguments: [4194], [], [], [], [], [], [], [], [], [], [], []
Error 600 happened during db open, shutting down database
USER (ospid: 29344): terminating the instance due to error 600
Instance terminated by USER, pid = 29344
ORA-1092 signalled during: ALTER DATABASE OPEN...
opiodr aborting process unknown ospid (29344) as a result of ORA-1092
Fri May 18 18:42:14 2018
ORA-1092 : opitsk aborting process
```

### 解决ORA-600[4193]错误
方法和解决ORA-600[4193]一样，尝试设置free block pool，如果不知道，就假设free block pool无可用的undo block，方法如下：
* ktuxcnfb设置为0x00
* kubadba 设置为0x00000000
``` perl
BBED> set file 1 block 128
        FILE#           1
        BLOCK#          128

BBED> p ktuxc
struct ktuxc, 104 bytes                     @4148    
   struct ktuxcscn, 8 bytes                 @4148    
      ub4 kscnbas                           @4148     0x00152d1b
      ub2 kscnwrp                           @4152     0x0000
   struct ktuxcuba, 8 bytes                 @4156    
      ub4 kubadba                           @4156     0x00400215
      ub2 kubaseq                           @4160     0x001a
      ub1 kubarec                           @4162     0x13
   sb2 ktuxcflg                             @4164     1 (KTUXCFSK)
   ub2 ktuxcseq                             @4166     0x001a
   sb2 ktuxcnfb                             @4168     1            # 修改这项
   ub4 ktuxcinc                             @4172     0x00000000
   sb2 ktuxcchd                             @4176     76
   sb2 ktuxcctl                             @4178     72
   ub2 ktuxcmgc                             @4180     0x8002
   ub4 ktuxcopt                             @4188     0x7ffffffe
   struct ktuxcfbp[0], 12 bytes             @4192    
      struct ktufbuba, 8 bytes              @4192    
         ub4 kubadba                        @4192     0x00400215    # 修改这项
         ub2 kubaseq                        @4196     0x001a
         ub1 kubarec                        @4198     0x2e
      sb2 ktufbext                          @4200     2
      sb2 ktufbspc                          @4202     1202
   struct ktuxcfbp[1], 12 bytes             @4204    
      struct ktufbuba, 8 bytes              @4204    
         ub4 kubadba                        @4204     0x00000000
         ub2 kubaseq                        @4208     0x0019
         ub1 kubarec                        @4210     0x21
      sb2 ktufbext                          @4212     1
      sb2 ktufbspc                          @4214     4548
   struct ktuxcfbp[2], 12 bytes             @4216    
      struct ktufbuba, 8 bytes              @4216    
         ub4 kubadba                        @4216     0x00000000
         ub2 kubaseq                        @4220     0x0018
         ub1 kubarec                        @4222     0x02
      sb2 ktufbext                          @4224     0
      sb2 ktufbspc                          @4226     7778
   struct ktuxcfbp[3], 12 bytes             @4228    
      struct ktufbuba, 8 bytes              @4228    
         ub4 kubadba                        @4228     0x00000000
         ub2 kubaseq                        @4232     0x0000
         ub1 kubarec                        @4234     0x00
      sb2 ktufbext                          @4236     0
      sb2 ktufbspc                          @4238     0
   struct ktuxcfbp[4], 12 bytes             @4240    
      struct ktufbuba, 8 bytes              @4240    
         ub4 kubadba                        @4240     0x00000000
         ub2 kubaseq                        @4244     0x0000
         ub1 kubarec                        @4246     0x00
      sb2 ktufbext                          @4248     0
      sb2 ktufbspc                          @4250     0

BBED> modify /x 00 offset 4168
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 128              Offsets: 4168 to 4679           Dba:0x00400080
------------------------------------------------------------------------
 00000000 00000000 4c004800 02800100 68000000 feffff7f 15024000 1a002e00 
 0200b204 00000000 19002100 0100c411 00000000 18000200 0000621e 00000000 
 00000000 00000000 00000000 00000000 00000000 14000000 13024000 75b81f00 
 00000000 09004a00 00000000 00000000 00000000 01000000 00000000 14000000 
 13024000 78b81f00 00000000 09000a00 00000000 00000000 00000000 01000000 
 00000000 14000000 12024000 416a1f00 00000000 09000700 00000000 00000000 
 00000000 01000000 00000000 14000000 12024000 b42d1500 00000000 09000200 
 00000000 00000000 00000000 01000000 00000000 14000000 12024000 ae2d1500 
 00000000 09005100 00000000 00000000 00000000 01000000 00000000 14000000 
 11024000 a82d1500 00000000 09005e00 00000000 00000000 00000000 01000000 
 00000000 14000000 13024000 90b81f00 00000000 09001700 00000000 00000000 
 00000000 01000000 00000000 14000000 13024000 71b81f00 00000000 09006000 
 00000000 00000000 00000000 01000000 00000000 14000000 13024000 84b81f00 
 00000000 09000e00 00000000 00000000 00000000 01000000 00000000 14000000 
 12024000 b22d1500 00000000 09000300 00000000 00000000 00000000 01000000 
 00000000 14000000 13024000 79b81f00 00000000 09001300 00000000 00000000 

 <32 bytes per line>

BBED> modify /x 00000000 4192
 File: /u01/app/oracle/oradata/orcl/system01.dbf (1)
 Block: 128              Offsets: 4192 to 4703           Dba:0x00400080
------------------------------------------------------------------------
 00000000 1a002e00 0200b204 00000000 19002100 0100c411 00000000 18000200 
 0000621e 00000000 00000000 00000000 00000000 00000000 00000000 14000000 
 13024000 75b81f00 00000000 09004a00 00000000 00000000 00000000 01000000 
 00000000 14000000 13024000 78b81f00 00000000 09000a00 00000000 00000000 
 00000000 01000000 00000000 14000000 12024000 416a1f00 00000000 09000700 
 00000000 00000000 00000000 01000000 00000000 14000000 12024000 b42d1500 
 00000000 09000200 00000000 00000000 00000000 01000000 00000000 14000000 
 12024000 ae2d1500 00000000 09005100 00000000 00000000 00000000 01000000 
 00000000 14000000 11024000 a82d1500 00000000 09005e00 00000000 00000000 
 00000000 01000000 00000000 14000000 13024000 90b81f00 00000000 09001700 
 00000000 00000000 00000000 01000000 00000000 14000000 13024000 71b81f00 
 00000000 09006000 00000000 00000000 00000000 01000000 00000000 14000000 
 13024000 84b81f00 00000000 09000e00 00000000 00000000 00000000 01000000 
 00000000 14000000 12024000 b22d1500 00000000 09000300 00000000 00000000 
 00000000 01000000 00000000 14000000 13024000 79b81f00 00000000 09001300 
 00000000 00000000 00000000 01000000 00000000 14000000 13024000 7bb81f00 

 <32 bytes per line>

BBED> sum apply
Check value for File 1, Block 128:
current = 0x729e, required = 0x729e

# 重启数据库
SQL> startup
ORACLE instance started.

Total System Global Area 4392697856 bytes
Fixed Size                  2260368 bytes
Variable Size             855638640 bytes
Database Buffers         3523215360 bytes
Redo Buffers               11583488 bytes
Database mounted.
Database opened.
# 数据库正常打开，日志也没有报错
```
BBED kubarec直接改回原值，不一定能正常启动数据库

## ORA-600[4193]/[4194]错误的解决思路
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_11.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_12.png)
![](http://p2c0rtsgc.bkt.clouddn.com/0518_oracle_dsi_13.png)
