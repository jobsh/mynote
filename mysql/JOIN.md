# JOIN

## 目标

* 熟练掌握各种join的区别
* 熟练掌握NLJ、BNLJ的原理
* 了解BKA、 HASH JOIN的原理 

## join分类

* left join
* right join
* full outer join
* inner join 
* cross join  — 求笛卡尔积(两表的乘积)
  * 如果 cross join带有in子句,就相当于 inner join

![image-20200906174428183](http://typicture.loopcode.online/image/image-20200906174428183.png)

## JOIN算法

### Nested-Loop Join (NLJ算法)

### Block Nested-Loop Join (BNLJ)：使用了join buffer

* 扫描行数计算公式

<img src="http://typicture.loopcode.online/image/image-20200906231824987.png" alt="image-20200906231824987" style="zoom:50%;" />

* join buffer使用条件
  * 连接类型是ALL、 index或 range
  * 第一个 nonconst table不会分配join buffer,即使类型是ALL或者index
  * join buffer只会缓存需要的字段,而非整行数据
  * 可通过join_ buffer_size变量设置join buffer大小
  * 每个能被缓存的join都会分配个 join buffer,一个查询可能拥有多个 join buffer
  * join buffer在执行联接之前会分配,在查询完成后释放。

### Batched Key Access Join (BKA)

* Mysql5.6 引入

* BKA的基石: Muti Range Read(MRR)

* MRR参数

  * optimizer_switch的子参数

    * mrr:是否开启mrr,on开启,of关闭	

      ```mysql
      set optimizer_switch = 'mrr=off';   -- 默认打开
      ```

      

    * mrr_cost_ based:表示是否要开启基于成本计算的MRR,on开启,of关闭	

      ```mysql
      set optimizer_switch = 'mrr_cost-based=off';  -- 默认打开
      ```

  * read_rnd_ buffer_size:指定mrr缓存大小

  * MRR核心：将随机IO转换成顺序IO,从而提升性能

* BKA流程

  ![image-20200906233930539](http://typicture.loopcode.online/image/image-20200906233930539.png)

* BKA参数

  * batched_key_ access : on开启, of关闭

### Hash-join算法（mysql 8.0.18引入）

join buffers缓存外部循环的hash表,内层循环遍历时到hash表匹配

MYSQL8.0.18オ引入,且有很多限制,比如不能作用于外连接,如 left join/ /right join等等。从8.0.20开始,限制少了很多,建议用8.0.20或更高版本

从 MYSQL8.0.18开始, hash join的 join buffer是递增分配的,这意味着,你可以为将join_ buffer_size设置得比较大。而在 MYSQL 8.0.18中
如果你使用了外连接,外连接没法用 hash join,此时join_ buffer_size会按照你设置的值直接分配内存。因此 join_buffer_size还是得谨慎设置。

从8.0.20开始,BNLJ已被删除了,用 hash join替代了BNLJ

## sql语句优化

```mysql
USE `foodie-shop-dev`
select * from users a right join orders b on a.id = b.user_id

select * from users a cross join orders b;
-- 156行 【笛卡尔连接】
select count(*)from users;
select count(*)from orders;
select 6*26;

-- 如果cross join带有on子句，就相当于inner join
select * from users a cross join orders b on a.id = b.user_id;


show variables like 'join_buffer_size';

set global join_buffer_size = 1024*1024*50;

-- Using join buffer (Block Nested Loop) => 使用了BNLJ
explain select * from users a left join orders b on a.id = b.user_id;

-- 可能会伴随大量的随机IO=> 数据按照主键排列，而不是from_date字段排列
/*
 * [1979-06-06,1980-06-06,(30000,1979-06-06)]
 * [1978-06-06,1979-06-06,(20000,1978-06-06)]
 * [1968-06-06,1969-06-06,(80000,1968-06-06)]
 * 按照主键排序之后：
 * [1978-06-06,1979-06-06,(20000,1978-06-06)]
 * [1979-06-06,1980-06-06,(30000,1979-06-06)]
 * [1968-06-06,1969-06-06,(80000,1968-06-06)]
 * -- 一旦开启MRR，会在extra里面展示Using MRR
 */
explain
select * from salaries
where from_date <='1980-01-01';

show variables like '%optimizer_switch%';

--  index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on,index_condition_pushdown=on,mrr=on,mrr_cost_based=on,block_nested_loop=on,batched_key_access=off,materialization=on,semijoin=on,loosescan=on,firstmatch=on,duplicateweedout=on,subquery_materialization_cost_based=on,use_index_extensions=on,condition_fanout_filter=on,derived_merge=on,use_invisible_indexes=off,skip_scan=on,hash_join=on
show variables like '%read_rnd_buffer_size%';

set optimizer_switch ='mrr_cost_based=off';

set optimizer_switch ='batched_key_access=on';

-- 当使用BKA的时候，会在extra里面展示Using join buffer (Batched Key Access)
explain
select * from salaries a,employees b
where a.from_date = b.birth_date;


-- MySQL 8.0.20 Using join buffer (hash join)
explain select * from users a left join orders b on a.id = b.user_id;
```

### 优化思路

* 用小表驱动大表
  * 一般无需人工考虑,关联查询优化器会自动选择最优的执行书序
  * 如果优化器抽风,可使用 STRAIGHT]OIN
  * 如果有 where条件,应当要能够使用索引,并尽可能地减少外层循环的数据量
* join的字段尽量创建索引
  * join字段的类型要保持一致
* 尽量减少扫描的行数( explain-rows)
  * 尽量控制在百万以内(经验之谈,仅供参考)
* 参与join的表不要太多
  * 阿里编程规约建议不超过3张 —— 业务装配
* 如果被驱动表的join字段用不了索引,且内存较为充足，可以考虑把 join buffer 设置得大一些

