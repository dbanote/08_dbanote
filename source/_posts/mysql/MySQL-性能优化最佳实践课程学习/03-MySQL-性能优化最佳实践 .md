---
title: MySQL性能优化最佳实践 - 03 MySQL服务器优化
date: 2017-08-25
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

## 如何配置一个进程只跑在单个CPU上？
CPU性能调优方法之一：把进程或线程绑定在单个CPU上，这可以增加进程的CPU缓存速度，提高它的内存I/O性能。方法如下：

``` perl
# 启动MySQL，将该进程绑定到CPU1上
taskset -c 1 mysqld_safe --defaults-file=/u01/conf/my3306.cnf &

# 使用sysbench压测工具查看CPU绑定后效果
drop database test;
create database test;

sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/select.lua \
--oltp-table-size=100000 --mysql-table-engine=innodb --db-driver=mysql \
--mysql-user=root --mysql-password=root123 --mysql-port=3306 \
--mysql-host=10.245.231.202 --mysql-db=test \
--events=0 --time=60 --oltp-tables-count=20 --report-interval=10 --threads=2 prepare

sysbench /usr/local/share/sysbench/tests/include/oltp_legacy/select.lua \
--oltp-table-size=100000 --mysql-table-engine=innodb --db-driver=mysql \
--mysql-user=root --mysql-password=root123 --mysql-port=3306 \
--mysql-host=10.245.231.202 --mysql-db=test \
--events=0 --time=60 --oltp-tables-count=20 --report-interval=10 --threads=2 run
```

<!-- more -->
使用`top`命令，查看每个CPU的负载(按1查看)，截图如下：
![](http://oligvdnzp.bkt.clouddn.com/0825_mysql_01.png)
从上图中可以看出`MYSQL`的进程绑定到`CPU1`上了

## 基于Linux中可用内存即将耗尽时为了释放更多内存会采取的步骤
``` perl
# 释放前
free -m
#------------------------------------------------------------------------------
             total       used       free     shared    buffers     cached
Mem:          7982       5165       2816          0        180       3661
-/+ buffers/cache:       1324       6658
Swap:        16143         19      16124
#------------------------------------------------------------------------------

cat /proc/sys/vm/drop_caches 
0

sync
echo 3 > /proc/sys/vm/drop_caches
cat /proc/sys/vm/drop_caches 
3

# drop_caches 的值可以是 0-3 之间的数字，代表不同的含义：
# 0：不释放（系统默认值）
# 1：释放页缓存
# 2：释放 dentries 和 inodes
# 3：释放所有缓存

# 释放后
free -m
#------------------------------------------------------------------------------
             total       used       free     shared    buffers     cached
Mem:          7982       1196       6785          0          2         28
-/+ buffers/cache:       1165       6816
Swap:        16143         19      16124
#------------------------------------------------------------------------------

# 释放完内存后改回去让系统重新自动分配内存。
echo 0 >/proc/sys/vm/drop_caches
```

## 逻辑I/O和物理I/O有什么区别？随机I/O和连续I/O有什么区别？
### 逻辑I/O和物理I/O区别
- 逻辑I/O是指指读取内存中的数据
- 物理I/O是指直接读取物理磁盘中的文件

### 随机I/O和连续I/O区别
- 随机I/O需要花费昂贵的磁头旋转和定位来查找
- 连续IO访问的速度远远快于随机IO
- 连续I/O一般只需扫描一次数据，缓存连续I/O的用处不大
- 缓存随机I/O可以节省更多的workload

## 描述一个网络接口工作超负荷会发生什么，包括对应用程序性能的影响。
ping值过高，丢包严重，应用程序连接访问缓慢

## 杂记

### 解决free内存不足和numa架构内存分配不均
1. 保证系统有足够多free内存，这个可以通过设置内存参数
 - vm.min_free_kbytes= NNNNN    当系统free内存少于这个时，内核会启动回收内存，进程在分配内存时也会启动回收内存
 - vm.extra_free_kbytes= NNNNN  当系统free内存少于这个时，内核会从pagecache回收内存（用户进程不会回收内存）
2. 在突然大量连接到来之前保留足够free内存
3. 采用交叉内存分配模式启动mysql或其它需要大内存的系统，保持多个节点之间内存分配平衡`numactl--interleave all command`
4. 优化mysql的%buffer%等参数内存分配，避免过大不合理参数

### 其他
``` perl
cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
      8  Intel(R) Xeon(R) CPU E5-2640 v3 @ 2.60GHz

numactl --hardware

# 内存带宽计算工具
/ptu40_005_lin_intel64/bin/vtbwrun-c -A

# 下载地址：
# http://www.processorportal.com/Download_Intel_Performance_Tuning_Utility_4_0_Update_5/tree3f-aggregator_news_item--103398-/
```