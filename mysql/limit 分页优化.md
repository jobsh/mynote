# limit 分页优化

很多时候，业务上会有分页操作的需求，对应的 SQL 类似下面这条：

```mysql
-- 查询第一页的时候，花费92ms
-- 查询第30001页的时候，花费174ms
select * from employees limit 300000,10;
```

表示从表 employees中取出从 300000 行开始的 10 行记录。看似只查询了 10 条记录，实际这条 SQL 是先读取 30010 条记录，然后抛弃前 30000 条记录，然后读到后面 10 条想要的数据。因此要查询一张大表比较靠后的数据，执行效率是非常低的。本节内容就一起研究下，是否有办法去优化分页查询。

* 方案一：尽可能使用上覆盖索引

```mysql
-- 覆盖索引 (108ms)
select emp_no from employees limit 300000,10;
```

* **方案二：覆盖索引+join**

```mysql
-- 覆盖索引+join(109ms)
select *
from 
	employees e
inner join
     (select emp_no from employees limit 300000,10) t
using (emp_no);
```

* **方案三：覆盖索引+子查询**

```mysql
-- 覆盖索引+子查询（126ms）
select *
from employees
where emp_no >=
      (select emp_no from employees limit 300000,1)
limit 10;
```

- **方案四：范围查询+limit语句**

如果我们获得上次查询得到的最后一个id，可以使用此方法，前提是id必须是自增的

```mysql
select *
from employees
limit 10;

select *
from employees
where emp_no > 10010  -- 此emp_no可以让前端传过来
limit 10;
```

- **方案五：如果能获得起始主键值 & 结束主键值**

```mysql
select *
from employees
where emp_no between 20000 and 20010;
```

- **方案六：禁止传入过大的页码：以百度为例**

  百度如果传入过大的页码，直接转成第一页。





