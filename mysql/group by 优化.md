# group by 优化

## mysql处理group by语句的三种方式

​	性能一次递减

1. 松散索引扫描( Loose Index Scan)
2. 紧凑索引扫描( Tight Index Scan)
3. 临时表( Temporary table)

## 松散索引扫描

无需扫描满足条件的所有索引键即可返回结果

使用组合索引（emp_no, salary）注意emp_no在前，salary在后。如果（salary, emp_no）则不使用松散索引扫描。

```mysql
-- 查询每个员工拿到过的最少的工资是多少[比惨大会]
/*
 * 分析这条SQL如何执行：
 * [emp_no, salary] 组合索引
 * [10001,50000]
 * [10001,51000]
 * ...
 * [10002,30000]
 * [10002,32000]
 * ...
 * 1. 先扫描emp_no = 10001的数据，并计算出最小的salary是多少，[10001,50000]
 * 2. 扫描emp_no = 10002，并计算出最小的salary是多少，[10002,30000]
 * 3. 遍历出每个员工的最小薪资，并返回
 ===
 * 改进：（松散索引扫描）
 * 1. 先扫描emp_no = 10001的数据，取出第一条 => 就是这个员工工资最小的数据
 * 2. 直接跳过所有的emp_no = 10001的数据，继续扫描emp_no = 10002的数据，取第一条
 * 3. 以此类推
 * explain的extra展示Using index for group-by => 说明使用了松散索引扫描
 */
explain
select emp_no, min(salary)
from salaries
group by emp_no
```

### 使用条件

* 查询作用在单张表上
* GROUP指定的所有字段要符合最左前缀原则，且没有其他字段，比如有索引 index(c1, c2, c3)，如果 GROUP BY c1, c2则可以使用松
  散索引扫描；但 GROUP BY c2, c3、 GROUP BY c1, c2, C4则不能使用
* 如果存在聚合函数**,只支持MIN() / MAX()**,并且如果同时使用了MIN()和MAX(),则必须作用在同一个字段。聚合函数作用的字段必须在索引中,并且要**紧跟 GROUP BY所指定的字段**
  * 比如有索引` index(c1,c2,c3)`， `SELECT C1,c2,MIN(c3),MAX(c3) FROM t1 GROUP BY c1,c2`可使用松散索引扫描
* 如果查询中存在除 GROUP BY指定的列以外的其他部分,则必须以常量的形式出现
  * SELECT c1,c3 FROM t1 GROUP BY c1,c2：select的字段中有c3字段，而c3字段不再group by子句中，所以不能使用
  * SELECT c1,C3 FROM TLWHERE **c3=3** GROUP BY c1,c2：可以使用松散索引扫描（c3此时变成了常量）。
* 索引必须索引整个字段的值,不能是前缀索引
  * 比如有字段**c1 VARCHAR(20),**但如果该字段使用的是前缀索引 **index(c1(10)）**而不是 **index(c1)**，则**无法使用松散索引描**

### 能使用松散索引扫描的SQLー览

<img src="http://typicture.loopcode.online/image/image-20200909200218742.png" alt="image-20200909200218742" style="zoom:70%;" />

### 不能使用松散索引扫描的SQL一览

<img src="http://typicture.loopcode.online/image/image-20200909200454127.png" alt="image-20200909200454127" style="zoom:74%;" />

### 特定聚合函数用法能用上松散索引扫描的条件

<img src="http://typicture.loopcode.online/image/image-20200909201615387.png" alt="image-20200909201615387" style="zoom:67%;" />

假设有 index(c1,c2,c3) 作用在表t1(c1,c2,c3,c4)上，下面这些SQL都能使用松散索引扫描:

```mysql
SELECT COUNT(DISTINCT C1), SUM(DISTINCT C1)FROM t1
SELECT COUNT(DISTINCT Cl, C2), COUNT(DISTINCT C2, C1)FROM t1
```

## 紧凑索引扫描

* 需要扫描满足条件的所有索引键才能返回结果
* 性能一般比松散索引扫描差，但一般都可接受

```mysql
-- 使用了紧凑索引扫描，explain-extra中没有明显的标志。
explain
select emp_no, sum(salary)
from salaries
group by emp_no;
```



## 临时表

紧凑索引扫描也没有办法使用的话，MYSQL将会读取需要的数据，并创建一个临时表，用临时表实现 GROUP BY操作

```mysql
-- 一旦出现临时表，将会在explain-extra显示Using temporary
explain
select max(hire_date)
from employees
group by hire_date;
```

## GROUP BY 语句优化

如果 GROUP BY使用了临时表，想办法用上松散索引扫描或紧凑索引扫描

## DISTINCT优化

DISTINCT是在 GROUP BY操作之后,每组只取1条

和GROUP BY优化思路一样