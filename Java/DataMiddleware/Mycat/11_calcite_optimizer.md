
# 1. 为什么选择 Calcite

Mycat2 选择 Apache Calcite 作为 SQL 解析引擎，而非自研解析器：

| 维度 | 自研解析器（Mycat1 的 Druid） | Apache Calcite |
|------|---------------------------|---------------|
| SQL 兼容性 | 需逐个语法适配 | 业界标准，兼容性好 |
| 查询优化 | 简单改写 | RBO + CBO 完整优化 |
| 维护成本 | 高（需持续跟进 MySQL 新语法） | 低（社区维护） |
| 开发效率 | 低 | 高 |

# 2. Calcite 集成架构

```
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

# 3. SQL 解析（Parser）

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

# 4. RBO（基于规则的优化）

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

# 5. CBO（基于成本的优化）

Calcite 的 CBO 根据**表统计信息**选择最优执行计划：

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

# 6. Mycat2 自定义规则

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

# 7. 自适应查询计划

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

# 8. 性能对比

```
测试：SELECT * FROM orders WHERE user_id = ? ORDER BY create_time DESC LIMIT 100

| 方案            | 4 分片  | 16 分片 | 64 分片 |
|----------------|---------|---------|---------|
| 无优化（全量取） | 120ms   | 450ms   | 2100ms  |
| LIMIT 下推      | 35ms    | 80ms    | 180ms   |
| 精确分片 + 下推 | 8ms     | 8ms     | 8ms     |
```
