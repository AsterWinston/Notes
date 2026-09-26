
# 1. JDBC 集成

SkyWalking Agent 通过拦截 `java.sql.Driver` 接口自动追踪数据库访问：

```java
// 自动拦截的 JDBC 操作：
Connection.prepareStatement()        → 创建 Exit Span
PreparedStatement.executeQuery()     → 执行 SQL
PreparedStatement.executeUpdate()    → 执行更新
```

## 1.1 Span 标签

| 标签 | 说明 | 示例 |
|------|------|------|
| `db.type` | 数据库类型 | `mysql`, `postgresql` |
| `db.instance` | 数据库名 | `order_db` |
| `db.statement` | SQL 语句 | `SELECT * FROM t_order WHERE id = ?` |
| `db.bind_variables` | 绑定参数 | `[1]`（需开启） |

## 1.2 配置

```properties
# agent.config
plugin.jdbc.trace_sql_parameters=true         # 追踪 SQL 参数
plugin.jdbc.sql_parameters_max_length=512     # 参数最大长度
```

# 2. 慢 SQL 识别

```properties
# 超过 100ms 标记为慢 SQL
plugin.jdbc.slow_sql_millis_threshold=100
```

```java
// 效果：在 Span 中标记
// Tags: db.slow_sql=true, db.slow_sql_threshold=100
```

# 3. 连接池监控

| 连接池 | 支持 |
|--------|------|
| HikariCP | ✓ 自动 |
| Druid | ✓ 自动 |
| DBCP2 | ✓ 自动 |
| Tomcat JDBC Pool | ✓ 自动 |

连接池指标通过 JMX 采集，在 SkyWalking UI 的 Instance Dashboard 中查看。

# 4. Redis 集成

## 4.1 支持的客户端

| 客户端 | 支持 |
|--------|------|
| Jedis | ✓ |
| Lettuce | ✓ |
| Redisson | ✓ |

## 4.2 Span 标签

| 标签 | 说明 |
|------|------|
| `db.type` | `Redis` |
| `db.statement` | Redis 命令（如 `GET key`） |
| `db.redis.cluster` | `true`/`false` |

```properties
# agent.config
plugin.redis.trace_redis_parameters=true
```

# 5. MongoDB 集成

```properties
# agent.config
plugin.mongodb.trace_param=true
plugin.mongodb.filter_length_limit=256
```

| 标签 | 说明 |
|------|------|
| `db.type` | `MongoDB` |
| `db.instance` | 数据库名 |
| `db.statement` | 查询语句 |

# 6. Elasticsearch 集成

| 客户端 | 支持 |
|--------|------|
| RestHighLevelClient | ✓ |
| RestClient | ✓ |
| TransportClient | ✓ |

# 7. 消息队列集成

| MQ | 支持 | 追踪方式 |
|----|------|---------|
| Kafka | ✓ | Producer + Consumer Span |
| RocketMQ | ✓ | Producer + Consumer Span |
| RabbitMQ | ✓ | 发送/消费 Span |

```
Kafka Producer:
  Span: Kafka/topic-name/Producer

Kafka Consumer:
  Span: Kafka/topic-name/Consumer
  sw8 自动注入到 Kafka Header
```

# 8. 性能开销

| 操作 | 额外开销 |
|------|---------|
| JDBC | < 5μs（约 0-1%） |
| Redis | < 3μs（可忽略） |
| Kafka | < 10μs（含 Header 处理） |

# 9. 最佳实践

1. **生产环境关闭 SQL 参数**：`trace_sql_parameters=false` 防止敏感数据泄露
2. **SQL 参数长度限制**：生产开启时设置 `sql_parameters_max_length=256`
3. **Redis Pipeline**：Pipeline 模式下所有命令合并为一个 Span
4. **MQ 消费端 Span**：消费端 Span 的 TraceId 来自生产端，自动串联
