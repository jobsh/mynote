# count优化一

## 数据准备

```mysql
create table user_test_count
(
    id       int primary key not null auto_increment,
    name     varchar(45),
    age      int,
    email    varchar(60),
    birthday date
) engine 'innodb';

insert into user_test_count (id, name, age, email, birthday)
values (1, '张三', 20, 'zhangsan@imooc.com', '2000-01-01');
insert into user_test_count (id, name, age, email, birthday)
values (2, '李四', 30, 'lisi@imooc.com', '1990-01-01');
insert into user_test_count (id, name, age, email, birthday)
values (3, '王五', 40, null, null);
insert into user_test_count (id, name, age, email, birthday)
values (4, '大目', 18, null, null);
```

## count相关知识点

```mysql
/*
 * 1. 当没有非主键索引时，会使用主键索引
 * 2. 如果存在非主键索引的话，会使用非主键索引 user_test_count_email_index 243
 * 3. 如果存在多个非主键索引，会使用一个最小的非主键索引
 * 为什么？
 * -innodb非主键索引：叶子节点存储的是：索引+主键
 * 主键索引叶子节点：主键+表数据
 * 在1个page里面，非主键索引可以存储更多的条目，对于一张表，1000000数据，
 * 使用非主键索引 扫描page 100 ，主键索引 500
 */
explain select count(*)
        from user_test_count;

/*
 * count(字段)只会针对该字段统计，使用这个字段上面的索引（如果有的话）
 * count(字段)会排除掉该字段值为null的行
 * count(*)不会排除
 */
explain select count(email)
        from user_test_count;

/*
 * count(*)和count(1)没有区别，详见官方文档：https://dev.mysql.com/doc/refman/8.0/en/group-by-functions.html#function_count
 * 对于MyISAM引擎，如果count(*)没有where条件(形如 select count(*) from 表名)，查询会非常的快
 * 对于MySQL 8.0.13，InnoDB引擎，如果count(*)没有where条件(形如 select count(*) from 表名)，查询也会被优化，性能有所提升
 */
explain select count(1)
        from user_test_count;
```

## count优化思路

```mysql
select count(*)
from salaries;
-- 120ms
-- innodb 版本8.0.18 > 8.0.13，可以针对无条件的count语句去优化
show create table salaries;
select version();
-- mysql 5.6，相同数据量，相同SQL需要花费841ms


explain
select count(*)
from salaries;

-- 方案1：创建一个更小的非主键索引
-- 方案2：把数据库引擎换成MyISAM => 实际项目用的很少，一般不会修改数据库引擎
-- 方案3：汇总表 table[table_name, count] => employees, 2000000
-- 好处：结果比较准确 table[emp_no, count]
-- 缺点：增加了维护的成本
-- 方案4：sql_calc_found_rows
select *
from salaries
limit 0,10;
select count(*)
from salaries;

-- 在做完本条查询之后，自动地去执行COUNT
select sql_calc_found_rows *
from salaries
limit 0,10;
select found_rows() as salary_count;
-- 缺点：mysql 8.0.17已经废弃这种用法，未来会被删除
-- 注意点：需要在MYSQL终端执行，IDEA无法正常返回结果。

-- 方案5：缓存 select count(*) from salaries; 存放到缓存
-- 优点：性能比较高；结果比较准确，有误差但是比较小（除非在缓存更新的期间，新增或者删除了大量数据）
-- 缺点：引入了额外的组件，增加了架构的复杂度

-- 方案6：information_schema.tables
select *
from `information_schema`.TABLES
where TABLE_SCHEMA = 'employees'
  and TABLE_NAME = 'salaries';
-- 好处：不操作salaries表，不论salaries有多少数据，都可以迅速地返回结果
-- 缺点：估算值，并不是准确值

-- 方案7：
show table status where Name = 'salaries';
-- 好处：不操作salaries表，不论salaries有多少数据，都可以迅速地返回结果
-- 缺点：估算值，并不是准确值

-- 方案8：
explain
select *
from salaries;
-- 好处：不操作salaries表，不论salaries有多少数据，都可以迅速地返回结果
-- 缺点：估算值，并不是准确值

explain
select count(*)
from salaries
where emp_no > 10010;

-- 799ms
select min(emp_no)
from salaries;

explain select count(*) - (select count(*) from salaries where emp_no <= 10010)
        from salaries;
```

