---
title: MySQL性能优化最佳实践 - 01 MySQL优化方法论
date: 2017-08-04
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

## MySQL优化方法的关键是？
* MySQL参数优化，innodb_buffer_pool_size/innodb_flush_log_at_trx_commit/sync_binlog
* SQL开发规范
* 分库分表

<!-- more -->

## 如何权衡吞吐率(Throughput) 和延时(Latency)？
- 吞吐率：一般使用单位时间内服务器处理的请求数来描述其并发处理能力,称之为吞吐率（Throughput），单位是`req/s`。吞吐率特指Web服务器单位时间内处理的请求数。
- 延时：延时是描述操作里用来等待服务的时间。在某些情况下，它可以指的是整个操作时间，等同于响应时间。例如，一次应用程序请求、一次数据库查询、一次文件系统操作，等等。举个例子，延时可以表示从点击链接
到屏幕显示整个网页加载完成的时间。
- 目标：在用户能够接受的延时下，尽可能的提高吞吐量。

![](http://oligvdnzp.bkt.clouddn.com/0804_mysql_01.png)

## 如何理解5分钟法则和临近法则？
* 5分钟法则：数据访问过一次，如果要在5分钟之内再次被访问到，就把数据缓存起来。提高资源利用率，降低成本。MySQL中buffer pool就应用了5分钟法则。
* 临近法则：数据被访问之后，跟该数据相连的数据被访问的概率很大，所以把相连的一些数据都预读缓存起来，下次访问相连数据时就可从缓存里找到，提高了效率。

![](http://oligvdnzp.bkt.clouddn.com/0804_mysql_02.png)
