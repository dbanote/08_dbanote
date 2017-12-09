---
title: Oracle 12c特性解读-容器数据库和灾备-04 CDB PDB对象管理
date: 2017-05-24
tags:
- oracle
- 12c
categories:
- Oracle 12c特性解读-容器数据库和灾备
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
``` perl
# 连接到PDB1中，创建测试数据
conn lyj/lyj@pdb1
create table test_mv tablespace users as select * from all_objects;
insert into test_mv select * from test_mv;  #多执行几次
commit;
set timing on
select count(*) from test_mv;

  COUNT(*)
----------
   2180000

Elapsed: 00:00:00.16

create index ind_test_mv_objid on test_mv(object_id) tablespace users;

# 查看segment所占空间
select SEGMENT_NAME,TABLESPACE_NAME,BYTES/1024/1024 SIZE_M from user_segments;

SEGMENT_NAME                   TABLESPACE_NAME                    SIZE_M
------------------------------ ------------------------------ ----------
IND_TEST_MV_OBJID              USERS                                  39
TEST_MV                        USERS                                 336

select OBJECT_NAME,STATUS from user_objects;

OBJECT_NAME                                        STATUS
-------------------------------------------------- -------
IND_TEST_MV_OBJID                                  VALID
TEST_MV                                            VALID

# 删除数据后查看segment所占空间，没有任何变动，查询的时间也没有变短
delete from test_mv where object_id<>2;
2179968 rows deleted
Elapsed: 00:00:37.29

commit;
select SEGMENT_NAME,TABLESPACE_NAME,BYTES/1024/1024 SIZE_M from user_segments;

SEGMENT_NAME                   TABLESPACE_NAME                    SIZE_M
------------------------------ ------------------------------ ----------
IND_TEST_MV_OBJID              USERS                                  39
TEST_MV                        USERS                                 336

select count(*) from test_mv;

  COUNT(*)
----------
        32

Elapsed: 00:00:00.26

# 执行move table online后，segment所占空间空小，表查询速度变快
alter table test_mv move tablespace users online;
# 或
alter table test_mv move online;

select SEGMENT_NAME,TABLESPACE_NAME,BYTES/1024/1024 SIZE_M from user_segments;

SEGMENT_NAME                   TABLESPACE_NAME                    SIZE_M
------------------------------ ------------------------------ ----------
IND_TEST_MV_OBJID              USERS                               .0625
TEST_MV                        USERS                               .0625

select count(*) from test_mv;

  COUNT(*)
----------
        32

Elapsed: 00:00:00.00
```

总结：12.2执行在线表移动后，不用像在11g时还需要rebuild index

## 创建一个序列test_seq,把它的nextval值修改为2017
``` perl
# 创建序列 起始编号是1，增量为1，最大值10000000
create sequence test_seq start with 1 increment by 1 maxvalue 10000000;
select test_seq.nextval from dual;

   NEXTVAL
----------
         1

select 2017 - test_seq.nextval - 1 seq_id from dual;
    SEQ_ID
----------
      2014

alter sequence test_seq increment by 2014;

SELECT test_seq.nextval FROM dual;

   NEXTVAL
----------
      2016

SELECT test_seq.currval FROM dual;

   CURRVAL
----------
      2016

# 以下SQL脚本可以实现自动重置序列
vi reset_sequence.sql
#--------------------------------------------------------------------------------
UNDEFINE seq_name
UNDEFINE reset_to
PROMPT "sequence name" ACCEPT '&&seq_name'
PROMPT "reset to value" ACCEPT &&reset_to
COL seq_id NEW_VALUE hold_seq_id
COL min_id NEW_VALUE hold_min_id
--
SELECT &&reset_to - &&seq_name..nextval - 1 seq_id
FROM dual;
--
SELECT &&hold_seq_id - 1 min_id FROM dual;
--
ALTER SEQUENCE &&seq_name INCREMENT BY &hold_seq_id MINVALUE &hold_min_id;
--
SELECT &&seq_name..nextval FROM dual;
--
ALTER SEQUENCE &&seq_name INCREMENT BY 1;
#--------------------------------------------------------------------------------
```

## 新知识总结
### 数据字典
* CDB_   All of the objects in the CDB across all PDBs
* DBA_   All of the objects in a container or PDB
* ALL_   Objects accessible by the current user
* USER_  Objects owned by the current user

### Application Container中的表
* sharing=data
* sharing=metadata
* sharing=extended data