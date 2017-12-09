---
title: MySQL性能优化最佳实践 - 11 MySQL锁优化分析
date: 2017-11-9
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

## 不同索引加锁顺序的问题，模拟重现死锁
走索引的SQL语句，会涉及两把锁，在特定场景下就会产生交差锁等待
### 测试表结构及相关语句
``` perl
mysql> show create table employees\G;
*************************** 1. row ***************************
       Table: employees
Create Table: CREATE TABLE `employees` (
  `emp_no` int(11) NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) NOT NULL,
  `last_name` varchar(16) NOT NULL,
  `gender` enum('M','F') NOT NULL,
  `hire_date` date NOT NULL,
  `base_salaries` decimal(8,2) NOT NULL,
  `id` int(11) NOT NULL AUTO_INCREMENT,
  PRIMARY KEY (`id`),
  KEY `idx_emp_hire` (`hire_date`),
  KEY `idx_emp_no` (`emp_no`)
) ENGINE=InnoDB AUTO_INCREMENT=300151 DEFAULT CHARSET=utf8

alter table employees drop primary key;
create index idx_emp_no on employees(emp_no);
alter table employees add id int;
alter table employees change id id int not null auto_increment primary key; 
```

<!-- more -->

### 模拟重现死锁
```perl
# 打开两个会话
# 1. 会话1
mysql> start transaction;
mysql> select * from employees where emp_no=99075 for update;
+--------+------------+------------+-----------+--------+------------+---------------+-------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | base_salaries | id    |
+--------+------------+------------+-----------+--------+------------+---------------+-------+
|  99075 | 1961-08-14 | Filipp     | Covnot    | M      | 1999-01-02 |          0.00 | 89075 |
+--------+------------+------------+-----------+--------+------------+---------------+-------+

mysql> explain select * from employees where emp_no=99075;
+----+-------------+-----------+------+---------------+------------+---------+-------+------+-------+
| id | select_type | table     | type | possible_keys | key        | key_len | ref   | rows | Extra |
+----+-------------+-----------+------+---------------+------------+---------+-------+------+-------+
|  1 | SIMPLE      | employees | ref  | idx_emp_no    | idx_emp_no | 4       | const |    1 | NULL  |
+----+-------------+-----------+------+---------------+------------+---------+-------+------+-------+

# 2. 会话2
mysql> start transaction;
mysql> select * from employees where emp_no=108961 for update;
+--------+------------+------------+-----------+--------+------------+---------------+-------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | base_salaries | id    |
+--------+------------+------------+-----------+--------+------------+---------------+-------+
| 108961 | 1962-11-13 | Jianhua    | Piveteau  | F      | 1999-01-02 |       1000.00 | 98961 |
+--------+------------+------------+-----------+--------+------------+---------------+-------+

# 3. 会话1
mysql> update employees set base_salaries=base_salaries+1000 where emp_no=108961;
## 这时会产生锁等待，新开一个会话可以查看锁的状态
mysql> select * from information_schema.innodb_locks;
+--------------------+-------------+-----------+-----------+-------------------------+------------+------------+-----------+----------+---------------+
| lock_id            | lock_trx_id | lock_mode | lock_type | lock_table              | lock_index | lock_space | lock_page | lock_rec | lock_data     |
+--------------------+-------------+-----------+-----------+-------------------------+------------+------------+-----------+----------+---------------+
| 26909:208:1522:918 | 26909       | X         | RECORD    | `employees`.`employees` | idx_emp_no |        208 |      1522 |      918 | 108961, 98961 |
| 26910:208:1522:918 | 26910       | X         | RECORD    | `employees`.`employees` | idx_emp_no |        208 |      1522 |      918 | 108961, 98961 |
+--------------------+-------------+-----------+-----------+-------------------------+------------+------------+-----------+----------+---------------+

# 4. 会话2
mysql> update employees set base_salaries=base_salaries+1000 where emp_no=99075;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
## 产生死锁

# 5. 会话1
mysql> update employees set base_salaries=base_salaries+1000 where emp_no=108961;
Query OK, 1 row affected (29.87 sec)
Rows matched: 1  Changed: 1  Warnings: 0
## 解锁，update语句执行成功
mysql> select * from employees where emp_no=108961;
+--------+------------+------------+-----------+--------+------------+---------------+-------+
| emp_no | birth_date | first_name | last_name | gender | hire_date  | base_salaries | id    |
+--------+------------+------------+-----------+--------+------------+---------------+-------+
| 108961 | 1962-11-13 | Jianhua    | Piveteau  | F      | 1999-01-02 |       2000.00 | 98961 |
+--------+------------+------------+-----------+--------+------------+---------------+-------+

# 6. 在新开会话中查看最后一次死锁信息
mysql> show engine innodb status\G;
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
....

------------------------
LATEST DETECTED DEADLOCK
------------------------
2017-11-21 16:46:52 7fc588cb9700
*** (1) TRANSACTION:
TRANSACTION 26909, ACTIVE 71 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 4 lock struct(s), heap size 1184, 3 row lock(s)
MySQL thread id 157583, OS thread handle 0x7fc588c78700, query id 2221313 localhost root updating
update employees set base_salaries=base_salaries+1000 where emp_no=108961
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 208 page no 1522 n bits 1272 index `idx_emp_no` of table `employees`.`employees` trx id 26909 lock_mode X locks rec but not gap waiting
Record lock, heap no 918 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8001a9a1; asc     ;;
 1: len 4; hex 80018291; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 26910, ACTIVE 32 sec starting index read
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1184, 3 row lock(s)
MySQL thread id 157584, OS thread handle 0x7fc588cb9700, query id 2221315 localhost root updating
update employees set base_salaries=base_salaries+1000 where emp_no=99075
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 208 page no 1522 n bits 1272 index `idx_emp_no` of table `employees`.`employees` trx id 26910 lock_mode X locks rec but not gap
Record lock, heap no 918 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8001a9a1; asc     ;;
 1: len 4; hex 80018291; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 208 page no 1514 n bits 1272 index `idx_emp_no` of table `employees`.`employees` trx id 26910 lock_mode X locks rec but not gap waiting
Record lock, heap no 656 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80018303; asc     ;;
 1: len 4; hex 80015bf3; asc   [ ;;
......
```

## TMS死锁分析和处理，模拟重现死锁
重现死锁
``` perl
show variables like '%isolation%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| tx_isolation  | REPEATABLE-READ |
+---------------+-----------------+

show variables like '%lock_wait%';
+--------------------------+----------+
| Variable_name            | Value    |
+--------------------------+----------+
| innodb_lock_wait_timeout | 20       |
| lock_wait_timeout        | 31536000 |
+--------------------------+----------+

show variables like '%autocommit%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+

# 会话1
use lyj
create table t2 (
    a int not null default 0,
    b int default null,
    primary key (a));

insert into t2 (a,b) values(2,2);
insert into t2 (a,b) values(3,3);
insert into t2 (a,b) values(4,4);
commit;
select * from t2;
+---+------+
| a | b    |
+---+------+
| 2 |    2 |
| 3 |    3 |
| 4 |    4 |
+---+------+

# 下面为了方便区分，将提示符加上时间显示
## 启动mysql时添加 --prompt="\\u@\\h:\\d \\r:\\m:\\s> "
root@localhost:lyj 10:35:42> insert into t2 (a,b) values(5,5);
Query OK, 1 row affected (0.00 sec)

# 会话2
root@localhost:lyj 10:35:37> use lyj
Database changed
root@localhost:lyj 10:36:30> select * from t2;
+---+------+
| a | b    |
+---+------+
| 2 |    2 |
| 3 |    3 |
| 4 |    4 |
+---+------+

## 这时有锁等待
root@localhost:lyj 06:07:24> delete from t2 where b=2;
## 查看锁信息
select * from information_schema.innodb_locks;
+---------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| lock_id       | lock_trx_id | lock_mode | lock_type | lock_table | lock_index | lock_space | lock_page | lock_rec | lock_data |
+---------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+
| 29509:211:3:5 | 29509       | X         | RECORD    | `lyj`.`t2` | PRIMARY    |        211 |         3 |        5 | 5         |
| 29504:211:3:5 | 29504       | X         | RECORD    | `lyj`.`t2` | PRIMARY    |        211 |         3 |        5 | 5         |
+---------------+-------------+-----------+-----------+------------+------------+------------+-----------+----------+-----------+

# 会话1
root@localhost:lyj 06:07:32>insert into t2 (a,b) values(6,6);
Query OK, 1 row affected (0.00 sec)

root@localhost:lyj 06:08:13>insert into t2 (a,b) values(1,1);
Query OK, 1 row affected (0.00 sec)

# 会话2，刚才的等待出现死锁错误
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```
死锁分析：在RR事务隔离级别下，由于DELETE没有用主键或者唯一索引，和INSERT形成死锁，无法避免。
处理办法：将事务隔离级别修改为RC，或才在RR事务隔离级别，将`innodb_locks_unsafe_for_binlog`设置为`on`也可以。

## MySQL如何避免死锁
### 注意程序的逻辑（最重要）
根本的原因是程序逻辑的顺序，最常见的是交差更新
``` perl
Transaction 1: 更新表A -> 更新表B
Transaction 2: 更新表B -> 更新表A
```
这类问题要从程序上避免，所有的更新需要按照一定顺序

### 保持事务的轻量
越是轻量的事务，占有越少的锁资源，这样发生死锁的几率就越小
1. 提高运行速度，避免使用子查询，尽量使用主键等
2. 尽量快提交事务，减少持有锁的时间

死锁是数据库对事务的保护机制，一旦发生死锁，mysql会选择相对小的事务(undo较少的)进行回滚。死锁的发生与否，并不在于事务中有多少条SQL语句，死锁的关键在于：两个或两个以上的session加锁的顺序不一致。分析MySQ每条SQL的加锁规则、顺序，检查多个并发SQL间是否存在相反的顺序加锁的情况，就可以分析出各种潜在的死锁情况，也可以分析出线上死锁发生的原因。