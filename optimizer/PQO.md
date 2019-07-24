# Parametric Query Optimization
## 背景
### PQO
在 OLTP 系统中，SQL 语句常常有特定的 SQL 模板。对于具有相同 SQL 模板的 SQL 语句，SQL 引擎可以先将 SQL 参数化为 SQL 模板，然后复用中间结果。<br>
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
由于优化本身也有代价，因此需要更长的 start-up time（每次执行前的耗时）。

**Dynamic-plan Optimization**: 优化器在 compile-time 产生多个备选计划，每个计划都在至少一个场景下（比如特定的 selectivity 范围内）是最优的。在每次执行前，先根据实际的统计信息（通过参数值）找出最优计划，再执行。<br>
与 Static Optimization 相比，Dynamic-plan Optimization 每次都能获得最优计划执行。<br>
与 Run-time Optimization 相比，Dynamic-plan Optimization 执行时避免了重复的优化。

PQO 主要研究 Dynamic-plan Optimization。也就是说，PQO 结合 Plan Cache 和 CBO，取二者之所长。

### PQO 的假设
PQO 基于几个假设：<br>
- Run-time 得到的统计信息都是准确的。
- 表的 row count、NDV 等在 compile-time 是已知的，仅与参数相关的统计信息未知，如 selectivity。
- 计划的代价公式在参数空间内是线性函数或分段线性函数。比如 selectivity 越大，代价越大。<br>

还有一些关于代价公式的其他假设，后面会提到。这些假设并非总是成立，但对于多数 OLTP 场景是成立的。

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

### 运行时参数
论文在提出时，并没有把该算法称为 PQO，因为它的 run-time 参数可以是：<br>
- 运行时的资源，包括内存使用量、CPU 占用率等。例如系统在 CPU-bound 时，计划的 CPU 代价应比内存、IO 代价有更高的权重。
- 用户自定义的变量，例如数据库的系统参数。
- 子查询的结果，这里特指使用 Apply 算子的子查询算法。
- prepared statement 的 predicate 的 selectivity，其中 predicate 包含未绑定变量。<br>
而 PQO 中的参数，近似于最后一条的定义。<br>

虽然 choose-plan 算子的应用范围很广，但在后续其他人的研究中，都把 choose-plan 当作一种 PQO 的算法来参考。本文也把参数定义为 SQL 模板中的绑定变量，不考虑运行时资源等其他变量。

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
实验表明，Dynamic-plan 的子树的数量、start-up time 都随参数的数量呈指数上涨，导致搜索复杂度扩大，优化器的耗时、算子树的内存占用都比较高。<br>
为此，本文提出一些优化手段：<br>

**剪枝优化 (branch-and-bound pruning)**：同 Volcano 优化器一样，choose-plan 的代价模型也可以使用剪枝优化。但是由于代价只在可比较时才能使用剪枝优化，所以远达不到同 Volcano 的效果。<br>

**subplan 共享**：因为算子树会共用大量的子计划，而子计划的代价不用重复估算，所以把计划组织成 DAG 图而不是树状，相同的子计划只出现一次。<br>

**清理低频子计划**：长时间没有被使用的子计划，可以从算子树中移除，可以减少内存占用和一定的 start-up time。<br>

## 基于 POSP 的 PQO
### POSP
PQO 的每个参数范围都能匹配到一个最优计划，那么把所有参数范围内的这些最优计划全部收集起来，就是 parametric optimal set of plans(POSP)。上述 choose-plan 算法里的 Plan Cache 就缓存了每个查询的一整套 POSP。<br>
POSP 是对查询计划的一种非常简洁有效的建模。因此，它在学术中经常被研究，包括 PQO、Re-optimization、AQP（自适应查询处理）等领域。<br>

在 PQO 中，假如能在 compile-time 把 POSP 及每个最优计划覆盖的参数范围求出来，并把 POSP 放到 Plan Cache 中，那么在 run-time 只用去 Plan Cache 里根据参数查找就能找到最优计划。<br>
因此，学术中涌现出大量关于如何统计出 POSP 的算法。下面介绍几个典型的算法。

### 通过几何形状统计 POSP
*Design and Analysis of Parametric Query Optimizer Algorithms* 提出一种用凸面体的特征来求 POSP 的算法。<br>

首先，它对 POSP 做了一些理想化的假设，并在此基础上推导。<br>
假如计划 p 的代价函数是 n 元线性函数，那么：<br>
- 假如 p 在 u 和 v 点都是最优的，那么 p 在 u 和 v 连线之间的点上也是最优的。
- p 的最优区间是一个凸面体。<br>
所以，n 维参数空间可以被划分为多个凸面体。求解在 n 维空间中的最优解的分布，其实就是找出这些凸面体。<br>
两个凸面体交界处，就是两个计划的代价函数相交的地方，它是一个 n-1 维的平面。<br>

论文针对一元线性代价函数、二元线性代价函数、非线性代价函数这三种情况，分别给出了如何把参数空间划分成多个最优计划的空间（parametric optimal set）的算法。它们的思路都是，按逆时针方向围绕着凸面体，求出各个边界。这些边界也就是平面。<br>
对于一元线性代价函数，最优计划集包含 14 到 22 个计划；对于二元线性代价函数，POSP 包含不到 40 个计划。

### 通过二分法统计 POSP
*Parametric Query Optimization for Linear and Piecewise Linear Cost Functions* 提出一种不同的思路。<br>
它大量参照了 *Design and Analysis of Parametric Query Optimizer Algorithms*，包括问题的定义、推论、算法的衡量标准等。它同样把问题分为两类：代价函数是线性的，或是非线性的。并对其中每一个问题的求解，进行算法推导。<br>
但是这些论文的算法与上篇论文有几处不同：<br>
- 上篇论文更加依赖优化器的实现，它假设优化器能某个点的所有最优计划。而本篇论文假设优化器输出最优计划中的一个。
- 上篇论文仅针对一元和二元的线性代价函数做了推算。本篇论文提出 cost polytope 算法，适用于多个维度的参数空间，而且对传统的优化器逻辑没有侵入。
- 本篇论文把非线性的代价函数近似看成分段的线性函数，因为只要分段够多，它总能拟合非线性函数。作者修改了 Volcano 优化器，以实现该算法。<br>

对算法好坏的衡量标准是，它在 compile-time 调用优化器的次数。论文证明了一个 lower bound，在某些假设成立的情况下，调用优化器的次数接近于这个 lower bound。

## PPQO
### PPQO 与 PQO 的差异
以上的 PQO 算法都有两大缺陷：<br>
- PQO 假设物理计划的代价公式在参数空间是线性或分段线性的，并且最优计划的参数空间是连续且凸面的，然而现实中并不是这样。
- PQO 在 compile-time 的代价过高，然而有些 SQL 模板很少出现，或者有些参数很少出现，每次为每种参数计算最优计划很浪费。<br>

因此 2009 年的论文 *Progressive Parametric Query Optimization* 提出了渐近式的 PQO —— PPQO。之所以称为渐近式，是因为 PPQO 并不是在 compile-time 把所有潜在的最优计划都计算并缓存起来，而是每次在执行前从 Plan Cache 里找有没有合适的计划，没有的情况下，才调用优化器算出最优计划，并加到 Plan Cache 中。<br>

PPQO 与 PQO 相比，有几个明显的不同点：<br>
- PPQO 的目标并不是找到最优计划，而是找出一个近优的计划即可。也就是说，选择的计划与最优计划的代价只要不差太多就行。而这个“太多”的定义，用户可以通过自定义参数来调整。现代数据库对于稍复杂的查询，不可能枚举所有计划，通常目标不是找到最优计划，而是避开最差计划。所以寻找近优解的方式是合理的。<br>
- PPQO 为每个查询缓存了多条计划，每个计划都至少在参数空间的某一个点是最优的，但这些计划并不一定覆盖整个参数空间。<br>
- Plan Cache 中的计划并不是在 compile-time 一次全部加进去的，而是随着查询的增加，Plan Cache 容纳越来越多的计划。一方面，这样可以把 compile-time 的开销分摊到 start-up time；另一方面，对于没有出现过的参数范围，不需要消耗额外的时间和空间。<br>

因为 PPQO 用选择度空间代替参数空间，所以 predicate 中不仅可以包含等值查询，还可以包含范围查询，只要 predicate 能表示为 selectivity 的表达式就可以。

### 算法
PPQO 有两个互相矛盾的目标。为此，文中提出了两种 PPQO 算法，分别侧重于这两个目标：<br>
- 选择的计划更优。前面提到，PPQO 选择的是近优的计划，所以近优计划的代价越接近最优计划越好。Bounded-PPQO 的目标是尽可能找到更优的计划。
- 调用优化器的次数降到最少。与 PQO 不同，PPQO 在 run-time 找不到近优计划时，会重新调用优化器。Ellipse-PPQO 倾向于使 PPQO 调用优化器的次数尽量少。<br>

令 T 为查询 q 对应的 Plan Cache，T 里的每个计划 p，都附带它的参数的选择度 s、代价 c。
Bounded-PPQO 和 Ellipse-PPQO 都要实现 addPlan 和 getPlan 函数。
- addPlan：调用优化器后如果生成了一个新的计划 p，addPlan 负责把 p 添加到 T 中。
- getPlan：getPlan 的输入是 SQL 的参数值，它通过统计信息到 T 中寻找近优的计划 p。如果没找到，需要重新调用优化器，找到最优的计划。

#### Bounded-PPQO
该算法要求 getPlan 返回的计划总是符合 `cost(p) < cost(optimal) * M + A`。其中 p 是返回的计划，optimal 是当前实际的最优计划，M 和 A 是用户自定义的值，M >=1, A >= 0。<br>

令 getPlan 的输入是查询 q'，q' 对应的选择度为 s'。那么 getPlan 步骤为：<br>
1. 把 T 里的计划按代价从低到高排序。因为代价是随选择度单调递增的，所以排序后选择度也是从低到高有序的。
2. 找到小于 s' 的最大选择度 s1 和对应代价 c1，以及大于 s' 的最小选择度 s2 和对应代价 c2。
3. 如果满足 `c1 <= c2 <= c1 * M + A`，则返回 p2。
4. 否则调用优化器，找出最优计划，并调用 addPlan 添加该计划。<br>

简单来说，假如 T 里已经有两个邻近的计划 p1 和 p2，选择度满足 s1 <= s' <= s2，并且 c1 和 c2 还很接近，那么就返回 p2。因为这样，p2 与实际的最优计划的代价必然更接近。<br>
随着查询次数的增加，最终 Plan Cache 的命中率越来越高，即调用优化器的频率越来越低。

#### Ellipse-PPQO
已知计划 p 在选择度 s1 和 s2 都是最优解，用户指定参数 delta，其中 `0 < delta < 1`。<br>
令 getPlan 的输入是查询 q'，q' 对应的选择度为 s'。<br>
那么假如 s' 满足 `|s1 - s2| / (|s1 - s'| + |s2 - s'|) >= delta`，就返回计划 p。<br>

在一维选择度空间（SQL 模板里仅有 1 个参数）中，p 的选择度范围是一条线段。也就是说，只要 s 在这条线段内，那么返回的计划都是 p。<br>
在二维选择度空间（SQL 模板里有 2 个参数）中，p 的选择度范围是一个椭圆。因为上述公式改写到二维空间中，就是椭圆的公式，其中 s1 和 s2 是椭圆的焦点。<br>
所以，Ellipse-PPQO 的含义是，已知计划 p 在两个选择度 s1 和 s2 上都是最优的，那么对于 s1 和 s2 附近的选择度范围内，都可以使用计划 p。<br>

同样的，随着查询次数的增加，Plan Cache 的命中率越来越高。

### 测试
在 SQL Server 上测 TPC-H。Bounded-PPQO 的参数：M=1.1, A=0；Ellipse-PPQO 的参数：delta=0.95。<br>
主要的几个指标：<br>
- HitRate：Plan Cache 的命中率。
- OptRate：getPlan 返回最优计划的比例。
- Number of plans：Plan Cache 中的计划数量。
- Number of points：内存中记录的（selectivity，plan, cost）三元组的数量。
- Quality of returned plans：最优计划的代价 / PPQO 返回的计划的代价。

## 工程评价
从学术的角度，PQO 基于理想化的假设，主要研究以下二者的平衡：<br>
- 从 Plan Cache 的角度，要求尽量少调用优化器。
- 从 CBO 的角度，要求选择的计划尽量更优。<br>

接下来的讨论中，我们也假设如下几点：<br>
- 统计信息都是准确的。
- 参数空间可以简化为选择度空间。<br>

从工程的角度，PQO 还需要解决以下几点问题，才够实用：<br>
- compile-time 和 start-up time。
- Plan Cache 占用的内存空间。
- 关于统计信息和代价模型的假设是否过于理想化。
- 对优化器逻辑的侵入。
- PQO 的适用场景。

### compile-time 和 start-up time
compile-time 在以下几个时间点会进行：
- 数据库进程启动后，SQL 引擎在第一次执行特定 SQL 模板时。
- 执行 DDL 后，Plan Cache 可能失效。例如加索引。
- 统计信息刷新后，Plan Cache 可能也会失效。例如 row count 会变化、selectivity 的取值范围会变化。<br>
在实际的数据库系统中，compile-time 过长，会使数据库的执行时间不稳定，甚至出现卡顿。<br>

几种 PQO 算法都是在 compile-time 把所有最优计划求出来。choose-plan 算法裁剪了较少的子树；基于 POSP 的 PQO 需要调用数次优化器。在有多个参数时，这两类算法都占用较长的 compile-time。<br>
当然，如果用户能提供 SQL 模板，可以离线预计算好 POSP，只针对特定的 SQL 模板使用 PQO。这与 SQL Plan Management 有一些相似之处。<br>

而 PPQO 是把调用优化器的开销分摊到多次执行中。当查询次数足够多时，因为 Plan Cache 的命中率接近 100%，所以 start-up time 也接近为 0。

### Plan Cache 占用的内存空间
前面提到，随着参数的增加，POSP 的容量呈指数上升。当参数仅有 2 个时，计划已经多达数十个了；当有 3 个时，内存就是明显的问题了。<br>

由于 PQO 总是会求出整个 POSP，所以在 compile-time 的内存占用就很高。<br>
与基于 POSP 的 PQO 相比，choose-plan 占用的内存更少，因为子计划是共享的。而基于 POSP 的 PQO 想要做到子计划共享，逻辑会更复杂。<br>
但是二分法的 PQO 其实可以改善内存问题，它不需要求出整个 POSP，只在把参数空间划分到一定粒度后就停止继续划分。例如，每个维度（即每个 selectivity）只划分为 4 个区间，这次能够控制总量。当然，会牺牲一定精度，导致计划质量下降。<br>

PPQO 在初始时占用内存很少，随着查询次数的参加，占用内存会升高。<br>
但是，因为 PPQO 提供了配置参数，所以用户可以通过调整参数达到计划质量与内存占用的平衡，非常灵活。极端情况下，可以达到与 Static Optimization 或 Run-time Optimization 等价的效果。<br>

### 关于代价模型的假设
compile-time 的耗时和 Plan Cache 的内存占用决定了 PQO 能不能用，而关于代价模型的假设，则决定了 PQO 好不好用。如果这些假设过于理想化，那么很多实际场景下 PQO 并不能取得更好的效果。

#### 统计信息
PQO 假设 selectivity、NDV 总是准确的，因为 PQO 认为优化器返回的最优计划就是真正的最优计划。<br>
然而实际情况并不理想。有一些原因，造成统计信息并不准确：<br>
- 统计信息过时。多数数据库的统计信息都不是实时的，而是定时刷新的，那么读到的统计信息就会延迟。
- 依赖于属性值独立（AVI）的假设。多数代价函数都假设属性之间是相互独立的，它们的取值都是独立事件。然而事实并非如此。
- 统计信息的粒度太粗。为了让统计信息不占用过多的空间，把大量信息压缩到较小的数据量。为了不占用过多的时间，通常使用采样。这样必然会丢失精度。
- 算子树中的错误传递。子节点的误差会传递到父节点，又会叠加父节点本身的误差。研究表明，多项误差叠加之后的误差是指数级增加的。<br>

目前即使是成熟的商业数据库，统计信息通常也不准确。为此，学术上除了提升统计信息准确性的算法外，还涌现出一批 AQP（自适应查询处理）的研究，使优化器不要强依赖于统计信息。但是这些研究距离工程使用还有很长的路要走。

#### 参数空间
基于 POSP 的 PQO 和 PPQO 都把参数空间简化，把参数空间等同于选择度空间。<br>
对各类统计信息进行分析：<br>
- 表的 row count。因为表名没有参数化，所以 row count 在 compile-time 是已知的。
- NDV，多用于 group by。因为列名没有参数化，所以 NDV 在 compile-time 是已知的。
- 值的平均长度，也是在 compile-time 已知。<br>
可以发现，参数空间可以简化为选择度空间。<br>

#### 代价函数
在论文 *Analyzing Plan diagrams of database query optimizers* 中提到，PQO 关于代价函数的几个假设，事实上都不成立：<br>
- 凸性：如果计划 p 在参数空间的 a 点和 b 点都是最优的，那么在 a 和 b 之间的任意点都是最优的。
- 唯一性：在整个选择度空间，单个计划只可能在一个连续区域上是最优的，不会是两个不连通的区域。
- 同质性：单个计划在它围起来的整个区域内都是最优的，不会存在“孤岛”。<br>

而基于 POSP 的 PQO 算法都强依赖于这些假设，choose-plan 算子和 PPQO 弱依赖于这些假设。

### 对优化器的逻辑侵入
尝试 PQO 的优化器必然也具备 CBO 的功能，也算是相对成熟了。而对于这样一个复杂的 SQL 引擎，修改其逻辑必须要考虑代码的侵入。<br>
如果数据库用的是 Run-time Optimization，那么任意一种 PQO 算法，必然都先要把 SQL 语句参数化，再调用优化器。<br>
因此，PQO 对于 Static Optimization 或 Dynamic-plan Optimization 的改动更小。<br>

接下来我们来简单比较上述几种 PQO 算法对 Static Optimization 的逻辑侵入。<br>
- choose-plan 算子的思维方式更接近于 Volcano 优化器，但是它对优化器的改动却相当大，因为代价模型完全不一样了。
- 基于 POSP 的 PQO 并不需要改动优化器内部，只需要在 compile-time 实现一套统计 POSP 的算法。但是优化器必须提供接口，能获得每个计划的代价函数。
- PPQO 的改动量就很小了，只需要实现 addPlan 和 getPlan 接口。不管是 Bounded-PPQO 还是 Ellipse-PPQO，实现起来都不算复杂。

### PQO 的实用性
目前还鲜有商业数据库使用 PQO。除了以上几点外，个人认为还有一些现实的因素。

#### 优化器的发展阶段
多数数据库还在研究如何提升 CBO 的质量、降低 CBO 的开销的阶段。无论是在计划枚举的高效性、统计信息的准确性、代价函数的科学性，都还不够成熟。<br>
很容易理解，如果 CBO 得出的计划都不是最优的，那 PQO 也无从谈起。

#### PPQO 的适用范围
- 对于简单的 OLTP 查询，有时 RBO + Plan Cache 就已经足够，连 CBO 都用不着。
- 对于复杂的查询，通常又使用 OLAP 系统，执行时间远大于优化时间，使用 Run-time Optimization 即可。
- 对于 ad-hoc 查询，或 OLAP 查询，很少有固定的 SQL 模板，用 Run-time Optimization 即可。<br>
可见，PQO 的应用范围并没有那么广。<br>

### 总结
通过上面的比较，个人比较看好 PPQO。它可以非常灵活地调整 compile-time、内存占用、start-up time 之间的平衡，并且对优化器的侵入也很小，尝试成本很低。<br>

关于 PPQO 的局限，也非常明确。虽然学术上有大量研究，但目前很少有数据库尝试，主要因为 SQL 引擎还没发展到可以完全依赖 CBO 的阶段。<br>
但是我相信，等未来 CBO 足够成熟了，PPQO 或类 PPQO 的算法，将大有用武之地。