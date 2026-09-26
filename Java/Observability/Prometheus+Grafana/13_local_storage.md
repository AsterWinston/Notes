
# 1. TSDB 概述

Prometheus 2.x 使用自研的 **TSDB（Time Series Database）** 作为本地存储引擎，设计灵感来自 Facebook 的 Gorilla TSDB 和 LevelDB。

## 1.1 设计目标

- **高效写入**：每秒数百万样本写入
- **压缩存储**：高效压缩时序数据
- **查询快速**：支持按时间范围快速检索
- **操作简单**：无外部依赖，单进程运行

# 2. 存储架构

```
Prometheus TSDB
├── WAL (Write-Ahead Log)
│   ├── wal/
│   │   ├── 000001
│   │   └── 000002
│   └── checkpoint/
│       └── ...
├── Blocks (数据块)
│   ├── 01EM6Q6A1YPX4G9X6R6XG9X6R6/
│   │   ├── chunks/
│   │   │   └── 000001
│   │   ├── index
│   │   ├── meta.json
│   │   └── tombstones
│   ├── 01EM6Q6A1YPX4G9X6R6XG9X6R7/
│   │   └── ...
│   └── ...
└── lock
```

## 2.1 目录结构

| 路径 | 说明 |
|------|------|
| `data/wal/` | Write-Ahead Log，用于故障恢复 |
| `data/<block_id>/` | 数据块目录 |
| `data/<block_id>/chunks/` | 压缩后的样本数据 |
| `data/<block_id>/index` | 索引文件（倒排索引） |
| `data/<block_id>/meta.json` | 块元数据（时间范围、采样数等） |
| `data/<block_id>/tombstones` | 墓碑文件（标记已删除数据） |
| `data/lock` | 防止多个进程同时打开同一个 TSDB |

# 3. Block（数据块）

Prometheus 每 **2 小时** 生成一个 Block，每个 Block 是数据的一个独立单元。

## 3.1 Block 生命周期

```
内存中数据
    |
    |-- 每 2h 落盘
    v
Head Block (内存中 + WAL)
    |
    |-- 2h 后变为 immutable block
    v
Immutable Block (只读)
    |
    |-- 后台 compaction
    v
Compacted Block (合并后)
```

## 3.2 meta.json

```json
{
  "ulid": "01EM6Q6A1YPX4G9X6R6XG9X6R6",
  "minTime": 1600000000000,
  "maxTime": 1600007200000,
  "stats": {
    "numSamples": 1000000,
    "numSeries": 50000,
    "numChunks": 20000
  },
  "compaction": {
    "level": 1,
    "sources": ["01EM6Q6A1YPX4G9X6R6XG9X6R6"],
    "parents": []
  }
}
```

# 4. WAL（Write-Ahead Log）

WAL 是 TSDB 的**预写日志**，确保数据持久性。

## 4.1 WAL 工作流程

```
1. 样本数据到达 TSDB
2. 写入 WAL（顺序写，高性能）
3. 写入内存中的 Head Block
4. 定期 Checkpoint（合并 WAL 段）
5. Head Block 满 2h 后，数据落盘为新 Block
6. 清理已落盘的 WAL 段
```

## 4.2 WAL 恢复流程

```
Prometheus 崩溃重启：
1. 读取 WAL 目录下的所有段
2. 从最近的 checkpoint 开始恢复
3. 重放 WAL 中的样本到 Head Block
4. 丢弃已落盘 Block 对应的 WAL 数据
5. 恢复完成，继续正常写入
```

# 5. 数据压缩

## 5.1 Gorilla 压缩算法

Prometheus TSDB 使用 Facebook Gorilla 论文中的压缩算法：

| 数据类型 | 压缩方式 | 平均压缩比 |
|----------|----------|-----------|
| Timestamp | Delta-of-Delta 编码 | 每值 0.1-1.5 字节 |
| Value | XOR 编码 | 每值 1.5-5 字节 |

## 5.2 压缩效果

- 原始样本：16 字节（8 字节时间戳 + 8 字节值）
- 压缩后：平均 1.37 字节/样本（Prometheus 官方基准）
- 压缩比：约 12:1

## 5.3 Chunk 编码类型

| 编码 | 适用 |
|------|------|
| XOR | 浮点数（通用） |
| Histogram | 直方图数据 |
| Integer | 整数 Gauge（优化压缩） |

# 6. Compaction（压缩合并）

Compaction 负责将多个小 Block 合并为更大的 Block，并执行数据清理。

## 6.1 Compaction 级别

```
Level 0: Head Block → 初始 Block (2h)
Level 1: 10个 2h Block → 20h Block
Level 2: 3个 20h Block → 60h Block
Level 3: ... 以此类推
```

## 6.2 Compaction 触发条件

- 达到 Block 数量阈值（默认每级别最多 3 个 Block）
- 达到时间阈值
- 手动触发

## 6.3 Compaction 优化

```bash
--storage.tsdb.min-block-duration=2h
--storage.tsdb.max-block-duration=48h
--storage.tsdb.retention.time=15d
```

# 7. 数据保留（Retention）

## 7.1 基于时间的保留

```bash
--storage.tsdb.retention.time=15d  # 保留 15 天
```

## 7.2 基于大小的保留

```bash
--storage.tsdb.retention.size=100GB  # 保留最近 100GB 数据
```

## 7.3 同时设置

```bash
# 两个条件都设置时，满足任一条件即删除旧数据
--storage.tsdb.retention.time=30d
--storage.tsdb.retention.size=500GB
```

# 8. 索引结构

TSDB 使用**倒排索引（Inverted Index）** 加速查询。

## 8.1 索引结构

```
Postings List（倒排表）:
  {__name__="http_requests_total"} → [series_id1, series_id2, ...]
  {method="GET"}                  → [series_id1, series_id3, ...]
  {status="200"}                  → [series_id2, series_id3, ...]

查询：http_requests_total{method="GET", status="200"}
  → 取交集：series_id2
```

## 8.2 索引文件

| 部分 | 内容 |
|------|------|
| Symbol Table | 字符串去重表 |
| Series | 序列列表（标签集 → ID） |
| Label Indices | 标签名 → 标签值 映射 |
| Postings | 倒排索引（标签 → 序列 ID 列表） |
| Postings Offset Table | 倒排索引偏移表 |

# 9. 内存管理

## 9.1 Head Block 内存

Head Block 大部分数据在内存中，内存使用量约：

```
内存 ≈ 序列数 × 每个序列的样本数 × 1.37 字节
```

## 9.2 内存优化

```bash
# 控制最大内存使用
--storage.tsdb.max-block-duration=2h  # 更频繁落盘，减少内存占用
--storage.tsdb.retention.size=50GB    # 限制总存储大小

# 限制内存中的序列数量
--storage.tsdb.max-exemplars=10000
```

# 10. 存储指标监控

Prometheus 自身暴露 TSDB 状态指标：

```promql
# TSDB 数据块数量
prometheus_tsdb_blocks_loaded

# 当前 Head Block 中的序列数
prometheus_tsdb_head_series

# WAL 中的样本量
prometheus_tsdb_wal_storage_size_bytes

# 数据删除操作
prometheus_tsdb_tombstone_cleanups_total

# 压缩失败次数
prometheus_tsdb_compactions_failed_total

# 数据总样本数
prometheus_tsdb_head_samples_appended_total

# 存储大小
prometheus_tsdb_storage_blocks_bytes
```

## 10.1 存储健康检查

```promql
# 检查 WAL 是否正常写入
rate(prometheus_tsdb_wal_writes_total[5m])

# 检查 Compaction 延迟
time() - max(prometheus_tsdb_lowest_timestamp{job="prometheus"})

# 检查序列数是否异常增长
prometheus_tsdb_head_series > 1000000  # 超过 100 万需要注意
```

# 11. 数据管理操作

## 11.1 创建快照

```bash
# 启用 admin API
--web.enable-admin-api

# 创建快照
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot
```

## 11.2 删除数据

```bash
# 删除特定序列
curl -X POST \
  --data 'match[]=http_requests_total{job="old-app"}' \
  http://localhost:9090/api/v1/admin/tsdb/delete_series

# 清理墓碑文件（释放磁盘空间，需重启）
```

## 11.3 手动清理

```bash
# 停止 Prometheus
# 删除特定时间范围的 Block
# 启动 Prometheus
```
