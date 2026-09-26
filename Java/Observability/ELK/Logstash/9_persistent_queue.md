
# 1. 为什么需要 PQ

```
Memory Queue（默认）：
  Input → Memory Queue → Filter → Output
               │
               ▼ 进程重启/崩溃
           数据丢失 ✗

Persistent Queue：
  Input → Persistent Queue (磁盘) → Filter → Output
               │
               ▼ 进程重启
           数据从 Checkpoint 恢复 ✓
```

# 2. PQ 配置

```yaml
# logstash.yml
queue.type: persisted
path.queue: /var/lib/logstash/queue
queue.max_bytes: 2gb                  # PQ 最大磁盘
queue.max_events: 0                   # 不限制事件数（0=仅限制字节）
queue.checkpoint.writes: 1024         # 每 1024 次写入做 checkpoint
queue.checkpoint.interval: 1000       # 强制 checkpoint 间隔(ms)
queue.checkpoint.retry: true
```

# 3. PQ 内部结构

```
/var/lib/logstash/queue/
  ├── checkpoint             # Checkpoint 文件（标记已处理位置）
  └── page.0                 # Page File（64MB × N）
      page.1
      page.2
```

```
每个 Page File 结构:
  ┌──────────────────────────────────────┐
  │ Header (序列号, 创建时间)             │
  ├──────────────────────────────────────┤
  │ Event 1 (序列化后的完整 Event)        │
  │ Event 2                              │
  │ ...                                   │
  │ Event N                              │
  └──────────────────────────────────────┘
```

# 4. Checkpoint 机制

```
Checkpoint 记录:
  - 每个 Page File 的读取进度
  - 已确认写入 Page File 的序列号

写入流程:
  1. Event 追加到当前 Page File
  2. 达到 checkpoint.writes 阈值 → 更新 checkpoint
  3. 崩溃重启 → 从上次 checkpoint 位置恢复

注意:
  - checkpoint 不是每条都写（性能优化）
  - 崩溃可能丢失最后几个未 checkpoint 的事件
  - checkpoint.writes=1 每条都写（最安全，最慢）
```

# 5. PQ 性能影响

```
性能基准（1000 events/s, 1KB/event）:
  Memory Queue: 100% (基准)
  PQ (checkpoint=1024): ~85%
  PQ (checkpoint=1): ~60%

建议:
  - 生产环境开启 PQ（数据安全 > 性能）
  - checkpoint.writes=1024（平衡性能与安全）
  - 使用 SSD 减少磁盘延迟
```

# 6. 死信队列 (DLQ)

```yaml
# logstash.yml
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 1gb
path.dead_letter_queue: /var/lib/logstash/dead_letter_queue
```

```
Output 写入失败（如 ES 不可达）:
  1. 重试（retry）N 次
  2. 仍失败 → 写入 DLQ
  3. DLQ 存储原始 Event + 失败原因 + 时间戳

重新处理 DLQ:
  input { dead_letter_queue { path => "/var/lib/logstash/dead_letter_queue" } }
  output { elasticsearch { ... } }
```

## 6.1 DLQ 读回

```ruby
# dlq_read.conf
input {
  dead_letter_queue {
    path => "/var/lib/logstash/dead_letter_queue"
    commit_offsets => true        # 处理后标记为已处理
    clean_consumed => true        # 处理完删除
  }
}

filter {
  # 可选：修复 Event
  mutate { add_field => { "reprocessed" => true } }
}

output {
  elasticsearch {
    hosts => ["http://es:9200"]
    index => "recovered-%{+YYYY.MM.dd}"
  }
}
```

# 7. PQ + DLQ 组合

```
最佳实践:
  1. 生产环境开启 PQ（防止进程崩溃丢数据）
  2. 生产环境开启 DLQ（防止 ES 不可用丢数据）
  3. 监控 PQ 大小（超过阈值告警）
  4. 定期检查 DLQ（有数据说明下游异常）
```
