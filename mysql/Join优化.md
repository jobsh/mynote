# Join优化

## MySQL中的几种Join语句

1. left join
   1. a left join b where b.key is null 
2. right join
   1. a right join b where a.key is null 
3. inner join
4. full outter join
   1. a full outer join where a.key is null or b.key is null
5. cross join：
   1. 只用cross join就是单纯的在做笛卡尔积，一般很少使用
   2. A cross join B 的得到的记录数 =  A 表记录数 * B表记录数
6. cross join + on = inner join

下图展示了 LEFT JOIN、RIGHT JOIN、INNER JOIN、OUTER JOIN 相关的 7 种用法。

![](http://typicture.loopcode.online/image/sql-join.png)

> join有三种算法，分别是Nested Loop Join，Hash join，Sort Merge Join。
>
> 但是MySQL官方文档中提到，MySQL只支持Nested Loop Join这一种join algorithm
>
> MySQL resolves all joins using a nested-loop join method. This means that MySQL reads a row from the first table, and then finds a matching row in the second table, the third table, and so on.
>
> **所以我们重点讨论的是Nested-Loop Join（NLJ）算法**

## Nested-Loop Join（NLJ）

### 驱动表和被驱动表

在学习NLJ相关算法时，我们先要弄明白两个概念——驱动表和被驱动表。

以 `select * from A left join B` 为例

驱动表：主动去连接的表，即上面的A表

被驱动表：被连接的表：即上面的B表

需要说明的是，对于left join 和 right join , 哪个作为驱动表是由sql本身的含义决定的，left join即左外连接，那么一定就是位于left join 左边的表作为驱动表，left join右边的表作为被驱动表；同样对于right join而言，就是right join右边的表作为驱动表，right join左边的表作为被驱动表。

这里值得注意的是inner join：mysql会对inner join类型自动进行优化，在没有强制声明使用哪张表作为驱动表时，mysql会优先使用记录数少的表作为驱动表。**所以使用 inner join 时，前面的表并不一定就是驱动表。**

其实Nested-Loop Join是代表一类所谓，这类算法的核心就是循环嵌套（至于为什么叫循环嵌套，我们可以从后面的执行流程中得知），实际上，NLJ有三种不同的表现形式：

* Simple Nested-Loop Join：SNLJ，简单嵌套循环连接
* Index Nested-Loop Join：INLJ，索引嵌套循环连接
* Block Nested-Loop Join：BNLJ，缓存块嵌套循环连接

之所以有这三种表现形式，是因为他们的逻辑大抵都是相同的，只是区别于是否使用了“辅助”—— 比如索引或者join buffer 缓存区。一般而言我们对于这两者都没有使用的Simple Nested-Loop Join，将其简称为NLP。下面我们对于Simple Nested-Loop Join就简称为NLJ了。

一个简单的**NLJ**算法一次一行循环地从**驱动表**中读取行，在这行数据中取到关联字段，根据关联字段在另一张表（**被驱动表**）里取出满足条件的行，然后**取出两张表的结果合集**。

我们试想一下，**如果在被驱动表中这个关联字段没有索引**，那么每次取出驱动表的关联字段在被驱动表查找对应的数据时，都会对被驱动表做一次全表扫描，成本是非常高的（比如驱动表数据量是 s，被驱动表数据量是 r，则扫描行数为 s * r）。

**因此 MySQL 在关联字段有索引时，才会使用 NLJ，我们也可以称这种情况下的NLJ为INLJ（Index Nested-Loop Join），如果没索引，就会使用 Block Nested-Loop Join**，等下会细说这个算法。我们先来看下在有索引情况的情况下，使用 Nested-Loop Join 的场景（称为：**Index Nested-Loop Join**）。

因为 MySQL 在关联字段有索引时，才会使用 NLJ，因此本文后面的内容所用到的 NLJ 都表示 Index Nested-Loop Join。

如何确定使用的是NLJ：一般 join 语句中，如果执行计划 Extra 中**未出现 Using join buffer**；则表示使用的 join 算法是 NLJ。

### NLJ 算法

```mysql
select * from t1 inner join t2 on t1.a = t2.a;       /* sql1 */
```

explain 分析执行计划：

![image-20210329150354509](http://typicture.loopcode.online/image/image-20210329150354509.png)

 sql执行流程：

1. 从表 t2 中读取一行数据；
2. 从第 1 步的数据中，取出关联字段 a，到表 t1 中查找；
3. 取出表 t1 中满足条件的行，跟 t2 中获取到的结果合并，作为结果返回给客户端；
4. 重复上面 3 步。

在这个过程中会读取 t2 表的所有数据，因此这里扫描了 100 行，然后遍历这 100 行数据中字段 a 的值，根据 t2 表中 a 的值索引扫描 t1 表中的对应行，这里也扫描了 100 行。因此整个过程扫描了 200 行。

### Block Nested-Loop Join 算法

删除索引之后

![image-20210329151321894](http://typicture.loopcode.online/image/image-20210329151321894.png)

发现多了Using where; Using join buffer (hash join)

>我们可以发现，Using join buffer后面还有Hash Join，hash join 是mysql8.0的新特性，在等值连接的情况下会被处罚，Hash join 不需要索引的支持。大多数情况下，hash join 比之前的 Block Nested-Loop 算法在没有索引时的等值连接更加高效。如果任何连接语句（ON）中没有使用等值连接条件，将不会采用 hash join 连接方式，将会采用性能更慢的 block nested loop 连接算法，这与 MySQL 8.0.18 之前版本中没有索引时的情况是一样的。
>
>Hash join 连接同样适用于不指定查询条件时的笛卡尔积
>
>默认配置时，MySQL 所有可能的情况下都会使用 hash join。同时提供了两种控制是否使用 hash join 的方法：
>
>在全局或者会话级别设置服务器系统变量 optimizer_switch 中的 hash_join=on 或者 hash_join=off 选项。默认为 hash_join=on。
>在语句级别为特定的连接指定优化器提示 HASH_JOIN 或者 NO_HASH_JOIN。
>
>PS：在MySQL 8.0.20和更高版本中，为了使用哈希联接，连接不再需要包含至少一个等值连接条件。

在这个过程中，对表 t1 和 t2 都做了一次全表扫描，因此扫描的总行数为10000(表 t1 的数据总量) + 100(表 t2 的数据总量) = 10100。并且 join_buffer 里的数据是无序的，因此对表 t1 中的每一行，都要做 100 次判断，所以内存中的判断次数是 100 * 10000= 100 万次。

如果被驱动表的关联字段没索引，为什么会选择使用 BNL 算法而不继续使用 Nested-Loop Join 呢？

在被驱动表的关联字段没索引的情况下，比如 sql2：

如果使用 Nested-Loop Join，那么扫描行数为 100 * 10000 = 100万次，这个是磁盘扫描。

如果使用 BNL，那么磁盘扫描是 100 + 10000=10100 次，在内存中判断 100 * 10000 = 100万次。

显然后者磁盘扫描的次数少很多，因此是更优的选择。因此对于 MySQL 的关联查询，如果被驱动表的关联字段没索引，会使用 BNL 算法。

**关于Join buffer做几点补充：**

1. 如果join_buffer_size空间足够大（可以放下所有要缓存的数据），那么此时**扫描的次数**就是两次，这两次分别是t1表数据全部放入缓存，t2表中的每行数据去对比join buffer中的t1表的每条数据。因此是两次

2. 如果join_buffer_size空间无法放下所有的要缓存的数据呢？这时候的扫描次数有如下公式：

   扫描行数 = S * C / join_buffer_size + 1

   S：缓存中的一行的数据大小

   C：需要缓存的所有数据

   join_buffer_size：join buffer可以提供的缓存空间大小

3. join buffer的使用条件：

   1. 连接类型是：ALL、index或range。
   2. 第一个nonconst table（非常量表）不会分配join buffer，即便连接类型是以上类型。
   3. join buffer只会缓存需要的字段而非整行数据。

4. 可通过join_buffer_size变量设置join_buffer大小。

5. 每个被缓存的join都会分配一个join buffer，一个查询可能拥有多个join buffer。

6. join buffer在执行连接前分配，在查询完成后释放。

### Batched Key Access 算法

在学了 NLJ 和 BNL 算法后，你是否有个疑问，如果把 NLJ 与 BNL 两种算法的一些优秀的思想结合，是否可行呢？

比如 NLJ 的关键思想是：被驱动表的关联字段有索引。

而 BNL 的关键思想是：把驱动表的数据批量提交一部分放到 join_buffer 中。

从 MySQL 5.6 开始，确实出现了这种集 NLJ 和 BNL 两种算法优点于一体的新算法：[Batched Key Access(BKA)](https://dev.mysql.com/doc/refman/5.7/en/bnl-bka-optimization.html)。

其原理是：

1. 将驱动表中相关列放入 join_buffer 中
2. 批量将关联字段的值发送到 Multi-Range Read(MRR) 接口
3. MRR 通过接收到的值，根据其对应的主键 ID 进行排序，然后再进行数据的读取和操作
4. 返回结果给客户端

> **这里补充下 MRR 相关知识：**
>
> 当表很大并且没有存储在缓存中时，使用辅助索引上的范围扫描读取行可能导致对表有很多随机访问。
>
> 而 Multi-Range Read 优化的设计思路是：查询辅助索引时，对查询结果先按照主键进行排序，并按照主键排序后的顺序，进行顺序查找，从而减少随机访问磁盘的次数。
>
> 使用 MRR 时，explain 输出的 Extra 列显示的是 Using MRR。
>
> optimizer_switch 中 mrr_cost_based 参数的值会影响 MRR。
>
> 如果 mrr_cost_based=on，表示优化器尝试在使用和不使用 MRR 之间进行基于成本的选择。
>
> 如果 mrr_cost_based=off，表示一直使用 MRR。
>
> 更多 MRR 信息请参考官方手册：https://dev.mysql.com/doc/refman/5.7/en/mrr-optimization.html。

下面尝试开启 BKA ：

```sql
set optimizer_switch='mrr=on,mrr_cost_based=off,batched_key_access=on';
```

这里对上面几个参数做下解释：

- mrr=on 开启 mrr
- mrr_cost_based=off 不需要优化器基于成本考虑使用还是不使用 MRR，也就是一直使用 MRR
- batched_key_access=on 开启 BKA

然后再看 sql1 的执行计划：

```sql
explain select * from t1 inner join t2 on t1.a = t2.a;
```

![图片描述](http://typicture.loopcode.online/image/5d3aae3a00015d6f16110183.png)在 Extra 字段中发现有 Using join buffer (Batched Key Access)，表示确实变成了 BKA 算法。

PS：mysql8.0中同时也已经不存在BKA，代替的是hash join

