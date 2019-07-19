# Parametric Query Optimization
## 背景
### PQO
在 OLTP 系统中，SQL 语句常常有特定的 SQL 模板。对于具有相同 SQL 模板的 SQL 语句，数据库系统可以先将 SQL 参数化为 SQL 模板，然后复用中间结果。<br>
例如<br>
```sql
# SQL 语句
SELECT order.price, customer.name
	FROM order JOIN customer ON order.customer_id = customer.id
	WHERE customer.city = 'BeiJing' AND order.create_time > '2019-1-1';
	
# SQL 模板
SELECT order.price, customer.name
	FROM order JOIN customer ON order.customer_id = customer.id
	WHERE customer.city = ? AND order.create_time > ?;
```

这类优化称作 Parametric Query Optimization(PQO)。<br>

PQO 的流程分为 compile-time 和 run-time。<br>
- compile-time 的输入是 SQL 模板，调用一次优化器，产生中间结果。
- run-time 的输入是参数值，调用优化器和执行器，输出运行结果。<br>

如何使 run-time 的耗时最短，是 PQO 的主题。

### Dynamic-plan Optimization
按优化的流程，可以分为以下几种：<br>

**Static Optimization**: 在 compile-time 产生一次计划，之后每次运行都复用这个计划，也就是使用常说的 Plan Cache。<br>
由于没有考虑到统计信息的变化，所以实际执行的代价时高时低，甚至会跨越数个数量级。

**Run-time Optimization**: compile-time 不做优化，每次在执行前重新优化。<br>
由于优化本身也有代价，因此需要更长的 start-up time。

**Dynamic-plan Optimization**: 优化器在 compile-time 产生多个备选计划，每个计划都在至少一个场景下（比如特定的 selectivity 范围内）是最优的。在每次执行前，先根据实际的统计信息（通过参数值）找出最优计划，再执行。<br>
与 Static Optimization 相比，Dynamic-plan Optimization 每次都能获得最优计划执行。<br>
与 Run-time Optimization 相比，Dynamic-plan Optimization 执行时避免了重复的优化。

PQO 主要研究 Dynamic-plan Optimization。

### PQO 的假设
PQO 有几个假设：<br>
- Run-time 得到的统计信息都是准确的。
- 表的 row count 在 compile-time 是已知的。
- 计划的代价公式在参数空间内是线性函数或分段线性函数。<br>

还有一些关于代价公式的其他假设，这些假设并非总是成立，但对于多数 TP 场景是成立的。

## choose-plan 算子
### 基本思路
1994 年的论文 *Optimization of Dynamic Query Evaluation Plans* 提出了 choose-plan 算子，它是 PQO 的典型代表。<br>
它的基本思路是，保留所有可能最优的子计划，使用 choose-plan 算子把它们关联起来，在执行时再选择实际最优的子树。<br>

该算法与传统的 Volcano 优化器不同。Volcano 在自顶向下搜索时，要求算子的代价都是确定的值，总是可以比较的，即代价是全序的。<br>
而在 PQO 的 compile-time，优化器因为没有实际的参数值，也就得不到确定的代价。作者把代价估算成一个范围，只有范围没有重叠才可以比较，否则就不可比较，所以代价是偏序的。<br>
例如 `Cost(A) = [1, 5]`,  `Cost(B) = [2, 7]`,  `Cost(C) = [6, 9]`<br>
那么可以确定 Cost(A) < Cost(C)，但 Cost(A) 与 Cost(B) 是无法比较的。<br>

对于无法比较代价的算子树，不必立即判断，而是把子树都保留，并在上层加一个 choose-plan 算子，作为它们的父节点。choose-plan 算子可以出现在任意算子之上，出现任意多次。<br>
在执行前，结合实际的统计信息，算出各个子树实际的代价，并选择代价最低的那一个子树。

### 代价模型

令算子树 A 和 B 的代价<br>
`Cost(A) = [CAmin, CAmax]`<br>
`Cost(B) = [CBmin, CBmax]`<br>
那么<br>
`Cost(A + B) = [CAmin + CBmin, CAmax + CBmax]`<br>
`Cost(A - B) = CAmin - CBmin`（用来做剪枝优化）<br>

choosen-plan 本质上也是一个物理算子，它本身也有代价。令 choose-plan 的代价<br>
`Cost(choose-plan) = [CPmin, CPmax]`<br>
那么 choose-plan 子树的代价是<br>
`Cost(subplan) = [min(CAmin, CBmin) + CPmin, min(CAmax, CBmax) + CPmax]`<br>
而不是<br>
`Cost(subplan) = [min(CAmin, CBmin) + CPmin, max(CAmax, CBmax) + CPmax]`<br>
因为 choose-plan 算子总是选择代价更小的子计划。

### 算法优化
因为代价从确定的值变成了一个范围，所以会保留大量子树。这时优化器的剪枝优化 (branch-and-bound pruning) 达不到理想的效果，导致搜索复杂度扩大，优化器的耗时、算子树的内存占用都比较高。<br>
实验表明，Dynamic-plan 的子树的数量、start-up time 都随参数的数量呈指数上涨。<br>
为此，除了剪枝优化以外，本文还提出其他的一些优化手段：<br>

**subplan 共享**：因为算子树会共用大量的子计划，而子计划的代价不用重复估算，所以把计划组织成 DAG 图而不是树状，相同的子计划只出现一次。<br>

**清理低频子计划**：长时间没有被使用的子计划，可以从算子树中移除。<br>

## PPQO
### PPQO 简介
以 choose-plan 算子为代表的 PQO 有两大缺陷：<br>
- PQO 假设物理计划的代价公式在参数空间是线性或分段线性的，并且最优计划的参数空间是连续且凸面的，然而现实中并不是这样。
- PQO 在 compile-time 的代价过高，然而有些查询模板很少出现，或者有些参数很少出现，每次为每种参数计算最优计划很浪费。<br>

因此 2009 年的论文 *Progressive Parametric Query Optimization* 提出了渐近式的 PQO —— PPQO。<br>
PPQO 的目标并不是找到最优计划，而是找出一个近优的计划即可。也就是说，选择的计划与最优计划的代价只要不差太多就行。而这个“太多”的定义，用户可以通过自定义参数来调整。<br>
PPQO 仍然基于对代价函数的一些假设。它假设随着选择度（selectivity) 的增大，代价是单调递增且连续的。这也符合大多情况。<br>

PPQO 有两个互相矛盾的目标。为此，文中提出了两种 PPQO 算法，分别侧重于这两个目标：<br>
- 选择的计划更优。前面提到，PPQO 选择的是近优的计划，所以近优计划的代价越接近最优计划越好。Bounded-PPQO 的目标是尽可能找到更优的计划。
- 调用优化器的次数降到最少。与 PQO 不同，PPQO 在 run-time 找不到近优计划时，会重新调用优化器。Ellipse-PPQO 倾向于使 PPQO 调用优化器的次数尽量少。<br>

PPQO 为每个查询缓存了多条计划，每个计划都至少在一个场景下是最优的。<br>
令 T 为查询 q 对应的 Plan Cache，T 里的每个计划 p，都附带它的参数的选择度 s、代价 c。
Bounded-PPQO 和 Ellipse-PPQO 都要实现 addPlan 和 getPlan 函数。
- addPlan：调用优化器后如果生成了一个新的计划 p，addPlan 负责把 p 添加到 T 中。
- getPlan：getPlan 的输入是 SQL 的参数值，它通过统计信息到 T 中寻找近优的计划 p。如果没找到，需要重新调用优化器，找到最优的计划。

### Bounded-PPQO
该算法要求 getPlan 返回的计划总是符合 `cost(p) < cost(optimal) * M + A`。其中 p 是返回的计划，optimal 是当前实际的最优计划，M 和 A 是用户自定义的值，M >=1, A >= 0。<br>

令 getPlan 的输入是查询 q'，q' 对应的选择度为 s'。那么 getPlan 步骤为：<br>
1. 把 T 里的计划按代价从低到高排序。因为代价是随选择度单调递增的，所以排序后选择度也是从低到高有序的。
2. 找到小于 s' 的最大选择度 s1 和对应代价 c1，以及大于 s' 的最小选择度 s2 和对应代价 c2。
3. 如果满足 `c1 <= c2 <= c1 * M + A`，则返回 p2。
4. 否则调用优化器，找出最优计划，并调用 addPlan 添加该计划。<br>

简单来说，假如 T 里已经有两个邻近的计划 p1 和 p2，选择度满足 s1 <= s' <= s2，并且 c1 和 c2 还很接近，那么就返回 p2。因为这样，p2 与实际的最优计划的代价必然更接近。<br>
随着查询次数的增加，最终 Plan Cache 的命中率越来越高，即调用优化器的频率越来越低。

### Ellipse-PPQO
已知计划 p 在选择度 s1 和 s2 都是最优解，用户指定参数 delta，其中 `0 < delta < 1`。<br>
令 getPlan 的输入是查询 q'，q' 对应的选择度为 s'。<br>
那么假如 s' 满足 `|s1 - s2| / (|s1 - s'| + |s2 - s'|) >= delta`，就返回计划 p。<br>

在一维选择度空间（SQL 模板里仅有 1 个参数）中，p 的选择度范围是一条线段。也就是说，只要 s 在这条线段内，那么返回的计划都是 p。<br>
在二维选择度空间（SQL 模板里有 2 个参数）中，p 的选择度范围是一个椭圆。因为上述公式改写到二维空间中，就是椭圆的公式，其中 s1 和 s2 是椭圆的焦点。<br>
所以，Ellipse-PPQO 的含义是，已知计划 p 在两个选择度 s1 和 s2 上都是最优的，那么对于 s1 和 s2 附近的选择度范围内，都可以使用计划 p。<br>

同样的，随着查询次数的增加，Plan Cache 的命中率越来越高。

### 测试
在 SQL Server 上测 TPC-H。Bounded-PPQO 的参数：M=1.1, A=0；Ellipse-PPQO 的参数：delta=0.95。<br>
主要的两个指标：<br>
HitRate：Plan Cache 的命中率。<br>
OptRate：getPlan 返回最优计划的比例。<br>
Number of plans：Plan Cache 中的计划数量。<br>
Number of points：内存中记录的（selectivity，plan, cost）三元组的数量。<br>
Quality of returned plans：最优计划的代价 / PPQO 返回的计划的代价。