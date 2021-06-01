# Order By调优

## 最好的做法是:利用索引避免排序

我们前面学过 B+Tree的存储结构，如果我们是根据索引字段排序的，那么就会直接按照索引的顺序直接返回，从而避免了再次排序。

本质上是：利用索引本身的有序性,让 MYSQL跳过排序过程。

<img src="http://typicture.loopcode.online/image/image-20200908150813283.png" alt="image-20200908150813283" style="zoom:80%;" />



## 如何判断order by使用了索引

我们重点关注explain执行计划中的type和extra字段

如果type为ALL （说明发生了全表扫描）并且 extra字段有明确指出使用了`Using filesort`（文件排序）,那么这种情况下就是没有使用索引，如下图：

![image-20210330221344245](http://typicture.loopcode.online/image/image-20210330221344245.png)

如果type为index或者range（索引范围扫描），Extra中没有出现提示`Using filesort`，那么说明order by使用了索引

## 哪些情况下 ORDER BY子句能用索引,哪些情况不能

总的来说，order by 如果想要使用索引，必须符合最左前缀匹配的原则，如果想要分析出order by是否会使用索引，那就必须要十分熟悉索引的数据结构，尤其是当order by多个字段排序时，或者order by与where配合使用时，很好的掌握组合索引的内部存储结构，对于索引的分析起着基础性的作用。想要了解组合索引的底层结构，可以查看《04联合索引的数据结构》。

我们来看几个例子，分析一下以下情况是否会使用索引：

employees表数据结构：

```mysql
CREATE TABLE `employees` (
  `emp_no` int(11) NOT NULL,
  `birth_date` date NOT NULL,
  `first_name` varchar(14) NOT NULL,
  `last_name` varchar(16) NOT NULL,
  `gender` enum('M','F') NOT NULL,
  `hire_date` date NOT NULL,
  PRIMARY KEY (`emp_no`)
) ENGINE=InnoDB DEFAULT COLLATE=utf8mb4_0900_ai_ci;
```



```mysql
explain select * from employees order by first_name, last_name
```

执行上面的语句我们发现，并没有使用索引，我们明明在first_name, last_name上创建了组合索引，但是仍然没有使用呢，这是由于我们排序的行数太多了，mysql优化器经过分析发现，还没有不用索引（全表扫面）的效率高呢，所以就没有使用索引。执行计划如下：

![image-20210330221344245](http://typicture.loopcode.online/image/image-20210330221344245.png)

如果我们在上面语句中限制返回的行数呢？

```mysql
explain select * from employees order by first_name, last_name limit 10;
```

这条语句加了limite 10我们可以看一下执行计划，发现type为index,使用了索引，并且Extra为Null，没有出现Using file sort

![image-20210330221713773](http://typicture.loopcode.online/image/image-20210330221713773.png)

* 当order by只有单个字段时，该字段创建了索引，就可能会使用索引进行排序，具体有mysql优化器分析成本去决定。

* 当order by排序字段有多个时：

  需要符合如下条件：

  1. order by字段的顺序必须符合最左前缀匹配原则。
  2. 需要注意的是，如果order by中如果有两种以上的索引，必须是一个组合索引的索引的列 + 最后一个位置是主键，因为二级索引存储的数据是主键，对于（first_name, last_name）索引来说，其实相当于（first_name, last_name, emp_no）
  3. order by字段的顺序必须一致

  ```mysql
  -- 1.可以使用索引，符合最左前缀原则，且都是升序排列
  explain select * from employees order by first_name, last_name limit 10;
  
  -- 2.无法利用索引避免排序（排序字段存在于多个索引中），原因见第二条，相当于破坏了最左前缀原则（跨过了last_name字段）
  explain select * from employees order by first_name, emp_no limit 10;
  
  -- 3.无法利用索引避免排序【升降序不一致】
  explain select * from employees order by first_name desc, last_name asc limit 10;
  ```

* 当order by 和where一起使用时，要明白sql的执行顺序时限where先选出符合条件的，再进行order by排序，因此 where 字段和 order by排序字段，放在一起要符合最左前缀匹配，如下：

```mysql
/*
 * 可以使用索引避免排序
 * [Bader,last_name1, emp_no]
 * [Bader,last_name2, emp_no]
 * [Bader,last_name3, emp_no]
 * [Bader,last_name4, emp_no]
 * [Bader,last_name5, emp_no]
 * ..
 */
explain
select *
from employees
where first_name = 'Bader'
order by last_name;

/*
 * 可以使用索引避免排序
 * ['Angel', lastname1, emp_no1]
 * ['Anni', lastname1, emp_no1]
 * ['Anz', lastname1, emp_no1]
 * ['Bader', lastname1, emp_no1]
 */
explain
select *
from employees
where first_name < 'Bader'
order by first_name;

/*
 * 可以使用索引避免排序
 */
explain
select *
from employees
where first_name = 'Bader'
  and last_name > 'Peng'
order by last_name;


/*
 * 无法利用索引避免排序【使用key_part1范围查询，使用key_part2排序】
 * ['Angel', lastname1, emp_no1]
 * ['Anni', lastname1, emp_no1]
 * ['Anz', lastname1, emp_no1]
 * ['Bader', lastname1, emp_no1]
 */
explain
select *
from employees
where first_name < 'Bader'
order by last_name;


```



## MySQL排序原理

### 排序模式1：rowid排序（常规排序）

又叫双路排序，又叫回表排序模式

1、从表中获取满足 WHERE条件的记录。
2、对于每条记录,将记录的主键及排序键(id, order_column)取出放入 sort buffer(由 sort_buffer_size控制)。

3、如果 sort buffere能存放所有满足条件的(id, order_column)，就直接在sort buffer中进行排序；

​	  否则 在sort buffer放满数据后，在sort buffer中排好序，然后**写到临时文件**。注意，排序永远都是在sort buffer中，即在内存中排序，内存中的排序使用的是快速排序算法。

4、若排序中产生了临时文件,需要利用**归并排序算法**对多个临时文件进行归并排序，从而保证整体的记录有序。

5、循环执行上述过程,直到所有满足条件的记录全部参与排序。

6、扫描排好序的(id, order_ column)对，并利用id去取SELECTF需要返回的其他字段。（这里需要回表，所以叫又做回表排序模式）

7、返回结果集。

特点：

- 看 sort buffer是否能存放结果集里面的所有(id, order_column),如果不满足，就会产生临时文件。

- 一次排序需要两次IO

  第二步:把(id, order column)扔到sort_ buffer

  第六步通过id去获取需要返回的其它字段，由于返回结果是按照order_colume排序的，所以主键id是乱序的，会存在随机IO的问题，MYSQL内部对这种情况做了个优化,在用ID取数数据前，会再按照ID排序并放到一个缓存里面，这个缓存的大小由read_rnd_buffer_size控制，接着再去读取记录，从而把随机IO转化为顺序IO。

### 排序模式2：全字段排序

这种排序方式是对模式1的一种有优化，并不会所有的排序都是用rowid排序，而是由max_length_for_sort_data来动态地决定使用哪一种排序方式，当order by 整个sql语句中出现的字段的总长度小于该值时，使用全字段排序，否则使用rowid排序（也就是说，有了全字段排序并不是意味着MySQL废弃了rowid排序）。这个参数mysql中默认256KB，我们一般不对其进行修改。

和rowid排序的步骤大体上是一样的，区别在于全字段排序是**直接取出所需要的全部字段**，放到sort buffer中（区别于rowid排序模式的步骤2），由于 sort buffer已经包含了查询需要的所有字段，因此**在sort buffer中排序完成后可直接返回，避免了二次回表**，当然如果sort buffer中无法一次性完成排序，也会用到临时文件，再对临时文件使用归并排序。

### 排序模式3：打包字段排序

全字段排序的一种优化，工作原理一样，但是会把返回的列数据进行压缩处理，例如：

varchar(255) "yes" 不打包的情况下占用255个字节，打包后只需要2 + 3个字节

我们对以上出现的参数进行一下汇总：+

| 变量                     | 作用                                                         |
| ------------------------ | ------------------------------------------------------------ |
| sort_buffer_size         | 控制sort buffer 的大小                                       |
| max_length_for_sort_data | 当order by 整个sql语句中出现的字段的总长度小于该值时，使用全字段排序，否则使用rowid排序 |
| read_rnd_buffer_size     | 按照主键排序后放到缓存区的大小、                             |
| max_sort_length          | 排序时最多截取多少个字节，调小该值有助于减少sort buffer的占用 |

下面一起来通过实验验证参数 max_length_for_sort_data 对排序模式的影响：

```sql
set session optimizer_trace="enabled=on",end_markers_in_json=on;

SET max_length_for_sort_data = 20;

select *
from employees
where first_name < 'Bader'
order by last_name;   

SELECT * FROM information_schema.OPTIMIZER_TRACE like '%Bader%'
```

OPTIMIZER_TRACE 结果中排序信息如下图：

![图片描述](http://typicture.loopcode.online/image/5d3aac0100016e5c07900149.png)

这里解释一下上面一些参数的含义：

- rows：预计扫描的行数
- examined_rows：参与排序的行
- number_of_tmp_files：使用临时文件的个数
- sort_buffer_size：sort_buffer 的大小
- sort_mode：排序模式

## ORDER BY调优原则与技巧

```mysql
-- sort buffer = 256k
-- 满足条件的(id, order_column) = 100m
-- [(10001,'Angel'),(88888,'Keeper'),(100001,'Zaker')] => file1
-- [(77777,'Jim'),(99999,'Lucy'),(5555, 'Hanmeimei')] => file2
-- [(10001,'Angel'),(5555, 'Hanmeimei'),(77777,'Jim'),(88888,'Keeper'),(99999,'Lucy'),(100001,'Zaker')]


SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;
SET optimizer_trace_offset=-30, optimizer_trace_limit=30;

select *
from employees
where first_name < 'Bader'
order by last_name;

select * from `information_schema`.OPTIMIZER_TRACE
where QUERY like '%Bader%';

show status like '%sort_merge_passes%'

-- 调优之前1.588	s

set sort_buffer_size = 1024*1024;
-- 调优之后168ms
```



## Order By调优实战

