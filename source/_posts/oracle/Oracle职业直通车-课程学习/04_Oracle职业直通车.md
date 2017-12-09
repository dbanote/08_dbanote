---
title: Oracle职业直通车-04 复杂一点的SQL/PLSQL
date: 2015-03-16
tags:
- oracle
categories:
- Oracle职业直通车
---

## 写一个子查询的SQL语句。
``` perl
select b.dname,a.ename
from
  (select deptno,ename from emp) a,
  (select deptno,dname from dept) b
where a.deptno=b.deptno order by 1,2;
```

<!-- more -->
## 分别写一个内连接，左连接，右连接的SQL语句。
``` perl
# 内连接 inner join
select a.dname,b.ename,b.mgr
from dept a, emp b
where a.deptno=b.deptno;

DNAME          ENAME             MGR
-------------- ---------- ----------
ACCOUNTING     CLARK            7839
ACCOUNTING     KING
ACCOUNTING     MILLER           7782
RESEARCH       JONES            7839
RESEARCH       FORD             7566
RESEARCH       ADAMS            7788
RESEARCH       SMITH            7902
RESEARCH       SCOTT            7566
SALES          WARD             7698
SALES          TURNER           7698
SALES          ALLEN            7698
SALES          JAMES            7698
SALES          BLAKE            7839
SALES          MARTIN           7698

# 外连接 outer join - 左连接 左边集合的全集
select a.dname,b.ename,b.mgr
from dept a, emp b
where a.deptno=b.deptno(+);
DNAME          ENAME             MGR
-------------- ---------- ----------
ACCOUNTING     CLARK            7839
ACCOUNTING     KING
ACCOUNTING     MILLER           7782
RESEARCH       JONES            7839
RESEARCH       FORD             7566
RESEARCH       ADAMS            7788
RESEARCH       SMITH            7902
RESEARCH       SCOTT            7566
SALES          WARD             7698
SALES          TURNER           7698
SALES          ALLEN            7698
SALES          JAMES            7698
SALES          BLAKE            7839
SALES          MARTIN           7698
OPERATIONS
IT

# 外连接 outer join - 右连接 右边集合的全集
select a.dname,b.ename,b.mgr
from dept a, emp b
where a.deptno(+)=b.deptno;
```

## 写一个标量子查询的SQL。
``` perl
select 
  (select dname from dept b where b.deptno=a.deptno),ename
from emp a
order by 1,2;
```

## 写一个with语法的SQL语句。
``` perl
# 找出工资大于平均工资的员工
select ename, sum(sal) from emp group by ename having sum(sal)>=(select sum(sal)/14 from emp);

# 使用WITH
with t as (select ename, sal from emp)
select ename,sal from t where sal>=(select sum(sal)/14 from emp);

with t as (select ename,sum(sal) sal from emp group by ename)
select ename,sal from t where sal>=(select sum(sal)/14 from emp);
```

## 用case或者decode语法做一个行列转换的列子，给出SQL语句。
``` perl
# case
select 
  case
    when deptno=10 then 'ACCOUNTING'
    when deptno=20 then 'RESERCH'
    when deptno=30 then 'SALES'
  end,
  sum(sal) from emp
  group by deptno;

# decode
select 
  decode(deptno,
         10,'ACCOUNTING',
         20,'RESERCH',
         30,'SALES'),
  sum(sal) from emp
  group by deptno;

# 行转列
select job,ename,sal from emp where job='MANAGER';

JOB       ENAME             SAL
--------- ---------- ----------
MANAGER   JONES            2975
MANAGER   BLAKE            2850
MANAGER   CLARK            2450

select job,
       sum(decode(ename,'JONES',SAL)) JONES,
       sum(decode(ename,'BLAKE',SAL)) BLAKE,
       sum(decode(ename,'CLARK',SAL)) CLARK
from emp
where job='MANAGER' group by job;

JOB            JONES      BLAKE      CLARK
--------- ---------- ---------- ----------
MANAGER         2975       2850       2450
```

## 写个存储过程，在在屏幕中打印出scott.dept表的内容。
``` perl
create or replace procedure print_dept is
begin
for i in (select * from scott.dept)
loop
  dbms_output.put_line(i.deptno||','||i.dname||','||i.loc);
end loop;
end;
/

set serverout on
exec print_dept
```

## 写个存储过程，通过使用传入的表名作为参数,删除这张表。
``` perl
create table emp1 as select * from emp;

create or replace procedure droptable (v_tablename varchar2) is
begin
  execute immediate 'drop table ' || v_tablename ||' purge';
end;
/

exec droptable('emp1');
```

## 写个函数，通过“表空间名称”作为传入参数，打印出表空间的大小。
``` perl
conn / as sysdba

# 函数
create or replace function f_tablespace_size(v_tablesapce varchar2)  
return varchar2 is
v_size varchar2(10);
begin
  select sum(bytes/1024/1024)||'M' into v_size from dba_data_files where TABLESPACE_NAME=v_tablesapce;
  return v_size;
end;
/

select f_tablespace_size('USERS') from dual;

# 存储过程
create or replace procedure tablespace_size(v_tablespace varchar2) is
v_size varchar2(10);
begin
  select sum(bytes/1024/1024)||'M' into v_size from dba_data_files where TABLESPACE_NAME=v_tablespace;
  dbms_output.put_line(v_tablespace||'表空间的大小为'||v_size);
end;
/

exec tablespace_size('USERS')
```

## 写个触发器，当向一张表中插入数据时，同时向另一张表中插入数据。
``` perl
conn scott/scott
create table t1 (id int);
create table t2 (id int);

create or replace trigger ins_talbe
after insert on t1
for each row
begin
  insert into t2(id) values(:new.id);
end;
/
```


## 课堂练习
``` perl
begin
  for i in 1 .. 10 loop
    null;
  end loop;
end;
/

set serveroutput on;
begin
  for i in 1 .. 10 loop
    dbms_output.put_line('hello,world!');
  end loop;
end;
/

conn scott/scott
drop table t purge;
create table t (id int);

begin
  for i in 1 .. 100 loop
    insert into t values(i);
  end loop;
  commit;
end;
/

# 定义变量的匿名PL/SQL块
set serveroutput on
declare
x varchar2(40):='my first PL/SQL block';
begin
  dbms_output.put_line(x);
end;
/

# 游标cursor
declare
x t.id%type;
cursor c is select * from t;
begin
  open c;
  loop
    fetch c into x;
    exit when c%notfound;
    dbms_output.put_line('id is '|| x);
  end loop;
  close c;
end;
/

# 删除指定员工记录
create or replace procedure delemp(v_empno IN emp.empno%TYPE) as
  no_result exception;
begin
  delete from emp where empno=v_empno;

  if sql%notfound then
    raise no_result;
  end if;
  commit;
  dbms_output.put_line('编码为'||v_empno||'的员工已被删除！');
exception
  when no_result then
    dbms_output.put_line('您需要的数据不存在！');
  when others then
    dbms_output.put_line('其他错误！');
end;
/

exec delemp(100)

# 存储过程的参数 --IN,OUT,IN OUT
CREATE OR REPLACE PROCEDURE ModeTest (
p_InParameter IN NUMBER,
p_OutParameter OUT NUMBER,
p_InOutParameter IN OUT NUMBER) IS
v_LocalVariable NUMBER;
BEGIN
v_LocalVariable := p_InParameter; -- Legal
--p_InParameter := 7; -- Illegal
p_OutParameter := 7; -- Legal
--v_LocalVariable := p_outParameter; -- Illegal
v_LocalVariable := p_InOutParameter; -- Legal
p_InOutParameter := 7; -- Legal
END ModeTest;
/


```