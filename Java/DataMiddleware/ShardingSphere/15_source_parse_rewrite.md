# 1. 解析引擎架构

解析引擎是 ShardingSphere 一切增值能力的地基：**没有语法树就没有路由，也没有安全的改写**——分片键提取、表名替换、分页放大、聚合拆解全部建立在 SQL 被完整解析为 AST 的前提上。整体流水线：

```text
SQL 字符串
  │
  ▼
┌──────────────────────────────┐
│        ANTLR 4 Parser        │
│  MySQLLexer / PgLexer ...    │
│  MySQLParser / PgParser ...  │
└──────────────┬───────────────┘
               │ Parse Tree
               ▼
┌──────────────────────────────┐
│     SQLVisitor (方言适配)     │
│  MySQLStatementVisitor        │
│  PostgreSQLStatementVisitor   │
└──────────────┬───────────────┘
               │ SQLStatement
               ▼
┌──────────────────────────────┐
│   ShardingSphere SQLStatement │
│  SelectStatement              │
│  InsertStatement ...          │
│  UpdateStatement ...          │
└──────────────────────────────┘
```

注意输出物的命名：ShardingSphere 内部统一叫 **SQLStatement（语法树/语法片段 Segment 模型）**，社区文章里常说的"AST"是泛指；严格说 ANTLR 产出的 `ParserRuleContext` 是 parse tree，Visitor 遍历后丢弃无关节点、组装成带语义的 `SQLStatement` 才是引擎使用的树。

## 1.1 解析器选型：为什么是 ANTLR 而不是其他

选型对比是源码篇必问点，三条路线的取舍：

| 方案 | 代表 | 优势 | 劣势 | 结论 |
|------|------|------|------|------|
| 自研解析器 | ShardingSphere 3.x/4.x | 无新依赖 | 语法覆盖靠手写、多方言不可维护 | 5.0 整体废弃 |
| ANTLR 4 + g4 语法文件 | ShardingSphere 5.x | 多方言复用工具链、语法可读可测 | 冷启动慢、jar 偏大、极端 SQL 性能一般 | **现状** |
| SQL 解析器库（JSqlParser/Druid） | Calcite / 部分中间件 | 开箱即用 | 抽象树信息粒度不够改写；Druid 协议问题 | 仅 Translator 用 JSqlParser |

社区讨论（BSF 提案与 Calcite 集成评估）里反复权衡的是**要不要用 Calcite 的 SqlParser 替代 ANTLR**：Calcite 语法更 SQL 标准、且联邦查询已经在用 Calcite 核心，但它对 MySQL/PG 方言扩展语法的覆盖需要自己写 parser extension，改造成本远高于收益。最终格局是**分工而非替代：ANTLR 管方言解析与改写，Calcite 只管联邦查询执行计划**（见 14 篇），一个引擎内两套 SQL 处理栈并存。

## 1.2 缓存与冷启动优化

ANTLR 首次构建语法预测表（ATN）较慢，5.x 做了两项工程优化：SQL 解析结果缓存（`!SQL_PARSER` 规则的 cache 配置，相同 SQL 模板跳过重复解析）与解析失败快速降级（超长/极端 SQL 解析代价过高时直接放弃，见第 8 节）——**大 IN 列表、批量 VALUES 类超长 SQL 是解析失败重灾区**，也是"某条大 INSERT 没触发主键改写"类问题的常见根因。

# 2. ANTLR 语法文件

```antlr
// MySQL 语法片段
grammar MySQL;

select
    : SELECT selectSpec* selectElements
      fromClause? whereClause? orderByClause? limitClause?
    ;

fromClause
    : FROM tableReferences
    ;

whereClause
    : WHERE expr
    ;
```

语法文件按方言分目录维护（`MySQL.g4`、`PostgreSQL.g4`、`SQL92.g4` 等），生成 Lexer/Parser 后由 Visitor 层消费。**g4 文件是理解"什么 SQL 能进分片链路"的唯一权威文档**：排查改写不生效时，先在语法文件里确认该子句是否存在——例如某版本的 g4 未覆盖 `WITH` 公共表表达式，所有 CTE 查询都走直通。语法升级节奏跟大版本走，方言新版本特性（MySQL 8.0 窗口函数、PG 15 MERGE）的支持有滞后窗口，选型评估时要把"语法覆盖清单"当作验收项。

# 3. SQLStatement 模型

```java
public abstract class SQLStatement {

    // 数据库类型
    private DatabaseType databaseType;

    // 参数标记数量
    private int parameterCount;

    // 表段
    private Collection<SimpleTableSegment> tables = new LinkedList<>();

    // 获取所有表名
    public Collection<SimpleTableSegment> getTables() { ... }
}

// SelectStatement 示例
public class MySQLSelectStatement extends SelectStatement {

    private SelectSegment select;
    private FromSegment from;
    private WhereSegment where;
    private OrderBySegment orderBy;
    private LimitSegment limit;

    // 获取 WHERE 条件中的分片键值
    public Optional<ShardingCondition> getShardingCondition() {
        if (where != null) {
            return extractShardingValue(where.getExpr());
        }
        return Optional.empty();
    }
}
```

## 3.1 Segment 模型的三个设计要点

**① 片段（Segment）携带位置信息**：每个 Segment 记录 `startIndex/stopIndex`，这是改写引擎能"原位替换"的前提——改写不需要重新拼 SQL，只按位置替换子串，原始 SQL 的风格（注释、大小写、空白）得以保留。

**② 表达式树独立建模**：WHERE/ON 中的条件抽象为 `ExpressionSegment` 树（`BinaryOperationExpression`、`InExpression`、`BetweenExpression` 等），分片值提取就是对该树做一次遍历，遇到 `column = ?` 且 column 命中分片键就记录参数下标——**参数下标而非字面值**，因为 Prepared 场景值在参数包里。

**②.5 子查询递归**：`SubquerySegment` 自引用，路由引擎递归进子查询提取分片值，因此"子查询里的分片键"也能被利用；但递归深度受解析器保护，超深嵌套直接解析失败。

**③ 抽象语句 + 方言语句双层**：`SelectStatement` 是规则引擎面对的抽象基类，`MySQLSelectStatement` 等继承并携带方言扩展信息。规则引擎永远只 import 抽象类，这是新增方言不动业务引擎的隔离边界。

> **设计考量**：为什么不用成熟的通用 AST（Calcite SqlNode、JSqlParser 树）？因为改写需要**精确回写位置**和**参数占位下标**，通用 AST 面向"语义求值"设计，不保留源码偏移量；ShardingSphere 的 Segment 模型是"可回写的语法树"，与纯求值树是两种目标函数，这是它自研树形的根本原因。

# 4. 改写引擎核心

改写引擎的模型是"**收集 Token → 排序 → 拼接**"，而非递归修改语法树：

```java
// SQLRewriteEngine — 改写入
public final class SQLRewriteEngine {

    private final SQLStatement sqlStatement;
    private final List<SQLToken> sqlTokens = new LinkedList<>();

    public SQLRewriteEntry rewrite(String sql, RouteContext routeContext) {
        // 1. 收集所有改写 Token
        collectTokens(routeContext);

        // 2. 按位置排序
        sqlTokens.sort(Comparator.comparingInt(SQLToken::getStartIndex));

        // 3. 生成改写后的 SQL
        StringBuilder result = new StringBuilder();
        int lastIndex = 0;
        for (SQLToken token : sqlTokens) {
            result.append(sql, lastIndex, token.getStartIndex());
            result.append(token.accept(generator));  // 替换内容
            lastIndex = token.getStopIndex() + 1;
        }
        result.append(sql.substring(lastIndex));
        return new SQLRewriteEntry(result.toString(), sqlTokens);
    }
}
```

5.x 重构后的实际形态：改写按 `SQLRewriteEntry`（SQL + 参数列表）输出，同一逻辑 SQL 对每个路由单元各生成一份——**参数也要改写**（主键补值、加密替换、分页参数放大都涉及参数列表变化），所以排序键是 Token 位置、载荷是"文本段 + 参数段"二元组。Token 由 `TokenGenerateEngine`（按语句类型分发 Insert/Select/Update/Delete）统一生成，天然支持多 Token 叠加：一条 INSERT 同时触发表名替换 + 主键补列两个 Token，互不冲突，因为位置不同。

# 5. 表名改写

```java
public class TableNameToken implements SQLToken {

    private final String logicalTable;
    private final String actualTable;

    @Override
    public String toString() {
        return actualTable;  // t_order → t_order_0
    }
}

// 改写逻辑
// SELECT * FROM t_order WHERE id = 1
// → SELECT * FROM t_order_0 WHERE id = 1
//                  ^^^^^^^^ 替换
```

表名改写有两个易被忽略的细节。**其一，改写只针对路由单元内确定的表映射**：UPDATE 语句默认不允许广播多路由（避免一条逻辑 UPDATE 改 N 个分片，需显式接受该语义），DELETE 同理受路由引擎约束。**其二，别名与多实例**：同一逻辑表在 SQL 中出现多次（自关联 `t_order a JOIN t_order b`），每个出现位置都独立生成 Token 替换为同一实际表名，自关联在分片下按路由单元内闭环执行——**自关联的分片键必须能同时约束两侧**，否则退化为笛卡尔积路由。

# 6. 自增主键改写

```java
public class GeneratedKeyInsertToken implements SQLToken {

    private final String column;
    private final Comparable<?> generatedValue;  // Snowflake ID

    @Override
    public String toString() {
        // INSERT INTO t_order (user_id) VALUES (?)
        // → INSERT INTO t_order_0 (order_id, user_id) VALUES (?, ?)
        //                           ^^^^^^^^             ^
        return column + ", ";  // 添加主键列
    }
}
```

批量插入（多 VALUES 组）时每个值组独立取号：`INSERT INTO t_order (user_id) VALUES (?),(?),(?)` 改写成三列三值，一次号段取 3 个。**主键补值的插入点在"主键列缺失"判定之后**：若 SQL 显式给了 order_id（如 MP 的 ASSIGN_ID 客户端生成），生成器不触发，改写只换表名——这就是"两套雪花并存"问题的源码依据（13 篇）。与数据库自增的协作：JDBC `getGeneratedKeys()` 在分片下返回的是 ShardingSphere 自己维护的生成值集合，而不是逐物理连接的 last_insert_id，跨分片批量插入的 `getGeneratedKeys` 结果顺序与输入值组一一对应。

# 7. 分页改写

```java
// 分页优化：LIMIT OFFSET → LIMIT
// MySQL:
// SELECT * FROM t_order LIMIT 100, 10
// → 每个分片: SELECT * FROM t_order_N LIMIT 0, 110
//   （取 110 = 100 + 10）
//   归并时跳过前 100 条，取 10 条返回

// PostgreSQL:
// SELECT * FROM t_order LIMIT 10 OFFSET 100
// → 同上逻辑
```

**正确性依据**：全局第 101-110 名必然位于某个分片的前 110 名之内，所以每片取前 offset+limit 条再归并即可无损。改写动作是把 `offset=100, limit=10` 改为 `offset=0, limit=110`（参数版则改参数值），归并层的 `SkipEnumerationDecoratorMergedResult` 负责丢弃。代价模型要直白：扇出 k 个分片时物理层共读 `k × (offset+limit)` 行，**深分页的总数据量随分片数线性放大**——10000 偏移 × 16 分片 = 16 万行传输，这就是"分片环境禁深度 OFFSET"的源码级解释（19 篇游标分页、8 篇 order-by 优化联动）。

## 7.1 聚合改写

```java
// AVG 改写
// 原始: SELECT AVG(amount) FROM t_order
// → 改写为: SELECT SUM(amount), COUNT(amount) FROM t_order_N
//   各分片结果归并: AVG = SUM(SUM) / SUM(COUNT)
```

聚合改写的完整家族：**AVG 拆成 SUM+COUNT**（唯一必须改写的函数，因为平均不可二次平均）；**SUM/MAX/MIN/COUNT 不改写**，各分片算局部值、归并层再聚合（`AverageAggregationUnit`、`SumAggregationUnit` 等对应 17 篇归并函数族）；**DISTINCT 聚合（COUNT(DISTINCT x)）无法精确改写**——各片局部去重后求和会重复计数，5.x 的处理是广播取回明细在内存去重（大基数直接内存风险）或提示走近似/ES；**GROUP_CONCAT/STRING_AGG 这类有序拼接函数默认不支持分片聚合**，解析段虽存在但归并语义无对应单元，结果按直通片顺序返回不保证全局顺序。

# 8. 解析失败降级：直通策略

解析不是硬依赖，5.x 的降级语义是**"解析失败 → SQLStatement 为空 → 跳过改写按原 SQL 直通单数据源"**：

```text
解析链路
  ├─ 解析成功 ──▶ 提取分片值 ──▶ 路由 ──▶ 改写 ──▶ 执行
  └─ 解析失败 ──▶ SQLStatement = null
                    ├─ 单数据源场景：直通，功能正常，无分片能力
                    └─ 多数据源/分片表场景：
                        无法提取分片值 → 全分片广播（DML 危险）
                        或报 SqlParseException（按 props 配置行为）
```

`sql-parse-failure-continue`（旧版 `system-log-level` 相关行为）控制失败时是继续直通还是抛错；日志侧降级打印 WARN。**直通不等于安全**：SELECT 广播多片后归并结果仍正确（只是代价大），但 UPDATE/DELETE 广播 = 一次执行改 N 片，所以路由引擎对无分片值的 DML 有拦截开关（`allow-range-query-with-inline-sharding` 管范围，DML 广播需显式确认）。工程建议：**生产监控 SQL 解析失败计数，把它当告警项**——某类 SQL 持续解析失败意味着它在无声地走广播或跳过主键生成，等数据出问题再发现就晚了。

> **行业实践**：大厂内部推 ShardingSphere 时通常配三件套防线：sql-show 日志抽样审计改写正确性、解析失败告警、以及上线前对全量业务 SQL 做 `PARSE SQL`/`PREVIEW` 批量预检（DistSQL 篇），把"语法覆盖盲区"在发布前而不是故障时暴露。

# 9. 小结

1. **ANTLR 管方言解析、Calcite 只做联邦执行**，选型核心是 Segment 树保留回写位置
2. **路由在改写之前**：Token 按位置替换 + 参数同步改写，是物理 SQL 与逻辑 SQL 差异的来源
3. **主键补值仅在列缺省时触发**，ORM 已显式给 ID 则生成器不介入
4. **分页改写=每片取 offset+limit，聚合改写=AVG 拆 SUM+COUNT**，其余按结合律直接下推
5. **解析失败静默降级直通**，生产必须把解析失败计数做成告警项