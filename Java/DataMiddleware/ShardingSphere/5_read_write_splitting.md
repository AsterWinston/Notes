
# 1. 概述

ShardingSphere 提供透明读写分离，写操作自动路由到主库，读操作负载均衡到从库。

```
Application (JDBC / Proxy)
    │ SQL
    ▼
ShardingSphere
    │
    ├── INSERT/UPDATE/DELETE → writeDataSource (Master)
    └── SELECT → readDataSource (Slave) 负载均衡
```

# 2. 静态读写分离

```yaml
rules:
  - !READWRITE_SPLITTING
    dataSources:
      readwrite_ds:
        staticStrategy:
          writeDataSourceName: ds_master
          readDataSourceNames:
            - ds_slave0
            - ds_slave1
        loadBalancerName: round_robin

    loadBalancers:
      round_robin:
        type: ROUND_ROBIN
      random:
        type: RANDOM
      weight:
        type: WEIGHT
        props:
          ds_slave0: 10
          ds_slave1: 5
```

## 2.1 负载均衡算法

| 算法 | 说明 |
|------|------|
| `ROUND_ROBIN` | 轮询（默认） |
| `RANDOM` | 随机 |
| `WEIGHT` | 按权重分配 |

# 3. 动态读写分离

动态模式基于**数据库发现**自动感知主从拓扑变化：

```yaml
rules:
  - !READWRITE_SPLITTING
    dataSources:
      readwrite_ds:
        dynamicStrategy:
          autoAwareDataSourceName: ds_master      # 数据库发现名称
          writeDataSourceQueryEnabled: true
          readDataSourceNames: []
        loadBalancerName: round_robin

  - !DB_DISCOVERY
    dataSources:
      ds_master:
        dataSourceNames:
          - ds_master
          - ds_slave0
          - ds_slave1
        discoveryHeartbeatName: mgr_heartbeat
        discoveryTypeName: mgr

    discoveryHeartbeats:
      mgr_heartbeat:
        props:
          keep-alive-cron: "0/5 * * * * ?"

    discoveryTypes:
      mgr:
        type: MGR     # MySQL Group Replication
```

**支持的数据库发现类型**：

| 类型 | 说明 |
|------|------|
| `MGR` | MySQL Group Replication |
| `MySQL.MASTER_SLAVE` | 传统主从复制 |

# 4. 事务内读写策略

```yaml
props:
  readwrite-splitting-hint-enable: true
```

```
事务中的读写行为：
  BEGIN → SELECT (走从库)
       → INSERT (走主库)
       → SELECT (也走主库！事务一致性保护)
  COMMIT
```

**默认策略**：同一事务中，一旦有写操作，后续读操作也走主库，避免读到不一致数据。

# 5. Hint 强制路由

```java
// Java 代码强制主库
HintManager hintManager = HintManager.getInstance();
hintManager.setWriteRouteOnly();       // 强制走主库
try {
    List<User> users = jdbcTemplate.query(
        "SELECT * FROM t_user", rowMapper);
} finally {
    hintManager.close();
}
```

# 6. 与分片的组合

```yaml
rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: readwrite_ds.t_order_${0..1}   # 引用读写分离数据源
        ...
  - !READWRITE_SPLITTING
    dataSources:
      readwrite_ds:
        staticStrategy:
          writeDataSourceName: ds_master
          readDataSourceNames: [ds_slave0, ds_slave1]
        loadBalancerName: round_robin
```

**效果**：每个分片都有独立的主从集群，分片路由到目标主从组后，再按读写分离策略选择主或从。

# 7. 从库故障处理

```
从库心跳检测失败 → 标记为不可用 → 从读负载中摘除
恢复后 → 自动检测 → 重新加入读负载
```

```yaml
props:
  slave-connection-healthy-check-enabled: true
```

# 8. 最佳实践

1. **关键读强制主库**：支付回调检查、订单状态验证等场景使用 Hint
2. **从库延迟容忍**：非关键查询（列表/统计）读从库，关键查询（详情/校验）读主库
3. **权重设置**：高性能从库设高权重，低配从库设低权重
4. **组合使用**：分片 + 读写分离，每个分片独立主从
