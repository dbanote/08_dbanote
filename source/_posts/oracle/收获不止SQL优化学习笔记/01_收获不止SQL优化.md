---
title: 收获不止SQL优化读书笔记 - 第一章 全局在胸--用工具对SQL整体优化
date: 2017-07-25
tags:
- oracle
- sql优化
---

## SQL优化不同场景对应工具

### 数据库的整体分析调优工具
1. **AWR**：关注数据库的整体性能报告，主要关注点
 * DB Time：这个指标主要用来判断当前系统有没有遇到相关瓶颈，是否较为繁忙导致等待时长很长
 * load profile：这个指标主要用来展现当前系统的一些指示性能的总体参数
 * efficiency percentages：是一些命中率指标，其中Buffer Hit、Library Hit等都表示SGA(System global area)的命中率
 * Top 10 Foreground Events by Total Wait Time：等待事件是衡量数据库整体优化情况的重要指标
 * SQL Statistics：分别从几个维度来罗列出TOP的SQL
 * Segment Statistics
2. **ASH**：数据库中的等待事件与哪些SQL具体对应的报告，时间更精准
3. **ADDM**：Oracle给出的一些建议
4. **AWRDD**：不同时段的性能对比报告
 * 不同时期load profile的比较
 * 不同时期等待事件的比较
 * 不同时期TOP SQL的比较
5. **AWRSQRPT**：sql在某两个快照间隔内，SQL的统计信息（总消耗的cpu时间，执行次数，逻辑读，物理读等）与执行计划的报告

<!-- more -->
### 数据库的局部分析调优工具
1. explain plan for
2. set autotrace on
3. statistics_level=all
4. 直接通过sql_id获取
5. 10046 trace
6. awrrpt.sql

## 五大性能报告的获取
### AWR
``` perl
@?/rdbms/admin/awrrpt.sql

# 或者
set pagesize 0
set linesize 121
spool /tmp/awr.html
select output from table(dbms_workload_repository.awr_report_html
(98185708, 1 , 27488, 27490));
spool off
# 98185708  =  dbid
# 1         =  instance_number
# 27488     =  min_snap_id
# 27490     =  max_snap_id

# 手工生成断点
exec dbms_workload_repository.create_snapshot();
```

> **小经验：**
1. AWR报告DB Time中，Elapsed时间乘以CPU个数的时间如果大于DB Time，系统压力不大，反之则压力较大。
2. AWR报告load profile中，Redo size就是用来显示平均每秒的日志尺寸和平均每个事务的日志尺寸，结合Transactions这个每秒事务数的指标，就可以分析出当前事务的繁忙程度。 Transactions一般在200上下比较正常，超过1000就属于非常繁忙了。
3. AWR报告的efficiency percentages中Soft Parse指标表示共享池的软解析率，在OLTP系统中如果该指标低于90%应当引起你的注意， 这表示存在未使用绑定变量的情况
4. AWR报告Top 10 Foreground Events中Event和%DB time两列，可以非常直观地看出当前数据库面临的主要等待事件是什么。
5. AWR报告Tablespace IO Stats中Av Rd(ms)项表示平均一次物理读花费的时间，Av Rd(ms)的值大于7一般就说明系统有严重的IO问题

### ASH
``` perl
@?/rdbms/admin/ashrpt.sql
# 说明：
## 1. 如果一路回车，就是获取最近5分钟的ASH报告。
## 2. 如果根据Oldest ASH sample available时间，然后回车，选择的是目前可收集的最长ASH运行情况。
## 3. 可以选择Oldest ASH sample available和Latest ASH sample available之间时间，
##    然后输入时长，比如30 表示30 分钟，取你要取的任何时段的ASH 报告。
## 4. ASH 报告的获取不同于AWR 的地方在于，快照之间有无重启动作不影响报告的获取。

set pagesize 0
set linesize 121
spool /tmp/ash_rpt.html
select output from table(dbms_workload_repository.ash_report_html(977587123,1,SYSDATE- 30/1440 , SYSDATE-1/1440));
spool off
```

### ADDM
``` perl
@?/rdbms/admin/addmrpt.sql

SELECT dbms_advisor.get_task_report('TASK_30670','TEXT','ALL') FROM DUAL ;
```

### AWRDD
``` perl
@?/rdbms/admin/awrddrpt.sql

SELECT dbms_advisor.get_task_report('TASK_30670','TEXT','ALL') FROM DUAL ;
```

### AWRDD
``` perl
# 获取AWR SQRPT 报告的关键之处在于，交互部分要输入所要分析的SOL的SOL_ID(可以从AWR 报告中获取)
@?/rdbms/admin/awrsqrpt.sql

SELECT dbms_advisor.get_task_report('TASK_30670','TEXT','ALL') FROM DUAL ;
```