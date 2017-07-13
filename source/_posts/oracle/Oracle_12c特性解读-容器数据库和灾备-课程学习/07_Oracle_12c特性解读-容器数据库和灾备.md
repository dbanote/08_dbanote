---
title: Oracle 12c特性解读-容器数据库和灾备-07 升级数据库至12c
date: 2017-06-16
tags:
- oracle
- 12c
---

## 11g升级到12.2
### 查看当前系统环境
``` perl
df -hT
Filesystem           Type   Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                     ext4    43G   20G   22G  48% /
tmpfs                tmpfs  3.9G  109M  3.8G   3% /dev/shm
/dev/sda1            ext4   477M   85M  364M  19% /boot

free
             total       used       free     shared    buffers     cached
Mem:       8174784    5810232    2364552     111112     196636     731784
-/+ buffers/cache:    4881812    3292972
Swap:     16777212          0   16777212

su - grid
opatch lspatches
22502505;ACFS Patch Set Update : 11.2.0.4.160419 (22502505)
23054319;OCW Patch Set Update : 11.2.0.4.160719 (23054319)
24732075;Database Patch Set Update : 11.2.0.4.170418 (24732075)

exit
su - oracle
sqlplus / as sysdba
select PROPERTY_VALUE from database_properties where PROPERTY_NAME='NLS_RDBMS_VERSION';

PROPERTY_VALUE
--------------------------------------------------
11.2.0.4.0

select name,db_unique_name,open_mode,database_role from v$database;

NAME      DB_UNIQUE_NAME                 OPEN_MODE            DATABASE_ROLE
--------- ------------------------------ -------------------- ----------------
DGT       dgts                           READ WRITE           PRIMARY
```

<!-- more -->
### 升级GI
``` perl
# 关闭数据库 
shutdown immediate
cd /oradata/software
unzip V839960-01.zip

# root用户下，安装cvuqdisk
## cvuqdisk存于oracle安装介质的rpm目录下，解压缩database的安装介质即可看到此包
cd /oradata/software/database/rpm/
export CVUQDISK_GRP=asmadmin
rpm -ivh cvuqdisk-1.0.10-1.rpm

# 创建GI的新HOME目录
su - grid
mkdir -p /u01/app/12.2.0/grid
chown -R grid:oinstall /u01/app/12.2.0
chmod -R 775 /u01/app/12.2.0

cd /u01/app/12.2.0/grid
unzip /oradata/software/V840012-01.zip

sqlplus / as sysasm
# 如果不设置这个参数，11.2默认是到/dev/raw/*查找asm disk，但12C 默认是到/dev/sd*去查
alter system set asm_diskstring='/dev/raw/*' scope=both;

# 不修改compatible安装时会报错
col COMPATIBILITY form a10
col DATABASE_COMPATIBILITY form a10
col NAME form a20
select group_number, name,compatibility, database_compatibility from v$asm_diskgroup;
GROUP_NUMBER NAME                 COMPATIBIL DATABASE_C
------------ -------------------- ---------- ----------
           1 DATA                 11.2.0.0.0 10.1.0.0.0
           2 FRA                  11.2.0.0.0 10.1.0.0.0

asmcmd setattr -G DATA compatible.asm 11.2.0.4.0
asmcmd setattr -G FRA compatible.asm 11.2.0.4.0

select group_number, name,compatibility, database_compatibility from v$asm_diskgroup;

GROUP_NUMBER NAME                 COMPATIBIL DATABASE_C
------------ -------------------- ---------- ----------
           1 DATA                 11.2.0.4.0 10.1.0.0.0
           2 FRA                  11.2.0.4.0 10.1.0.0.0

# 开始图形化安装
export DISPLAY=10.240.4.150:0.0
unset ORACLE_HOME
unset ORACLE_BASE
unset ORACLE_SID
cd /u01/app/12.2.0/grid
./gridSetup.sh
```

![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_01.png)
确保数据库实例已经关闭后，点YES
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_02.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_03.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_04.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_05.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_06.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_07.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_08.png)
``` perl
# root用户下执行，交互确认都输入y，时间较长，耐心等待完成
/u01/app/12.2.0/grid/rootupgrade.sh
Overwrite it? (y/n) y # 
...
2017/06/21 17:45:31 CLSRSC-482: Running command: 'srvctl upgrade model -s 11.2.0.4.0 -d 12.2.0.1.0 -p first'
2017/06/21 17:45:46 CLSRSC-482: Running command: 'srvctl upgrade model -s 11.2.0.4.0 -d 12.2.0.1.0 -p last'

dgts     2017/06/21 17:45:48     /u01/app/12.2.0/grid/cdata/dgts/backup_20170621_174548.olr     0     

dgts     2017/06/08 18:10:20     /u01/app/11.2.0.4/grid/cdata/dgts/backup_20170608_181020.olr     -     
CRS-2791: Starting shutdown of Oracle High Availability Services-managed resources on 'dgts'
CRS-2673: Attempting to stop 'ora.evmd' on 'dgts'
CRS-2677: Stop of 'ora.evmd' on 'dgts' succeeded
CRS-2793: Shutdown of Oracle High Availability Services-managed resources on 'dgts' has completed
CRS-4133: Oracle High Availability Services has been stopped.
CRS-4123: Oracle High Availability Services has been started.

2017/06/21 18:03:27 CLSRSC-327: Successfully configured Oracle Restart for a standalone server
```

![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_09.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_10.png)

注：若之前不修改compatible安装时检查会报以下错
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_11.png)

``` perl
# root 用户下，完成后修改Grid帐号下的ORACLE_HOME值
vi /home/grid/.bash_profile
export ORACLE_HOME=/u01/app/12.2.0/grid

# 修改root帐号下GRID_HOME值
vi /root/.bash_profile
export GRID_HOME=/u01/app/12.2.0/grid
export PATH=$PATH:$GRID_HOME/bin:$GRID_HOME/OPatch

su - grid
opatch lsinventory

# 可以使用asmca工具查看asm状态
export DISPLAY=10.240.4.150:0.0
asmca
```

![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_12.png)
``` perl
# 查看GI状态
crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.DATA.dg
               ONLINE  ONLINE       dgts                     STABLE
ora.FRA.dg
               ONLINE  ONLINE       dgts                     STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       dgts                     STABLE
ora.asm
               ONLINE  ONLINE       dgts                     Started,STABLE
ora.ons
               OFFLINE OFFLINE      dgts                     STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        ONLINE  ONLINE       dgts                     STABLE
ora.diskmon
      1        OFFLINE OFFLINE                               STABLE
ora.evmd
      1        ONLINE  ONLINE       dgts                     STABLE
--------------------------------------------------------------------------------
```

### 安装12c数据库软件
``` perl
# oracle用户下，创建oracle软件安装的HOME目录
mkdir -p /u01/app/oracle/product/12.2.0/db_1
chown -R oracle:oinstall /u01/app/oracle/product/12.2.0
chmod -R 775 /u01/app/oracle/product/12.2.0

cd /oradata/software/database
export DISPLAY=10.240.4.150:0.0
unset ORACLE_BASE
unset ORACLE_HOME
unset ORACLE_SID
./runInstaller

# 注意修改软件安装目录为/u01/app/oracle/product/12.2.0/db_1
# 只安装database software，单实例数据库，这步没什么特别的，就不上图了
```

### 升级数据库
``` perl
# 执行预升级脚本检查
java -jar $ORACLE_BASE/product/12.2.0/db_1/rdbms/admin/preupgrade.jar
Preupgrade generated files:
    /u01/app/oracle/cfgtoollogs/dgts/preupgrade/preupgrade.log
    /u01/app/oracle/cfgtoollogs/dgts/preupgrade/preupgrade_fixups.sql
    /u01/app/oracle/cfgtoollogs/dgts/preupgrade/postupgrade_fixups.sql

SQL> @/u01/app/oracle/cfgtoollogs/dgts/preupgrade/preupgrade_fixups.sql
#--------------------------------------------------------------------------------------------
Executing Oracle PRE-Upgrade Fixup Script

Auto-Generated by:       Oracle Preupgrade Script
                         Version: 12.2.0.1.0 Build: 1
Generated on:            2017-06-17 00:54:45

For Source Database:     DGT
Source Database Version: 11.2.0.4.0
For Upgrade to Version:  12.2.0.1.0

                          Fixup
Check Name                Status  Further DBA Action
----------                ------  ------------------
dictionary_stats          Passed  None

PL/SQL procedure successfully completed.
#--------------------------------------------------------------------------------------------

# 使用dbua图形化工具升级DB
cd /u01/app/oracle/product/12.2.0/db_1/bin
export DISPLAY=10.240.4.150:0.0
./dbua
```

![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_14.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_15.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_16.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_17.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_18.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_19.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_20.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_21.png)
![](http://oligvdnzp.bkt.clouddn.com/0616_12c_upgrade_22.png)

``` perl
# 没有执行preupgrade_fixups.sql，按提示手动执行以下命令
EXECUTE DBMS_STATS.GATHER_DICTIONARY_STATS;
EXECUTE DBMS_STATS.GATHER_FIXED_OBJECTS_STATS;

# 停止原11g数据库
shutdown immediate

vi /home/oracle/.bash_profile
export ORACLE_HOME=$ORACLE_BASE/product/12.2.0/db_1

sqlplus / as sysdba
#--------------------------------------------------------------------------------------------
SQL*Plus: Release 12.2.0.1.0 Production on Sat Jun 17 01:50:54 2017

Copyright (c) 1982, 2016, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.2.0.1.0 - 64bit Production
SQL> 
#--------------------------------------------------------------------------------------------

col COMP_ID for a10
col COMP_NAME for a40
select substr(comp_id,1,15) comp_id,substr(comp_name,1,30)
      comp_name,substr(version,1,10) version,status
from dba_registry order by modified;

COMP_ID    COMP_NAME                                VERSION              STATUS
---------- ---------------------------------------- -------------------- ----------------------
CATALOG    Oracle Database Catalog Views            12.2.0.1.0           VALID
CATPROC    Oracle Database Packages and T           12.2.0.1.0           INVALID
OWM        Oracle Workspace Manager                 12.2.0.1.0           VALID
XDB        Oracle XML Database                      12.2.0.1.0           VALID


# 检查orattab
cat /etc/oratab
+ASM:/u01/app/12.2.0/grid:N             # line added by Agent
dgts:/u01/app/oracle/product/12.2.0/db_1:N
```





