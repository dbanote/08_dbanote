---
title: GoldenGate 12.1.2 新特性参数部分解释
tags:
- oracle
- goldengate
---

## LOGALLSUPCOLS
Writes all supplementally logged columns to the trail, including those required for conflict detection and resolution and the scheduling columns required to support integrated Replicat. (Scheduling columns are primary key, unique index, and foreign key columns.) You configure the database to log these columns with GGSCI commands.
使用该参数可以记录所有附加日志信息并写入trail文件中。


## UPDATERECORDFORMAT COMPACT
Combines the before and after images of an UPDATE operation into a single record in the trail. This parameter is valid for Oracle databases version 12c and later to support Replicat in integrated mode. Although not a required parameter, UPDATERECORDFORMAT COMPACT is a best practice and significantly improves Replicat performance.
UPDATERECORDFORMAT参数可以控制抽取进程兼容更新操作的前镜像和后镜像信息并写入到一个trail文件中，但是一定要注意的是该参数必须在数据库版本是11.2.0.4或者12c版本。