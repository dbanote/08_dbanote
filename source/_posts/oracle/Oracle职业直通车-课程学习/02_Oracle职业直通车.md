---
title: Oracle职业直通车-02 从最简单的SQL语句开始
date: 2015-03-02
tags:
- oracle
categories:
- Oracle职业直通车
---

## 教材第二章课后作业 1，2，3，4题
### 创建一查询，显示与BLAKE在同一部门工作的雇员的项目和受雇日期，但是BLAKE不包含在内。
``` perl
SQL> conn scott/scott
Connected.
SQL> select * from cat;

TABLE_NAME                     TABLE_TYPE
------------------------------ -----------
BONUS                          TABLE
DEPT                           TABLE
EMP                            TABLE
SALGRADE                       TABLE

select ENAME,JOB,to_char(HIREDATE,'YYYY-mm-dd') as HIREDATE from emp 
  where DEPTNO = (select DEPTNO from emp where ENAME='BLAKE')
    and ENAME<>'BLAKE';

ENAME      JOB       HIREDATE
---------- --------- ----------
ALLEN      SALESMAN  1981-02-20
WARD       SALESMAN  1981-02-22
MARTIN     SALESMAN  1981-09-28
TURNER     SALESMAN  1981-09-08
JAMES      CLERK     1981-12-03
```

<!-- more -->
### 显示位置在Dallas的部门内的雇员姓名、变化以及工作
``` perl
select e.EMPNO,e.ENAME,e.JOB from EMP e, DEPT d
  where e.DEPTNO=d.DEPTNO and LOC='DALLAS';
# 或者
select EMPNO,ENAME,JOB from emp 
  where DEPTNO=(select deptno from dept where LOC='DALLAS');

     EMPNO ENAME      JOB
---------- ---------- ---------
      7369 SMITH      CLERK
      7566 JONES      MANAGER
      7788 SCOTT      ANALYST
      7876 ADAMS      CLERK
      7902 FORD       ANALYST
```

### 显示被 KING 直接管理的雇员的姓名以及工资
``` perl
select ENAME,SAL
  from emp
  where MGR=(select EMPNO from emp where ENAME='KING');
# 或者
SELECT E.ENAME,E.SAL FROM EMP E,EMP EP WHERE E.MGR=EP.EMPNO AND EP.ENAME='KING';

ENAME             SAL
---------- ----------
JONES            2975
BLAKE            2850
CLARK            2450
```

### 创建一查询，显示能获得与 Scott 一样工资和奖金的其他雇员的姓名、受雇日期以及工资。
``` perl
select ENAME,to_char(HIREDATE,'YYYY-MM-DD') HIREDATE,SAL
from emp
where SAL=(select SAL from emp where ENAME='SCOTT')
  and nvl(COMM,0)=(select nvl(COMM,0) from emp where ENAME='SCOTT')
  and ENAME<>'SCOTT';

ENAME      HIREDATE          SAL
---------- ---------- ----------
FORD       1981-12-03       3000
```

##  Oracle企业版和标准版有什么区别？
标准版是为中小型企业提供的功能全面的数据库，它以较低的成本为中小企业提供了世界一流数据库的性能、可用性、可伸缩性和安全性。

企业版在集群化和单一服务器配置方面提供了企业级的性能、可伸缩性和可靠性。它提供了全面的功能来支持要求最严格的事务处理、商务智能和内容管理软件。防止服务器故障、站点故障和人为错误的发生，并减少了计划内的宕机时间利用独特的行级安全性、细粒度审计和透明的数据加密技术来确保数据安全包括了高性能的数据仓库、在线分析处理和数据挖掘功能。

企业版的功能和组件是最全的，标准版的功能和组件有些没有，如分区功能、RAC、 OLAP、DATA MINING 、SPATIAL、ADVANCED REPLICATION等等功能是企业版有标准版没有的。

## 描述ORACLE_BASE,ORACLE_HOME,ORACLE_SID的含义以及它们的用途，并给出你本机的这三个变量的值。
ORACLE_BASE是Oracle产品的基目录，Oracle提供了大量企业级软件，并不单单是Oracle数据库，该目录用于存放所有安装的Oracle产品
ORACLE_HOME是Oracle数据库目录
ORACLE_SID是Oracle数据库实例名

## 在安装过程中如果选择一般事务或者数据仓库，将会有什么不同？
一般事务即适于OLTP，数据库系统善于短事务，高并发，读写频繁，对并发数，内存，变量绑定要求比较高，主要用于在线交易。

数据仓库即适于OLAP，数据库系统善于长事务，低并发，多读而少写，动态采样要求比较高，主要用于高层数据分析。并行在安装过程中选取的模式不一样，那么数据库会针对不同的模式配置不同的参数，以使得数据库运行的更加稳定，更加可靠。

Oracle数据库大多数参数都是可以调节的，但是有一些参数是无法调整的，比如db_block_size，即块大小，这个参数在OLTP和OLAP中设置不一样。对于OLTP数据库，一般不会将db_block_size设置太大，以避免读写时的I/O浪费；而在OLAP数据库中则一般设置较大。

OLTP(on-linetransaction processing)：联机事务处理，表示事务多，但执行大多较短，并发量大的数据库，如日常的进销存操作等；OLTP（在线交易）对并发、内存、变量绑定、优化器等有特殊要求。OLTP适用于用户多、连接数大,资源要求并发量高，内存效率高，SQL解析要求高

OLAP(On-LineAnalytical Processing): 联机分析处理，表示事务较少，但执行大多较长，并发量较小的数据库，如基于数据仓库的操作；OLAP（在线分析）：对I/O，并行，动态采样，优化器有特殊要求OLAP适用于用户不多、数据量大、查询复杂的报表系统，资源要求中I/O要求高、并行（多核CPU处理性能）、SQL执行计划要求高。

## 分别用pfile生成spfile和用spfile生成pfile；分别用这两个参数启动数据库。
``` perl
SQL> conn / as sysdba
Connected.
SQL> show parameter spfile

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      /u01/app/oracle/11.2.0.4/db_1/
                                                 dbs/spfileorcl.ora
SQL> create pfile from spfile;

File created.

SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> ! ls -l /u01/app/oracle/11.2.0.4/db_1/dbs/
total 39768
-rw-r-----. 1 oracle oinstall 10158080 Aug 29  2016 c-1445641855-20160829-01
-rw-r-----. 1 oracle oinstall 10158080 Aug 29  2016 c-1445641855-20160829-03
-rw-r-----. 1 oracle oinstall 10158080 Aug 29  2016 c-1445641855-20160829-04
-rw-r-----. 1 oracle oinstall 10223616 Aug 29  2016 c-1445641855-20160829-06
-rw-rw----. 1 oracle oinstall     1544 Jul 18 10:30 hc_orcl.dat
-rw-r--r--. 1 oracle oinstall     2851 May 15  2009 init.ora
-rw-r--r--  1 oracle oinstall      827 Jul 18 10:30 initorcl.ora
-rw-r-----  1 oracle oinstall       24 Jul 18 08:43 lkORCL
-rw-r-----  1 oracle oinstall     1536 Jul 18 08:46 orapworcl
-rw-r-----  1 oracle oinstall     2560 Jul 18 08:47 spfileorcl.ora

# 使用pfile启动数据库
SQL> startup pfile='/u01/app/oracle/11.2.0.4/db_1/dbs/initorcl.ora';
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             905970328 bytes
Database Buffers         3456106496 bytes
Redo Buffers               11661312 bytes
Database mounted.
Database opened.

SQL> show parameter spfile

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string

# 使用SPFILE启动数据库（默认）
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> create spfile from pfile;

File created.

SQL> startup
ORACLE instance started.

Total System Global Area 4375998464 bytes
Fixed Size                  2260328 bytes
Variable Size             905970328 bytes
Database Buffers         3456106496 bytes
Redo Buffers               11661312 bytes
Database mounted.
Database opened.
SQL> show parameter pfile

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      /u01/app/oracle/11.2.0.4/db_1/
                                                 dbs/spfileorcl.ora
```

## 给表空间增加一个数据文件
``` perl
SQL> col FILE_NAME for a50
SQL> col TABLESPACE_NAME for a15
SQL> select FILE_NAME,TABLESPACE_NAME,BYTES/1024/1024 size_m from dba_data_files;

FILE_NAME                                          TABLESPACE_NAME     SIZE_M
-------------------------------------------------- --------------- ----------
/u01/app/oracle/oradata/orcl/users01.dbf           USERS                    5
/u01/app/oracle/oradata/orcl/undotbs01.dbf         UNDOTBS1                90
/u01/app/oracle/oradata/orcl/sysaux01.dbf          SYSAUX                 520
/u01/app/oracle/oradata/orcl/system01.dbf          SYSTEM                 750
/u01/app/oracle/oradata/orcl/example01.dbf         EXAMPLE            313.125

SQL> alter tablespace USERS add datafile '/u01/app/oracle/oradata/orcl/users02.dbf' size 5M;

Tablespace altered.
```

## 创建一个新的UNDO表空间，并使用它。
``` perl
SQL> create undo tablespace UNDOTBS2 datafile '/u01/app/oracle/oradata/orcl/undotbs02.dbf' size 200M;

Tablespace created.

SQL> alter system set undo_tablespace='UNDOTBS2';

System altered.

SQL> show parameter undo

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_management                      string      AUTO
undo_retention                       integer     900
undo_tablespace                      string      UNDOTBS2
```

## 给当前redo增加一组新的redo group
``` perl
SQL> select GROUP#,MEMBERS,BYTES/1024/1024 size_m from v$log;

    GROUP#    MEMBERS     SIZE_M
---------- ---------- ----------
         1          1         50
         2          1         50
         3          1         50

SQL> col MEMBER for a50
SQL> select GROUP#,MEMBER from v$logfile;

    GROUP# MEMBER
---------- --------------------------------------------------
         3 /u01/app/oracle/oradata/orcl/redo03.log
         2 /u01/app/oracle/oradata/orcl/redo02.log
         1 /u01/app/oracle/oradata/orcl/redo01.log

SQL> alter database add logfile group 4 ('/u01/app/oracle/oradata/orcl/redo04.log') size 50M;

Database altered.

SQL> select GROUP#,MEMBERS,BYTES/1024/1024 size_m from v$log;

    GROUP#    MEMBERS     SIZE_M
---------- ---------- ----------
         1          1         50
         2          1         50
         3          1         50
         4          1         50
```

## 计算当前数据库中所有数据文件的总计大小
```
SQL> select sum(BYTES)/1024/1024 size_m from v$datafile;

    SIZE_M
----------
  1883.125
```

## 杂记
### dual是什么？
* 是Oracle下的一个字典表
* 属于sys用户
* 用于构造一个标准的SQL
* 对优化器有一定影响

``` perl
alter session set nls_date_format='yyyy-mm-dd hh24:mi:ss';
select sysdate from dual;
select user from dual;
```

### SQL语句的种类
* DML: Data Manipulation language
 - SELECT
 - INSERT
 - DELETE
 - UPDATE
* DDL: Data Definition Language
 - CREATE
 - DROP
 - TRUNCATE
 - ALTER
* DCL: Data Control Language
 - GRANT
 - REVOKE

``` perl
create user lyj identified by lyj default tablespace users;
grant connect, resource to lyj;
conn lyj/lyj
create table t1 (id int);
truncate table t1;
alter table t1 add name varchar2(10);
insert into t1 values(1,'lyj');
delete from t1 where id=1;
drop table t1 purge;
conn / as sysdba
drop user lyj cascade;

conn scott/scott
select job,max(sal),min(sal),avg(sal),sum(sal) from emp group by job;

alter database backup controlfile to trace;
```