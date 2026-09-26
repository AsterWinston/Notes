
# 1. 解析引擎架构

```
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
│  InsertStatement              │
│  UpdateStatement ...          │
└──────────────────────────────┘
```

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

# 4. 改写引擎核心

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

# 8. 聚合改写

```java
// AVG 改写
// 原始: SELECT AVG(amount) FROM t_order
// → 改写为: SELECT SUM(amount), COUNT(amount) FROM t_order_N
//   各分片结果归并: AVG = SUM(SUM) / SUM(COUNT)
```
