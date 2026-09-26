
# 1. 调优全景图

```
性能调优维度:
┌────────────┬────────────┬────────────┬────────────┐
│  JVM 层     │  管道层     │  Filter 层  │  Output 层  │
├────────────┼────────────┼────────────┼────────────┤
│ 堆内存/GC   │ Worker 数   │ Grok 优化   │ ES 批量     │
│ 堆外内存    │ Batch 大小  │ Dissect 替代│ 连接池      │
│ 启动参数    │ Queue 类型  │ 条件过滤    │ 重试策略    │
└────────────┴────────────┴────────────┴────────────┘
```

# 2. JVM 调优

```bash
# config/jvm.options
-Xms2g -Xmx2g          # 堆内存相同（避免 resize 开销）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+DisableExplicitGC

# 堆外内存（Direct Memory）
# Kafka Input 使用堆外内存
-XX:MaxDirectMemorySize=512m
```

# 3. 管道 Workers 与 Batch

```yaml
# logstash.yml
pipeline.workers: 4            # Worker 线程（= CPU 核数）
pipeline.batch.size: 125       # 每批事件数（默认 125）
pipeline.batch.delay: 50       # 批等待时间(ms)
```

```
Workers 计算:
  Workers = CPU 核心数 × 1-2
  过多 → 上下文切换开销 > 并行收益
  过少 → 无法充分利用 CPU

Batch 计算:
  batch.size 太小 → 频繁刷新 Output（ES 写入压力大）
  batch.size 太大 → 内存占用高，延迟增加
```

# 4. Filter 优化

```ruby
# 1. 条件过滤（避免不必要处理）
if [type] == "nginx" {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
}

# 2. Grok 锚定（^ 减少回溯）
grok { match => { "message" => "^%{COMBINEDAPACHELOG}" } }

# 3. Dissect 替代 Grok（分隔符明确时）
dissect { mapping => { "message" => "%{ts} [%{level}] %{msg}" } }

# 4. 提前删除不需要的字段
mutate { remove_field => ["@version", "beat", "input", "host"] }

# 5. 避免过多正则
# ✗: 5 个 Grok 规则
# ✓: 1 个 Grok + Mutate 字段操作
```

# 5. Output 优化

```ruby
output {
  elasticsearch {
    hosts => ["es1:9200", "es2:9200"]
    flush_size => 500               # 增大批量
    idle_flush_time => 2            # 减少空刷新
    sniffing => false               # 关闭嗅探（减少延迟）
    manage_template => false        # 手动管理模板（减少请求）
  }
}
```

# 6. 管道线程监控

```bash
# Logstash API
curl http://localhost:9600/_node/stats/pipelines?pretty

# 关注指标:
# - events.in: 输入事件速率
# - events.out: 输出事件速率
# - events.filtered: 过滤后事件数
# - queue.events_count: 队列堆积事件数（> 10000 需关注）
# - pipeline.plugins.outputs[].bulk_requests.latency: ES 写入延迟
```

# 7. 生产调优配置

```yaml
# 高性能配置
pipeline.workers: 8
pipeline.batch.size: 250
pipeline.batch.delay: 25
queue.type: persisted
queue.max_bytes: 4gb
queue.checkpoint.writes: 2048
```

# 8. 性能基准

```
环境: 4C8G, Filebeat → Logstash (Grok+GeoIP) → ES

| 配置                 | 吞吐量     | CPU  |
|---------------------|-----------|------|
| Workers=2, Batch=125 | 5000 eps  | 45%  |
| Workers=4, Batch=250 | 12000 eps | 75%  |
| Workers=8, Batch=500 | 18000 eps | 90%  |

eps = events per second (每秒事件数)
```

# 9. 常见瓶颈

| 瓶颈 | 现象 | 解决 |
|------|------|------|
| Grok 正则回溯 | CPU 100%, 吞吐低 | 锚定 ^、用 Dissect |
| ES 写入慢 | queue 堆积 | 增大 flush_size、加 ES 节点 |
| JVM GC 频繁 | 吞吐波动大 | 增大堆、换 G1GC |
| PQ 磁盘慢 | 吞吐下降 | 换 SSD、增大 checkpoint 间隔 |
