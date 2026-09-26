
# 1. JVM 参数

```bash
# OAP Server JVM 调优
java -Xms4g -Xmx8g \
     -Xmn2g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:+DisableExplicitGC \
     -jar oap-server.jar
```

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `-Xms/-Xmx` | 4g/8g | 中小规模 |
| `-Xmn` | 2g | 年轻代（L1 聚合主要在年轻代） |
| GC | G1GC | 低暂停 |

# 2. gRPC 线程池

```yaml
# config/application.yml
core:
  default:
    grpcPoolSize: 16              # gRPC 线程池
    restPoolSize: 8               # HTTP 线程池
    maxConcurrentCallsPerConnection: 50   # 单连接最大并发
    maxMessageSize: 10485760       # 10MB 最大消息
```

# 3. 分析 Worker 配置

```yaml
receiver-register:
  default:
    maxAnalyzeCount: 8            # 分析线程数（建议 = CPU 核数）

receiver-trace:
  default:
    bufferPath: /tmp/skywalking    # trace 数据缓冲
    bufferFileSize: 64M
    bufferSegmentMaxSize: 500
```

# 4. L1/L2 聚合优化

```yaml
# 聚合参数
metrics-batch-pool-size: 4        # 批处理线程
metrics-batch-size: 1000          # 批大小

# OAL 引擎
oal-engine:
  max-threads: 4
```

# 5. 存储写入优化

```yaml
storage:
  elasticsearch:
    batchActions: 5000            # 批量写入
    flushInterval: 10             # 刷新间隔
    concurrentRequests: 4         # 并发写入线程
    syncBulkActions: 10           # 同步写入批量
```

# 6. OAP 集群调优

```
Agent 负载均衡策略：
  - 所有 OAP 节点注册到同一个 Service (DNS/LB)
  - Agent 通过 gRPC 长连接随机选择一个节点
  - 如果连接断开 → 自动切换到其他节点

OAP 节点扩容：
  - 增加 OAP 实例 → Agent 负载自动分散
  - 瓶颈在存储层：先扩容 ES/BanyanDB，再扩容 OAP
```

# 7. 压测基准

```
测试环境：OAP 8C16G，ES 3 节点，4 个 Spring Boot 应用

| Agent 实例 | 每秒 Trace | OAP CPU | ES IOPS |
|-----------|-----------|---------|---------|
| 10        | 1000      | 20%     | 200     |
| 50        | 5000      | 45%     | 800     |
| 100       | 10000     | 70%     | 1500    |
| 200       | 20000     | 90%     | 3000    |
```

# 8. 快速调优清单

```
□ JVM: -Xms4g -Xmx8g -XX:+UseG1GC
□ gRPC: grpcPoolSize = CPU 核数 × 2
□ Analysis: maxAnalyzeCount = CPU 核数
□ Storage: batchActions ≥ 2000
□ ES: indexShardsNumber = 2, indexReplicasNumber = 1
□ Trace TTL: 7 天
□ Metrics TTL: 30 天
□ 日志级别: WARN（生产）
```
