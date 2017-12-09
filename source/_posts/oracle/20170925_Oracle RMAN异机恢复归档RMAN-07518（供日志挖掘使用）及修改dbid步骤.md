---
title: Oracle RMAN异机恢复归档RMAN-07518（供日志挖掘使用）及修改dbid步骤
date: 2017-09-25
tages:
- rman
---


``` perl
# 原库：CRM (DBID=3657993631)  异机：DGT (DBID=1562574039)
sqlplus / as sysdba
exec dbms_backup_restore.nidbegin('dgt','DGT','3657993631','1562574039',0,0,10);

variable a number;
variable b number;
variable c number;

exec dbms_backup_restore.nidprocessdf(0,0,:a,:b,:c);
exec dbms_backup_restore.nidprocesscf(:a,:b);
exec dbms_backup_restore.nidend;

select dbid from v$database;

rman target /
catalog start with '/oradata/nfs/crmpn_rman/arc_temp';

run
{allocate channel ci type disk;
set archivelog destination to '/u01/';
restore archivelog all;
release channel ci;
}
```

<!-- more -->
参考：
1. [Oracle RMAN异机恢复归档RMAN-07518（供日志挖掘使用）及修改dbid步骤](http://blog.csdn.net/lk_db/article/details/51931638)
2. [使用dbms_backup_restore修改DBID](http://blog.csdn.net/lk_db/article/details/51931638

```
PROCEDURE NIDBEGIN
 Argument Name                  Type                    In/Out Default?
 ------------------------------ ----------------------- ------ --------
 NEWDBNAME                      VARCHAR2                IN
 OLDDBNAME                      VARCHAR2                IN
 NEWDBID                        NUMBER                  IN
 OLDDBID                        NUMBER                  IN
 DOREVERT                       BINARY_INTEGER          IN
 DORESTART                      BINARY_INTEGER          IN
 EVENTS                         NUMBER                  IN
PROCEDURE NIDEND
PROCEDURE NIDGETNEWDBID
 Argument Name                  Type                    In/Out Default?
 ------------------------------ ----------------------- ------ --------
 DBNAME                         VARCHAR2                IN
 NDBID                          NUMBER                  OUT
PROCEDURE NIDPROCESSCF
 Argument Name                  Type                    In/Out Default?
 ------------------------------ ----------------------- ------ --------
 CHGDBID                        BINARY_INTEGER          OUT
 CHGDBNAME                      BINARY_INTEGER          OUT
PROCEDURE NIDPROCESSDF
 Argument Name                  Type                    In/Out Default?
 ------------------------------ ----------------------- ------ --------
 FNO                            NUMBER                  IN
 ISTEMP                         BINARY_INTEGER          IN
 SKIPPED                        BINARY_INTEGER          OUT
 CHGDBID                        BINARY_INTEGER          OUT
 CHGDBNAME                      BINARY_INTEGER          OUT
 ```