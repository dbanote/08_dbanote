---
title: Oracle 12c特性解读-容器数据库和灾备-05 PDB迁移和克隆
date: 2017-06-02
tags:
- oracle
- 12c
categories:
- Oracle 12c特性解读-容器数据库和灾备
---

## 本地克隆创建PDB
### 创建一个新的PDB from SEED
``` perl
alter session set container=PDB$SEED;
select file_name from dba_data_files;

FILE_NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/orcl1/pdbseed/system01.dbf
/u01/app/oracle/oradata/orcl1/pdbseed/sysaux01.dbf
/u01/app/oracle/oradata/orcl1/pdbseed/undotbs01.dbf

conn / as sysdba
create pluggable database pdb2
  admin user pdb_mgr identified by oracle
  file_name_convert=('/u01/app/oracle/oradata/orcl1/pdbseed','/u01/app/oracle/oradata/orcl1/pdb2');

alter pluggable database pdb2 open;
show pdbs

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO
         4 PDB2                           READ WRITE NO
```

<!-- more -->

### 克隆一个已有的PDB
``` perl
conn / as sysdba
create pluggable database pdb3 from pdb2
  file_name_convert=('/u01/app/oracle/oradata/orcl1/pdb2','/u01/app/oracle/oradata/orcl1/pdb3');

alter pluggable database pdb3 open;
show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO
         4 PDB2                           READ WRITE NO
         5 PDB3                           READ WRITE NO
```

## 远程克隆创建PDB
``` perl
源端CDB: orcl1    源端PDB: pdb2
目标CDB: orcl2    目标PDB: pdb2_new

# 在源端
alter session set container=pdb2;
create tablespace users datafile '/u01/app/oracle/oradata/orcl1/pdb2/users01.dbf' size 1g;
create user lyj identified by lyj default tablespace users;
grant connect,resource to lyj;
grant unlimited tablespace to lyj;
conn lyj/lyj@pdb2
create table test as select * from all_objects;

# 在目标
conn / as sysdba
create public database link orcl1_pdb2_lyj 
  connect to lyj identified by lyj
  using '(DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.245.231.205)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = pdb2)
    ))';

# 如tnsnames.ora中已有配置，也可以换成以下写法
create public database link orcl1_pdb2_lyj 
  connect to lyj identified by lyj
  using 'PDB2';

# 验证DBLINK创建是否成功
select count(*) from test@orcl1_pdb2_lyj;  

# 注意：file_name_convert中第一个是源端的目录地址
create pluggable database pdb2_new from pdb2@orcl1_pdb2_lyj
  file_name_convert=('/u01/app/oracle/oradata/orcl1/pdb2','/u01/app/oracle/oradata/orcl2/pdb2_new');

alter pluggable database pdb2_new open;
alter session set container=pdb2_new;
select count(*) from lyj.test;

  COUNT(*)
----------
     56325
```

## 远程克隆NON-CDB创建PDB
``` perl
# 创建一个NON-CDB
dbca -silent -createDatabase -templateName $ORACLE_HOME/assistants/dbca/templates/General_Purpose.dbc \
-gdbname noncdb -sid noncdb -characterSet ZHS16GBK -sysPassword oracle -systemPassword oracle

export ORACLE_SID=noncdb
sqlplus / as sysdba
select name,cdb,con_id from v$database;
NAME      CDB     CON_ID
--------- --- ----------
NONCDB    NO           0

# 在目标CDB中创建DB LINK
export ORACLE_SID=orcl2
sqlplus / as sysdba

create public database link noncdb_link connect to system identified by oracle 
  using '(DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.245.231.205)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = noncdb)
    ))';

# 验证DBLINK创建是否成功
select name,cdb,con_id from v$database@noncdb_link;
NAME      CDB     CON_ID
--------- --- ----------
NONCDB    NO           0

select file_name from dba_data_files@noncdb_link;
FILE_NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/noncdb/users01.dbf
/u01/app/oracle/oradata/noncdb/undotbs01.dbf
/u01/app/oracle/oradata/noncdb/system01.dbf
/u01/app/oracle/oradata/noncdb/sysaux01.dbf

# 克隆non-cdb
create pluggable database noncdb_new from noncdb@noncdb_link
  file_name_convert=('/u01/app/oracle/oradata/noncdb','/u01/app/oracle/oradata/orcl2/noncdb_new');

show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB2_NEW                       READ WRITE NO
         4 NONCDB_NEW                     MOUNTED

# 从NON-CDB克隆完后直接OPEN报错
alter pluggable database noncdb_new open;
Warning: PDB altered with errors.

# 执行noncdb_to_pdb脚本
alter pluggable database noncdb_new close;
alter session set container=noncdb_new;
@?/rdbms/admin/noncdb_to_pdb.sql

# 这个过程较慢，执行期间可以查看alert.log，其中可以看到类似的以下信息
tail -f $ORACLE_BASE/diag/rdbms/"$ORACLE_SID"/"$ORACLE_SID"/trace/alert_"$ORACLE_SID".log 
#-----------------------------------------------------------------------------------------------
2017-06-04T13:52:11.740697+08:00
NONCDB_NEW(4):Resize operation completed for file# 15, old size 517120K, new size 522240K
2017-06-04T13:52:15.526573+08:00
NONCDB_NEW(4):Resize operation completed for file# 15, old size 522240K, new size 527360K
2017-06-04T13:52:17.018421+08:00
NONCDB_NEW(4):Resize operation completed for file# 15, old size 527360K, new size 532480K
2017-06-04T13:52:20.209732+08:00
NONCDB_NEW(4):Resize operation completed for file# 15, old size 532480K, new size 537600K
2017-06-04T13:52:28.960499+08:00
NONCDB_NEW(4):Resize operation completed for file# 15, old size 537600K, new size 542720K
#-----------------------------------------------------------------------------------------------

SQL> alter database open;

Database altered.

SQL> show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         4 NONCDB_NEW                     READ WRITE NO

# 在克隆non-cdb创建PDB时，如果报以下错误，可能是NON-CDB数据库的SGA过大导致（超过目标端）
#-----------------------------------------------------------------------------------------------
ERROR at line 1:
ORA-65169: error encountered while attempting to copy file
ORA-12801: error signaled in parallel query server
#-----------------------------------------------------------------------------------------------
```

## 练习PDB的plugging和unpluged操作
```perl
# 练习一：datafile原位置插拔
# unplug PDB
show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO
         4 PDB2                           READ WRITE NO
         5 PDB3                           READ WRITE NO

alter pluggable database pdb3 close;
alter pluggable database pdb3 unplug into '/tmp/pdb3.xml';

# 删除PDB(不含文件)
drop pluggable database pdb3;

# plug PDB
create pluggable database pdb3 using '/tmp/pdb3.xml' nocopy;

# 练习二：datafile位置改变
export ORACLE_SID=orcl2
sqlplus / as sysdba
show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB2_NEW                       READ WRITE NO
         4 NONCDB_NEW                     READ WRITE NO

alter pluggable database PDB2_NEW close;
alter pluggable database PDB2_NEW unplug into '/tmp/pdb2_new.xml';

export ORACLE_SID=orcl1
sqlplus / as sysdba
create pluggable database pdb2_new using '/tmp/pdb2_new.xml' copy
  file_name_convert=('/u01/app/oracle/oradata/orcl2/pdb2_new','/u01/app/oracle/oradata/orcl1/pdb2_new');

alter pluggable database PDB3 open;
alter pluggable database PDB2_NEW open;

show pdbs;

    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO
         4 PDB2                           READ WRITE NO
         5 PDB3                           READ WRITE NO
         6 PDB2_NEW                       READ WRITE NO

# nocopy选项
CREATE PLUGGABLE DATABASE salespdb USING '/disk1/usr/salespdb.xml' NOCOPY TEMPFILE REUSE;

# clone,nocopy选项
CREATE PLUGGABLE DATABASE salespdb AS CLONE USING '/disk1/usr/salespdb.xml' NOCOPY TEMPFILE REUSE;
```


### 多租户环境中相关的数据字典
* [G]V$SYSSTAT
* [G]V$SYS_TIME_MODEL
* [G]V$SYSTEM_EVENT
* [G]V$SYSTEM_WAIT_CLASS
* [G]V$CON_SYSSTAT
* [G]V$CON_SYS_TIME_MODEL
* [G]V$CON_SYSTEM_EVENT
* [G]V$CON_SYSTEM_WAIT_CLASS


### sqlnet.ora中的兼容设置
``` perl
# 参数用来限制可以连接到数据库服务器上的最小客户端版本，
# 比如设置值为10，即10g，11g等以上客户端版本可以连接到数据库服务器上
SQLNET.ALLOWED_LOGON_VERSION_SERVER=10
SQLNET.ALLOWED_LOGON_VERSION_CLIENT=10

# sec_case_sensitive_logon的值不能是false
SQL> show parameter sensitive

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
sec_case_sensitive_logon             boolean     TRUE
```

### 多租户环境中相关的视图
``` perl
SELECT SYS_CONTEXT ('USERENV', 'CON_NAME') FROM DUAL; 
select CON_NAME_TO_ID('CDB$ROOT') from dual;

alter session set container=pd1;
alter system set max_iops=1000 container=current;
```

### 多租户环境中的PDB状态
``` perl
alter pluggable database tcymob0 save state;
alter pluggable database all save state;
alter pluggable database pdbhr1 discard state;
alter pluggable database ALL EXCEPT hrpdb2, hrpdb1 save state;
```

### 多租户环境下的批量数据变更
``` perl
UPDATE CONTAINERS(sales.customers) ctab 
SET ctab.city_name='MIAMI' 
WHERE ctab.CON_ID IN(7,8) AND CUSTOMER_ID=3425;
```

### 克隆远程PDB常见错误1
``` perl
ORA-17628: Oracle error 65035 returned by remote Oracle server
ORA-65035: unable to create pluggable database from

$ oerr ora 65035
65035, 00000, "unable to create pluggable database from %s"
// *Cause: An attempt was made to clone a pluggable database that did not have
// local undo enabled.
// *Action: Enable local undo for the PDB and and retry the operation.

# 解决办法：share undo 转local undo
select PROPERTY_NAME,PROPERTY_VALUE from database_properties where property_name='LOCAL_UNDO_ENABLED';

shutdown immediate;
startup upgrade;
show con_name
alter database local undo on;
shutdown immediate;
startup;
show pdbs
alter session set container=tcymob1;
select name from v$datafile where name like ‘%undo%‘;
```

### 克隆远程PDB常见错误2
``` perl
ERROR at line 1:
ORA-17627: ORA-01033: ORACLE initialization or shutdown in progress
ORA-17629: Cannot connect to the remote database server
```

### 克隆远程PDB常见错误3
``` perl
ERROR at line 1:
ORA-65005: missing or invalid file name pattern for file -
/U01/app/oracle/oradata/new12c/NEW12C/tcymob1/system01.dbf
```

### 克隆远程PDB常见错误4
``` perl
ORA-19504: failed to create file "/U01/app/oracle/oradata/test12cs/tcymob1"
ORA-27038: created file already exists
Additional information: 1

# 文件映射路径问题，文件夹—文件夹 文件—文件
CREATE PLUGGABLE DATABASE tcymob1_test12cs FROM tcymob1@tcymob1_new12c
file_name_convert=('/U01/app/oracle/oradata/new12c/NEW12C/tcymob0/NEW12C/4FB0C84834AE542DE0537D857F0AE8F5/datafile/o1_mf_undo_1_dloqnnn9_.dbf','/U01/app/oracle/oradata/test12cs/tcymob1/undotbs1.dbf','/U01/app/oracle/oradata/new12c/NEW12C/tcymob1','/U01/app/oracle/oradata/test12cs/tcymob1') ;
```

### Plugging in non-cdb
``` perl
SET SERVEROUTPUT ON
DECLARE compatible CONSTANT VARCHAR2(3) := CASE
DBMS_PDB.CHECK_PLUG_COMPATIBILITY( pdb_descr_file => '/tmp/ncdb12c_actvdb.xml', pdb_name => 'NCDB12C')
WHEN TRUE THEN 'YES' ELSE 'NO'
END;
BEGIN
DBMS_OUTPUT.PUT_LINE(compatible);
END;
/

select name,cause,type,message,status from PDB_PLUG_IN_VIOLATIONS where name='NCDB12C';
```