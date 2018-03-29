---
title: oracle 11g修改自动收集统计信息计划任务的执行时间
date: 2018-03-29
tags:
- oracle
---

>oracle 11g默认的自动收集统计信息的时间是22:00--2:00。
但这个时段在公司大型活动期间是业务的高峰期，给本已紧张的系统带来更大的负担。所以，需要把自动执行的时间改到空闲的时段。

## 获得当前自动收集统计信息的执行时间
``` perl
col WINDOW_NAME for a20
col REPEAT_INTERVAL for a60
col DURATION for a30
set linesize 120
SELECT t1.window_name, t1.repeat_interval, t1.duration,enabled
	FROM dba_scheduler_windows t1, dba_scheduler_wingroup_members t2
		WHERE t1.window_name = t2.window_name
			AND t2.window_group_name IN
			('MAINTENANCE_WINDOW_GROUP', 'BSLN_MAINTAIN_STATS_SCHED');

WINDOW_NAME          REPEAT_INTERVAL                                              DURATION                       ENABL
-------------------- ------------------------------------------------------------ ------------------------------ -----
MONDAY_WINDOW        freq=daily;byday=MON;byhour=22;byminute=0; bysecond=0        +000 04:00:00                  TRUE
TUESDAY_WINDOW       freq=daily;byday=TUE;byhour=22;byminute=0; bysecond=0        +000 04:00:00                  TRUE
WEDNESDAY_WINDOW     freq=daily;byday=WED;byhour=22;byminute=0; bysecond=0        +000 04:00:00                  TRUE
THURSDAY_WINDOW      freq=daily;byday=THU;byhour=22;byminute=0; bysecond=0        +000 04:00:00                  TRUE
FRIDAY_WINDOW        freq=daily;byday=FRI;byhour=22;byminute=0; bysecond=0        +000 04:00:00                  TRUE
SATURDAY_WINDOW      freq=daily;byday=SAT;byhour=6;byminute=0; bysecond=0         +000 20:00:00                  TRUE
SUNDAY_WINDOW        freq=daily;byday=SUN;byhour=6;byminute=0; bysecond=0         +000 20:00:00                  TRUE

# WINDOW_NAME：任务名
# REPEAT_INTERVAL：任务重复间隔时间
# DURATION：持续时间
```

<!-- more -->
## 修改自动收集统计信息的执行时间
### 停止任务
``` perl
BEGIN
DBMS_SCHEDULER.DISABLE(
name=>'"SYS"."MONDAY_WINDOW"',
force=>TRUE);
END;
/

BEGIN
DBMS_SCHEDULER.DISABLE(
name=>'"SYS"."TUESDAY_WINDOW"',
force=>TRUE);
END;
/

BEGIN
DBMS_SCHEDULER.DISABLE(
name=>'"SYS"."WEDNESDAY_WINDOW"',
force=>TRUE);
END;
/

BEGIN
DBMS_SCHEDULER.DISABLE(
name=>'"SYS"."THURSDAY_WINDOW"',
force=>TRUE);
END;
/

BEGIN
DBMS_SCHEDULER.DISABLE(
name=>'"SYS"."FRIDAY_WINDOW"',
force=>TRUE);
END;
/

BEGIN
DBMS_SCHEDULER.DISABLE(
name=>'"SYS"."SATURDAY_WINDOW"',
force=>TRUE);
END;
/

BEGIN
DBMS_SCHEDULER.DISABLE(
name=>'"SYS"."SUNDAY_WINDOW"',
force=>TRUE);
END;
/
```

### 修改任务的持续时间，单位是分钟
``` perl
BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."MONDAY_WINDOW"',
attribute=>'DURATION',
value=>numtodsinterval(180, 'minute'));
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."TUESDAY_WINDOW"',
attribute=>'DURATION',
value=>numtodsinterval(180, 'minute'));
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."WEDNESDAY_WINDOW"',
attribute=>'DURATION',
value=>numtodsinterval(180, 'minute'));
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."THURSDAY_WINDOW"',
attribute=>'DURATION',
value=>numtodsinterval(180, 'minute'));
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."FRIDAY_WINDOW"',
attribute=>'DURATION',
value=>numtodsinterval(180, 'minute'));
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."SATURDAY_WINDOW"',
attribute=>'DURATION',
value=>numtodsinterval(180, 'minute'));
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."SUNDAY_WINDOW"',
attribute=>'DURATION',
value=>numtodsinterval(180, 'minute'));
END;
/
```

### 开始执行时间，BYHOUR=4,表示4点开始执行
``` perl
BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."MONDAY_WINDOW"',
attribute=>'REPEAT_INTERVAL',
value=>'freq=daily;byday=MON;byhour=4;byminute=0; bysecond=0');
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."TUESDAY_WINDOW"',
attribute=>'REPEAT_INTERVAL',
value=>'freq=daily;byday=TUE;byhour=4;byminute=0; bysecond=0');
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."WEDNESDAY_WINDOW"',
attribute=>'REPEAT_INTERVAL',
value=>'freq=daily;byday=WED;byhour=4;byminute=0; bysecond=0');
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."THURSDAY_WINDOW"',
attribute=>'REPEAT_INTERVAL',
value=>'freq=daily;byday=THU;byhour=4;byminute=0; bysecond=0');
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."FRIDAY_WINDOW"',
attribute=>'REPEAT_INTERVAL',
value=>'freq=daily;byday=FRI;byhour=4;byminute=0; bysecond=0');
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."SATURDAY_WINDOW"',
attribute=>'REPEAT_INTERVAL',
value=>'freq=daily;byday=SAT;byhour=4;byminute=0; bysecond=0');
END;
/

BEGIN
DBMS_SCHEDULER.SET_ATTRIBUTE(
name=>'"SYS"."SUNDAY_WINDOW"',
attribute=>'REPEAT_INTERVAL',
value=>'freq=daily;byday=SUN;byhour=4;byminute=0; bysecond=0');
END;
/
```

### 开启任务
``` perl
BEGIN
DBMS_SCHEDULER.ENABLE(
name=>'"SYS"."MONDAY_WINDOW"');
END;
/

BEGIN
DBMS_SCHEDULER.ENABLE(
name=>'"SYS"."TUESDAY_WINDOW"');
END;
/

BEGIN
DBMS_SCHEDULER.ENABLE(
name=>'"SYS"."WEDNESDAY_WINDOW"');
END;
/

BEGIN
DBMS_SCHEDULER.ENABLE(
name=>'"SYS"."THURSDAY_WINDOW"');
END;
/

BEGIN
DBMS_SCHEDULER.ENABLE(
name=>'"SYS"."FRIDAY_WINDOW"');
END;
/

BEGIN
DBMS_SCHEDULER.ENABLE(
name=>'"SYS"."SATURDAY_WINDOW"');
END;
/

BEGIN
DBMS_SCHEDULER.ENABLE(
name=>'"SYS"."SUNDAY_WINDOW"');
END;
/
```

## 查看修改后自动收集统计信息的执行时间
``` perl
col WINDOW_NAME for a20
col REPEAT_INTERVAL for a60
col DURATION for a30
set linesize 120
SELECT t1.window_name, t1.repeat_interval, t1.duration,enabled
	FROM dba_scheduler_windows t1, dba_scheduler_wingroup_members t2
		WHERE t1.window_name = t2.window_name
			AND t2.window_group_name IN
			('MAINTENANCE_WINDOW_GROUP', 'BSLN_MAINTAIN_STATS_SCHED');

WINDOW_NAME          REPEAT_INTERVAL                                              DURATION                       ENABL
-------------------- ------------------------------------------------------------ ------------------------------ -----
MONDAY_WINDOW        freq=daily;byday=MON;byhour=4;byminute=0; bysecond=0         +000 03:00:00                  TRUE
TUESDAY_WINDOW       freq=daily;byday=TUE;byhour=4;byminute=0; bysecond=0         +000 03:00:00                  TRUE
WEDNESDAY_WINDOW     freq=daily;byday=WED;byhour=4;byminute=0; bysecond=0         +000 03:00:00                  TRUE
THURSDAY_WINDOW      freq=daily;byday=THU;byhour=4;byminute=0; bysecond=0         +000 03:00:00                  TRUE
FRIDAY_WINDOW        freq=daily;byday=FRI;byhour=4;byminute=0; bysecond=0         +000 03:00:00                  TRUE
SATURDAY_WINDOW      freq=daily;byday=SAT;byhour=4;byminute=0; bysecond=0         +000 03:00:00                  TRUE
SUNDAY_WINDOW        freq=daily;byday=SUN;byhour=4;byminute=0; bysecond=0         +000 03:00:00                  TRUE
```
