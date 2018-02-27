---
title: Oracle特殊恢复原理与实战_01 Oracle特殊恢复入门
date: 2018-02-25
tags:
- oracle
- DSI
categories:
- Oracle特殊恢复原理与实战
---

> **本课程就是基于Oracle DSI403e学习Oracle特殊恢复**

## DSI介绍
DSI是Data Server Internals的缩写，是Oracle公司内部用来培训Oracle售后工程师使用的教材
DSI课程系统包括：
* DSI303  Advanced Backup, Restore and recovery Techniques
* DSI401  Dumps Crashes and Corruptions
* DSI402  Space and Transaction Management
* DSI402e Data types and block structures
* DSI403e Recovery Architecture Components
* DSI404e Query Optimizer
* DSI405  Performance TUning
* DSI408  Real Application clusters Internals


## BBED工具介绍
* BBED stands for **<font color=red>B</font>**lock **<font color=red>B</font>**rower and **<font color=red>ED</font>**itor
* BBED只是一款工具，类似于ultraEdit，单纯的会用BBED来修改数据没有任何意义！关键是要知道为什么要这么改！
* 在充分了解Block格式和Oracle的各种机制的基础上广泛使用BBED，用它来帮你构造测试案例，用它来验证测试结果，**<font color=red>用它来帮你深入理解Oracle！</font>**

<!-- more -->

## BBED工具使用
### 安装bbed工具
上传sbbdpt.o、ssbbded.o、bbedus.msb(拷贝oracle的linux64版本)
``` perl
cp sbbdpt.o $ORACLE_HOME/rdbms/lib/ssbbded.o
cp ssbbded.o $ORACLE_HOME/rdbms/lib/sbbdpt.o
cp bbedus.msb $ORACLE_HOME/rdbms/mesg/bbedus.msb

cd $ORACLE_HOME/rdbms/lib
make -f $ORACLE_HOME/rdbms/lib/ins_rdbms.mk BBED=$ORACLE_HOME/bin/bbed $ORACLE_HOME/bin/bbed
#----------------------------------------------------------------------------------------------------------------------
inking BBED utility (bbed)
rm -f /u01/app/oracle/product/11.2.0/db_1/bin/bbed
gcc -o /u01/app/oracle/product/11.2.0/db_1/bin/bbed -m64 -z noexecstack -L/u01/app/oracle/product/11.2.0/db_1/rdbms/lib/
 -L/u01/app/oracle/product/11.2.0/db_1/lib/ -L/u01/app/oracle/product/11.2.0/db_1/lib/stubs/  
 /u01/app/oracle/product/11.2.0/db_1/lib/s0main.o /u01/app/oracle/product/11.2.0/db_1/rdbms/lib/ssbbded.o 
 /u01/app/oracle/product/11.2.0/db_1/rdbms/lib/sbbdpt.o `cat /u01/app/oracle/product/11.2.0/db_1/lib/ldflags`    
 -lncrypt11 -lnsgr11 -lnzjs11 -ln11 -lnl11 -ldbtools11 -lclntsh  `cat /u01/app/oracle/product/11.2.0/db_1/lib/ldflags`    
 -lncrypt11 -lnsgr11 -lnzjs11 -ln11 -lnl11 -lnro11 `cat /u01/app/oracle/product/11.2.0/db_1/lib/ldflags`    
 -lncrypt11 -lnsgr11 -lnzjs11 -ln11 -lnl11 -lnnz11 -lzt11 -lztkg11 -lclient11 -lnnetd11  -lvsn11 -lcommon11 
 -lgeneric11 -lmm -lsnls11 -lnls11  -lcore11 -lsnls11 -lnls11 -lcore11 -lsnls11 -lnls11 -lxml11 -lcore11 -lunls11 
 -lsnls11 -lnls11 -lcore11 -lnls11 `cat /u01/app/oracle/product/11.2.0/db_1/lib/ldflags`    -lncrypt11 -lnsgr11 
 -lnzjs11 -ln11 -lnl11 -lnro11 `cat /u01/app/oracle/product/11.2.0/db_1/lib/ldflags`    
 -lncrypt11 -lnsgr11 -lnzjs11 -ln11 -lnl11 -lclient11 -lnnetd11  -lvsn11 -lcommon11 -lgeneric11   
 -lsnls11 -lnls11  -lcore11 -lsnls11 -lnls11 -lcore11 -lsnls11 -lnls11 -lxml11 -lcore11 -lunls11 -lsnls11 
 -lnls11 -lcore11 -lnls11 -lclient11 -lnnetd11  -lvsn11 -lcommon11 -lgeneric11 -lsnls11 -lnls11  -lcore11 
 -lsnls11 -lnls11 -lcore11 -lsnls11 -lnls11 -lxml11 -lcore11 -lunls11 -lsnls11 -lnls11 -lcore11 -lnls11   
 `cat /u01/app/oracle/product/11.2.0/db_1/lib/sysliblist` -Wl,-rpath,/u01/app/oracle/product/11.2.0/db_1/lib -lm    
 `cat /u01/app/oracle/product/11.2.0/db_1/lib/sysliblist` -ldl -lm   -L/u01/app/oracle/product/11.2.0/db_1/lib `
#------------------------------------------------------------------------------------------------------------------------

ll $ORACLE_HOME/bin/bbed
-rwxr-xr-x 1 oracle oinstall 255110 Feb 25 12:04 /u01/app/oracle/product/11.2.0/db_1/bin/bbed
```

### 配置使用BBED
#### 直接登入
``` perl
$ bbed
Password: blockedit   

BBED: Release 2.0.0.0.0 - Limited Production on Sun Feb 25 12:14:21 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

************* !!! For Oracle Internal Use only !!! ***************

BBED> exit
```

#### 配置文件登入
``` perl
sqlplus / as sysdba
SQL> select file#||chr(9)||name||chr(9)||bytes from v$datafile;

FILE#||CHR(9)||NAME||CHR(9)||BYTES
----------------------------------------------------------------------------
1       /u01/app/oracle/oradata/dgt/system01.dbf        734003200
2       /u01/app/oracle/oradata/dgt/sysaux01.dbf        629145600
3       /u01/app/oracle/oradata/dgt/undotbs01.dbf       209715200
4       /u01/app/oracle/oradata/dgt/users01.dbf 5242880
5       /u01/app/oracle/oradata/dgt/dsi01.dbf   524288000
SQL> exit

cd ~
vi par.txt
#-------------------------------------------------------
blocksize=8192
listfile=filelist.txt
mode=edit
#-------------------------------------------------------

vi filelist.txt
#-------------------------------------------------------
1       /u01/app/oracle/oradata/dgt/system01.dbf        734003200
2       /u01/app/oracle/oradata/dgt/sysaux01.dbf        629145600
3       /u01/app/oracle/oradata/dgt/undotbs01.dbf       209715200
4       /u01/app/oracle/oradata/dgt/users01.dbf 5242880
5       /u01/app/oracle/oradata/dgt/dsi01.dbf   524288000
#-------------------------------------------------------

# 用别名的方式直接登录
echo "alias bbed='rlwrap bbed parfile=par.txt password=blockedit'" >> /home/oracle/.bash_profile

source /home/oracle/.bash_profile

bbed
#-------------------------------------------------------
BBED: Release 2.0.0.0.0 - Limited Production on Sun Feb 25 12:34:15 2018

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

************* !!! For Oracle Internal Use only !!! ***************

 
```

#### BBED常用命令
``` perl
常用命令：set、find、dump、modify、sum apply、examine、map 、print、verity，示例如下：

# 查看访问的数据文件列表
BBED> info
#---------------------------------------------------------------------------------
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /u01/app/oracle/oradata/dgt/system01.dbf                         89600
     2  /u01/app/oracle/oradata/dgt/sysaux01.dbf                         76800
     3  /u01/app/oracle/oradata/dgt/undotbs01.dbf                        25600
     4  /u01/app/oracle/oradata/dgt/users01.dbf                            640
     5  /u01/app/oracle/oradata/dgt/dsi01.dbf                            64000
#---------------------------------------------------------------------------------

set file 5 block 1 # 访问5号文件的1号块
set dba 0x01000020
set offset 0       # 0表示第一个字节开始
set block 1        # 1表示第一个块开始
set count 8192     # 默认是显示512字节

find /x 05d67g     # 查指定的字符串在指定数据块中的具体位置
                   # f --find的简写，表示继续从当前位置开始往下查询字符串05d67g
dump               # 十六进制查看block
dump /v            # 查看十六进制内容的同时以文本方式“翻译”十六进制显示的内容，相当于对当前block执行strings命令

modify /x d43      # 修改指定block,指定offset的数据块块内记录的内容
sum apply          # 计算修改后的数据块的checksum值，然后写入数据块的offset为16-17的位置

map /v             # 查看块结构
```

### BBED解析DB_NAME
DB_NAME数据库名，长度不能超出8个字符，记录在datafile,redolog和control file文件头中
``` perl
BBED> set file 5 block 1
        FILE#           5
        BLOCK#          1
```

kcvfh：Kernel Cache recoVer File Header
![](http://p2c0rtsgc.bkt.clouddn.com/0225_oracle_dsi_01.png)

从下图可以验证DB_NAME最长只能为8个字节，DB_NAME是DGT
![](http://p2c0rtsgc.bkt.clouddn.com/0225_oracle_dsi_02.png)

用dump方式查看DB_NAME
![](http://p2c0rtsgc.bkt.clouddn.com/0225_oracle_dsi_03.png)
``` perl
SQL> select chr(to_number(substr('4447540000000000',rownum*2-1,2),'xxxxxxxx')) from dba_objects where rownum<=9;

CH
--
D
G
T
```

## 诊断trace files
``` sql
alter session set events 'immediate trace name controlf level N';
alter session set events 'immediate trace name file_hdrs level N';
alter session set events 'immediate trace name redohdr level N';
alter session set events 'immediate trace name treedump level 64952';

alter system dump datafile 3 block 256;
alter system dump logfile '/u01/app/oracle/oradata/dgt/redo01.log';
alter system dump undo header "_SYSSMU17_3012809736$";
```

详见：[Oracle数据库event事件与dump文件介绍](http://www.cnblogs.com/rootq/archive/2008/11/12/1332239.html)

## Recovery算法
Oracle为了在任何情况下都能顺利执行recovery操作，采用了一系列的算法/设计来保证recovery的成功实施：
* Page Fix  块修复
* Write-Ahead-Log  日志优先写
* Log-force-at-commit  提交必须把日志写到日志文件
* Online-Log Switch Management  日志切换管理
* Checkpointing  检查点
* Thread-Open Flag  线程打开的标记
* Incremental Checkpointing  增量检查点
* Two-Pass Recovery  两次恢复

### Page Fix Rule
![](http://p2c0rtsgc.bkt.clouddn.com/0225_oracle_dsi_04.png)

### Write Ahead Logging
![](http://p2c0rtsgc.bkt.clouddn.com/0225_oracle_dsi_05.png)

### Online Log Switching
![](http://p2c0rtsgc.bkt.clouddn.com/0225_oracle_dsi_06.png)

### Checkpoint
![](http://p2c0rtsgc.bkt.clouddn.com/0225_oracle_dsi_07.png)
RBA(Redo Block Address)，由三部分组成，80-日志序列号，20-日志第几个block，16-在这个块上的第几个字节

## Recovery方法
* Recovery for a thread starts at last checkpoint
* Recovery progresses, redo record by redo record, until all available redo has been applied
 - Find the block
 - Block time > Redo record time: Skip block
 - Block time = Redo record time: Apply changes
 - Block time < Redo record time: Stuck recovery(ORA-600[3020])
* No generation of redo

### 通过BBED工具修改文件头信息，使数据库不能正常打开
``` perl
# 关闭数据库
shutdown immedaite

# 通过BBED工具修改文件头信息
bbed

BBED> info
 File#  Name                                                        Size(blks)
 -----  ----                                                        ----------
     1  /u01/app/oracle/oradata/dgt/system01.dbf                         89600
     2  /u01/app/oracle/oradata/dgt/sysaux01.dbf                         76800
     3  /u01/app/oracle/oradata/dgt/undotbs01.dbf                        25600
     4  /u01/app/oracle/oradata/dgt/users01.dbf                            640
     5  /u01/app/oracle/oradata/dgt/dsi01.dbf                            64000

BBED> set file 5 block 1
        FILE#           5
        BLOCK#          1

BBED> p kcvfhcpc
ub4 kcvfhcpc                                @140      0x00000003

BBED> p kcvfhccc
ub4 kcvfhccc                                @148      0x00000002

BBED> modify /x 1 offset 140
 File: /u01/app/oracle/oradata/dgt/dsi01.dbf (5)
 Block: 1                Offsets:  140 to  651           Dba:0x01400001
------------------------------------------------------------------------
 01000000 00000000 02000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 05000000 03004453 49000000 00000000 00000000 00000000 00000000 00000000 
 00000000 05000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 961b4300 00000000 
 7547c239 01000000 44000000 d2970400 10004964 02000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 0d000d00 0d000100 

 <32 bytes per line>

BBED> sum apply
Check value for File 5, Block 1:
current = 0x82d5, required = 0x82d5

BBED> p kcvfhcpc
ub4 kcvfhcpc                                @140      0x00000001

BBED> p kcvfhccc
ub4 kcvfhccc                                @148      0x00000002

# kcvfhcpc（检查点的计数器）的值要大于kcvfhccc（检查点的控制文件备份的计数器）

# 这时已经不能打开数据库了
SQL> startup
ORACLE instance started.

Total System Global Area 4409401344 bytes
Fixed Size                  2260408 bytes
Variable Size            1140851272 bytes
Database Buffers         3254779904 bytes
Redo Buffers               11509760 bytes
Database mounted.
ORA-01113: file 5 needs media recovery
ORA-01110: data file 5: '/u01/app/oracle/oradata/dgt/dsi01.dbf'

# alter日志中有以下错误提示
Errors in file /u01/app/oracle/diag/rdbms/dgtp/dgt/trace/dgt_ora_10717.trc:
ORA-01113: file 5 needs media recovery
ORA-01110: data file 5: '/u01/app/oracle/oradata/dgt/dsi01.dbf'
ORA-1113 signalled during: ALTER DATABASE OPEN...
```

## 参考文件：
* [利用BBED恢复数据文件头](https://www.2cto.com/database/201406/309329.html)

## 编译安装lrzsz
``` perl
tar xvpf lrzsz-0.12.20.tar.gz
cd lrzsz-0.12.20
./configure --prefix=/usr/local/lrzsz
make && make install
cd /usr/bin
ln -s /usr/local/lrzsz/bin/lrz rz
ln -s /usr/local/lrzsz/bin/lsz sz
```
