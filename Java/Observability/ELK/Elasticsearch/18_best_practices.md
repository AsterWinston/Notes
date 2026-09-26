
# 1. 容量规划

```
分片数计算:
  单分片推荐大小: 30-50GB（搜索密集）/ 50-100GB（日志密集）
  分片数 ≈ 总数据量(TB) × 1024 / 单分片大小(GB)

  示例: 5TB 订单数据, 单分片 50GB → 约 100 个主分片

节点数计算:
  每个节点 ≤ 600 分片（总主+副）
  节点数 ≥ (总分片数 / 600) × 1.5（冗余）

内存计算:
  堆内存: 节点内存的 50%, 上限 32GB（压缩指针限制）
  OS Cache: 用于 Lucene 文件缓存（至少留 50% 内存）
```

# 2. JVM 调优

```bash
# config/jvm.options
-Xms16g -Xmx16g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=8m
-XX:InitiatingHeapOccupancyPercent=30
-XX:+DisableExplicitGC

# 堆内存 ≤ 32GB（compressed oops 限制）
# 堆内存 = 物理内存的 50%
```

# 3. Mapping 设计原则

```json
{
  "_doc": {
    // ✓ 字符串优先 keyword（精确匹配）
    "order_no": { "type": "keyword" },

    // ✓ 全文搜索用 text
    "description": { "type": "text" },

    // ✓ 关闭不需要的功能节省空间
    "status": {
      "type": "keyword",
      "doc_values": true,   // 聚合用
      "norms": false        // keyword 不需要评分
    },

    // ✓ 禁止动态映射（生产环境）
    "dynamic": "strict"
  }
}
```

# 4. 集群安全

```
□ 开启 X-Pack Security（认证 + 授权）
□ 启用 TLS（传输 + HTTP 两层）
□ 非 root 用户运行 ES 进程
□ IPTables / Security Group 限制访问 9200/9300
□ 定期审计日志
□ 固定版本（不用 :latest 镜像）
□ 备份 snapshot 每天自动
□ 监控磁盘水印（disk watermark）
```

# 5. ILM 策略

```json
{
  "hot": { "max_age": "3d", "max_size": "50GB", "rollover": {} },
  "warm": { "min_age": "3d", "forcemerge": {}, "shrink": {} },
  "cold": { "min_age": "30d", "searchable_snapshot": {} },
  "delete": { "min_age": "90d" }
}
```

# 6. 监控告警

```
必须监控的指标:
  □ 集群状态（必须 green 或 yellow）
  □ 未分配分片数（> 0 告警）
  □ 磁盘使用率（> 85% 告警）
  □ JVM 堆使用率（> 75% 告警）
  □ 写入拒绝率（> 0 告警）
  □ 搜索延迟 P99（> 1s 告警）
  □ 节点离线告警
```

# 7. DON'T

```
✗ 不要混用不同版本的 ES 节点
✗ 不要让堆内存超过 32GB（丢失 compressed oops）
✗ 不要在单节点部署多个角色
✗ 不要在生产环境用 dynamic mapping
✗ 不要设置过大的 from+size（> 10000）
✗ 不要在高负载时 forcemerge
✗ 不要在磁盘满时继续写入（flood_stage 会只读）
✗ 不要只用 1 个 Master 节点（脑裂风险）
✗ 不要在已运行的索引上修改主分片数
```

# 8. 生产配置模板

```yaml
cluster.name: prod-cluster
node.name: data-node-1
node.roles: [data, ingest]

path.data: /data/elasticsearch
path.logs: /var/log/elasticsearch

network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

discovery.seed_hosts: ["master1:9300", "master2:9300", "master3:9300"]
cluster.initial_master_nodes: ["master1", "master2", "master3"]

xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true

indices.memory.index_buffer_size: 20%
bootstrap.memory_lock: true
```
