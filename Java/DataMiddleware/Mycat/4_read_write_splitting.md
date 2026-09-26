
# 1. 概述

Mycat2 的读写分离通过**集群配置**定义主从关系，自动将写操作路由到 Master，读操作路由到 Slave。

```
Application
    │ SQL
    ▼
┌──────────┐
│  Mycat2  │
│  Router  │
└────┬─────┘
     │
     ├── INSERT/UPDATE/DELETE → Master (写)
     └── SELECT → Slave(s) (读，负载均衡)
```

# 2. 集群配置

## 2.1 conf/clusters.json

```json
{
  "clusters": {
    "c0": {
      "clusterType": "SINGLE_NODE",
      "master": {
        "name": "master0",
        "url": "jdbc:mysql://192.168.1.10:3306/test_db",
        "user": "mycat",
        "password": "mycat123",
        "type": "JDBC",
        "minCon": 2,
        "maxCon": 20,
        "weight": 10
      }
    },
    "rw_cluster": {
      "clusterType": "MASTER_SLAVE",
      "readBalanceType": "BALANCE_ALL",
      "master": {
        "name": "m1",
        "url": "jdbc:mysql://192.168.1.10:3306/test_db",
        "user": "mycat",
        "password": "mycat123",
        "type": "JDBC",
        "initSQLs": ["SET NAMES utf8mb4"]
      },
      "replicas": [
        {
          "name": "s1",
          "url": "jdbc:mysql://192.168.1.11:3306/test_db",
          "user": "mycat",
          "password": "mycat123",
          "type": "JDBC",
          "weight": 10
        },
        {
          "name": "s2",
          "url": "jdbc:mysql://192.168.1.12:3306/test_db",
          "user": "mycat",
          "password": "mycat123",
          "type": "JDBC",
          "weight": 5
        }
      ]
    }
  }
}
```

# 3. 读写分离类型

| 类型 | 配置值 | 说明 |
|------|--------|------|
| 单节点 | `SINGLE_NODE` | 只有一个 Master，无读写分离 |
| 普通主从 | `MASTER_SLAVE` | 写走 Master，读走 Slave |
| Galera 集群 | `GARELA_CLUSTER` | 多主集群，任意节点可读写 |
| MGR 集群 | `MGR` | MySQL Group Replication |

# 4. 读负载均衡策略

| 策略 | 配置值 | 说明 |
|------|--------|------|
| 全部均衡 | `BALANCE_ALL` | 所有 Slave 参与读负载 |
| 不均衡 | `BALANCE_NONE` | 所有读也走 Master（慎用） |
| 权重均衡 | `BALANCE_ALL_READ` | 按 `weight` 权重分配读请求 |
| 就近 | `BALANCE_NEARBY` | 优先选择同一机房的 Slave |

**权重配置示例**：

```json
"replicas": [
    { "name": "s1", "weight": 10 },   // 高性能 Slave，承载 67% 读流量
    { "name": "s2", "weight": 5 }     // 低配 Slave，承载 33% 读流量
]
```

# 5. Schema 引用集群

```json
// conf/schemas.json
{
  "schemas": {
    "test_db": {
      "tables": {
        "users": {
          "type": "sharding",
          "dataNodes": "rw_cluster.users"
        }
      }
    }
  }
}
```

**dataNodes 格式**：`{clusterName}.{tableName}`，其中 `clusterName` 对应 `clusters.json` 中定义的集群。

# 6. 读写分离判断规则

```java
// Mycat2 路由判断伪代码
public DataNode route(String sql) {
    SQLType type = parseSQLType(sql);

    switch (type) {
        case SELECT:
            // 读操作 → 负载均衡到 Slave
            return loadBalanceSlave();
        case INSERT:
        case UPDATE:
        case DELETE:
        case REPLACE:
            // 写操作 → 强制路由到 Master
            return getMaster();
        case SET:
        case SHOW:
            // 特殊语句 → Master
            return getMaster();
        default:
            return getMaster();
    }
}
```

**强制走 Master 的 SQL**：
- 所有写操作（INSERT/UPDATE/DELETE/REPLACE）
- 包含 `FOR UPDATE` 的 SELECT
- 显式事务中的语句（`BEGIN ... COMMIT`）
- `LAST_INSERT_ID()` 调用

# 7. Hint 强制路由

```sql
-- 强制走 Master（即使 SELECT）
/*!mycat:db_type=master*/ SELECT * FROM users WHERE id = 1;

-- 强制走 Slave（即使写，通常不用）
/*!mycat:db_type=slave*/ SELECT * FROM users WHERE id = 1;
```

# 8. 主从延迟处理

## 8.1 问题

主从复制存在延迟，刚写入 Master 的数据可能在 Slave 上尚未同步。此时立即查询 Slave 可能读到旧数据。

## 8.2 解决方案

```json
// 方案 1：关键读走 Master
/*!mycat:db_type=master*/ SELECT * FROM orders WHERE user_id = 1;

// 方案 2：配置读写分离策略
// BALANCE_ALL_READ: 根据权重均衡读
// 设置新写入的 Slave 权重为 0，延迟追上后再恢复

// 方案 3：延迟检测 + 自动切换
{
  "heartbeat": {
    "maxRetry": 3,
    "minSwitchTimeInterval": 30000,
    "slaveThreshold": 60,    // Slave 延迟超过 60s 自动摘除
    "heartbeatTimeout": 1000
  }
}
```

# 9. 心跳检测

```json
// 心跳配置（datasources.json 或 clusters.json）
{
  "heartbeat": {
    "heartbeatSQL": "SELECT 1",
    "retryCount": 3,
    "intervalSeconds": 10,
    "timeoutSeconds": 5
  }
}
```

心跳检测到 Slave 故障后：
1. 自动从读负载均衡列表中摘除
2. 日志告警
3. 恢复后自动重新加入

# 10. 多主多从

```json
{
  "clusters": {
    "multi_master": {
      "clusterType": "GARELA_CLUSTER",
      "masters": [
        {
          "name": "m1",
          "url": "jdbc:mysql://192.168.1.10:3306/test_db",
          "type": "JDBC",
          "weight": 10
        },
        {
          "name": "m2",
          "url": "jdbc:mysql://192.168.1.11:3306/test_db",
          "type": "JDBC",
          "weight": 10
        }
      ]
    }
  }
}
```

Galera/MGR 集群模式下，所有节点均可读写，Mycat2 按权重轮询分配请求。
