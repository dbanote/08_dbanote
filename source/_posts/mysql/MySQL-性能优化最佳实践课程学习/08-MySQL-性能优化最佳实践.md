---
title: MySQL性能优化最佳实践 - 08 SQL EXPLAIN解析
date: 2017-10-17
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

## 什么是归并排序？
如果需要排序的数据超过了sort_buffer_size的大小，说明无法在内存中完成排序，就需要写到临时文件中。若排序中产生了临时文件，需要利用归并排序算法保证临时文件中的记录是有序的。归并排序算法是分批将数据放到文件中进行排序，然后逐一按序合并。
简单来说是把在内存中无法直接排序的数据进行分批，每批已排序的结果分别放到文件中。用每个已排序的文件中第一行数据做进行比较，取出最小的值放到最终的合并排序文件中。重复以上操作，直到所有文件文件数据都放到合并排序文件中。
<!-- more -->

## 执行计划中Using temporary与using filesort的区别？
Using temporary：表示MySQL需要使用临时表来存储结果集，常见于排序和分组查询。
Using filesort：MySQL中无法利用索引完成的排序操作称为“文件排序”，Using filesort只能在一张表中进行
示例：
``` perl
mysql> explain
    -> select id from sbtest1
    -> union
    -> select id from sbtest2
    -> order by id;
+----+--------------+------------+-------+---------------+------+---------+------+--------+---------------------------------+
| id | select_type  | table      | type  | possible_keys | key  | key_len | ref  | rows   | Extra                           |
+----+--------------+------------+-------+---------------+------+---------+------+--------+---------------------------------+
|  1 | PRIMARY      | sbtest1    | index | NULL          | k_1  | 4       | NULL | 986400 | Using index                     |
|  2 | UNION        | sbtest2    | index | NULL          | k_2  | 4       | NULL | 986400 | Using index                     |
| NULL | UNION RESULT | <union1,2> | ALL   | NULL          | NULL | NULL    | NULL |   NULL | Using temporary; Using filesort |
+----+--------------+------------+-------+---------------+------+---------+------+--------+---------------------------------+
# Using temporary：表示使用临时表，将sbtest1和sbtest2数据放到临时表中
# Using filesort：表示将这个临时表进行排序，因Using filesort只能在一张表中进行，所以需要把数据汇总到临时表中，然后再进行排序
```

## 表中只有一条数据，为什么在执行计划中也会有using filesort？（实验测试）
``` perl
mysql> select * from test;
+----+------+------+
| id | name | dept |
+----+------+------+
|  1 | lyj  | dnb  |
+----+------+------+

mysql> show create table test\G;
*************************** 1. row ***************************
       Table: test
Create Table: CREATE TABLE `test` (
  `id` int(11) NOT NULL DEFAULT '0',
  `name` varchar(30) DEFAULT NULL,
  `dept` varchar(30) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

mysql> explain select * from test order by name;
+----+-------------+-------+------+---------------+------+---------+------+------+----------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra          |
+----+-------------+-------+------+---------------+------+---------+------+------+----------------+
|  1 | SIMPLE      | test  | ALL  | NULL          | NULL | NULL    | NULL |    1 | Using filesort |
+----+-------------+-------+------+---------------+------+---------+------+------+----------------+
```
filesort表示MySQL需要进行实际的排序操作，而不能通过索引获得已排序数据。

实际上，只要一条 SQL 语句需要进行排序操作，都会显示“Using filesort”，这并不表示就会有文件排序操作。

## explain执行计划包含的信息
### id：执行顺序
- id相同，执行顺序由上至下
- 子查询id的序号会递增，id值越大优先级越高，越先被执行

### select _type：查询类型
1. SIMPLE：查询中不包含子查询或者UNION
2. 查询中若包含任何复杂的子部分，最外层查询则被标记为：PRIMARY
3. 在SELECT或WHERE列表中包含了子查询，该子查询被标记为：SUBQUERY
4. 在FROM列表中包含的子查询被标记为：DERIVED（衍生）
5. 若第二个SELECT出现在UNION之后，则被标记为UNION；若UNION包含在FROM子句的子查询中，外层SELECT将被标记为：DERIVED
6. 从UNION表获取结果的SELECT被标记为：UNION RESULT

### table：表
- 每个执行步骤执行时引用的表
- derivedN--有时不是真实的表名字,看到的是derivedN(N是id值,指第几步执行的结果)
- unionM,N(行记录引用的UNIO连接的多个ID值)

### type：搜索类型
表示MySQL在表中找到所需行的方式，又称“访问类型”，常见类型如下：
- ALL(全表)：Full Table Scan，MySQL将遍历全表以找到匹配的行
- index(索引扫描)：Full Index Scan，index与ALL区别为index类型只遍历索引树
- range(范围扫描)：索引范围扫描，对索引的扫描开始于某一点，返回匹配值域的行，常见于between、<、>等的查询
- ref(关联/参考)：非唯一性索引扫描，返回匹配某个单独值的所有行。常见于使用非唯一索引即唯一索引的非唯一前缀进行的查找
- eq_ref(等值关联/参考)：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描
- const, system(常量)：当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量
- NULL(空)：MySQL在优化过程中分解语句，执行时甚至不用访问表或索引

由上至下，响应时间(RT)由最差到最好,through output不一定
 
``` perl
mysql> explain select id,c,pad from sbtest1 where pad='';
+----+-------------+---------+------+---------------+------+---------+------+--------+-------------+
| id | select_type | table   | type | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+---------+------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | sbtest1 | ALL  | NULL          | NULL | NULL    | NULL | 986400 | Using where |
+----+-------------+---------+------+---------------+------+---------+------+--------+-------------+

mysql> explain select count(*) from sbtest1;
+----+-------------+---------+-------+---------------+------+---------+------+--------+-------------+
| id | select_type | table   | type  | possible_keys | key  | key_len | ref  | rows   | Extra       |
+----+-------------+---------+-------+---------------+------+---------+------+--------+-------------+
|  1 | SIMPLE      | sbtest1 | index | NULL          | k_1  | 4       | NULL | 986400 | Using index |
+----+-------------+---------+-------+---------------+------+---------+------+--------+-------------+

mysql> explain select id,c,pad from sbtest1 where id between 50 and 100;
+----+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table   | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+---------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | sbtest1 | range | PRIMARY       | PRIMARY | 4       | NULL |   51 | Using where |
+----+-------------+---------+-------+---------------+---------+---------+------+------+-------------+

mysql>  explain select * from test where dept='zb';
+----+-------------+-------+------+----------------+----------------+---------+-------+------+--------------------------+
| id | select_type | table | type | possible_keys  | key            | key_len | ref   | rows | Extra                    |
+----+-------------+-------+------+----------------+----------------+---------+-------+------+--------------------------+
|  1 | SIMPLE      | test  | ref  | test_dept_name | test_dept_name | 93      | const |    5 | Using where; Using index |
+----+-------------+-------+------+----------------+----------------+---------+-------+------+--------------------------+

mysql> explain select * from (select * from test where id=1) d1;
+----+-------------+------------+--------+---------------+---------+---------+-------+------+-------+
| id | select_type | table      | type   | possible_keys | key     | key_len | ref   | rows | Extra |
+----+-------------+------------+--------+---------------+---------+---------+-------+------+-------+
|  1 | PRIMARY     | <derived2> | system | NULL          | NULL    | NULL    | NULL  |    1 | NULL  |
|  2 | DERIVED     | test       | const  | PRIMARY       | PRIMARY | 4       | const |    1 | NULL  |
+----+-------------+------------+--------+---------------+---------+---------+-------+------+-------+
```

### possible-keys：可能用到的主键/索引
指出MySQL能使用哪个索引在表中找到行，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用

### key：实际用到的主键/索引
显示MySQL在查询中实际使用的索引，若没有使用索引，显示为NULL。 <br><font color='red'>查询中若使用了覆盖索引，则该索引仅出现在key列表中</font>

### key_len：key的长度
表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度。 <br><font color='red'>key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的</font>

### ref：参考（字段的关联）
表示上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值

### row：估计结果行数
表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数

## MySQL执行计划的局限
- EXPLAIN不会告诉你关于触发器、存储过程的信息或用户自定义函数对查询的影响情况
- EXPLAIN不考虑各种Cache
- EXPLAIN不能显示MySQL在执行查询时所作的优化工作
- 部分统计信息是估算的，并非精确值(如:对Innodb行数的估算)
- EXPALIN只能解释SELECT操作，其他操作要重写为SELECT后查看执行计划（5.6.*版本已经可以对update，delete操作做EXPALIN）

## MySQL5.6支持OPTIMIZER_TRACE
开启可查看步骤：
``` perl
# 开启
mysql> show variables like '%trace%';
+------------------------------+----------------------------------------------------------------------------+
| Variable_name                | Value                                                                      |
+------------------------------+----------------------------------------------------------------------------+
| optimizer_trace              | enabled=off,one_line=off                                                   |
| optimizer_trace_features     | greedy_search=on,range_optimizer=on,dynamic_range=on,repeated_subselect=on |
| optimizer_trace_limit        | 1                                                                          |
| optimizer_trace_max_mem_size | 16384                                                                      |
| optimizer_trace_offset       | -1                                                                         |
+------------------------------+----------------------------------------------------------------------------+

set optimizer_trace='enabled=on';
set optimizer_trace_max_mem_size=1000000;
set end_markers_in_json=on;

# 查看
mysql> select * from information_schema.optimizer_trace\G;
*************************** 1. row ***************************
                            QUERY: explain select id,pad from sbtest1 where id<100
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `sbtest1`.`id` AS `id`,`sbtest1`.`pad` AS `pad` from `sbtest1` where (`sbtest1`.`id` < 100)"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
......
```