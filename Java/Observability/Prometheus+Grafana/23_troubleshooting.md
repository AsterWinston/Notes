
# 1. 启动与配置问题

## 1.1 Prometheus 无法启动

**现象：**
```
level=error ts=... caller=main.go:... msg="Unable to load configuration file" file=prometheus.yml err="..."
```

**排查步骤：**

```bash
# 1. 检查配置文件语法
promtool check config prometheus.yml

# 2. 检查 YAML 格式（缩进是否正确）
# 3. 检查文件权限
ls -la prometheus.yml

# 4. 检查端口是否被占用
netstat -ano | findstr :9090
netstat -tlnp | grep 9090

# 5. 检查数据目录权限
ls -la /data/prometheus
```

## 1.2 常见配置错误

```yaml
# 错误：YAML 缩进错误
scrape_configs:
  - job_name: 'node'
  static_configs:       # 缩进不对
    - targets: ['...']

# 正确：一致的缩进
scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['...']
```

# 2. 采集问题

## 2.1 目标状态为 DOWN

**检查步骤：**

```bash
# 1. 检查采集目标状态
curl http://localhost:9090/api/v1/targets | jq .

# 2. 直接测试目标是否可达
curl http://target:9100/metrics

# 3. 检查网络连通性
ping target
telnet target 9100

# 4. 检查防火墙规则
iptables -L -n | grep 9100
```

**常见原因：**

| 原因 | 现象 | 解决方法 |
|------|------|----------|
| 目标未启动 | TCP 连接被拒绝 | 启动目标 Exporter |
| 防火墙阻挡 | 连接超时 | 开放防火墙端口 |
| DNS 解析失败 | 无法解析主机名 | 检查 DNS 配置 |
| 证书错误 | TLS 握手失败 | 配置正确的证书 |
| 认证失败 | 401 Unauthorized | 配置 Basic Auth |
| 采集超时 | context deadline exceeded | 增大 scrape_timeout |

## 2.2 指标不完整

```yaml
# 检查是否超过样本限制
scrape_configs:
  - job_name: 'node'
    sample_limit: 10000  # 如果指标数超过此值，超出的部分被丢弃
```

```promql
# 查看采集时被丢弃的样本数
rate(prometheus_target_scrapes_exceeded_sample_limit_total[5m])
rate(prometheus_target_scrapes_sample_duplicate_drop_total[5m])
```

## 2.3 指标值为 NaN 或空

```promql
# 原因：Counter 在时间窗口内是平缓的（没有新增）
rate(http_requests_total[5m]) == 0  # 可能是正常的

# 原因：时间窗口内没有数据点
rate(some_counter[30s])  # 时间窗口小于采集间隔可能无数据
```

# 3. 查询问题

## 3.1 查询超时

```bash
--query.timeout=2m
--query.max-concurrency=20
--query.max-samples=50000000
```

```promql
# Bad：全量扫描，容易超时
http_requests_total

# Good：缩小范围
http_requests_total{job="api", method="GET", status="200"}

# Good：使用聚合
sum(rate(http_requests_total[5m])) by (job)
```

## 3.2 查询结果不准确

```promql
# Counter 重置导致 rate 计算异常
# 检查 Counter 是否被重置
resets(http_requests_total[1h])

# 增加时间窗口平滑跳变
rate(http_requests_total[15m])  # 大窗口更平滑
```

## 3.3 时间序列数据缺失

```promql
# 检查时间序列最新数据
prometheus_tsdb_head_min_time
prometheus_tsdb_head_max_time

# 计算数据延迟（当前时间 - 最新数据时间）
time() - prometheus_tsdb_head_max_time / 1000
```

# 4. 性能问题

## 4.1 内存过高

**现象：** Prometheus OOM 或被 OOM Killer 杀死

**原因排查：**

```promql
# 1. 检查指标总数
prometheus_tsdb_head_series

# 2. 检查高基数 Job
# 在 Prometheus UI -> Status -> TSDB Status 查看
# 或查询 API：
curl http://localhost:9090/api/v1/status/tsdb

# 3. 查找高基数标签
topk(10, count by (__name__) ({__name__=~".+"}))
```

**解决方案：**

```yaml
# 1. 增加 Relabel 丢弃高基数指标
scrape_configs:
  - job_name: 'problematic'
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'high_cardinality_metric_.*'
        action: drop

# 2. 限制内存
--storage.tsdb.max-block-duration=2h
--storage.tsdb.retention.time=7d

# 3. 增加系统内存
```

## 4.2 磁盘 IO 过高

**原因：**
- Compaction 频繁
- WAL 写入压力大
- 查询大量历史数据

**优化：**

```bash
--storage.tsdb.wal-compression       # 启用 WAL 压缩
--storage.tsdb.min-block-duration=2h  # 默认
--storage.tsdb.max-block-duration=48h # 减少 Compaction 频率
```

## 4.3 CPU 过高

**原因：**
- 查询并发过高
- 规则评估频繁
- 大量指标需要 Compaction

**优化：**

```bash
--query.max-concurrency=10    # 降低查询并发
--storage.tsdb.max-block-duration=48h  # 减少 Compaction 频率
```

# 5. 存储问题

## 5.1 磁盘空间不足

```bash
# 1. 检查当前磁盘使用
du -sh /data/prometheus/*
du -sh /data/prometheus/chunks_head

# 2. 缩短保留时间
--storage.tsdb.retention.time=7d

# 3. 设置存储大小限制
--storage.tsdb.retention.size=100GB

# 4. 手动清理（需停止 Prometheus）
rm -rf /data/prometheus/01EM6Q6A1YPX4G9X6R6XG9X6R6
```

```promql
# 查看各 Block 的大小
prometheus_tsdb_storage_blocks_bytes

# 查看 Block 数量
prometheus_tsdb_blocks_loaded
```

## 5.2 TSDB 损坏

**现象：**
```
level=error ts=... msg="...corruption..." 
level=error ts=... msg="...checksum mismatch..."
```

**恢复步骤：**

```bash
# 1. 停止 Prometheus
systemctl stop prometheus

# 2. 备份数据
cp -r /data/prometheus /data/prometheus_backup

# 3. 使用 promtool 修复
promtool tsdb repair /data/prometheus

# 4. 启动 Prometheus
systemctl start prometheus

# 5. 如果修复失败，删除最近的 Block 后启动
# 6. 如果仍然失败，使用快照或备份恢复
```

# 6. 高基数爆炸

## 6.1 识别高基数

```promql
# 查看每个指标的序列数
count by (__name__) ({__name__=~".+"})

# 查看特定指标
count({__name__="http_requests_total"})

# 查找高基数标签
topk(10, count by (method) ({__name__="http_requests_total"}))
```

## 6.2 常见高基数来源

| 来源 | 示例 | 解决方案 |
|------|------|----------|
| URL Path 带参数 | `/api/users/12345` | 使用路径模板替换参数 |
| HTTP Header 泄露 | `User-Agent`、`Referer` | 过滤不需要的标签 |
| 容器指标 | `container_label_*` | 只保留有意义的容器标签 |
| 系统指标 | `node_arp_entries` | 禁用对应 Collector |
| 动态标签 | `timestamp`、`random_id` | 在 Relabel 中丢弃 |

## 6.3 紧急处理

```yaml
# 紧急丢弃高基数指标
scrape_configs:
  - job_name: 'high-cardinality'
    metric_relabel_configs:
      # 丢弃可能导致基数爆炸的指标
      - source_labels: [__name__]
        regex: '.*_created|.*_info'
        action: drop
      # 丢弃高基数标签
      - regex: 'container_label_.*|id|name'
        action: labeldrop
```

# 7. WAL 问题

## 7.1 WAL 文件过大

```bash
# 检查 WAL 大小
du -sh /data/prometheus/wal/

# 启用 WAL 压缩
--storage.tsdb.wal-compression

# 减少 WAL 段大小
--storage.tsdb.wal-segment-size=64MB
```

## 7.2 WAL 恢复失败

```bash
# 删除损坏的 WAL 段（会丢失少量数据）
# 停止 Prometheus
# 删除 WAL 目录
rm -rf /data/prometheus/wal/
# 启动 Prometheus（丢失 WAL 中的未落盘数据）
```

# 8. 网络问题

## 8.1 Too Many Open Files

```bash
# 检查当前文件句柄数
lsof -p $(pgrep prometheus) | wc -l

# 增大限制
ulimit -n 65536

# systemd 配置
[Service]
LimitNOFILE=65536
```

## 8.2 TCP Time Wait 过多

```bash
# 检查 TIME_WAIT 连接数
netstat -an | grep TIME_WAIT | wc -l

# 优化内核参数
sysctl -w net.ipv4.tcp_fin_timeout=30
sysctl -w net.ipv4.tcp_tw_reuse=1
```

# 9. Promtool 调试命令

```bash
# 检查配置
promtool check config prometheus.yml

# 检查规则
promtool check rules rules/*.yml

# 测试告警规则
promtool test rules rules_test.yml

# TSDB 修复
promtool tsdb repair /data/prometheus

# TSDB 分析
promtool tsdb analyze /data/prometheus

# 验证指标
promtool query instant http://localhost:9090 'up'
```
