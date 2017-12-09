---
title: 收获不止SQL优化读书笔记 - 第三章 循规蹈矩--如何读懂SQL执行计划
date: 2017-07-28
tags:
- oracle
- sql优化
---

## 执行计划分析概述
### SQL执行计划
同一条SQL词句，可以有不同的执行计划，但一次只能有一种访问路径。
哪种执行开销更低，性能更好，速度更快，就选哪一种，这个过程叫作Oracle的解析过程。
解析后的SQL执行计划保存到SGA的Shared Pool里，后续再执行同样的SQL只需要在Shared Pool里获取，不需要再次分析了。
SQL执行计划选定选定依据是统计信息

### 查看表和索引的统计信息
``` perl
# 表的相关统计信息
select table_name,num_rows,blocks,last_analyzed from user_tables
where table_name in ('&table_name');

# 索引的相关统计信息
select table_name,index_name,blevel,num_rows,leaf_blocks,last_analyzed from user_indexes
where table_name in ('&table_name');
```

### 数据库统计信息的收集
``` perl
# 收集表统计信息
exec dbms_stats.gather_table_stats(ownname=>'&owner',tabname=>'&table_name',estimate_percent=>10,method_opt=>'for all indexed columns');

# 收集索引统计信息
exec dbms_stats.gather_index_stats(ownname=>'&owner',indname=>'&index_name',estimate_percent=>'10',degree=>'4');

# 收集表和索引统计信息
exec dbms_stats.gather_table_stats(ownname=>'&owner',tabname=>'&table_name',estimate_percent=>10,method_opt=>'for all indexed columns',cascade=>TRUE);

# 收集分区表指定分区统计信息
exec dbms_stats.gather_table_stats(ownname=>'&owner',tabname=>'&table_name',partname='&partition_name',estimate_percent=>10,method_opt=>'for all indexed columns',cascade=>TRUE);
```

## 读懂执行计划的关键


## 从案例中辨别低效SQL


