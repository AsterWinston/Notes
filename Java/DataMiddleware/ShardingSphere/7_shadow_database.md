
# 1. 概述

影子库（Shadow Database）是 ShardingSphere 的**全链路压力测试**方案。通过将压测流量路由到影子库，实现生产数据隔离的压测。

```
┌─────────────────────────────────────────┐
│              Application                 │
│                                         │
│  ┌──────────┐          ┌──────────┐    │
│  │ 正常流量  │          │ 压测流量  │    │
│  │ (shadow=false)│      │ (shadow=true)│ │
│  └────┬─────┘          └────┬─────┘    │
│       │                     │           │
└───────┼─────────────────────┼───────────┘
        │                     │
        ▼                     ▼
┌────────────┐       ┌──────────────┐
│ 生产数据库  │       │  影子数据库   │
│ (真实数据)  │       │ (压测数据)    │
└────────────┘       └──────────────┘
```

# 2. 配置

## 2.1 数据源配置

```yaml
dataSources:
  ds_prod:
    url: jdbc:mysql://prod-db:3306/order_db
    username: root
    password: root
  ds_shadow:
    url: jdbc:mysql://shadow-db:3306/order_db
    username: root
    password: root
```

## 2.2 影子规则

```yaml
rules:
  - !SHADOW
    dataSources:
      shadowDataSource:        # 生产-影子映射
        productionDataSourceName: ds_prod
        shadowDataSourceName: ds_shadow

    tables:
      t_order:
        dataSourceNames:
          - shadowDataSource
        shadowAlgorithmNames:
          - user_id_match

    shadowAlgorithms:
      user_id_match:
        type: REGEX_MATCH
        props:
          operation: insert
          column: user_id
          regex: "[1-9]\\d{5,}"
```

# 3. 影子算法

## 3.1 列值匹配（REGEX_MATCH）

```yaml
shadowAlgorithms:
  user_id_match:
    type: REGEX_MATCH
    props:
      column: user_id          # 判断列
      operation: insert        # 判断操作
      regex: "[1-9]\\d{5,}"    # 匹配的列值正则
```

## 3.2 Hint 方式

```java
// 代码中标记为压测流量
HintManager hintManager = HintManager.getInstance();
hintManager.setShadow(true);
try {
    jdbcTemplate.update("INSERT INTO t_order (...) VALUES (...)", ...);
} finally {
    hintManager.close();
}
```

```yaml
shadowAlgorithms:
  hint_match:
    type: HINT_MATCH
    props:
      shadow: true
```

# 4. 全链路压测流程

```
1. 构造压测数据 → 写入影子库
2. 压测工具（JMeter/Gatling）发起请求
3. 请求中携带压测标记（Header/Hint/参数）
4. ShardingSphere 识别压测流量
5. 读/写全部路由到影子库
6. 监控收集压测指标
7. 分析瓶颈，优化
```

# 5. 与数据分片组合

```yaml
rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds_prod.t_order_${0..1}

  - !SHADOW
    dataSources:
      shadow_ds0:
        productionDataSourceName: ds_prod
        shadowDataSourceName: ds_shadow
    tables:
      t_order:
        dataSourceNames: [shadow_ds0]

  - !READWRITE_SPLITTING
    # 读写分离可同时使用
```

# 6. 注意事项

1. **数据隔离**：影子库必须是独立的数据库实例，确保生产数据安全
2. **DDL 同步**：生产库的 DDL 变更需同步到影子库
3. **清理策略**：压测结束后清理影子库数据，避免存储膨胀
4. **监控区分**：生产监控和压测监控分开，避免告警混淆
5. **性能隔离**：确保影子库的资源不影响生产库
