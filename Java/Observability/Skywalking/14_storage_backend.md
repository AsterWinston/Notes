
# 1. 存储后端对比

| 维度 | Elasticsearch | MySQL | BanyanDB | H2 |
|------|--------------|-------|----------|-----|
| **生产推荐** | ★★★★★ | ★★★☆☆ | ★★★★☆ | ☆☆☆☆☆ |
| **写入性能** | 高 | 中 | 极高 | 低 |
| **查询性能** | 高（全文检索） | 中 | 高 | 低 |
| **水平扩展** | ✓ | 有限 | ✓ | ✗ |
| **运维复杂度** | 中 | 低 | 中 | 极低 |
| **资源消耗** | 高 | 中 | 中 | 低 |
| **适用场景** | 生产首选 | 小规模/已有 MySQL | 9.x 推荐 | 开发测试 |

# 2. Elasticsearch 配置

```yaml
# config/application.yml
storage:
  selector: elasticsearch
  elasticsearch:
    namespace: skywalking
    clusterNodes: es1:9200,es2:9200,es3:9200
    protocol: http
    user: elastic
    password: ${ES_PASSWORD}
    connectTimeout: 5000
    socketTimeout: 30000
    indexReplicasNumber: 1
    indexShardsNumber: 2
    recordDataTTL: 7       # Trace 数据保留天数
    metricsDataTTL: 30     # 指标数据保留天数
    dayStep: 1             # 每天新索引
    batchActions: 5000     # 批量写入大小
    flushInterval: 10      # 刷新间隔(秒)
    concurrentRequests: 2  # 并发请求数
```

## 2.1 ES 索引优化

```bash
# 提前创建 Index Template
curl -X PUT "localhost:9200/_template/skywalking-template" \
  -H "Content-Type: application/json" \
  -d '{
    "index_patterns": ["sw_*"],
    "settings": {
      "number_of_shards": 2,
      "number_of_replicas": 1,
      "refresh_interval": "30s"
    }
  }'
```

## 2.2 清理策略

```bash
# 自动按 TTL 清理
# 或手动删除过期索引
curl -X DELETE "localhost:9200/sw_metrics-all-20240101"
```

# 3. MySQL 配置

```yaml
storage:
  selector: mysql
  mysql:
    properties:
      jdbcUrl: jdbc:mysql://localhost:3306/skywalking
      dataSource.user: root
      dataSource.password: ${MYSQL_PASSWORD}
      dataSource.maxActive: 50
```

**MySQL 适用场景**：日 Trace < 1000 万条的中小规模环境。

# 4. BanyanDB 配置

```yaml
storage:
  selector: banyandb
  banyandb:
    host: localhost
    port: 17912
    maxBulkSize: 5000
    flushInterval: 15
```

BanyanDB 是 SkyWalking 自研的时序数据库，9.x 推荐为默认存储：

| 优势 | 说明 |
|------|------|
| **写优化** | LSM-Tree 结构，写入性能优于 ES |
| **原生支持** | 无需额外部署，与 OAP 集成 |
| **资源占用低** | 比 ES 省 50%+ 内存 |
| **自动分片** | 按时间自动分片 |

# 5. 存储选型决策

```
每日 Trace 量?
  ├── < 100 万 → MySQL / H2
  ├── 100 万 ~ 1000 万 → Elasticsearch / BanyanDB
  └── > 1000 万 → BanyanDB (写优化) 或 ES 集群

运维能力?
  ├── 有 ES 运维经验 → Elasticsearch
  ├── 有 MySQL DBA → MySQL
  └── 不想额外维护 → BanyanDB
```

# 6. 最佳实践

1. **TTL 设置合理**：Trace 保留 7 天，Metrics 保留 30 天
2. **ES 索引优化**：关闭 `_all` 字段，使用 `keyword` 类型
3. **MySQL 定期清理**：配置定时任务清理过期数据
4. **BanyanDB 推荐**：9.x 新项目优先 BanyanDB
5. **监控存储本身**：ES 集群/JVM 健康纳入监控
