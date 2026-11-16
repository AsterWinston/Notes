# 1. 为什么选择 Calcite

**Apache Calcite 是一套"不存储数据"的开源 SQL 引擎框架**，它把数据库前端中最难啃的部分——SQL 解析、语义校验、关系代数表示、查询优化——做成了可复用的组件，数据本身仍存在后端 MySQL 里。对 Mycat2 来说，中间件的核心诉求是**看懂 SQL**：用户查了哪些列、过滤了哪些条件、JOIN 了哪些表，才能决定路由到哪些分片、下发什么改写后的 SQL、回来之后怎么归并。

对比 Mycat 1.6.x：1.6.x 使用**阿里 Druid SQL Parser**，面对简单 CRUD 尚可，一旦涉及子查询、多表 JOIN、复杂表达式，能力快速退化为"提示不支持"。引入 Calcite 后，Mycat2 把中间件从"字符串级改写"升级为"计划级优化"——这是 1.x 与 2.x 在 SQL 处理上的本质差距，也是 2.x 能宣称"JOIN、子查询、聚合支持度大幅提升"的技术根源。

> **行业实践**：把 Calcite 作为 SQL 内核的不止 Mycat2——Flink 的 Table/SQL 层、Hive 3.x、Drill、Kylin 都构建在它之上。复用 Calcite 的好处是语法兼容性和优化规则由社区持续演进，代价是升级节奏受制于上游，且团队要吃透一套并不简单的 RelNode API。

| 维度 | 自研解析器（Mycat1 的 Druid） | Apache Calcite |
|------|---------------------------|---------------|
| SQL 兼容性 | 需逐个语法适配 | 业界标准，兼容性好 |
| 查询优化 | 简单改写 | RBO + CBO 完整优化 |
| 维护成本 | 高（需持续跟进 MySQL 新语法） | 低（社区维护） |
| 开发效率 | 低 | 高 |

# 2. Calcite 集成架构

Mycat2 的一条 SQL 经历四个阶段：**Parser** 把文本解析成 SqlNode 语法树；**Validator** 结合 Mycat2 注册的逻辑 Schema（表、列、类型、分片元数据）做语义校验；**SqlToRelConverter** 把 SqlNode 转成**关系代数计划树 RelNode**；**Optimizer**（HepPlanner 跑 RBO、VolcanoPlanner 跑 CBO）做等价变换；最后交给 Mycat2 自己的 Router/Executor，翻译成下发给各分片 MySQL 的语句。

```text
┌─────────────────────────────────────────┐
│              Mycat2 SQL 处理              │
│                                          │
│  SQL Request                             │
│      │                                   │
│      ▼                                   │
│  ┌──────────────────────────────────┐   │
│  │        Calcite Framework          │   │
│  │                                   │   │
│  │  ┌─────────┐  ┌─────────┐        │   │
│  │  │ Parser  │→│Validator│        │   │
│  │  └────┬────┘  └────┬────┘        │   │
│  │       │            │             │   │
│  │       ▼            ▼             │   │
│  │  ┌──────────────────────┐        │   │
│  │  │      Optimizer       │        │   │
│  │  │  ┌──────┐ ┌───────┐  │        │   │
│  │  │  │ RBO  │ │  CBO  │  │        │   │
│  │  │  └──┬───┘ └───┬───┘  │        │   │
│  │  │     │         │      │        │   │
│  │  │     └────┬────┘      │        │   │
│  │  │          ▼           │        │   │
│  │  │     Optimized Plan   │        │   │
│  │  └──────────┬───────────┘        │   │
│  └─────────────┼────────────────────┘   │
│                │                         │
│                ▼                         │
│        Mycat2 Router                     │
│        (分发到后端 MySQL)                 │
└─────────────────────────────────────────┘
```

注意 Mycat2 与 Calcite 的分工边界：**Calcite 只负责"生成更好的计划"，不负责"执行"**——哪个分片、什么连接、结果如何归并，全部由 Mycat2 在 RelNode 之上自己实现。Mycat2 向 Calcite 注册的是一个虚拟 Catalog，表元数据都来自 schemas 配置，这也是改表结构必须 reload 之后优化器才能"看见"的原因。

# 3. SQL 解析（Parser）

Parser 阶段的产物是 **SqlNode 语法树**，它只描述"写了什么"，不保证"是对的"——`SELECT nmae FROM users` 在 Parser 阶段完全合法，要到 Validator 阶段才因列不存在而报错。Mycat2 配置了 MySQL 兼容的 `SqlParser.Config`（反引号、大小写不敏感），这是 2.x 对 MySQL 8.0 语法支持度好于 1.6.x 的前提。

```java
// Calcite 将 SQL 解析为 AST
String sql = "SELECT id, name FROM users WHERE age > 18";

// 1. 解析
SqlParser parser = SqlParser.create(sql);
SqlNode sqlNode = parser.parseQuery();

// 2. AST 结构
// SqlSelect
//   ├── SqlSelectKeyword(SELECT)
//   ├── selectList: [SqlIdentifier(id), SqlIdentifier(name)]
//   ├── from: SqlIdentifier(users)
//   └── where: SqlBasicCall(>)
//         ├── SqlIdentifier(age)
//         └── SqlNumericLiteral(18)
```

校验后的 SqlNode 被翻译成 RelNode 计划树，这是优化器的真正操作对象。上面这条 SQL 对应一棵三层小树：

```text
LogicalProject(id, name)          ← 投影：只输出 id、name
      │
LogicalFilter(age > 18)           ← 过滤：WHERE 条件
      │
LogicalTableScan(users)           ← 扫描：逻辑表 users（背后是 N 个分片）
```

理解这棵树很关键：**优化规则的本质就是对这棵树做等价重排**。投影下推让 Project 贴近 Scan；过滤下推把 Filter 推到 Scan 之下（在分片侧先过滤）；路由则在 TableScan 这一层判断"扫描范围能否从全部分片收缩到部分分片"。

# 4. RBO（基于规则的优化）

**RBO（Rule-Based Optimization）不依赖统计信息，只做固定规则的等价变换**，在 Mycat2 中由 HepPlanner 按预设规则序列逐条应用。对分库分表中间件来说，RBO 的价值密度远高于 CBO：**决定中间件性能的头号因素不是 join 顺序，而是"这条 SQL 要不要跨分片、跨了之后传输量多大"**，而这两件事几乎都能靠规则解决。

Mycat2 内置优化规则：

| 规则 | 说明 | 示例 |
|------|------|------|
| **谓词下推** | 将 WHERE 条件尽可能下推到数据源 | `WHERE id=1 AND UPPER(name)='A'` → id=1 下推，UPPER 后续处理 |
| **列裁剪** | 只查询需要的列 | `SELECT name` → 不查询 age, city |
| **常量折叠** | 编译时计算常量表达式 | `WHERE 1+2 > age` → `WHERE 3 > age` |
| **子查询展开** | 展开非关联子查询 | `IN (SELECT ...)` → JOIN |
| **LIMIT 下推** | 将 LIMIT 推送到各分片 | `LIMIT 10` → 每个分片取 10 条而非全量 |

## 4.1 LIMIT 下推优化

```sql
-- 原始 SQL
SELECT * FROM users ORDER BY age DESC LIMIT 10;

-- 优化前（Mycat1）：
-- 每个分片取全量 → 归并排序 → 取 10 条
-- 代价：O(N × 分片数)

-- 优化后（Mycat2 + Calcite）：
-- 每个分片取 10 条 → 归并排序 → 取 10 条
-- 代价：O(10 × 分片数)
```

LIMIT 下推的正确性依赖一个隐含前提：**每个分片各取前 N 条后，全局前 N 条必然包含在这 N×分片数 条候选里**——这对 ORDER BY 全排序成立。但带 OFFSET 时下发的是 `LIMIT (offset+limit)`，深分页时每个分片仍要取 offset+limit 条，下推救不了深度分页，这是第 12 篇专门讨论的问题。

## 4.2 投影下推（列裁剪）

列裁剪决定"后端下发 SQL 的 SELECT 列表放哪些列"。直觉上 `SELECT name FROM users` 就只下发 name，但归并语义要求额外的列：**ORDER BY 的排序列、DISTINCT/GROUP BY 的分组列、聚合改写需要的中间列**都必须被"隐形"加进下发 SQL，在最终输出前再裁掉。裁剪收益按**行宽**计：分片表 30 列只查 2 列时，网络传输与归并内存能降一个量级。

## 4.3 过滤下推与不可下推的边界

**能下推的谓词：只涉及单表列、在后端 MySQL 语义下结果确定的表达式**。不能下推的典型情况有三类：涉及分片算法的路由条件（应转为路由计算而非原样下发）；下推后语义不等价的组合——最典型是**没有 ORDER BY 的 LIMIT**，每分片各取 10 条与全局取 10 条的结果集不等价；被函数包裹的分片键（`WHERE MOD(id,2)=1`）丧失路由能力。

> **注意**：分片键等值条件是过滤下推中特殊且最重要的一类——它不改变下发 SQL 本身，而是直接参与**路由计算**，把 TableScan 范围从全部分片收缩到一个或几个分片。谓词下推决定"每个分片扫什么"，分片键路由决定"哪些分片要扫"，二者是两层独立优化，EXPLAIN 排错时也要分开看。

## 4.4 JOIN 下推的条件

JOIN 能否"下推"到分片内本地执行，取决于两侧表的分布是否对齐：

| JOIN 场景 | 能否分片内完成 | 执行方式 |
|-----------|--------------|---------|
| ER 表（子表按父表分片键分布） | 能 | 同分片本地 JOIN，一次下发 |
| 同分片键、同算法的两张分片表 | 能 | 逐分片下发 JOIN |
| 分片表 JOIN 全局表 | 能 | 全局表每分片都有副本，随分片下发（广播 JOIN） |
| 不同分片键的两张分片表 | 不能 | 跨分片 JOIN：驱动侧路由 + 被驱动侧点查/拉取，代价高 |

不匹配时 JOIN 退化为中间件层的跨片匹配——后者在生产环境基本等同于要禁止的查询形态，这也解释了"JOIN 尽量走 ER 表或全局表"这条最佳实践的机制根源。

# 5. CBO（基于成本的优化）

**CBO（Cost-Based Optimization）在多个等价计划之间按估算代价选最优**，Calcite 用 VolcanoPlanner 搜索计划空间，代价由行数估算（选择率）与算子传输成本合成。在中间件场景里，CBO 主要影响**多表 JOIN 的驱动顺序**与广播/拉取策略的选择。Calcite 的 CBO 根据**表统计信息**选择最优执行计划：

```sql
-- 场景：分片表 users (1000万行) JOIN 全局表 dict_city (500行)

-- 计划 A：先扫描 users，再 JOIN dict_city
-- 计划 B：先扫描 dict_city，再 JOIN users
-- CBO 会选择 计划 B（小表驱动大表）
```

**统计信息**：

| 统计项 | 说明 |
|--------|------|
| `rowCount` | 预估行数 |
| `distinctRowCount` | 去重行数 |
| `nullCount` | NULL 值数量 |
| `avgRowSize` | 平均行大小 |

```sql
-- 更新统计信息
ANALYZE TABLE users;
```

> **设计争议**：中间件层到底值不值得做完整 CBO？反方认为：Mycat2 拿到的统计信息往往粗粒度甚至缺失，估算天然不准，而真正执行扫描的后端 MySQL 还会跑**自己的**优化器，两层优化器各按各的统计做决定，叠加效果未必最优；正方认为即使统计粗糙，CBO 对"全局表广播 vs 全量拉取"这类量级差极大的决策仍有价值。实践中 Mycat2 的行为是**RBO 为主、CBO 为辅**：路由与下推靠规则确定，仅在 join 顺序等局部决策上用代价估算——把 90% 的确定性收益拿满，不为剩下 10% 引入计划抖动的风险。

# 6. Mycat2 自定义规则

Calcite 的设计意图就是被二次开发：Mycat2 在标准规则集之外注册了一批自己的 RelOptRule，最有代表性的是把 Filter 条件里的**分片键抽取成路由目标**——这不是普通的改写，而是"优化器规则与路由器的接口"：规则识别出分片键等值条件后生成带定向扫描属性的算子，Executor 据此只连接对应分片。

```java
// Mycat2 注册自定义 Calcite Rule
public class MycatFilterPushDownRule extends RelOptRule {

    public MycatFilterPushDownRule() {
        super(operand(LogicalFilter.class,
            operand(MycatTableScan.class, none())),
            "MycatFilterPushDown");
    }

    @Override
    public void onMatch(RelOptRuleCall call) {
        LogicalFilter filter = call.rel(0);
        MycatTableScan scan = call.rel(1);

        // 将 filter 条件中的分片键条件提取
        // 用于精确路由到特定分片
        RexNode condition = filter.getCondition();
        if (isShardKey(condition)) {
            // 生成仅路由到目标分片的计划
            call.transformTo(new MycatTargetedScan(scan, condition));
        }
    }
}
```

**设计考量**：

- **为什么用规则形态做路由**：路由条件常嵌在 AND/OR 组合、子查询甚至 JOIN 条件深处，手写遍历容易漏；借用 Calcite 的 operand 模式匹配，"Filter over Scan"这种形状能被精确声明，规则可测试、可组合。
- **规则改写必须保持语义等价**：中间件比单体数据库更不能容忍"优化改变结果"，自定义规则上线前应用同数据双跑对比验证。

# 7. 自适应查询计划

优化器输出的计划树最终要落到**执行策略**上。Mycat2 根据路由结果（单节点/多节点/全节点）与查询形态（简单聚合/排序/JOIN）选择不同 Executor，这一步本质是"把计划树的物理算子映射到归并算法"，与第 12 篇结果归并直接衔接。

```java
// Mycat2 根据查询条件自适应选择执行策略
public ExecutionPlan optimize(RouteResult route, SqlNode sqlNode) {
    if (route.isSingleNode()) {
        // 单节点 → 直接转发，无需归并
        return new DirectExecutionPlan(route.getNode(), sqlNode);
    } else if (route.isAllNodes() && isSimpleAggregate(sqlNode)) {
        // 全分片聚合 → 二次聚合策略
        return new TwoPhaseAggregatePlan(route, sqlNode);
    } else if (route.isAllNodes() && hasOrderBy(sqlNode)) {
        // 全分片排序 → 归并排序策略
        return new MergeSortPlan(route, sqlNode);
    }
    return new DefaultExecutionPlan(route, sqlNode);
}
```

**设计考量**：**单分片短路是收益最大的分支**——命中单节点时中间件近似透明代理，延迟只增加一跳网络与协议解析，这就是"分片键设计决定 Mycat 性能上限"的机制根源，优化器再强也算不出没进 WHERE 的分片键。聚合改写决定了归并方式：`AVG(age)` 改写为每分片下发 `SUM(age), COUNT(age)`，归并后再相除；`COUNT(DISTINCT x)` 无法两阶段完成，只能拉明细去重，这是优化器能力边界的典型案例。

# 8. 性能对比

```text
测试：SELECT * FROM orders WHERE user_id = ? ORDER BY create_time DESC LIMIT 100

| 方案            | 4 分片  | 16 分片 | 64 分片 |
|----------------|---------|---------|---------|
| 无优化（全量取） | 120ms   | 450ms   | 2100ms  |
| LIMIT 下推      | 35ms    | 80ms    | 180ms   |
| 精确分片 + 下推 | 8ms     | 8ms     | 8ms     |
```

这张表最重要的信息不是绝对数字（不同硬件必然不同），而是**三个量级的分界**：无优化时耗时随分片数线性甚至超线性增长（传输量、归并内存同时膨胀）；LIMIT 下推把增长斜率压平；精确路由把曲线变成水平线——分片再多，单分片命中的查询成本不变。最后一行就是所有分库分表方案追求的目标形态。

# 9. 引入优化器的代价与使用边界

**天下没有免费的计划树**。Calcite 的代价主要有三：解析与优化的 CPU 开销比 1.6.x 的字符串级改写贵一个量级，因此 Mycat2 按 `schema + SQL 模板`（参数脱敏后）维护**执行计划缓存**，OLTP 场景同一模板只优化一次；优化器自身、元数据与缓存计划都占堆内存，排错需要能读懂 EXPLAIN 输出的算子树；Calcite 语法集与 MySQL 方言仍有差异，遇到解析失败要退回 MySQL 兼容子集写法。同时必须说清边界：跨不同分片键的大表 JOIN、无分片键的深分页、全分片 `COUNT(DISTINCT)`，无论优化器多聪明都要付出跨节点传输的硬成本——**优化器能"把能省的省掉"，不能"把搬数据变成不搬数据"**。

> **行业实践**：排查一条 SQL 在中间件里"为什么慢"，顺序应该是：**先看路由（EXPLAIN 输出的目标分片数），再看下发 SQL 形态（是否带隐形列、LIMIT 是否被改写），最后才看后端 MySQL 自身执行计划**。前两步决定中间件层的传输量与归并成本，第三步才是索引问题。Mycat 1.6.x 用 `explain` 命令查看改写后的下发语句，2.x 用 `EXPLAIN` 查看计划树，形态不同但排查思路一致。

# 10. 小结

| 要点 | 结论 |
|------|------|
| 为什么用 Calcite | 复用成熟 SQL 引擎前端，把中间件从"字符串改写"升级为"计划级优化"，换取 MySQL 语法兼容度与 JOIN/子查询支持 |
| SQL→RelNode | Parser(SqlNode) → Validator(逻辑 Schema 校验) → SqlToRelConverter(关系代数树) → 规则改写 → Executor 翻译成后端 SQL |
| RBO 是主力 | 谓词下推、列裁剪、LIMIT 下推、子查询展开；其中分片键等值/IN 条件参与路由收缩，收益最大 |
| JOIN 下推条件 | ER 表、同键同算法、全局表可分片内完成；不同分片键的 JOIN 退化为高代价跨片执行 |
| CBO 定位 | 依赖统计信息，仅用于 join 顺序等局部决策，RBO 为主 CBO 为辅 |
| 代价与边界 | 解析开销靠计划缓存摊薄；跨片传输的硬成本优化器救不了 |

一句话收束：**Calcite 让 Mycat2 的每一次"少扫一个分片、少传一列数据、少搬一行结果"都成为可证明的等价变换**，但分片键没进 WHERE 这件事，任何优化器都救不回来。下一篇（12 结果归并）继续往下：计划确定之后，多分片的结果是怎么被合回一条流的。
