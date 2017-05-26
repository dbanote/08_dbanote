---
title: Oracle 12c特性解读-容器数据库和灾备-03 创建与管理CDB和PDB
date: 2017-05-24
tags:
- oracle
- 12c
---

## 在多PDB环境中，创建common user，并赋予权限
查看状态
```sql
show pdbs;
    CON_ID CON_NAME                       OPEN MODE  RESTRICTED
---------- ------------------------------ ---------- ----------
         2 PDB$SEED                       READ ONLY  NO
         3 PDB1                           READ WRITE NO
         4 PDB2                           READ WRITE NO

show con_name
CON_NAME
------------------------------
CDB$ROOT
```

<!-- more -->
创建common user
```sql
-- 创建时报错
create user c##cdbadmin identified by oracle default tablespace users temporary tablespace temp;
create user c##cdbadmin identified by oracle default tablespace users temporary tablespace temp
*
ERROR at line 1:
ORA-65048: error encountered when processing the current DDL statement in pluggable database PDB1
ORA-00959: tablespace 'USERS' does not exist

-- 上面报错的原因是指定了CDB用户默认的表空间时，要求所有的PDB用户存在该表空间
-- 给pdb1和pdb2添加users表空间
alter session set container=pdb1;

col FILE_NAME for a50
select file_name from dba_data_files;
FILE_NAME
--------------------------------------------------
/u01/app/oracle/oradata/orcl2/pdb1/system01.dbf
/u01/app/oracle/oradata/orcl2/pdb1/sysaux01.dbf
/u01/app/oracle/oradata/orcl2/pdb1/undotbs01.dbf

create tablespace users datafile '/u01/app/oracle/oradata/orcl2/pdb1/users01.dbf' size 1G;

alter session set container=pdb2;
select file_name from dba_data_files;
FILE_NAME
--------------------------------------------------
/u01/app/oracle/oradata/orcl2/pdb2/system01.dbf
/u01/app/oracle/oradata/orcl2/pdb2/sysaux01.dbf
/u01/app/oracle/oradata/orcl2/pdb2/undotbs01.dbf

create tablespace users datafile '/u01/app/oracle/oradata/orcl2/pdb2/users01.dbf' size 1G;

--重新执行创建成功
conn / as sysdba
create user c##cdbadmin identified by oracle default tablespace users temporary tablespace temp;

-- 授权
grant connect,resource to c##cdbadmin;
grant connect,resource to c##cdbadmin container=all;
```

用新创建的common user测试登陆
``` perl
# 登陆CDB
sqlplus c##cdbadmin/oracle@orcl2

# 登陆PDB
sqlplus c##cdbadmin/oracle@pdb1
```

## 模拟测试12.2中的move tablespace online，简单总结
## 创建一个序列test_seq,把它的nextval值修改为2017



## 新知识总结
### 数据字典
* CDB_   All of the objects in the CDB across all PDBs
* DBA_   All of the objects in a container or PDB
* ALL_   Objects accessible by the current user
* USER_  Objects owned by the current user

数据字典表
desc user$
desc ts$
desc seg$

desc v$session
desc x$ksmsp
desc x$kccle


select status,count(*) from CDB_SCHEDULER_JOB_RUN_DETAILS where log_date>sysdate-1 group by status;

select count(*) from CDB_SCHEDULER_JOB_RUN_DETAILS where status='FAILED';