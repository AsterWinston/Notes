
# 1. 为什么需要分布式主键

在分库分表场景下，单库自增主键（AUTO_INCREMENT）无法保证全局唯一：

```
db0.users:  INSERT → id = 1 (auto_increment)
db1.users:  INSERT → id = 1 (auto_increment)
→ 两条记录 id 冲突！
```

分布式主键需要满足：
- **全局唯一**：跨所有分片不重复
- **趋势递增**：利于 MySQL InnoDB 聚簇索引性能
- **高性能**：高并发下不成为瓶颈

# 2. Mycat2 内置方案

| 方案 | 实现类 | 特点 |
|------|--------|------|
| Snowflake | `SnowflakeIdGenerator` | 推荐，高性能，趋势递增 |
| 数据库序列 | `DatabaseIdGenerator` | 基于 MySQL 自增，简单可靠 |
| UUID | `UUIDGenerator` | 全局唯一但无序 |

# 3. Snowflake 雪花算法

## 3.1 算法原理

```
64 位 long 型 ID:

┌─┬──────────────────────────┬──────────┬──────────┐
│0│     41 位时间戳 (ms)      │ 10 位机器│ 12 位序列 │
└─┴──────────────────────────┴──────────┴──────────┘

41 位时间戳 = 可用 69 年 (从自定义起始时间算起)
10 位机器码 = 1024 个节点
12 位序列号 = 每毫秒 4096 个 ID
```

## 3.2 配置

```json
{
  "tables": {
    "orders": {
      "type": "sharding",
      "autoIncrement": true,
      "sequence": {
        "clazz": "io.mycat.router.sequence.SnowflakeSequenceHandler",
        "properties": {
          "workerId": 1,
          "dataCenterId": 1,
          "startTimestamp": 1700000000000
        }
      }
    }
  }
}
```

## 3.3 使用

```sql
-- 插入时无需指定 ID，Mycat2 自动生成
INSERT INTO orders(user_id, product_id, amount)
VALUES (1, 100, 50.0);

-- 获取上次生成的 ID
SELECT LAST_INSERT_ID();
-- → 7234567890123456789
```

# 4. 数据库序列

```json
{
  "sequence": {
    "clazz": "io.mycat.router.sequence.DatabaseSequenceHandler",
    "properties": {
      "url": "jdbc:mysql://192.168.1.100:3306/mycat_seq",
      "user": "mycat",
      "password": "mycat123",
      "tableName": "sequence",
      "valueColumn": "current_value",
      "nameColumn": "seq_name",
      "step": 1000
    }
  }
}
```

```sql
-- 序列表
CREATE TABLE sequence (
    seq_name    VARCHAR(50) PRIMARY KEY,
    current_value BIGINT NOT NULL,
    increment   INT NOT NULL DEFAULT 1000
);

INSERT INTO sequence VALUES ('orders', 0, 1000);
```

**工作原理**：每个 Mycat2 实例从序列表批量获取（step=1000），缓存到本地内存分配，减少数据库访问。

# 5. 自定义主键生成器

```java
// 实现 SequenceHandler 接口
public class MySequenceHandler implements SequenceHandler {

    @Override
    public long nextId(String schema, String table) {
        // 自定义 ID 生成逻辑
        // 例如：基于 Redis INCR
        return redisTemplate.opsForValue()
            .increment("seq:" + schema + ":" + table);
    }

    @Override
    public void init(Properties properties) {
        // 初始化
    }
}
```

# 6. 主键策略对比

| 方案 | 全局唯一 | 趋势递增 | 性能 | 依赖 | 适用场景 |
|------|---------|---------|------|------|---------|
| Snowflake | ✓ | ✓ | 极高 | 无 | **推荐首选** |
| 数据库序列 | ✓ | ✓ | 中 | MySQL | 简单可靠 |
| UUID | ✓ | ✗ | 高 | 无 | 非聚簇索引表 |
| Redis INCR | ✓ | ✓ | 高 | Redis | 需要全局递增 |
| ZK 序列 | ✓ | ✓ | 中 | ZK | 需要强递增 |

# 7. 时钟回拨处理

Snowflake 依赖系统时钟，时钟回拨可能导致 ID 重复。

```
Mycat2 Snowflake 的容错策略：
1. 检测到时钟回拨（当前时间 < 上次生成时间）
2. 如果回拨 < 5ms → 等待追平
3. 如果回拨 > 5ms → 使用"未来时间"生成（容忍少量回拨）
4. 记录告警日志
```

# 8. 最佳实践

1. **Snowflake 首选**：无外部依赖，性能最高，满足大部分场景
2. **workerId 唯一**：每个 Mycat2 实例的 workerId 必须唯一，可通过 ZK/DB 自动分配
3. **避免对外暴露 ID 规律**：生产环境不要用自增 ID 暴露业务规模
4. **索引优化**：趋势递增 ID 对 InnoDB 聚簇索引最友好
5. **时钟同步**：Snowflake 依赖节点间时钟基本同步，建议配置 NTP
