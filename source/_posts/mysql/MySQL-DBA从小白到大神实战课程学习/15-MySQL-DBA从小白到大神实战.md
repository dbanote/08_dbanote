---
title: MySQL-DBA从小白到大神实战-15 MySQL源码初探
date: 2017-05-02
tags:
- mysql
categories:
- MySQL DBA从小白到大神实战
---

## 通过gdb工具分析mysqld进程启动的过程
``` perl
gdb --args /u01/mysql/bin/mysqld
#--------------------------------------------------------------------------------
GNU gdb (GDB) Red Hat Enterprise Linux (7.2-90.el6)
Copyright (C) 2010 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-redhat-linux-gnu".
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>...
Reading symbols from /u01/mysql/bin/mysqld...done.
(gdb) b mysqld_main   # 设置断点
Breakpoint 1 at 0x58c0e4: file /u01/mysql-5.6.35/sql/mysqld.cc, line 5245.
(gdb) r   # 运行
Starting program: /u01/mysql/bin/mysqld 
[Thread debugging using libthread_db enabled]

Breakpoint 1, mysqld_main (argc=1, argv=0x7fffffffe668) at /u01/mysql-5.6.35/sql/mysqld.cc:5245
5245    {
Missing separate debuginfos, use: debuginfo-install glibc-2.12-1.192.el6.x86_64 keyutils-libs-1.4-5.0.1.el6.x86_64 
krb5-libs-1.10.3-57.el6.x86_64 libcom_err-1.42.8-1.0.2.el6.x86_64 libgcc-4.4.7-17.el6.x86_64 
libselinux-2.0.94-7.el6.x86_64  libstdc++-4.4.7-17.el6.x86_64 nss-softokn-freebl-3.14.3-23.el6_7.x86_64 
openssl-1.0.1e-48.el6.x86_64 zlib-1.2.3-29.el6.x86_64
(gdb) n   # 向下
5250      my_progname= argv[0];
(gdb) n
5254      if (my_init())                 // init my_sys library & pthreads
(gdb) s   # 看源码位置
my_init () at /u01/mysql-5.6.35/mysys/my_init.c:69
69        if (my_init_done)
(gdb) n
74        my_umask= 0660;                       /* Default umask for new files */
(gdb) n
75        my_umask_dir= 0700;                   /* Default umask for new directories */
#--------------------------------------------------------------------------------
```

<!-- more -->

查看`/u01/mysql-5.6.35/sql/mysqld.cc`的line 5245
``` perl
{
  /*
    Perform basic thread library and malloc initialization,
    to be able to read defaults files and parse options.
  */
  my_progname= argv[0];
```

## 杂记
### MySQL代码结构
1. **Client：客户端工具集合**
比如mysql，mysqldump，mysqlbinlog等等
2. **Myisam：myisam引擎相关的代码**
mi_open.c：打开文件
mi_close.c：关闭文件
mi_update.c：更新记录
3. **Mysys：系统工具代码**
比如charset.c，mf_qsort.c等
4. **Sql：MySQL核心代码**
sql_lex.cc：词法解析
sql_yacc.yy: 语法解析
sql_select.c, sql_update, sql_delete.c, sql_insert.c等文件
5. **Vio：网络通信代码**
viosocket.c：socket通信
6. **第三方开源库**
Dbug，pstack，regex，strings，zlib
7. **Innobase**
InnoDB引擎代码
8. **Mysql-test**
MySQL testcase
9. **Scripts**
mysql_install_db，mysql_system_tables.sql
10. **win**
windows用到的

### 源码查看工具sourceinsight
下载地址：[http://pan.baidu.com/s/1bprLzZL ](http://pan.baidu.com/s/1bprLzZL)