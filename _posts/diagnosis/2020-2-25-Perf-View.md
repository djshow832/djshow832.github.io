---
layout: post
title:  "SQL Performance Views"
date:   2020-2-25 20:48:57 +0800
categories: diagnosis
---

# 前言

为了排查 SQL 的性能问题，一般数据库都会提供 SQL 性能视图。SQL 性能视图包含每种 SQL 的执行耗时、执行次数等信息，用于定位问题 SQL 以及原因。

下面调研了几种数据库的 SQL 性能视图。

# CockroachDB

CockroachDB 通过 [Statements Page](https://www.cockroachlabs.com/docs/stable/admin-ui-statements-page.html) 把统计信息以图表的形式展示在页面上，有助于定位频繁执行或延迟较高的 SQL。

它只统计一段时间内的 SQL，定时清空数据，默认的更新间隔是 1 小时。所以在清空之后查询，数据会很少。

以下是所有 SQL 的展示页面：

![crdb-statements-page](/media/diagnosis/crdb-statements-page.png)

以下是单条 SQL 的详情页面：

![crdb-details-page](/media/diagnosis/crdb-statements-details-page.png)

# OceanBase

OceanBase 提供了一系列的[系统表和视图](https://www.yuque.com/oceanbase-site/oceanbase/obinttable)。[SQL 监控](https://oceanbase.alipay.com/docs/oceanbase/OceanBase%E7%AE%A1%E7%90%86%E5%91%98%E6%89%8B%E5%86%8C/%E7%AC%AC%E4%B8%80%E9%83%A8%E5%88%86%20OceanBase%E5%9F%BA%E7%A1%80%E7%AE%A1%E7%90%86/dv8yhg)主要靠用户查询 `v$sql` 和 `v$sql_audit` 这两张视图。

[v$sql](https://oceanbase.alipay.com/docs/oceanbase/%E5%8F%82%E8%80%83%E7%B1%BB/%E5%86%85%E9%83%A8%E8%A1%A8/tntwp4) 以 plan 分类，汇总了单个 plan 多次执行的统计信息，包括来源 IP、SQL 文本、平均耗时等。

`gv$sql` 汇总了所有 OBServer 的 `v$sql` 信息。

[v$sql_audit](https://oceanbase.alipay.com/docs/oceanbase2.1/OceanBase%20SQL%E8%B0%83%E4%BC%98%E6%8C%87%E5%8D%97/SQL%E6%89%A7%E8%A1%8C%E6%80%A7%E8%83%BD%E7%9B%91%E6%8E%A7/sql_audit) 是基于虚拟表 `__all_virtual_sql_audit` 的视图，该虚拟表是一张内存表。它没有对 SQL 进行分类，记录了每一条 SQL 的来源 IP、影响行数、各阶段耗时、消耗内存等信息。

`gv$sql_audit` 汇总了所有 OBServer 的 `v$sql_audit` 信息。

查询 `v$sql_audit`：

![ob-sql-audit](/media/diagnosis/ob-sql-audit.png)

# Oracle

Oracle 实时统计监控数据，用 [Dynamic Performance View](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/about-dynamic-performance-views.html) 来查询。这些视图在 catalog.sql 中定义，用户必须运行 catalog.sql 才能创建这些视图。

V$ 开头的视图仅查询单个实例。在 Oracle Real Application Clusters，还有 GV$ 开头的视图，可以查询所有实例，同时 GV$ 视图也多出一列 `INST_ID`。

SQL 性能视图是 `V$SQL` 开头的视图，共 34 个，以下列举典型的几个。

* [V$SQL](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SQL.html) 按 `SQL_ID` 进行分类，也即一个 cursor 占一行。但是用户可以按 `PLAN_HASH_VALUE` 列进行 GROUP BY。
* [V$SQLAREA](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SQLAREA.html) 按 SQL 文本进行分类。
* [V$SQLAREA_PLAN_HASH](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SQLAREA_PLAN_HASH.html) 把 `V$SQL` 按 `SQL_ID` 和 `PLAN_HASH_VALUE` 聚合。
* [V$SQL_MONITOR](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-SQL_MONITOR.html) 类的视图，没有按 SQL 分类，且只监控部分 SQL。

`V$SQL_MONITOR` 满足以下一个条件就会被监控：

* SQL 以并行模式运行
* 单次执行消耗了 5 秒以上的 CPU 时间或 IO 时间
* SQL 中包含 `/*+ MONITOR */`

Oracle 没有把 SQL 规一化再进行分类的视图。它的视图都是从 SQL Area 里取的。

另外 Oracle 还提供了可视化的工具 [SQL Monitor](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/monitoring-database-operations.html) 。SQL Monitor 中可以显示 SQL 监控信息，下图是显示 `V$SQL_MONITOR` 的信息：

![oracle-sql-monitor](/media/diagnosis/oracle-sql-monitor.jpg)

点击 ID，就可以查看单条 SQL 的详细信息，甚至包含执行计划每阶段的监控：

![oracle-sql-monitor-details](/media/diagnosis/oracle-sql-monitor-details.gif)

# MySQL

MySQL 5.7.7 及以上版本提供了 sys schema，其中包含了一系列封装 performance_schema 的[视图](https://dev.mysql.com/doc/refman/5.7/en/sys-schema-views.html)，更便于查找问题。

其中 SQL 性能视图是 [statement_analysis](https://dev.mysql.com/doc/refman/5.7/en/sys-statement-analysis.html) （或 `x$statement_analysis`）。它按 SQL digest 分类，统计每一类 SQL 的执行次数、平均延迟、影响行数等。其中 SQL digest 是把 SQL 规一化之后生成的指纹。

`statement_analysis` 是基于 [performance_schema.events_statement_summary_by_digest](https://dev.mysql.com/doc/refman/5.7/en/statement-summary-tables.html) 来计算的。而 `events_statements_summary_by_digest` 的数据来源于 statement history，statement history 默认保留最近的 10000 条 SQL。在查询 SQL 性能视图时，会实时对 statement history 进行聚合。

此外，还有其他一些类似的视图：

* [statements_with_errors_or_warnings](https://dev.mysql.com/doc/refman/5.7/en/sys-statements-with-errors-or-warnings.html)
* [statements_with_full_table_scans](https://dev.mysql.com/doc/refman/5.7/en/sys-statements-with-full-table-scans.html)
* [statements_with_runtimes_in_95th_percentile](https://dev.mysql.com/doc/refman/5.7/en/sys-statements-with-runtimes-in-95th-percentile.html)
* [statements_with_sorting](https://dev.mysql.com/doc/refman/5.7/en/sys-statements-with-sorting.html)
* [statements_with_temp_tables](https://dev.mysql.com/doc/refman/5.7/en/sys-statements-with-temp-tables.html)

查询如下：

![mysql-stmt-summary](/media/diagnosis/mysql-stmt-summary.png)

# DRDS

DRDS 提供了 [SQL 审计与分析](https://help.aliyun.com/document_detail/95273.html) 功能。不仅支持历史 SQL 记录的查看，还能对 SQL 执行状况、性能指标、安全问题进行实时诊断分析。

DRDS 作为云服务，可以异步抓取日志，用专门的集群进行分析，再显示到控制台上。用户不需要任何额外操作，购买服务即可使用。

用户可以通过写查询语句来查询 SQL 日志：

![drds-audit-log](/media/diagnosis/drds-audit-log.png)

还可以查看每一类 SQL 的信息：

![drds-sql-audit](/media/diagnosis/drds-sql-audit.png)

# TiDB

TiDB 实现了与 MySQL 类似的 [statement summary tables](https://pingcap.com/docs-cn/stable/reference/performance/statement-summary/)。

与 MySQL 有几点显著不同：

* `events_statements_summary_by_digest` 表的数据会定时（默认每半小时）更新，历史数据填到 `events_statements_summary_by_digest_history` 表中。这样可以查询一段时间内的数据。
* 同时按 SQL digest 和 plan digest 聚合，这样可以排查 plan 变化导致的性能问题。

从 TiDB 4.0 起，statement summary tables 被集成到 TiDB Dashboard 中，以可视化的形式呈现。

# 总结

从这些数据库的 SQL 性能视图来看，有几种形式：

* 内核集成，以系统表或系统视图的形式提供能力。但系统表多为内存表，在查询它时实时生成结果。这样可以借助 SQL 强大的表达能力，对系统表做聚合、排序甚至 join 操作。
* 提供可视化界面，从内核中抽取数据并展现到界面上。这要求内核本身有统计 SQL 的能力。它的好处是图表非常直观，也不需要写 SQL 来查询。
* 内核打印日志，然后外部工具定时抽取日志、解析、聚合并展现到界面上。它不需要内核本身具有统计 SQL 的能力，但打印日志比较消耗资源。
