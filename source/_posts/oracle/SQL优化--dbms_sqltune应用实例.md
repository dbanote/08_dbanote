---
title: SQL优化--dbms_sqltune应用实例
date: 2017-06-28
tags:
- oracle
- sql优化
---

## 需要优化的SQL
``` perl
SELECT a.sjhm
FROM    bfcrm8.hyk_hyxx  b,
        bfcrm8.hyk_grxx  a,
        bfcrm8.hyxfjl    c,
        bfcrm8.hyxfjl_sp d
WHERE   a.hyid         =b.hyid
        AND a.hyid     =c.hyid
        AND c.xfjlid   =d.xfjlid
        AND b.hyktype IN ('101')
        AND b.status  >=0
        AND a.sex      ='1'
        AND d.bmdm LIKE '010104%'
        AND TO_CHAR(sysdate,'YYYY') - TO_CHAR(a.csrq,'YYYY')>='20'
        AND TO_CHAR(sysdate,'YYYY') - TO_CHAR(a.csrq,'YYYY') <'41'
GROUP BY a.sjhm
```

<!-- more -->
## 重新收集相关表的统计信息
``` perl
col OBJECT_NAME for a30
select OBJECT_NAME,OBJECT_TYPE from dba_objects where OBJECT_NAME in ('HYK_HYXX','HYK_GRXX','HYXFJL','HYXFJL_SP') and OWNER='BFCRM8';

OBJECT_NAME                    OBJECT_TYPE
------------------------------ -------------------
HYK_GRXX                       VIEW    # 需要找到对应的表
HYK_HYXX                       TABLE
HYXFJL                         TABLE
HYXFJL_SP                      TABLE

desc dbms_metadata
#--------------------------------------------------------------------
......
FUNCTION GET_DDL RETURNS CLOB
 Argument Name                  Type                    In/Out Default?
 ------------------------------ ----------------------- ------ --------
 OBJECT_TYPE                    VARCHAR2                IN
 NAME                           VARCHAR2                IN
 SCHEMA                         VARCHAR2                IN     DEFAULT
 VERSION                        VARCHAR2                IN     DEFAULT
 MODEL                          VARCHAR2                IN     DEFAULT
 TRANSFORM                      VARCHAR2                IN     DEFAULT
......
#--------------------------------------------------------------------

select dbms_metadata.get_ddl('VIEW','HYK_GRXX','BFCRM8') from dual;
#--------------------------------------------------------------------
......
from BFCRM8.HYK_HYXX X,BFCRM8.HYK_GKDA A
  where X.GKID=A.GKID
#--------------------------------------------------------------------

# 需要重新收集以下4张表
exec dbms_stats.gather_table_stats('BFCRM8','HYK_HYXX',cascade=>true,method_opt=>'for all columns size 1');
exec dbms_stats.gather_table_stats('BFCRM8','HYK_GKDA',cascade=>true,method_opt=>'for all columns size 1');
exec dbms_stats.gather_table_stats('BFCRM8','HYXFJL',cascade=>true,method_opt=>'for all columns size 1');
exec dbms_stats.gather_table_stats('BFCRM8','HYXFJL_SP',cascade=>true,method_opt=>'for all columns size 1');

select TABLE_NAME,NUM_ROWS from dba_tables 
where OWNER='BFCRM8' and TABLE_NAME in ('HYK_HYXX','HYK_GKDA','HYXFJL','HYXFJL_SP');
TABLE_NAME                       NUM_ROWS
------------------------------ ----------
HYXFJL_SP                        86697233
HYXFJL                           27315713
HYK_HYXX                          5962417
HYK_GKDA                           587940
```

## 查找对应的SQL_ID
``` perl
select sql_id from v$sql where sql_text like 'SELECT a.sjhm%';

SQL_ID
-------------
bgfs0u4xch7wu
```

## 根据SQL_ID进行优化
``` perl
set serveroutput on
declare
l_tuning_task varchar2(30);
begin
 l_tuning_task := dbms_sqltune.create_tuning_task(sql_id => '&SQL_ID');
 dbms_sqltune.execute_tuning_task(l_tuning_task);
 dbms_output.put_line(l_tuning_task);
end;
/

Enter value for sql_id: bgfs0u4xch7wu
old   4:  l_tuning_task := dbms_sqltune.create_tuning_task(sql_id => '&SQL_ID');
new   4:  l_tuning_task := dbms_sqltune.create_tuning_task(sql_id => 'bgfs0u4xch7wu');
TASK_40161

PL/SQL procedure successfully completed.
```

## 查看优化建议报告
``` perl
# 内容比较多，建议使用客端工具查看，格式化显示也比较好
set long 10000
select dbms_sqltune.report_tuning_task('&TASK_ID') as re from dual;

GENERAL INFORMATION SECTION
-------------------------------------------------------------------------------
Tuning Task Name   : TASK_40161
Tuning Task Owner  : SYS
Workload Type      : Single SQL Statement
Scope              : COMPREHENSIVE
Time Limit(seconds): 1800
Completion Status  : COMPLETED
Started at         : 06/28/2017 16:03:21
Completed at       : 06/28/2017 16:03:56

-------------------------------------------------------------------------------
Schema Name: SYS
SQL ID     : bgfs0u4xch7wu
SQL Text   : SELECT a.sjhm
             FROM    bfcrm8.hyk_hyxx  b,
                     bfcrm8.hyk_grxx  a,
                     bfcrm8.hyxfjl    c,
                     bfcrm8.hyxfjl_sp d
             WHERE   a.hyid         =b.hyid
                     AND a.hyid     =c.hyid
                     AND c.xfjlid   =d.xfjlid
                     AND b.hyktype IN ('101')
                     AND b.status  >=0
                     AND a.sex      ='1'
                     AND d.bmdm LIKE '010104%'
                     AND TO_CHAR(sysdate,'YYYY') -
             TO_CHAR(a.csrq,'YYYY')>='20'
                     AND TO_CHAR(sysdate,'YYYY') - TO_CHAR(a.csrq,'YYYY')
             <'41'
             GROUP BY a.sjhm

-------------------------------------------------------------------------------
FINDINGS SECTION (2 findings)
-------------------------------------------------------------------------------

1- SQL Profile Finding (see explain plans section below)
--------------------------------------------------------
  为此语句找到了性能更好的执行计划 2。选择以下 SQL 概要文件之一进行实施。

  Recommendation (estimated benefit: 90.68%)
  ------------------------------------------
  - 考虑接受推荐的 SQL 概要文件。
    execute dbms_sqltune.accept_sql_profile(task_name => 'TASK_40161',
            task_owner => 'SYS', replace => TRUE);

  Recommendation (estimated benefit: 99.83%)
  ------------------------------------------
  - 考虑接受建议的 SQL 概要文件, 以便对此语句使用并行执行。
    execute dbms_sqltune.accept_sql_profile(task_name => 'TASK_40161',
            task_owner => 'SYS', replace => TRUE, profile_type =>
            DBMS_SQLTUNE.PX_PROFILE);

  与 DOP 64 并行执行此查询会使 SQL 概要文件计划上的响应时间缩短 98.26%。但是, 启用并行执行时要付出一些代价。它将增加语句的资源消耗
  (预计为 11.07%), 这会导致系统吞吐量降低。此外, 由于在非常短的持续时间内消耗了这些资源, 因此如果没有足够可用的硬件容量,
  并发语句的响应时间将受到负面影响。

  The following data shows some sampled statistics for this SQL from the past
  week and projected weekly values when parallel execution is enabled.

                                 Past week sampled statistics for this SQL
                                 -----------------------------------------
  Number of executions                                                   0 
  Percent of total activity                                              0 
  Percent of samples with #Active Sessions > 2*CPU                       0 
  Weekly DB time (in sec)                                                0 

                              Projected statistics with Parallel Execution
                              --------------------------------------------
  Weekly DB time (in sec)                                                0 

2- Index Finding (see explain plans section below)
--------------------------------------------------
  通过创建一个或多个索引可以改进此语句的执行计划。

  Recommendation (estimated benefit: 99.33%)
  ------------------------------------------
  - 考虑运行可以改进物理方案设计的访问指导或者创建推荐的索引。
    create index BFCRM8.IDX$$_9CE10001 on BFCRM8.HYXFJL_SP("BMDM","XFJLID");

  Rationale
  ---------
    创建推荐的索引可以显著地改进此语句的执行计划。但是, 使用典型的 SQL 工作量运行 "访问指导"
    可能比单个语句更可取。通过这种方法可以获得全面的索引建议案, 包括计算索引维护的开销和附加的空间消耗。
.....
```

## 结合优化建议报告手动执行优化
``` perl
# 从报告文件可以看出，自动调优优化器生成了更优的执行计划，主要方案有以下两种（生产环境中不建议开并行）
## SQL Profile
execute dbms_sqltune.accept_sql_profile(task_name => 'TASK_40161',task_owner => user, replace => TRUE, force_match => true);

## Index
create index BFCRM8.IDX$$_9CE10001 on BFCRM8.HYXFJL_SP("BMDM","XFJLID") tablespace crm2_index;
```

## 使用OEM的SQL优化指导
``` perl
# SQL优化指导 -> 选择优化方案 -> 实施 —> 显示SQL
declare
cmd varchar2(400);
sname varchar2(400);
begin
cmd := 'create  index BFCRM8.IDX$$_9CE10001 on BFCRM8.HYXFJL_SP("BMDM","XFJLID") TABLESPACE "CRM2_INDEX"';
EXECUTE IMMEDIATE cmd;
sname := dbms_sqltune.accept_sql_profile(task_name => 'TASK_40161', object_id => 1);
END;
/
```

## 参考
* [sql_tuneadvisor的使用](http://www.2cto.com/database/201503/385677.html)
* [ORACLE概要文件--sqlprofile](http://www.2cto.com/database/201401/270997.html)
* [SQL优化----dbms_sqltune详解](http://blog.sina.com.cn/s/blog_61cd89f60102edi3.html)