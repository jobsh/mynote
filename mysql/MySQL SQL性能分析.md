# 1-11 MySQL SQL性能分析

# MySQL SQL性能分析

> **TIPS**
>
> - 本文基于MySQL 8.0

本文探讨如何深入SQL内部，去分析其性能，包括了三种方式：

- SHOW PROFILE
- INFORMATION_SCHEMA.PROFILING
- PERFORMANCE_SCHEMA

## SHOW PROFILE

SHOW PROFILE是MySQL的一个性能分析命令，可以跟踪SQL各种资源消耗。使用格式如下：

```mysql
SHOW PROFILE [type [, type] ... ]
    [FOR QUERY n]
    [LIMIT row_count [OFFSET offset]]

type: {
    ALL                     显示所有信息
  | BLOCK IO                显示阻塞的输入输出次数
  | CONTEXT SWITCHES				显示自愿及非自愿的上下文切换次数
  | CPU											显示用户与系统CPU使用时间
  | IPC											显示消息发送与接收的次数
  | MEMORY									显示内存相关的开销，目前未实现此功能
  | PAGE FAULTS							显示页错误相关开销信息
  | SOURCE									列出相应操作对应的函数名及其在源码中的位置(行)
  | SWAPS										显示swap交换次数
}
```

默认情况下，SHOW PROFILE只展示Status和Duration两列，如果想展示更多信息，可指定type。

使用步骤如下：

- 使用如下命令，查看是否支持SHOW PROFILE功能，yes标志支持。从MySQL 5.0.37开始，MySQL支持SHOW PROFILE。

  ```sql
  select @@have_profiling;
  ```

- 查看当前是否启用了SHOW PROFILE，0表示未启用，1表示已启用

  ```sql
  select @@profiling;
  ```

- 使用如下命令**为当前会话**开启或关闭性能分析，设成1表示开启，0表示关闭

  ```sql
  set profiling = 1;
  ```

- 使用SHOW PROFILES命令，可为最近发送的SQL语句做一个概要的性能分析。展示的条目数目由profiling_history_size会话变量控制，该变量的默认值为15。最大值为100。将值设置为0具有禁用分析的实际效果。

  ```sql
  -- 默认展示15条
  show profiles
  
  -- 使用profiling_history_size调整展示的条目数
  set profiling_history_size = 100;
  ```

- 使用show profile分析指定查询：

  ```sql
  mysql> SHOW PROFILES;
  +----------+----------+--------------------------+
  | Query_ID | Duration | Query                    |
  +----------+----------+--------------------------+
  |        0 | 0.000088 | SET PROFILING = 1        |
  |        1 | 0.000136 | DROP TABLE IF EXISTS t1  |
  |        2 | 0.011947 | CREATE TABLE t1 (id INT) |
  +----------+----------+--------------------------+
  3 rows in set (0.00 sec)
  
  mysql> SHOW PROFILE;
  +----------------------+----------+
  | Status               | Duration |
  +----------------------+----------+
  | checking permissions | 0.000040 |
  | creating table       | 0.000056 |
  | After create         | 0.011363 |
  | query end            | 0.000375 |
  | freeing items        | 0.000089 |
  | logging slow query   | 0.000019 |
  | cleaning up          | 0.000005 |
  +----------------------+----------+
  7 rows in set (0.00 sec)
  
  -- 默认情况下，只展示Status和Duration两列，如果想展示更多信息，可指定type。
  mysql> SHOW PROFILE FOR QUERY 1;
  +--------------------+----------+
  | Status             | Duration |
  +--------------------+----------+
  | query end          | 0.000107 |
  | freeing items      | 0.000008 |
  | logging slow query | 0.000015 |
  | cleaning up        | 0.000006 |
  +--------------------+----------+
  4 rows in set (0.00 sec)
  
  -- 展示CPU相关的开销
  mysql> SHOW PROFILE CPU FOR QUERY 2;
  +----------------------+----------+----------+------------+
  | Status               | Duration | CPU_user | CPU_system |
  +----------------------+----------+----------+------------+
  | checking permissions | 0.000040 | 0.000038 |   0.000002 |
  | creating table       | 0.000056 | 0.000028 |   0.000028 |
  | After create         | 0.011363 | 0.000217 |   0.001571 |
  | query end            | 0.000375 | 0.000013 |   0.000028 |
  | freeing items        | 0.000089 | 0.000010 |   0.000014 |
  | logging slow query   | 0.000019 | 0.000009 |   0.000010 |
  | cleaning up          | 0.000005 | 0.000003 |   0.000002 |
  +----------------------+----------+----------+------------+
  7 rows in set (0.00 sec)
  ```

- 分析完成后，记得关闭掉SHOW PROFILE功能：

  ```sql
  set profiling = 0;
  ```

> **TIPS**
>
> - MySQL官方文档声明SHOW PROFILE已被废弃，并建议使用Performance Schema作为替代品。
> - 在某些系统上，性能分析只有部分功能可用。比如，部分功能在Windows系统下无效（show profile使用了getrusage()这个API，而在Windows上将会返回false，因为Windows不支持这个API）；此外，性能分析是进程级的，而不是线程级的，这意味着其他线程的活动可能会影响到你看到的计时信息。

## INFORMATION_SCHEMA.PROFILING

INFORMATION_SCHEMA.PROFILING用来做性能分析。它的内容对应SHOW PROFILE和SHOW PROFILES 语句产生的信息。除非设置了 `set profiling = 1;` ，否则该表不会有任何数据。该表包括以下字段：

- QUERY_ID：语句的唯一标识
- SEQ：一个序号，展示具有相同QUERY_ID值的行的显示顺序
- STATE：分析状态
- DURATION：在这个状态下持续了多久（秒）
- CPU_USER，CPU_SYSTEM：用户和系统CPU使用情况（秒）
- CONTEXT_VOLUNTARY，CONTEXT_INVOLUNTARY：发生了多少自愿和非自愿的上下文转换
- BLOCK_OPS_IN，BLOCK_OPS_OUT：块输入和输出操作的数量
- MESSAGES_SENT，MESSAGES_RECEIVED：发送和接收的消息数
- PAGE_FAULTS_MAJOR，PAGE_FAULTS_MINOR：主要和次要的页错误信息
- SWAPS：发生了多少SWAP
- SOURCE_FUNCTION，SOURCE_FILE，SOURCE_LINE：当前状态是在源码的哪里执行的

> **TIPS**
>
> - SHOW PROFILE本质上使用的也是INFORMATION_SCHEMA.PROFILING表；
>
> - INFORMATION_SCHEMA.PROFILING表已被废弃，在未来可能会被删除。未来将可使用Performance Schema替代，详见 [“Query Profiling Using Performance Schema”](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-query-profiling.html)
>
> - 下面两个SQL是等价的：
>
>   ```sql
>   SHOW PROFILE FOR QUERY 2;
>   
>   SELECT STATE, FORMAT(DURATION, 6) AS DURATION
>   FROM INFORMATION_SCHEMA.PROFILING
>   WHERE QUERY_ID = 2 ORDER BY SEQ;
>   ```

## PERFORMANCE_SCHEMA

PERFORMANCE_SCHEMA是MySQL建议的性能分析方式，未来SHOW PROFILE、INFORMATION_SCHEMA.PROFILING都会废弃。据笔者研究，PERFORMANCE_SCHEMA在MySQL 5.6引入，因此，在MySQL 5.6及更高版本才能使用。可使用`SHOW VARIABLES LIKE 'performance_schema';` 查看启用情况，MySQL 5.7开始默认启用。

下面来用PERFORMANCE_SCHEMA去实现SHOW PROFILE类似的效果：

- 查看是否开启性能监控

  ```sql
  mysql> SELECT * FROM performance_schema.setup_actors;
  +------+------+------+---------+---------+
  | HOST | USER | ROLE | ENABLED | HISTORY |
  +------+------+------+---------+---------+
  | %    | %    | %    | YES     | YES     |
  +------+------+------+---------+---------+
  ```

  默认是开启的。

- 你也可以执行类似如下的SQL语句，只监控指定用户执行的SQL：

  ```sql
  mysql> UPDATE performance_schema.setup_actors
         SET ENABLED = 'NO', HISTORY = 'NO'
         WHERE HOST = '%' AND USER = '%';
  
  mysql> INSERT INTO performance_schema.setup_actors
         (HOST,USER,ROLE,ENABLED,HISTORY)
         VALUES('localhost','test_user','%','YES','YES');
  ```

  这样，就只会监控localhost机器上test_user用户发送过来的SQL。其他主机、其他用户发过来的SQL统统不监控。

- 执行如下SQL语句，开启相关监控项：

  ```sql
  mysql> UPDATE performance_schema.setup_instruments
         SET ENABLED = 'YES', TIMED = 'YES'
         WHERE NAME LIKE '%statement/%';
  
  mysql> UPDATE performance_schema.setup_instruments
         SET ENABLED = 'YES', TIMED = 'YES'
         WHERE NAME LIKE '%stage/%';
         
  mysql> UPDATE performance_schema.setup_consumers
         SET ENABLED = 'YES'
         WHERE NAME LIKE '%events_statements_%';
  
  mysql> UPDATE performance_schema.setup_consumers
         SET ENABLED = 'YES'
         WHERE NAME LIKE '%events_stages_%';
  ```

- 使用开启监控的用户，执行SQL语句，比如：

  ```sql
  mysql> SELECT * FROM employees.employees WHERE emp_no = 10001;
  +--------+------------+------------+-----------+--------+------------+
  | emp_no | birth_date | first_name | last_name | gender | hire_date |
  +--------+------------+------------+-----------+--------+------------+
  |  10001 | 1953-09-02 | Georgi     | Facello   | M      | 1986-06-26 |
  +--------+------------+------------+-----------+--------+------------+
  ```

- 执行如下SQL，获得语句的EVENT_ID。

  ```sql
  mysql> SELECT EVENT_ID, TRUNCATE(TIMER_WAIT/1000000000000,6) as Duration, SQL_TEXT
         FROM performance_schema.events_statements_history_long WHERE SQL_TEXT like '%10001%';
  +----------+----------+--------------------------------------------------------+
  | event_id | duration | sql_text                                               |
  +----------+----------+--------------------------------------------------------+
  |       31 | 0.028310 | SELECT * FROM employees.employees WHERE emp_no = 10001 |
  +----------+----------+--------------------------------------------------------+
  ```

  这一步类似于 SHOW PROFILES。

- 执行如下SQL语句做性能分析，这样就可以知道这条语句各种阶段的信息了。

  ```sql
  mysql> SELECT event_name AS Stage, TRUNCATE(TIMER_WAIT/1000000000000,6) AS Duration
         FROM performance_schema.events_stages_history_long WHERE NESTING_EVENT_ID=31;
  +--------------------------------+----------+
  | Stage                          | Duration |
  +--------------------------------+----------+
  | stage/sql/starting             | 0.000080 |
  | stage/sql/checking permissions | 0.000005 |
  | stage/sql/Opening tables       | 0.027759 |
  | stage/sql/init                 | 0.000052 |
  | stage/sql/System lock          | 0.000009 |
  | stage/sql/optimizing           | 0.000006 |
  | stage/sql/statistics           | 0.000082 |
  | stage/sql/preparing            | 0.000008 |
  | stage/sql/executing            | 0.000000 |
  | stage/sql/Sending data         | 0.000017 |
  | stage/sql/end                  | 0.000001 |
  | stage/sql/query end            | 0.000004 |
  | stage/sql/closing tables       | 0.000006 |
  | stage/sql/freeing items        | 0.000272 |
  | stage/sql/cleaning up          | 0.000001 |
  +--------------------------------+----------+
  ```

## 参考文档

- [SHOW PROFILE Statement](https://dev.mysql.com/doc/refman/8.0/en/show-profile.html)
- [The INFORMATION_SCHEMA PROFILING Table](https://dev.mysql.com/doc/refman/8.0/en/profiling-table.html)
- [Query Profiling Using Performance Schema](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-query-profiling.html)
- [配置详解 | performance_schema全方位介绍](http://blog.itpub.net/28218939/viewspace-2222168/)