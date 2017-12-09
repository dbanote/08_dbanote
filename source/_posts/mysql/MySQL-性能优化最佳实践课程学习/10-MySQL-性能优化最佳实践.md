---
title: MySQL性能优化最佳实践 - 10 MySQL写出高效SQL
date: 2017-11-9
tags:
- mysql
categories:
- MySQL性能优化最佳实践
---

## MySQL设计标准
1. 数据库命名规范、统一，如vip_xxxx
2. **<font color=red>表一旦设计好，字段只允许增加，不允许减少(drop column)</font>**
3. 统一使用INNODB存储引擎，UTF8编码（整个数据库的编码统一为`utf8_general_ci`，为此不需要建立表的DDL上加上`CHARACTER SET COLLATE utf8_general_ci）`
4. 需在设计阶段考虑如果访问量非常大，且不做`scale out(横向扩展)`表拆分的话，需读写分离，但读写分离注意主从复制有延迟的可能性
5. **<font color=red>禁用</font>**stored procedure(包括存储过程，函数，触发器)，容易将业务逻辑和DB耦合在一起，并且MySQL的存储过程、触发器、函数中存在一定的bug
6. **<font color=red>禁止使用</font>**`UUID()`，`USER()`这样的`MYSQL INSIDE`函数，对于复制来说是很危险的，会导致主备数据不一致，重要的是会严重影响mysql性能
7. 表必须有主键，建议统一由`auto_increment`字段生成整型，不建议使用组合主键， **<font color=red>另外`auto_increment`主键字段只作为虚拟主键，不建议与业务数据处理有关联关系，如果把控不好，会有问题</font>**
8. 库名、表名、字段名、索引名必须使用小写字母
9. 多表join写SQL的时候，一定要给每个字段指定表名做前缀
10. 如果应用使用的是长连接，应用必须具有自动重连的机制。但需避免每执行一个SQL去检查一次DB可用性
<!-- more -->
11. 如果应用使用的是长连接，应用应该具有连接的TIMEOUT检查机制，及时回收长时间没有使用的连接，TIMEOUNT时间一般建议为20min
12. 存储精确浮点数必须使用DECIMAL替代FLOAT和DOUBLE，DECIMAL类型具有更高的精度和更小的范围，它适合于财务和货币计算，如金额等
13. 表名、列名必须有注释（必须加上`COMMENT '<字段扼要解说>'`），表结构变更须由库表OWNER所在团队发起
14. SQL语句必须采用`PreparedStatement`技术，可以提升性能并且避免SQL注入，如果编程语言不支持`PreparedStatement`技术，需要做好特殊字符过滤，如不要前后有空串等
15. 尽可能不要使用TEXT、BLOB、CHAR字段类型，请使用`VARCHAR(N)`，N表示的是字符数不是字节数，比如`VARCHAR(255)`，可以最大存储255个汉字，需要根据实际宽度来设置N，**<font color=red>请注意同一表中，所有VARCHAR字段的长度加起来，不能大于65535</font>**
16. 每张表数据量建议控制在千万级别行以下，为此设计阶段需要考虑数据的归档
17. 如果字段只有`ture or false`，请使用tinyint(数值范围-128～127)，如果模块分类：1订单/2商品；删除标示：0正常/1删除
18. 存储时间（精确到秒）建议使用TIMESTAMP类型，因为TIMESTAMP使用4个字节，DATETIME使用8个字节，同时TIMESTAMP具有自动赋值以及自动更新的特性
19. 每个字段的默认值不能用NULL，禁止`default NULL`，数据类型用`default 0`，字符类型用`default ''`
20. **<font color=red>关键业务数据表，建议用`create_time`和`last_update_time`，方便后期数据分析，如订单表，库存表</font>**
21. **<font color=red>关键业务数据表，如订单表、用户信息表，钱包支付信息等，禁止硬删除，必须软删除，加上`is_deleted`字段，标注这条记录的状态</font>**
22. **<font color=red>避免使用</font>**`select col1,col2 from table where id in (select col from table) 这样的子查询`
23. 需要多表join的字段(**<font color=red>尽量避免多于两表join</font>**)，数据类型保持一致
24. **<font color=red>加字段禁止使用after，因为不能确定全局代码里面是否都采用了类似`insert into table(col,col,col...) values ....`，若中间插一个字段，就会导致数据偏移的问题，影响可大可小，同样`select *`的也可能会影响数值的偏移</font>**
25. **<font color=red>生产环境中，MySQL忽略大小写，配置了`lower_case_table_names=1`</font>**
26. select语句只获取需要的字段，禁止使用select * from语句，这是有效防止新增字段对应用逻辑的影响，还能减少对性能的影响
27. INSERT语句必须显式的指明字段名称，不使用INSERT INTO table values()
28. 禁止在where子句中对字段施加函数，如to_date(add_time)>xxxxx，应改为add_time>=unix_timestamp(date_add(str_to_date('20171109','%Y%m%d'),interval-29 day))
29. where条件中必须使用合适的类型，避免MySOL进行隐式类型转化，如ISENDED=1，字段类型是tinyint，则不能写成ISENDED='1'
30. 应用程序里的SQL语句，禁止一切DDL操作，如有特殊需要需与DBA协商，同意后方可使用
31. 避免在SQL语句进行数学运算或函数运算，容易将业务逻辑和DB耦合在一起
32. INSERT语句使用batch提交
33. INNODB表避免使用count(*)操作，计数统计实时要求较强可以使用memcache或redis，非实时统计可以使用单独统计表，定时更新
34. 使用%前缀模糊查询将不走索引，例如LIKE '%weibo'，不建议使用
35. 避免多余的排序，使用group by时，默认会进行排序，当不需要排序时，可以使用order by null
36. 不鼓励在数据库里排序，行数小的，建议放在应用服务上排序

## 事务的处理标准
1. 一个事务，处理的行数不能超过1000 rows/s，超出会导致主从复制延迟的问题
2. 禁止一些框架或定制化的底层类等使用`set autocommit=0; `、`set autocommit=1;`这样控制事务，应该由程序把控，需要时`begin;`，操作完后及时`commit;`

## 索引使用标准
1. 非唯一索引建议使用`idx_表缩写名称_字段缩写名称`进行命名
2. 唯一索引建议使用`uniq_表缩写名称_字段缩写名称`进行命名
3. 索引名称必须使用小写
4. 唯一键不和主键重复
5. 索引字段的顺序需要考虑字段去重之后的个数，个数多的放在前面，就是数据分布
6. 使用EXPLAIN判断SQL语句是否合理的使用索引，尽量避免extra列出现：`Using File Sort`，`Using Temporary`
7. UPDATE、DELETE语句需要根据WHERE条件添加索引
8. 合理创建联合索引（避免冗余），`(a,b,c)`相当于`(a)`、`(a,b)`、`(a,b,c)`
9. 合理利用覆盖索引（三星索引），如`select emain,uid from user_email where uid=xx`，如果uid不是主键，适当时候可以添加`(uid,email)`覆盖索引，以获得性能提升

## 约束设计
1. 主键的内容不能被修改
2. 禁用外键约束，外键约束一般不在数据库上创建，只表达一个逻辑的概念，由程序控制
3. unique约束命名`UK_列名`，check约束命名`CK_列名`

## 怎么写出高效SQL
1. 清晰无误的了知业务需求
2. 满足业务需求，不做无用功
3. 知道表数据量和索引基本情况
4. 知道完成SQL需要扫描的数据量级
5. SQL执行计划OK？SQL性能达到要求？
6. 调整索引和SQL，优化SQL

## 关于IN子查询
IN子查询容易导致问题，禁止使用，需改成join
1. in子查询包含group by，外部查询的值传不到in子查询中，会导致in子查询效率低下
2. 外部查询传入到in子查询中的值没法利用索引时，会导致in子查询效率低下
3. in子查询总是返回单个值时，用=，=和in处理机制不同
4. 多个filter类子查询，一般在SQL后面的先执行

## 选择正确的驱动表
1. 多表关联查询，选择正确驱动表是最重要一步,好的开始是成功的一半
2. 驱动表不一定是小表；表大，但过滤条件很强，能快速减少关联中间结果也可以选为驱动表
3. access vs filter
4. 关联字段需建索引


## 作业
### 为什么使用prepared statement能提高性能？
Statement主要用于执行静态SQL语句，即内容固定不变的SQL语句。Statement每执行一次都要对传入的SQL语句编译一次，效率较差。

PreparedStatement是Statement的子类，表示预编译的SQL语句的对象。在使用PreparedStatement对象执行SQL命令时，命令被数据库编译和解析，并放入命令缓冲区。缓冲区中的预编译SQL命令可以重复使用。

使用PreparedStatement时，SQL语句已提前编译，三种常用方法`execute`、`executeQuery`和`executeUpdate`已被更改，以使之不再需要参数。PreparedStatement 实例包含已事先编译的SQL语句，SQL语句可有一个或多个IN参数，IN参数的值在SQL语句创建时未被指定。该语句为每个IN留一个问号（“？”）作为占位符。 每个问号的值必须在该语句执行之前，通过适当的setInt或者setString 等方法提供。由于 PreparedStatement 对象已预编译过，所以其执行速度要快于Statement 对象。因此，多次执行的 SQL 语句经常创建为PreparedStatement 对象，以提高效率。

PreparedStatement的另外一个好处就是预防sql注入攻击。对JDBC而言，SQL注入攻击对PreparedStatement无效，因为PreparedStatement不允许在插入参数时改变SQL语句的逻辑结构。

### MySql优化的一般步骤是什么？举个例子详细说明。
#### 通过`show status`命令了解各种sql的执行效率
通过SHOW STATUS可以提供服务器状态信息，也可以使用mysqladmin extended-status命令获得。SHOW STATUS可以根据需要显示session级别的统计结果和global级别的统计结果。
``` perl
# 以下几个参数对Myisam和Innodb存储引擎都计数
Com_select  执行select操作的次数，一次查询只累加1
Com_insert  执行insert操作的次数，对于批量插入的insert操作，只累加一次
Com_update  执行update操作的次数
Com_delete　执行delete操作的次数

# 以下几个参数是针对Innodb存储引擎计数的，累加的算法也略有不同
Innodb_rows_read　   select查询返回的行数
Innodb_rows_inserted 执行insert操作插入的行数
Innodb_rows_updated  执行update操作更新的行数
Innodb_rows_deleted　执行delete操作删除的行数

# 以下几个参数便于了解数据库的基本情况
Connections  试图连接Mysql服务器的次数
Uptime　     服务器工作时间
Slow_queries 慢查询的次数

# 对于事务型的应用，通过Com_commit和Com_rollback可以了解事务提交和回滚的情况
# 对于回滚操作非常频繁的数据库，可能意味着应用编写存在问题
```

#### 定位执行效率较低的SQL语句
可以通过以下两种方式定位执行效率较低的SQL语句：
1. 可以通过慢查询日志定位那些执行效率较低的sql语句，用`--log-slow-queries[=file_name]`选项启动时，mysqld写一个包含所有执行时间超过`long_query_time`秒的SQL语句的日志文件。可以链接到管理维护中的相关章节。
2. 慢查询日志在查询结束以后才纪录，所以在应用反映执行效率出现问题的时候查询慢查询日志并不能定位问题，可以使用`show processlist`命令查看当前MySQL在进行的线程，包括线程的状态，是否锁表等等，可以实时的查看SQL执行情况，同时对一些锁表操作进行优化

#### 通过explain分析低效率的SQL语句的执行情况
通过以上步骤查询到效率低的SQL后，可以通过explain或者desc获取MySQL如何执行SELECT语句的信息，包括select语句执行过程表如何连接和连接的次序。
执行计划说明：
>select_type：select类型
>table：输出结果集的表
>type：表示表的连接类型
>　　1. 当表中仅有一行是type的值为system是最佳的连接类型；
>　　2. 当select操作中使用索引进行表连接时type的值为ref；
>　　3. 当select的表连接没有使用索引时，经常会看到type的值为ALL，表示对该表进行了全表扫描，这时需要考虑通过创建索引来提高表连接的效率。
>possible_keys：表示查询时,可以使用的索引列.
>key：表示使用的索引
>key_len：索引长度
>rows：扫描范围
>Extra：执行情况的说明和描述

#### 确定问题，并采取相应的优化措施
确认问题出现的原因后，可以根据情况采取相应的措施，进行优化提高执行的效率。 示例如下：
``` perl
mysql> explain select e.emp_no,e.first_name,e.last_name,e.hire_date,d.dept_no from employees e,dept_emp d 
       where d.emp_no=e.emp_no and d.dept_no='d005';
+----+-------------+-------+--------+---------------+---------+---------+--------------------+--------+-------------+
| id | select_type | table | type   | possible_keys | key     | key_len | ref                | rows   | Extra       |
+----+-------------+-------+--------+---------------+---------+---------+--------------------+--------+-------------+
|  1 | SIMPLE      | d     | ALL    | NULL          | NULL    | NULL    | NULL               | 331290 | Using where |
|  1 | SIMPLE      | e     | eq_ref | PRIMARY       | PRIMARY | 4       | employees.d.emp_no |      1 | NULL        |
+----+-------------+-------+--------+---------------+---------+---------+--------------------+--------+-------------+

# 这个例子，对表dept_emp全表扫描，效果不理想，对d表的dept_no字段创建了索引，查询需要扫描的行数明显较少
create index idx_dept_emp_dn on dept_emp(dept_no);

mysql> explain select e.emp_no,e.first_name,e.last_name,e.hire_date,d.dept_no from employees e,dept_emp d 
       where d.emp_no=e.emp_no and d.dept_no='d005';
+----+-------------+-------+--------+-----------------+-----------------+---------+--------------------+--------+-----------------------+
| id | select_type | table | type   | possible_keys   | key             | key_len | ref                | rows   | Extra                 |
+----+-------------+-------+--------+-----------------+-----------------+---------+--------------------+--------+-----------------------+
|  1 | SIMPLE      | d     | ref    | idx_dept_emp_dn | idx_dept_emp_dn | 12      | const              | 164256 | Using index condition |
|  1 | SIMPLE      | e     | eq_ref | PRIMARY         | PRIMARY         | 4       | employees.d.emp_no |      1 | NULL                  |
+----+-------------+-------+--------+-----------------+-----------------+---------+--------------------+--------+-----------------------+
```

### MySQL 这该死的 “IN (子查询)”，如下SQL怎么优化 “in(子查询)”？
``` perl
# 优化前
SELECT s1 FROM t1 WHERE s1 IN (SELECT s1 FROM t2 WHERE id = 1);

# 优化后
SELECT s1 from t1 a,(SELECT s1 FROM t2 WHERE id = 1) b where a.s1=b.s1;
```


``` perl
# 这是一个简单的示例，每1000个语句为一批插入提交，它避免了SQL注入和内存不足的问题
# 插入到数据库使用批处理上万条记录，有可能产生的OutOfMemoryError
import java.sql.Connection;
import java.sql.PreparedStatement;
...
String sql = "insert into employee (name, city, phone) values (?, ?, ?)";
Connection connection = new getConnection();
PreparedStatement ps = connection.prepareStatement(sql);
final int batchSize = 1000;
int count = 0;
for (Employee employee: employees) {
    ps.setString(1, employee.getName());
    ps.setString(2, employee.getCity());
    ps.setString(3, employee.getPhone());
    ps.addBatch();
    if(++count % batchSize == 0) {
        ps.executeBatch();
    }
}
ps.executeBatch(); // insert remaining records
ps.close();
connection.close();
```