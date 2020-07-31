---
layout: post
title:  "Troubleshooting MySQL through performance_schema"
date:   2020-7-15 20:48:57 +0800
categories: diagnosis
---

## 前言

虽然 MySQL 的实现比 Oracle 简单，但排查问题的工具远远没有 Oracle 强大。MySQL 的排查工具主要由系统库 `performance_schema` 提供，我们可以通过查询系统表的方法来排查性能问题。然而 MySQL 官网对 `performance_schema` 的介绍不够整体，用户往往不知道如何下手。

本文先搂清这些系统表的大致框架，然后从实践角度举例如何排查性能问题。

## 事件表

讲到 `performance_schema` 就绕不开“事件”这个概念。`performance_schema` 中有很多 `events_xxx` 表，查找性能问题主要从事件开始。

### 事件类型

MySQL 中的”事件“比我们常理解的事件更宽泛。它的事件主要有几类，粒度从大到小分别是：

* Transaction，即一个事务
* Statement，即执行一条 SQL 语句
* Stage，即 statement 中的每一个阶段
* Wait，即等待事件，例如 IO、锁等

其他事件例如 Error，本文不做介绍。

以上每种事件都有自己的 event id，它们可以根据 event id 相关联，后面会讲到。

### 设置记录事件

为了节省性能和内存，MySQL 并非默认记录所有的事件。在查看事件表前，需要先打开对应的开关。

开关不是由系统变量控制的，而是由系统表 `setup_instruments` 和 `setup_consumers` 控制的。

* `setup_instruments` 控制 MySQL 是否收集各类事件
* `setup_consumers` 控制各类事件表是否读取事件

我们可以从 `setup_instruments` 查看某类事件是否被打开。同时，直接 update 该表，就可以设置它的开关。

例如，通过 JDBC 执行 prepared statement 时，一般使用二进制协议，而默认只有文本协议 (statement/sql/execute_sql) 才被记录下来。要记录二进制协议的 prepared statement，就要使用如下语句打开该开关： 

```sql
UPDATE performance_schema.setup_instruments
	SET ENABLED = 'YES', TIMED = 'YES'
	WHERE NAME = 'statement/com/Execute';
```

否则在 `events_statements_xxx` 中查看不到使用二进制协议执行的 prepared statement。

### 事件历史表

Statement 等每种事件有多张表。常见的有：

* `events_xxx_current`：当前正在进行中的事件，每个线程占一行
* `events_xxx_history`：已经完成的事件，每个线程仅保存最近的 N 行
* `events_xxx_history_long`：已经完成的事件，保存全局最近的 N 行，不管是哪个线程产生的

其中 `events_xxx_history` 和 `events_xxx_history_long` 的大小由系统变量设置。

要查看这些表的数据，除了要打开 `setup_instruments` 中的事件开关，还要在 `setup_consumers` 中打开该表的开关。例如 `events_statements_history_long` 默认没有打开，可以用 update 语句打开：

```sql
UPDATE performance_schema.setup_consumers
	SET ENABLED = 'YES'
	WHERE NAME = 'events_statements_history_long';
```

### 事件间的关联

Transaction、Statement、Stage、Wait 事件是嵌套的关系，前者嵌套后者。它们可以根据历史表中的 `EVENT_ID` 和 `NESTING_EVENT_ID` 相关联。`NESTING_EVENT_ID` 字段是父事件的 `EVENT_ID`，例如 stage 的 `NESTING_EVENT_ID` 就是对应 statement 的 `EVENT_ID`。

但是，通常我们以 statement 作为排查的起点，而不是 transaction。在查找 statement 的瓶颈时，可以遵循以下步骤：

* 通过 `events_statements_history_long` 的 `EVENT_ID` 找到 statement 的事件 ID
* 通过上述 statement 事件 ID 在 `events_stages_history_long where NESTING_EVENT_ID={statement event id}` 找到 statement 的所有 stage，以及最耗时的 stage 的事件 ID
* 通过上述 stage 事件 ID 在 `events_waits_history_long where NESTING_EVENT_ID={stage event id}` 查看 stage 的所有等待事件，以及最耗时的等待事件

这样，就能找到整个 statement 中最耗时的等待事件。

### 事件汇总表

历史表中记录了每一个事件的详细信息，如果要查看每类事件的统计数据，例如耗时最长的 SQL、文件 IO 的平均等待时间等，可以查看事件汇总表。

每种事件有多张汇总表，分别以不同维度进行聚合。汇总表的命名方式是 `events_{event type}_summary_by_{agg column}`。例如常见的 `events_statements_summary_by_digest` 就是以 SQL 的 digest 聚合，这样可以查看每一类 SQL 的统计数据。

事件汇总表也有大小限制，也是由系统变量控制。

从汇总表中可以查找出耗时最高的 statement 或等待事件。但使用 SYS schema 中的视图更加便捷：

* 时间、内存等字段被格式化为字符串
* 字段数量被精简，字段名更易懂
* 按总耗时排序

例如几张常用的视图：

* `statement_analysis` 的基表是 `events_statements_summary_by_digest`，使用 `SUM_TIMER_WAIT` 排序，可以快速找到消耗资源最大的 SQL，改进业务
* `waits_global_by_latency` 的基表是 `events_waits_summary_global_by_event_name`，排除了 IDLE 事件，使用 `SUM_TIMER_WAIT` 排序，当很多 SQL 都很慢时，可以快速找到有问题的等待事件

## 诊断 SQL

有以上面的基本概念，下面演示如何诊断 SQL。

### 定位单条 SQL 的等待事件

要查看单条慢 SQL 的各项指标，一般使用 slow log。但在生产环境中，有时 MySQL 只开放了 3306 端口，并不能直接打开 slow log。这时可以查询 `events_statements_history` 来代替：

```sql
select * from performance_schema.events_statements_history 
	order by timer_wait desc limit 3\G
```

这样可以在最近的 N 条记录中找到耗时最高的 SQL。

既然能定位到 SQL，那么结合 stage 和 wait 历史表，可以定位到该语句最耗时的等待事件。

以下通过示例查看 `select * from test.t` 中的瓶颈在哪里：

```sql
mysql> select event_id, sys.format_time(timer_wait), sql_text from performance_schema.events_statements_history_long where sql_text like 'select * from test.t';
+----------+-----------------------------+----------------------+
| event_id | sys.format_time(timer_wait) | sql_text             |
+----------+-----------------------------+----------------------+
|      818 | 27.66 ms                    | select * from test.t |
+----------+-----------------------------+----------------------+
2 rows in set (0.00 sec)

mysql> select event_id, event_name, source, sys.format_time(timer_wait) from performance_schema.events_stages_history_long where nesting_event_id=818;
+----------+------------------------------------------------+----------------------------------+-----------------------------+
| event_id | event_name                                     | source                           | sys.format_time(timer_wait) |
+----------+------------------------------------------------+----------------------------------+-----------------------------+
|      819 | stage/sql/starting                             | init_net_server_extension.cc:101 | 58.00 us                    |
|      820 | stage/sql/Executing hook on transaction begin. | rpl_handler.cc:1119              | 1.00 us                     |
|      821 | stage/sql/starting                             | rpl_handler.cc:1121              | 5.00 us                     |
|      822 | stage/sql/checking permissions                 | sql_authorization.cc:2176        | 3.00 us                     |
|      823 | stage/sql/Opening tables                       | sql_base.cc:5591                 | 36.00 us                    |
|      824 | stage/sql/init                                 | sql_select.cc:677                | 4.00 us                     |
|      825 | stage/sql/System lock                          | lock.cc:331                      | 8.00 us                     |
|      828 | stage/sql/optimizing                           | sql_optimizer.cc:282             | 2.00 us                     |
|      829 | stage/sql/statistics                           | sql_optimizer.cc:502             | 14.00 us                    |
|      830 | stage/sql/preparing                            | sql_optimizer.cc:583             | 14.00 us                    |
|      831 | stage/sql/executing                            | sql_union.cc:1409                | 27.48 ms                    |
|      833 | stage/sql/end                                  | sql_select.cc:730                | 3.00 us                     |
|      834 | stage/sql/query end                            | sql_parse.cc:4606                | 2.00 us                     |
|      835 | stage/sql/waiting for handler commit           | handler.cc:1589                  | 6.00 us                     |
|      836 | stage/sql/closing tables                       | sql_parse.cc:4657                | 7.00 us                     |
|      837 | stage/sql/freeing items                        | sql_parse.cc:5330                | 22.00 us                    |
|      838 | stage/sql/cleaning up                          | sql_parse.cc:2184                | 1.00 us                     |
+----------+------------------------------------------------+----------------------------------+-----------------------------+
17 rows in set (0.00 sec)

mysql> select event_id, event_name, source, sys.format_time(timer_wait) from performance_schema.events_waits_history_long where nesting_event_id=831;
+----------+---------------------------+-----------------+-----------------------------+
| event_id | event_name                | source          | sys.format_time(timer_wait) |
+----------+---------------------------+-----------------+-----------------------------+
|      832 | wait/io/table/sql/handler | handler.cc:2965 | 26.20 ms                    |
+----------+---------------------------+-----------------+-----------------------------+
1 row in set (0.00 sec)
```

在上面的例子中，SQL 的总耗时是 27.66 ms，其中 `wait/io/table/sql/handler` 事件的耗时是 26.20 ms，所以瓶颈在 table IO 上。

### 定位系统全局等待事件

对单条 SQL 的分析通常不具代表性。既然有汇总表，那么可以通过汇总表找出整个系统中最慢的 SQL 以及事件。

通过 `statement_analysis` 找到总耗时最高的 SQL：

```sql
select * from sys.statement_analysis limit 3\G
```

另外，可以直接找出全局最耗时的等待事件：

```sql
mysql> select * from sys.waits_global_by_latency limit 3;
+--------------------------------------+-------------+---------------+-------------+-------------+
| events                               | total       | total_latency | avg_latency | max_latency |
+--------------------------------------+-------------+---------------+-------------+-------------+
| wait/io/table/sql/handler            | 14876385000 | 12.74 w       | 517.93 us   | 4.18 m      |
| wait/io/file/innodb/innodb_data_file |   497506557 | 15.09 h       | 109.16 us   | 2.00 s      |
| wait/io/file/sql/binlog              |    28923151 | 7.34 h        | 913.29 us   | 2.30 m      |
+--------------------------------------+-------------+---------------+-------------+-------------+
3 rows in set (0.18 sec)
```

这里 table IO 是最耗时的操作。

### 定位最近一段时间的瓶颈

汇总表只能代表整个系统运行至今的状况，并不能体现最近的状况。前面提到了，每类事件都有历史表，我们可以通过 `EVENT_ID` 的关联关系，定位最近一段时间最耗时的等待事件。

例如，可以先将 statement 历史表与 stage 历史表关联，找出单个 stage 耗时高的 SQL：

```sql
SELECT stages.event_name, stmts.sql_text, stages.timer_wait*1e-12
	FROM performance_schema.events_stages_history_long stages
		JOIN performance_schema.events_statements_history_long stmts
		ON stages.nesting_event_id = stmts.event_id
	ORDER BY stages.timer_wait DESC LIMIT 3\G
```

类似地，可以再用 stage 表和 wait 表关联。

然而在高并发时，历史表更新过快，必须调大历史表的容量。另外，stage 和 wait 的历史表默认关闭，需要权限才能打开。所以，更实用的方法是在业务运行时截取一小段时间，比较前后的汇总数据。

例如，上面已经得出 table IO 耗时高，可以比较 10s 内执行 SQL 的耗时和 table IO 的耗时：

```sql
mysql> select count_read, sum_timer_read*1e-12 from performance_schema.table_io_waits_summary_by_table where object_name='t' into @count_table_io, @time_table_io;
Query OK, 1 row affected (0.14 sec)

mysql> select sum(count_star), sum(sum_timer_wait)*1e-12 from performance_schema.events_statements_summary_by_digest where object_name='t'  into @count_sql, @time_sql;
Query OK, 1 row affected (0.10 sec)

mysql> do sleep(10);
Query OK, 0 rows affected (10.17 sec)

mysql> select count_read-@count_table_io, sum_timer_read*1e-12-@time_table_io from performance_schema.table_io_waits_summary_by_table where object_name='t';
+----------------------------+-------------------------------------+
| count_read-@count_table_io | sum_timer_read*1e-12-@time_table_io |
+----------------------------+-------------------------------------+
|                      57950 |                    75.8137493748218 |
+----------------------------+-------------------------------------+
1 row in set (0.11 sec)

mysql> select sum(count_star)-@count_sql, sum(sum_timer_wait)*1e-12-@time_sql from performance_schema.events_statements_summary_by_digest where object_name='t';
+----------------------------+-------------------------------------+
| sum(count_star)-@count_sql | sum(sum_timer_wait)*1e-12-@time_sql |
+----------------------------+-------------------------------------+
|           159.000000000000 |                     77.731274286285 |
+----------------------------+-------------------------------------+
1 row in set (0.11 sec)
```

这个例子中，10s 内的 SQL 耗时是 77s（因为有并发，所以大于 10s），其中 table IO 就占了 75s。

从次数上看，执行 159 条 SQL 就执行了 57K 次 table IO，说明平均每条 SQL 访问了 358 条数据。75 秒执行 57K 次 table IO，那么平均每次耗时 1.3ms。

用类似的方法还可以估算出 QPS。

## 诊断 IO

执行过程中最常见的瓶颈是 IO 和锁。本文以诊断 IO 为例，展现循序渐进找出问题的方法。

针对 IO 的诊断，有比 `events_waits_xxx` 更常用的几张系统表。

### table_io_waits_summary_xxx

`table_io_waits_summary_by_table` 由等待事件 `wait/io/table/sql/handler` 聚合而来，通过它能查看每张表的 table IO 的次数、耗时。这里的 table IO 是指访问表及索引中的数据，既包括访问硬盘，又包括访问 buffer pool。

`table_io_waits_summary_by_table` 里的统计字段比 `events_waits_xxx` 更详细：

* `XXX_WAIT`：所有 IO 之和
* `XXX_READ` / `XXX_FETCH`：读
* `XXX_WRITE`：所有写 IO 之和
* `XXX_INSERT`：插入
* `XXX_UPDATE`：更新
* `XXX_DELETE`：删除

这样，在找 SELECT 语句的瓶颈时，就只用查看 `XXX_READ` 字段，更有针对性。

`table_io_waits_summary_by_index_usage` 包含每个索引的逻辑 IO 的次数、耗时。`index_name` 字段就是索引名，如果是 Primary key，该字段就是 `PRIMARY`。所以，`table_io_waits_summary_by_index_usage` 的 IO 次数之和就是 `table_io_waits_summary_by_table` 的 IO 次数。

视图 `sys.schema_table_statistics` 的基表是 `table_io_waits_summary_by_table`，视图 `sys.schema_index_statistics` 的基表是 `table_io_waits_summary_by_index_usage`。

### file_summary_by_xxx

`table_io_waits_summary_xxx` 里包含的统计数据是 table IO，而 `file_summary_by_xxx` 里包含 file IO，即硬盘的存取。它由等待事件 `wait/io/file/xxx` 聚合而来。

`wait/io/file/xxx` 包含一组事件，其中最常见的是 `wait/io/file/innodb/innodb_data_file`，即访问 InnoDB 数据文件。

`file_summary_by_xxx` 统计的数据与 `table_io_waits_summary_xxx` 稍有差别：

* 没有 FETCH、INSERT、UPDATE、DELETE，只有 READ、WRITE
* `XXX_MISC` 代表读、写以外的文件操作，例如打开、关闭文件
* 包含读、写的字节数统计

视图 `sys.io_global_by_file_by_latency` 和 `sys.io_global_by_file_by_bytes` 基于表 `file_summary_by_instance`，分别可以查看每个文件的 IO 耗时及读写数据量。

通过对比 `table_io_waits_summary_by_table` 和 `file_summary_by_instance` 的中读写耗时，我们可以得知 file IO 占 table IO 的耗时比例。

```sql
mysql> select count_read, sys.format_time(sum_timer_read) from performance_schema.table_io_waits_summary_by_table where object_name='t';
+------------+---------------------------------+
| count_read | sys.format_time(sum_timer_read) |
+------------+---------------------------------+
|    1441810 | 2.34 s                          |
+------------+---------------------------------+
1 row in set (0.00 sec)

mysql> select file_name, count_read, sys.format_time(sum_timer_read) from performance_schema.file_summary_by_instance where file_name like '%t.ibd';
+---------------------------------+------------+---------------------------------+
| file_name                       | count_read | sys.format_time(sum_timer_read) |
+---------------------------------+------------+---------------------------------+
| /usr/local/var/mysql/test/t.ibd |          6 | 5.03 us                         |
+---------------------------------+------------+---------------------------------+
1 row in set (0.00 sec)
```

在这个例子中，table read IO 总耗时 2.34s，而 file read IO 总耗时仅 5.03us，显然 file IO 不是读取中的瓶颈。

### Buffer pool 命中率

对比 table IO 和 file IO 的次数之后，就可以算出该表的缓冲池命中率。

要知道 InnoDB 总的缓冲池命中率，需要从 global status 中查询：

```sql
show global status like 'innodb%read%';
```

或者用 `SHOW ENGINE INNODB STATUS`，结果更清晰。以下是从结果中截取的两段：

```sql
--------
FILE I/O
--------
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
40706 OS file reads, 287254 OS file writes, 135910 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s

----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137363456
Dictionary memory allocated 512425
Buffer pool size   8191
Free buffers       1024
Database pages     7150
Old database pages 2619
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 59089, not young 139969
0.00 youngs/s, 0.00 non-youngs/s
Pages read 40684, created 17215, written 79957
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 7150, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```

可以看到，file IO 的频率是 0，并且 buffer pool 的命中率是 100%。

### 查找 table IO 的瓶颈

Table IO 是一个比较特殊的等待事件，它会嵌套其他等待事件，例如 file IO。如果 table IO 耗时高而 file IO 的占比并不多，可以进一步查找 table IO 内的其他等待事件。

通过 `events_waits_history_long`，可以查询等待事件之间的嵌套关系。同样的，嵌套关系通过 `EVENT_ID` 和 `NESTING_EVENT_ID` 关联。

例如，查询 table IO 中嵌套的等待事件，以及每次事件占 table IO 的耗时比例：

```sql
select wait.event_name, sum(wait.timer_wait)*1e-12 from events_waits_history_long io, events_waits_history_long wait where io.event_name='wait/io/table/sql/handler' and wait.nesting_event_id=io.event_id group by wait.event_name;
+--------------------------------------+----------------------------+
| event_name                           | sum(wait.timer_wait)*1e-12 |
+--------------------------------------+----------------------------+
| wait/io/file/innodb/innodb_data_file |         3.5929532760000003 |
+--------------------------------------+----------------------------+
1 row in set (0.01 sec)
```

`wait/io/file/innodb/innodb_data_file` 是读写 InnoDB 数据文件。这个例子中，table IO 中除了 file IO 没有其他事件。

### 使用 strace 查看系统调用

Table IO 看不出瓶颈时，可以通过 strace 查看期间 MySQL 执行了哪些系统调用。但是 strace 要求能登上 MySQL 的宿主机执行 shell 命令。

执行以下命令，让 strace attach 到 mysqld 进程：

```
strace -o ./strace.log -T -tt -f -p `pidof mysqld`
```

其中常用参数：

- -T：显示每个系统调用所耗费的时间，其时间开销在输出行最右侧的尖括号内
- -tt：在每行输出前添加绝对时间戳信息，精确到微秒级

打印的结果中，包含每次系统调用的名字、参数、返回值、耗时等。例如，结果中主要包含两类系统调用：

```
pread64(33, "datadatadatadata"..., 16384, 6126534) = 16384 <0.000914>

sendto(32, "datadatadatadata"..., 16384, MSG_DONTWAIT, NULL, 0) = 16384 <0.000853>
```

`pread64` 是从文件读数据，一次读 16K 数据，每次耗时 914us。文件描述符是 33，可以通过 ls 查看对应于哪个文件：

```
ls -lrt /proc/`pidof mysqld`/fd/33
```

`sendto` 是向客户端返回数据，buffer 大小是 16K，每次耗时 853us。该例子中 `pread64` 和 `sendto` 交叉出现，即 MySQL 边处理数据边发结果，数据写满 16K 后就发。

由于 914us 的 `pread64` 与上面的 1.3ms 的 table IO 几乎匹配，所以大部分时间花在系统调用上。

## 总结

本文从 `performance_schema` 常见的一些系统表入手，介绍了系统表的结构以及它们之间的联系。然后举例展示了如何定位到 SQL、定位有问题的等待事件、定位等待事件慢的原因。

总结起来，有这些最佳实践：

- 各类事件表通过 `EVENT_ID` 和 `NESTING_EVENT_ID` 相互关联
- 历史表可用于查询最近一段时间耗时最高的等待事件
- 截取一段时间内的汇总数据，可以算出等待事件的耗时比例
- 对比 table IO 和 file IO 的耗时，可以确定访问磁盘的耗时比例
- 通过 strace 查看系统调用，可以快速发现系统级问题