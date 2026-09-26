
# 1. 过滤概述

Canal 的数据过滤分为两个层面：
1. **Server 端过滤**：在 EventParser/EventSink 层面对 binlog 事件进行过滤，减少不必要的数据传输
2. **Client 端过滤**：通过订阅规则，只在客户端接收感兴趣的数据

```
┌─────────────────────────────────────────────────────────┐
│                    过滤层次                              │
│                                                         │
│  ① Server 端过滤（instance.properties）                 │
│     canal.instance.filter.regex = .*\\..*                │
│     canal.instance.filter.black.regex =                  │
│                                                         │
│  ② Sink 层过滤（Aviator 表达式）                        │
│     可以在 Sink 阶段做复杂条件过滤                       │
│                                                         │
│  ③ Client 端过滤（客户端 subscribe）                    │
│     connector.subscribe("test\\..*")                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

# 2. Server 端表/库过滤

## 2.1 白名单过滤

```properties
# instance.properties

# 订阅所有库的所有表
canal.instance.filter.regex = .*\\..*

# 只订阅 test 库
canal.instance.filter.regex = test\\..*

# 只订阅 test 库的 user 表
canal.instance.filter.regex = test\\.user

# 订阅 test 库的所有表和 prod 库的 order 表
canal.instance.filter.regex = test\\..*,prod\\.order

# 订阅 test 库下以 t_ 开头的表
canal.instance.filter.regex = test\\.t_.*
```

## 2.2 黑名单过滤

```properties
# instance.properties

# 排除 test 库的 temp_ 开头的表
canal.instance.filter.black.regex = test\\.temp_.*

# 排除所有库的 backup_ 开头的表
canal.instance.filter.black.regex = .*\\.backup_.*

# 排除指定库的表
canal.instance.filter.black.regex = test\\.log_table,test\\.audit_table

# 黑名单支持复杂正则
canal.instance.filter.black.regex = test\\.temp_.*,.*\\._bak_.*
```

## 2.3 过滤实现源码

```java
public class AviaterRegexFilter {
    
    private Pattern pattern;
    private String regex;
    
    public AviaterRegexFilter(String regex) {
        this.regex = regex;
        if (StringUtils.isNotEmpty(regex)) {
            // 将多个正则用 | 连接
            String[] parts = regex.split(",");
            StringBuilder combined = new StringBuilder();
            for (String part : parts) {
                if (combined.length() > 0) combined.append("|");
                combined.append("(").append(part.trim()).append(")");
            }
            this.pattern = Pattern.compile(combined.toString(), Pattern.CASE_INSENSITIVE);
        }
    }
    
    /**
     * 判断是否匹配
     * @param value 格式: "schema.table"
     */
    public boolean filter(String value) {
        if (pattern == null) {
            return true;  // 没有过滤规则，全部通过
        }
        return pattern.matcher(value).matches();
    }
}

// 在 EntryEventSink 中使用
public class EntryEventSink {
    
    private AviaterRegexFilter filter;
    private AviaterRegexFilter blackFilter;
    
    private boolean applyFilter(CanalEntry.Entry entry) {
        if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN
                || entry.getEntryType() == EntryType.TRANSACTIONEND
                || entry.getEntryType() == EntryType.HEARTBEAT) {
            return true;  // 事务边界和心跳不拦截
        }
        
        String schemaName = entry.getHeader().getSchemaName();
        String tableName = entry.getHeader().getTableName();
        String fullName = schemaName + "." + tableName;
        
        // 先检查黑名单
        if (blackFilter != null && blackFilter.filter(fullName)) {
            return false;  // 黑名单命中，丢弃
        }
        
        // 再检查白名单
        if (filter != null && !filter.filter(fullName)) {
            return false;  // 白名单未命中，丢弃
        }
        
        return true;
    }
}
```

# 3. Aviator 表达式过滤

Canal 支持使用 Aviator 表达式引擎进行更复杂的条件过滤。Aviator 是一个高性能的 Java 表达式求值引擎。

## 3.1 基本语法

```properties
# 按表名过滤
canal.instance.filter.query.dsl = table.name == 'user'

# 按库名+表名过滤
canal.instance.filter.query.dsl = schema.name == 'test' && table.name == 'user'

# 多表过滤
canal.instance.filter.query.dsl = table.name in ('user','order','product')

# 按事件类型过滤（只订阅 INSERT 和 UPDATE）
canal.instance.filter.query.dsl = event.type in ('INSERT','UPDATE')
```

## 3.2 Aviater 过滤器实现

```java
public class AviaterFilter {
    
    private Expression expression;
    private String dsl;
    
    /**
     * 编译 Aviator 表达式
     */
    public AviaterFilter(String dsl) {
        this.dsl = dsl;
        if (StringUtils.isNotEmpty(dsl)) {
            this.expression = AviatorEvaluator.compile(dsl, true);
        }
    }
    
    /**
     * 对 Entry 进行条件过滤
     */
    public boolean filter(CanalEntry.Entry entry) {
        if (expression == null) {
            return true;
        }
        
        try {
            Header header = entry.getHeader();
            
            // 构建表达式上下文
            Map<String, Object> env = new HashMap<>();
            env.put("schema.name", header.getSchemaName());
            env.put("table.name", header.getTableName());
            env.put("event.type", header.getEventType().name());
            env.put("event.length", header.getEventLength());
            env.put("execute.time", header.getExecuteTime());
            
            // 如果是 ROWDATA，进一步提取信息
            if (entry.getEntryType() == EntryType.ROWDATA) {
                enrichRowDataContext(entry, env);
            }
            
            // 执行表达式
            Object result = expression.execute(env);
            return result instanceof Boolean ? (Boolean) result : true;
            
        } catch (Exception e) {
            logger.error("aviator filter error, allow by default", e);
            return true;  // 表达式执行异常时，默认通过
        }
    }
    
    /**
     * 将行数据信息注入表达式上下文
     */
    private void enrichRowDataContext(CanalEntry.Entry entry, Map<String, Object> env) {
        try {
            RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
            if (!rowChange.getRowDatasList().isEmpty()) {
                RowData rowData = rowChange.getRowDatasList().get(0);
                
                // 提取 after 列值
                for (Column column : rowData.getAfterColumnsList()) {
                    env.put("column." + column.getName(), column.getValue());
                    env.put("column." + column.getName() + ".updated", column.getUpdated());
                }
                
                // 提取 before 列值
                for (Column column : rowData.getBeforeColumnsList()) {
                    env.put("old." + column.getName(), column.getValue());
                }
            }
        } catch (Exception e) {
            // 忽略解析错误
        }
    }
}
```

## 3.3 Aviator 过滤示例

**示例1：只订阅特定用户的数据**

```java
// Aviator DSL
String dsl = "table.name == 'user' && column.status != 'deleted'";

// 或者过滤特定 ID 范围
String dsl = "table.name == 'order' && column.amount > 100";
```

**示例2：复杂条件组合**

```java
String dsl = """
    (table.name == 'user' && event.type == 'UPDATE' && column.status.updated == true)
    ||
    (table.name == 'order' && event.type == 'INSERT')
    """;
```

**示例3：按时间过滤**

```java
String dsl = "execute.time > 1672545600000";
```

# 4. 列级过滤

Canal 支持对特定表的特定列进行过滤，使用 `canal.instance.filter.field` 配置：

```properties
# field 过滤格式：schema.table:column1|column2
# 多个表用逗号分隔

# 只订阅 user 表的 name 和 email 列
canal.instance.filter.field = test.user:name|email

# 多个表配置
canal.instance.filter.field = test.user:id|name|email,test.order:id|status

# 排除特定列使用 ~ 前缀（保留除该列外的所有列）
# 注意：~ 功能在不同版本可能有差异
```

## 4.1 列过滤实现

```java
public class ColumnFilter {
    
    // schema.table -> set of column names
    private Map<String, Set<String>> columnFilterMap = new HashMap<>();
    private Map<String, Set<String>> columnExcludeMap = new HashMap<>();  // ~ 语法
    
    public ColumnFilter(String filterConfig) {
        if (StringUtils.isEmpty(filterConfig)) return;
        
        // 解析配置: test.user:name|email,test.order:id|status
        String[] tableConfigs = filterConfig.split(",");
        for (String config : tableConfigs) {
            String[] parts = config.split(":");
            if (parts.length == 2) {
                String key = parts[0].trim();
                String[] columns = parts[1].trim().split("\\|");
                
                Set<String> includes = new HashSet<>();
                Set<String> excludes = new HashSet<>();
                
                for (String col : columns) {
                    if (col.startsWith("~")) {
                        excludes.add(col.substring(1));
                    } else {
                        includes.add(col);
                    }
                }
                
                if (!includes.isEmpty()) {
                    columnFilterMap.put(key, includes);
                }
                if (!excludes.isEmpty()) {
                    columnExcludeMap.put(key, excludes);
                }
            }
        }
    }
    
    /**
     * 过滤 RowChange 中的列
     */
    public void filter(String schema, String table, RowData rowData) {
        String key = schema + "." + table;
        
        Set<String> includes = columnFilterMap.get(key);
        Set<String> excludes = columnExcludeMap.get(key);
        
        if (includes == null && excludes == null) {
            return;  // 没有过滤规则
        }
        
        // 处理 afterColumns
        if (!rowData.getAfterColumnsList().isEmpty()) {
            List<Column> filtered = filterColumns(rowData.getAfterColumnsList(), 
                                                     includes, excludes);
            rowData.clearAfterColumns();
            rowData.addAllAfterColumns(filtered);
        }
        
        // 处理 beforeColumns
        if (!rowData.getBeforeColumnsList().isEmpty()) {
            List<Column> filtered = filterColumns(rowData.getBeforeColumnsList(),
                                                     includes, excludes);
            rowData.clearBeforeColumns();
            rowData.addAllBeforeColumns(filtered);
        }
    }
    
    private List<Column> filterColumns(List<Column> columns,
                                        Set<String> includes,
                                        Set<String> excludes) {
        List<Column> result = new ArrayList<>();
        for (Column col : columns) {
            if (includes != null && !includes.isEmpty()) {
                if (includes.contains(col.getName())) {
                    result.add(col);
                }
            } else if (excludes != null && !excludes.isEmpty()) {
                if (!excludes.contains(col.getName())) {
                    result.add(col);
                }
            }
        }
        return result;
    }
}
```

# 5. 事件类型过滤

## 5.1 只订阅特定事件类型

```properties
# 只订阅 INSERT 和 UPDATE 事件
canal.instance.filter.query.dsl = event.type in ('INSERT','UPDATE')

# 排除 DELETE 事件
canal.instance.filter.query.dsl = event.type != 'DELETE'
```

## 5.2 客户端事件类型过滤

```java
public class EventTypeFilter {
    
    // 客户端层面过滤
    public boolean accept(Entry entry, Set<EventType> acceptedTypes) {
        if (entry.getEntryType() != EntryType.ROWDATA) {
            return true;
        }
        
        EventType eventType = entry.getHeader().getEventType();
        return acceptedTypes.contains(eventType);
    }
}

// 使用示例
Set<EventType> accepted = EnumSet.of(EventType.INSERT, EventType.UPDATE);

for (Entry entry : message.getEntries()) {
    if (eventTypeFilter.accept(entry, accepted)) {
        processEntry(entry);
    }
}
```

# 6. 动态过滤规则管理

## 6.1 通过 Canal Admin 动态修改

Canal Admin 提供了 Web 界面动态修改过滤规则的能力：

```
1. 登录 Canal Admin
2. 进入 Instance 管理 → 选择目标 Instance
3. 修改 "过滤规则" 字段
4. 保存并应用（无需重启）
```

## 6.2 通过 API 动态修改

```bash
# 查看当前过滤规则
curl http://localhost:8089/api/v1/canal/instance/log/example

# 修改过滤规则
curl -X PUT http://localhost:8089/api/v1/canal/instance/filter/example \
  -H "Content-Type: application/json" \
  -d '{
    "filter": "test\\..*,prod\\.order"
  }'
```

## 6.3 客户端动态修改

```java
// 运行时切换过滤规则
connector.unsubscribe();  // 先取消
connector.subscribe("test\\.user");  // 重新订阅
```

# 7. 过滤性能考虑

## 7.1 过滤时机优化

```
过滤时机对性能的影响：

EventParser 层过滤 ── 最早，最省资源（避免后续所有处理）
        │
EventSink 层过滤   ── 中等（Aviator 表达式在这里执行）
        │
Client 端过滤      ── 最晚，但最灵活
```

**建议**：
- 表/库级别的固定过滤 → 放在 Server 端（`filter.regex`）
- 复杂的行级条件过滤 → 放在 Sink 层（Aviator DSL）
- 动态的、业务相关的过滤 → 放在 Client 端

## 7.2 正则表达式性能

```java
// 预编译正则表达式（Canal 内部已做）
public class OptimizedRegexFilter {
    
    // 使用 Guava Cache 缓存编译后的 Pattern
    private static final Cache<String, Pattern> PATTERN_CACHE = 
        CacheBuilder.newBuilder()
            .maximumSize(1000)
            .expireAfterAccess(1, TimeUnit.HOURS)
            .build();
    
    public static boolean matches(String value, String regex) {
        Pattern pattern = PATTERN_CACHE.getIfPresent(regex);
        if (pattern == null) {
            pattern = Pattern.compile(regex, Pattern.CASE_INSENSITIVE);
            PATTERN_CACHE.put(regex, pattern);
        }
        return pattern.matcher(value).matches();
    }
}
```

## 7.3 Aviator 表达式性能

```java
// Aviator 表达式编译缓存
public class AviatorFilterCache {
    
    // Aviator 自带编译缓存，但也可以加一层
    private static final Cache<String, Expression> EXPRESSION_CACHE =
        CacheBuilder.newBuilder()
            .maximumSize(500)
            .build();
    
    public static boolean evaluate(String dsl, Map<String, Object> env) {
        Expression expression = EXPRESSION_CACHE.getIfPresent(dsl);
        if (expression == null) {
            expression = AviatorEvaluator.compile(dsl, true);  // true = 缓存
            EXPRESSION_CACHE.put(dsl, expression);
        }
        return (Boolean) expression.execute(env);
    }
}
```

# 8. 过滤场景实战

## 8.1 缓存同步场景（只关注核心表）

```properties
# 只订阅需要更新缓存的表
canal.instance.filter.regex = db_user\\.user,db_user\\.user_profile,db_product\\.product
```

## 8.2 ES 同步场景（排除大字段表）

```properties
# 只订阅需要建立索引的表
canal.instance.filter.regex = db_article\\.article
canal.instance.filter.black.regex = db_article\\.article_content  # 排除内容大字段表
```

## 8.3 数据归档场景（过滤已完成数据）

```java
// 只处理已完成的订单
String aviatorDsl = """
    table.name == 'order'
    && event.type == 'UPDATE'
    && column.status == 'COMPLETED'
    """;
```

## 8.4 多租户场景（按租户过滤）

```java
// 只处理租户 A 的数据
String aviatorDsl = """
    table.name == 'data'
    && column.tenant_id == 'tenant_a'
    """;
```
