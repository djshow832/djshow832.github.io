---
layout: post
title:  "PostgreSQL based multi-model databases"
date:   2020-5-30 21:08:57 +0800
categories: pg
---
## PG 扩展

### 基于 PG 的数据库

PostgreSQL（以下简称 PG）是当下最流行的数据库之一，很多数据库都是基于 PG 开发的。

有些数据库是基于 PG fork 修改源码的。例如 YugabyteDB，虽替换了存储引擎，但还是基于 PG 的 SQL 层进行改造，以减少开发量。由于需要修改源码，这一类的实现原理比较好理解。

有些数据库是基于 PG 扩展开发的，例如 Citus 是 PG-sharding 的分布式数据库，把 PG 作为存储节点，在计算层再实现分布式计算，把单机计算的部分下推给 PG。由于仍然是关系型数据库，这一类也比较好理解。

本文要介绍的几款数据库就很不一样，它们是以 PG 扩展的形式开发的，但数据模型又不是普通的关系模型：

* PostGIS 是空间数据库
* TimescaleDB 是时序数据库
* AgensGraph 是图数据库

从直觉上来说，这些数据库一般归为 NoSQL 数据库，它们的解析器、优化器、执行器、存储引擎等各个流程都与 RDBMS 不一样。而 PG 扩展是无法替换这些模块的，那么它们是如何实现的呢？

### 基于 PG 扩展的优势

不难理解，基于 PG 扩展的数据库，有以下优势：

* PG 非常成熟且稳定，PG 扩展会让用户更易于尝试
* 借助 PG 强大的生态，天然地支持了各类工具、插件
* 本身就支持 SQL，形成多模数据库，用户只用部署一个数据库实例
* PG 更新版本之后，用户可以及时地享受 PG 的新特性
* PG 更新版本之后，开发者几乎不需要关心与 PG 的兼容性
* 遵循 PG 规范，降低用户的学习成本

可以参考各数据库从 PG fork 转为 PG 扩展的文章：

* Citus：[What it means to be a Postgres extension](https://www.citusdata.com/blog/2017/10/25/what-it-means-to-be-a-postgresql-extension/)
* PipelineDB：[One More Release Until PipelineDB is a PostgreSQL Extension](https://www.pipelinedb.com/blog/pipelinedb-0-9-9-one-more-release-until-pipelinedb-is-a-postgresql-extension)

### PG 扩展简介

PG 流行的原因，除了开源，还有其强大的扩展性。它们帮助 PG 建立起了强大的生态圈，也正是生态圈使得更多数据库选择以 PG 扩展的形式存在。

开源社区可以在不修改 PG 代码的情况下定制各类扩展。PG 扩展可以是：

* 语言支持，例如 PL/pgSQL, PL/Python, PL/Java
* 扩展数据类型，例如 Hstore, cube, hstore
* FDW 扩展，例如 postgres_fdw, mysqldb_fdw, clickhousedb_fdw
* 其他扩展，种类相当多

其中 FDW 一般是把其他数据库当作 PG 的数据源，本文介绍的几款数据库不属于 FDW 扩展。

用户只需要简单的 `CREATE/DROP EXTENSION [extension_name]` 就可以安装/卸载扩展。

### PG 扩展的组成

创建扩展一般需要 4 种文件：

* Makefile 文件: 用来编译 C 代码
* Control 文件: 描述扩展的基本信息
* SQL 文件: 把 SQL 函数映射到 C 函数，调用 SQL 函数时自动调用扩展中的 C 函数
* C 代码: 以共享对象的形式被调用

其中 C 语言代码有三种函数：

* SQL 文件中 SQL 函数的具体实现
* 回调函数，通过给函数指针赋值来注册，当特定事件发生时会调该函数
* 不用注册，一些事件触发时会自动调用，例如 load/unload 该扩展

可见，PG 在设计时就充分考虑了扩展性。本文介绍的几款数据库，就是通过以上功能实现的。

## PostGIS

PostGIS 是当下最流行的空间数据库。PostGIS 最基础的三个模块是空间数据类型、空间索引和空间函数。PostGIS 通过向 PG 扩展以上三个模块以实现空间数据库。

### 空间数据类型

PostGIS 支持两种空间对象：

* 几何形状基于平面，对应的类型是 geometry，例如 geometry(POINT)。因为基于平面计算，geometry 的效率更高。如果空间仅局限于一个市，或更小的区域，建议用 geometry。
* 地理形状基于球体，对应的类型是 geography，例如 geography(POINT)。它存储经纬度，但是因为计算更复杂，所以可用的函数更少。

因为 PG 支持动态扩展数据类型，所以 PostGIS 可以方便地在 PG 上扩展这些类型。例如 geometry 的定义：

```sql
CREATE TYPE geometry (
        internallength = variable,
        input = geometry_in,
        output = geometry_out,
        ...
        storage = main
);
```

PostGIS 对 geometry 和 geography 都支持两种格式：Well-Known Text (WKT) 和 Well-Known Binary (WKB)。其中 WKT 是文本格式，易于阅读。WKB 是二进制格式，效率更高。生产环境一般用 WKB。两种对象可以相互转换，例如函数 `ST_AsBinary` 把 WKT 转换为 WKB。

文本可以很方便地转为 WKT 或 WKB 格式，例如 `ST_GeomFromText` 把文本转换为 WKT 格式的 geometry 对象：

```sql
CREATE TABLE geotable (geom geometry);
INSERT INTO geotable (geom) VALUES (ST_GeomFromText('POINT(-126.4 45.32)', 312));
```

PG 还有原生的语法进行类型转换，例如：

```sql
SELECT 'SRID=4326;POINT(0 0)'::geometry;
```

### 空间索引

对多边形进行计算，计算量非常大，但如果把它们简化为矩形，计算就快很多。即使是最复杂的多边形和线串（LineString) 也可以用一个简单的 bounding box（边界框）来表示。二维图形的 bounding box 是一个矩形，三维是一个长方体。

计算二维图形的 bounding box：

![bounding-box](/media/pg/bounding-box.png)

BTree 只能搜索一维数据，而空间索引需要检索多维数据。空间索引的基本思路是存储对象的 bounding box，检索时先用 bounding box 近似计算，过滤大部分对象，再进行精确计算，这样大大减少了读取的数据量。例如，“多边形内包含了哪些线段” 可以先近似为 “多边形的边界框内包含了哪些线段的边界框”。

PG 原生就支持 GiST 索引、SP-GiST 索引和 BRIN 索引。因此 PostGIS 可以直接借助这些索引实现空间索引。

#### GiST 索引

GiST(Generalized Search Trees，通用搜索树) 和 BTree 一样，是平衡树结构。GiST 准确地说并不算一种索引类型，而是一种模板，RTree 就是在 GiST 之上实现的。

RTree 和 BTree 类似，也是把数据集迭代地分割成更小的数据集，最终形成一棵平衡树。不同的是 RTree 是 BTree 在多维空间的扩展，可以分割多维空间。

RTree 索引分割二维空间的原理：

![rtree-index](/media/pg/rtree-index.png)

GiST 建索引速度较慢，写入性能不好，占的空间也比较大。只要 GiST 索引能在内存放得下，查询性能就很好。

语法：

```sql
CREATE INDEX [indexname] ON [tablename] USING GIST ( [geometryfield] ); 
```

#### SP-GiST 索引

SP-GiST(Space-Partitioned GiST)，也是一种模板，可以在它之上实现 partitioned search tree，例如四叉树、KD 树、字典树。它迭代地把数据集分成更小的区域，但与 GiST 不一样，它不是平衡树，每个子数据集的大小也不需要相等。

当有很多对象重叠的时候，SP-GiST 尤其有效。

语法：

```sql
CREATE INDEX [indexname] ON [tablename] USING SPGIST ( [geometryfield] ); 
```

#### BRIN 索引

BRIN(Block Range Index) 的思想是以 table block（或叫 range）为单位划分整张表，存储每个 range 中所有几何体的 bounding box。在查询时，可以快速过滤掉不符合条件的 range。它适用于数据有序的场景，这样 range 的 bounding box 互不相交，可以有效地过滤 range。

BRIN 用查询效率换取了写入效率。它的存储空间比 GiST 小很多，建索引速度快很多，但查询效率也低。

语法：

```sql
CREATE INDEX [indexname] ON [tablename] USING BRIN ( [geometryfield] ); 
```

### 空间函数

空间数据库需要有分析和处理 GIS 对象的能力。前面讲到，PG 支持扩展 SQL 函数，所以 PostGIS 在 PG 上扩展了很多函数，实现了这些能力。例如：

* `ST_DWithin(location, point, 1000)`：判断位置是否在某个点在 1km 以内
* `ST_Intersects(geom1, geom2)`：检查两个形状是否相交

SQL 文件中定义 SQL 函数与 C 函数的映射。例如 `ST_Intersects` 的定义如下：

```sql
CREATE OR REPLACE FUNCTION _ST_Intersects(geom1 geometry, geom2 geometry)
		RETURNS boolean
		AS 'MODULE_PATHNAME','ST_Intersects'
		LANGUAGE 'c' IMMUTABLE STRICT PARALLEL SAFE
		_COST_HIGH;
```

除函数外，还支持操作符进行近似计算。例如使用 `geom1 && geom2` 代替 `ST_Intersects(geom1, geom2)` 检查 `geom1` 和 `geom2` 否相交，会只通过 bounding box 进行近似查询。由于操作符是纯索引查询，不需要精确计算，所以效率高很多。

### 查询语言

空间数据库与关系型数据库的差异不算大，它仍然可以用关系表存储，用 SQL 查询。

例如空间连接（spatial joins）就是用空间关系作为条件进行表间 join：

```sql
SELECT * FROM geom_table1 INNER JOIN geom_table2 ON
		ST_Intersects(geom_table1.geom, geom_table2.geom);
```

### 总结

PostGIS 实现空间数据库离不开 PG 的几个功能：

* 扩展数据类型
* 扩展 SQL 函数
* GiST 和 RTree 索引

## TimescaleDB

TimescaleDB 是除 InfluxDB 外最流行的时序数据库。

时序数据库的特点是：

* 写入量巨大，所以写入效率很重要
* 绝大多数写入都是追加，极少需要更新
* 越新的数据查询越频繁
* 经常以时间或 metric 的维度进行实时聚合

TimescaleDB 必须要适应以上这些需求。

### 存储

由于时序数据库的数据量巨大，如果用单个 BTree 存储，当单表的数据量达到 TB 级别后写入性能必然严重下降。

TimescaleDB 通过 hypertable 存储数据。Hypertable 是 TimescaleDB 定义的一种抽象，它在逻辑上是一张表，但物理实现是多个 chunk，每个 chunk 是一张小表。PG 支持表继承，所以 TimescaleDB 使用表继承实现 chunk：每个 chunk 其实是一张子表，继承于父表 hypertable。

每隔一段时间，TimescaleDB 就会自动创建新 chunk，后续的写入写到新 chunk 里，新 chunk 常驻内存。这样，TimescaleDB 通过把表拆分成很多个 BTree 保证了写入性能。

TimescaleDB 官网上贴出了数据导入的性能对比：

![timescale-vs-postgres-insert](/media/pg/timescale-vs-postgres-insert.jpg)

建 hypertable 的步骤是先建一张普通表，然后用 `create_hypertable(table_name, key)` 函数把它转换为 hypertable，同时指定各种参数。其中参数 `chunk_time_interval` 是分割 chunk 的时间间隔，每隔 `chunk_time_interval` 就创建一个新的 chunk。

PG 的存储特性是只增不改，很适合时序数据库。PG 通过 AUTOVACUUM 清理无效数据，但时序数据库极少进行 delete 和 update，几乎没有无效数据，所以这些开销也可以忽略。但是由于当前写入都集中在同一个 chunk，所以会有写热点。

除 DML 外，hypertable 的 DDL 也是对用户透明的。例如对 hypertable 的 `ALTER` 语句都被传播给它的每一个 chunk。

但 chunk 也不是完全透明。例如用户可以通过 `move_chunk()` 移动 chunk 到某个 tablespace，这样可以把更久的数据移到更便宜的磁盘上。

查询最近的数据一般会选择更小的时间、更多的字段，用行存更合适；但是历史数据相反，用列存更合适。可以配置 TimescaleDB 异步地把行存压缩成列存。例如把 7 天前的数据进行压缩：

```sql
SELECT add_compress_chunks_policy('measurements', INTERVAL '7 days');
```

用户还可以指定列存的 main key，查询中通过 main key 过滤将非常高效。

### 查询语言

典型的时序数据库是 schema-free 的，例如 InfluxDB 和 OpenTSDB。为了处理半结构化数据，例如传感器会采集到各种度量 (metric)。TimescaleDB 可借助 JSON 和 JSONB 来达到此目的。

为了适应时序的场景，一些时序数据库会定制查询语言，例如 InfluxDB 使用 Flux。而 TimescaleDB 仍然使用 SQL 查询。

TimescaleDB 借助窗口函数实现实时聚合，例如：

* 用 `SUM(SUM(column)) OVER(ORDER BY group)` 实现累积和
* 用 `AVG(column) OVER(ORDER BY time ROWS BETWEEN ...)` 实现 moving average

有些通过自定义 SQL 函数实现，例如：

* `percentile_cont()`：百分位
* `time_bucket()`：按固定间隔采样，在降采样查询时使用
* `histogram()`：被聚合的一组数据的直方图

通过窗口函数和 SQL 函数，SQL 可以像 Flux 一样简洁地查询数据。

因为 PG 扩展不能改动解析器，TimescaleDB 实现很多 DDL 操作不是通过新增语法，而是 SQL 函数。用户需要在 SELECT 中调用该函数，例如前面提到的：

* `create_hypertable()`
* `add_compress_chunks_policy()`
* `move_chunk()`

不得不说有些反直觉。

### 总结

TimescaleDB 实现时序数据库离不开 PG 的几个功能：

* 表继承
* JSON/JSONB 类型
* 窗口函数
* 扩展 SQL 函数

## AgensGraph

AgensGraph 在图数据库中还不算流行，但也有不小的影响力。准确地说，AgensGraph 不算 PG 扩展，它是基于 PG 改造的图数据库，使用自己的存储引擎和图查询引擎。

![AG-Architecture](/media/pg/AG-Architecture.png)

但是 AgensGraph 已经开始调整方向，并发布了 [agensgraph-ext](https://github.com/bitnine-oss/agensgraph-ext)，一款基于 PG 扩展的图数据库。关于调整方向的原因，参考讨论：[AgensGraph postgres extension](https://github.com/bitnine-oss/agensgraph/issues/268)。

### 存储

图数据模型包含各类对象：graph、vertex、edge、vertex label、edge label、property 等。AgensGraph 使用图存储引擎存储这些对象，但 AgensGraph-ext 显然不能这么操作，只能用关系表存储。

存储的规则如下：

* 新建 graph 时，都会自动创建一个与该 graph 同名的库，同时向 `ag_catalog.ag_graph` 表中插入一条记录
* 新建 label（包括 vertex label 和 edge label）时，都会自动在 graph 库里创建一张与 label 同名的表，同时向 `ag_catalog.ag_label` 表中插入一条记录
* 插入 vertex / edge 时，在上述 label 表中插入一条记录

`ag_graph` 和 `ag_label` 都是系统表，存储 graph 和 label 的元信息。`ag_label` 中记录了 label 和名字、graph id、类型（vertex label 还是 edge label）、存储 vertex 或 edge 的表名。

创建图 `g`，以及 vertex label `v1`：

```sql
SELECT * FROM cypher('g', $$CREATE (:v1)$$) AS r(a agtype);
```

AgensGraph-ext 会自动创建库 `g` 和表 `g.v1`：

```sql
SELECT name, id, kind, relation FROM ag_label;
       name       | id | kind |      relation      
------------------+----+------+--------------------
 _ag_label_vertex |  1 | v    | g._ag_label_vertex
 _ag_label_edge   |  2 | e    | g._ag_label_edge
 v1               |  3 | v    | g.v1
```

Vertex label 对应的表中包含 vertex id、properties。其中 properties 是新定义的类型 `ag_type`，它实际是一个 map，内部的数据类型有些是 PG 原生类型，有些是 Cypher 自定义类型。

Edge label 对应的表中包含 edge id、start_id、end_id、properties。其中 start_id 和 end_id 是与它相连的 vertex 的 id。

这样，AgensGraph-ext 把图模型包含的对象都存储到了关系表中。

### 查询语言

由于图模型的特殊性，图数据库一般需要特殊的语言进行查询，例如 Cypher、Gremlin 等。AgensGraph 的查询语言很灵活，支持 SQL、Cypher 以及 SQL + Cypher 的混合查询。可以在 SQL 中嵌入 Cypher，也可以在 Cypher 中嵌入 SQL。

例如在 SQL 中嵌入 Cypher：

```sql
SELECT n->>'name' as name 
	FROM history, (MATCH (n:dev) RETURN n) as dev 
	WHERE history.year > (n->>'year')::int;
```

在 Cypher 中嵌入 SQL：

```sql
MATCH (n:dev)
	WHERE n.year < (SELECT year FROM history WHERE event = 'AgensGraph')
	RETURN properties(n) AS n;
```

但是 AgensGraph-ext 作为扩展，就不能这么灵活了，只能以函数的形式扩展语法。

创建图和删除图：

```sql
SELECT create_graph('example');
SELECT drop_graph('example', true);
```

AgensGraph-ext 通过 SQL 函数 `cypher()` 实现了在 SQL 中嵌入 Cypher：把 graph 名作为第一个参数；把 Cypher 语句像字符串一样用 `$$` 包起来作为第二个参数，AgensGraph-ext 内部解析 Cypher 语句。

使用 `CREATE` 子句插入数据：

```sql
SELECT * FROM cypher('example', $$
    CREATE (:v {id:"right rel, initial node"})-[:e {id:"right rel"}]->(:v {id:"right rel, end node"})
$$) AS (a agtype);
```

使用 `MATCH` 子句查询：

```sql
SELECT * FROM cypher('example', $$
	MATCH con_path=(a)-[]->()<-[]-()
	where a.id = 'initial'
	RETURN con_path
$$) AS (con_path agtype);
```

### 总结

AgensGraph-ext 实现图数据库离不开 PG 的几个功能：

* 扩展数据类型
* 扩展 SQL 函数

## 总结

以上分析了 PostGIS、TimescaleDB、AgensGraph 的大体实现方式，现在它们的原理很清楚了。

在存储上：

* PostGIS 充分利用到 GiST、SP-GiST、BRIN 索引
* TimescaleDB 对用户透明地把 hypertable 拆分成多个 chunk 以提升插入性能
* TimescaleDB 后台自动把历史数据转换为列存，以提升查询性能
* AgensGraph 把图数据以关系表的形式存储
* 半结构化的数据使用 JSON/JSONB 类型存储

在查询上：

* PostGIS 和 TimescaleDB 并非 NoSQL 模型，它们仍然是关系模型，却能适用于空间和时序的场景
* 用户仍然使用 SQL 查询，可以进行聚合、JOIN 等操作
* 不能用 SQL 语法表达的操作，就用 SQL 函数表达
* Cypher 语言通过字符串传递给 SQL 函数

我们也从中窥见了 PG 强大的扩展能力，其中必不可少的功能包括：

* 动态扩展数据类型
* 动态扩展 SQL 函数

但是同时看到，PG 扩展的形态也限制了这些数据库的能力：

* 只能用 SQL 函数的形式表达多样的操作，尤其是图查询语言
* 不能做极致的优化，尤其是时序数据库，仍然用 BTree 存储

总之，用 PG 扩展的形式实现多模数据库的便利性还是远大于其限制的。相信未来有更多的数据库会以 PG 扩展的形式开发。