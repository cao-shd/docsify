# MYSQL 性能优化

## 优化方向

- 数据库表设计
- 索引
- 分表技术
- 读写分离
- 存储过程
- 配置优化
- 硬件升级
- 定时整理

## 数据库表设计

- 三范式
- 反三范式

## 运行状态查询

- 运行状态：为了确认数据库各种操作更适合那种引擎

```sql
show status\G

show [session|global] like '<param>'

-- 查看启动时间 s
show status like 'uptime';

-- 执行过多少次查询
show global status like 'com_select';

-- 执行过多少次删除
show status like 'com_delete';

-- 执行过多少次更新
show status like 'com_update';

-- 查询连接数
show status like 'connections';

-- 显示慢查询
show status like 'slow_queries';
```

## 定位慢查询

- 配置数据库 my.ini 文件

```properties
[mysqld]
slow-query-log=ON
long_query_time=2
```

- 定位慢查询

```sql
-- 查看变量
show variables\G

-- 查看慢查询配置 默认 10s
show variables like 'long_query_time';

-- 修改慢查询配置
set long_query_time=1;

-- 显示慢查询
show status like 'slow_queries';
```

- 测试 构建大表 数据尽量随机

```sql
-- 创建数据库
create database if not exists mytest charset utf8;

-- 使用数据库
use mytest;

-- 创建部门表
create table if not exists dept (
    deptno MEDIUMINT UNSIGNED NOT NULL DEFAULT 0,
    dname VARCHAR(20) NOT NULL DEFAULT '',
    loc VARCHAR(13) NOT NULL DEFAULT ''
) DEFAULT CHARSET=UTF8;

-- 创建员工表
create table if not exists emp (
    empno MEDIUMINT UNSIGNED NOT NULL DEFAULT 0,
    ename VARCHAR(20) NOT NULL DEFAULT '',
    job VARCHAR(9) NOT NULL DEFAULT '',
    mgr VARCHAR(9) NOT NULL DEFAULT '',
    hiredate DATE NOT NULL,
    sal DECIMAL(7, 2) NOT NULL,
    com DECIMAL(7, 2) NOT NULL,
    deptno MEDIUMINT UNSIGNED NOT NULL DEFAULT 0
) DEFAULT CHARSET=UTF8;

-- 创建工资级别表
create table if not exists salgrade (
    grade MEDIUMINT UNSIGNED NOT NULL DEFAULT 0,
    losal DECIMAL(17, 2) NOT NULL,
    hisal DECIMAL(17, 2) NOT NULL
) DEFAULT CHARSET=UTF8;

-- 测试数据 
-- 工资级别
insert into salgrade values (1, 700, 1200);
insert into salgrade values (2, 1201, 1400);
insert into salgrade values (3, 1401, 2000);
insert into salgrade values (4, 2001, 3000);
insert into salgrade values (5, 3001, 9999);

-- 创建存过
-- 定义结束符
set global log_bin_trust_function_creators=TRUE;

delimiter $$

-- 删除自定义函数 rand_string
drop function if exists rand_string $$
create function rand_string(n INT) returns varchar(255)
begin
  declare chars_str varchar(100) default 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  declare return_str varchar(255) default '';
  declare i int default 0;
  while i < n do
    set return_str = concat(return_str, substring(chars_str, floor(1 + rand()*52), 1));
    set i = i + 1;
  end while;
  return return_str;
end $$

-- 创建自定义函数 rand_num
drop function if exists rand_num $$
create function rand_num() returns int(5)
begin
  declare i int default 0;
    set i = floor(1 + rand()*500);
  return i;
end $$

-- 创建自定义存过 insert_emp
drop procedure if exists insert_emp $$
create procedure insert_emp(in start int(10), in max_num int(10))
begin
  declare i int default 0;
  set autocommit = 0;  
  repeat
    set i = i + 1;
    insert into emp values ((start + i), rand_string(6), 'SALESMAN', 0001, curdate(), 2000, 400, rand_num());
  until i = max_num end repeat;
  commit;
end $$

-- 创建自定义存过 insert_dept
drop procedure if exists insert_dept $$
create procedure insert_dept(in start int(10), in max_num int(10))
begin
  declare i int default 0;
  set autocommit = 0;  
  repeat
    set i = i + 1;
    insert into dept values ((start + i), rand_string(10), rand_string(8));
  until i = max_num end repeat;
  commit;
end $$

-- 创建自定义存过 insert_salgrade
drop procedure insert_salgrade $$
create procedure insert_salgrade(in start int(10), in max_num int(10))
begin
  declare i int default 0;
  set autocommit = 0;  
  ALTER TABLE emp DISABLE KEYS;  
  repeat
    set i = i + 1;
    insert into salgrade values ((start + i), (start + i), (start + i));
  until i = max_num end repeat;
  commit;
end $$

-- 定义结束符
delimiter ;

-- 测试自定义函数
select rand_string(6);
select rand_num();

call insert_emp(100001, 4000000);

call insert_dept(100, 10);

call insert_salgrade(10000, 1000000);
```

## 执行计划

- 查看执行计划

```sql
explain SELECT XXX
```

## 索引

索引分类

- 主键索引
- 唯一索引
- 普通索引
- 全文索引

索引操作

- 添加索引

```sql
-- 建表语句添加索引
create table test_idx (
  -- 主键索引
  id int unsigned primary key auto_increment,
  -- 唯一索引 内容可以为 NULL
  name varchar(32) unique 
);

-- 建表后添加主键索引
alter table <table_name> add primary key(<column_name>);
create index <index_name> on <table_name>(<column_name>);
create unique index <index_name> on <table_name>(<column_name>);

create table test_idx1 (
  id int unsigned,
  name varchar(32)
);
alter table test_idx1 add primary key(id);
create index idx_test_idx1_1 on test_idx1(name);
create unique index uni_name on test_idx1(name);
```

- 查询索引

```sql
-- 查看表结构
-- 不能显示索引名字
desc <table_name>;
desc test_idx;

-- 查看索引
show indexes from <table_name>\G
show index from <table_name>\G
show keys from <table_name>\G

show indexes from test_idx\G
show index from test_idx\G
show keys from test_idx\G
```

- 删除索引

```sql
-- 删除普通索引
drop index <index_name> on <table_name>;
alter table <table_name> drop index <index_name>;

drop index uni_name on test_idx1;
alter table test_idx1 drop index uni_name;

--删除主键索引
alter table <table_name> drop primary key;

alter table test_idx1 drop primary key;
```

- 修改索引

```sql
-- 先删除再重建
drop index <index_name> on <table_name>;
create index uni_name on test_idx1(name);
```

- 索引使用

```latex
1. 作为查询条件
2. 查询较为频繁
3. 字段变化不频繁
4. 字段唯一性比较强
5. 组合索引最左匹配
6. 模糊查询'%xxx'不会使用索引
7. or 条件的所有查询条件都要有索引
8. 类型严格匹配才会使用索引
```

- 查看索引使用情况

```sql

show status like 'handler_read%'
-- 值越高 使用索引越高
show status like 'handler_read_key%'
show status like 'handler_read_rnd_next%'
```

- 数据导入小技巧

```
-- 导入前禁用索引
alter table <table_name> disable keys;

-- 导入后恢复索引
alter table <table_name> enable keys;

-- innodb
1. 事先按主键排序
2. set unique_checks=0; 关闭唯一检查
3. set autocommit=0;    关闭自动提交
```