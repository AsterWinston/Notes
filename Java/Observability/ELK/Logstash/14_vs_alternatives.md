
# 1. 对比矩阵

| 维度 | Logstash | Fluentd | Fluent Bit | Vector |
|------|----------|---------|-----------|--------|
| **语言** | JRuby (JVM) | C + Ruby | C | Rust |
| **内存** | 500MB+ | ~50MB | ~5MB | ~20MB |
| **CPU** | 中-高 | 中 | 极低 | 低 |
| **吞吐（单核）** | 2K-5K eps | 10K-20K eps | 50K+ eps | 50K+ eps |
| **插件数** | 200+ | 1000+ | 100+ | 80+ |
| **配置语法** | 自定义 DSL | 自定义 DSL | YAML-like | TOML |
| **License** | Elastic License | Apache 2.0 | Apache 2.0 | MPL-2.0 |
| **ELK 集成** | ★★★★★ | ★★★☆☆ | ★★★★☆ | ★★★☆☆ |
| **K8s 友好** | ★★☆☆☆ | ★★★★☆ | ★★★★★ | ★★★★★ |
| **持久化队列** | ✓ (PQ) | ✓ (Buffer) | ✓ | ✓ |
| **数据处理** | 强（Grok/Mutate/GeoIP） | 中 | 弱 | 中 |

# 2. 性能基准

```
测试: Nginx access log, Grok 解析, 输出到 ES

| 工具        | 单核吞吐   | 4 核吞吐   | 内存 (4核) |
|------------|-----------|-----------|-----------|
| Logstash   | 2K eps    | 8K eps    | 1.5GB     |
| Fluentd    | 8K eps    | 30K eps   | 200MB     |
| Fluent Bit | 30K eps   | 100K+ eps | 15MB      |
| Vector     | 40K eps   | 150K+ eps | 35MB      |
```

# 3. 架构对比

```
Logstash:
  Input Plugin → Queue → Filter Chain → Output Plugin

Fluentd/Fluent Bit:
  Input Plugin → Parser → Filter → Buffer → Output Plugin
  
  Buffer = 类似 PQ 的磁盘缓冲层
  Parser = 在 Input 阶段解析（非 Filter 阶段）

Vector:
  Source → Transform → Sink
  (更现代的 Rust 实现, 零拷贝管道)
```

# 4. 选择决策树

```
需要复杂数据处理 (Grok/GeoIP/Mutate)?
  ├── 是
  │     ├── 已使用 ELK 生态? → Logstash
  │     └── 否 → Fluentd
  └── 否
        ├── K8s 环境 / 边缘设备 / 低资源?
        │     └── Fluent Bit
        └── 追求最高性能 / Rust 生态?
              └── Vector

混合架构 (推荐):
  边缘: Fluent Bit (轻量采集)
    → 中心: Logstash (重量处理 + 丰富)
      → Elasticsearch
```

# 5. 迁移路径

```
从 Logstash 迁移到 Fluentd/Fluent Bit:
  1. 保留 Logstash 的核心 Filter 逻辑
  2. 边缘/容器层用 Fluent Bit 采集
  3. Fluent Bit 直接输出到 ES (轻量日志)
  4. 需要复杂处理的仍发 Logstash
  5. 逐步缩减 Logstash 使用范围
```

# 6. 总结

| 场景 | 推荐 |
|------|------|
| ELK 完整生态，复杂 ETL | Logstash |
| 通用数据管道，多输出目标 | Fluentd |
| K8s 日志采集，边缘设备 | Fluent Bit |
| 极致性能，Rust 技术栈 | Vector |
