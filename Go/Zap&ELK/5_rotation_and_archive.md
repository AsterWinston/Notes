日志文件如果只增不减，迟早会把磁盘写满；而磁盘一满，不只是日志写不进去，数据库、临时文件、甚至进程本身都会出问题。轮转（rotation）、压缩（compression）、保留（retention）和归档（archive）是一套组合拳，缺一环都会在某个凌晨以事故的形式找上门。本篇从策略讲到 Lumberjack 的落地配置。

# 1. 为什么需要轮转

## 1.1 单文件的问题

一个长期运行的服务，如果只往 `app.log` 追加：

| 问题 | 后果 |
|------|------|
| 文件无限增长 | 单文件几十 GB，`tail`、`grep` 都卡 |
| 磁盘写满 | 进程 panic、数据库损坏、系统异常 |
| 无法删除旧数据 | 想清理只能删整个文件 |
| Filebeat 重读成本高 | 大文件从头扫描慢，inode 变化时更麻烦 |
| 备份困难 | 无法按天/按大小归档 |

## 1.2 轮转的三个维度

```text
app.log                ← 当前写入
app-2026-11-06.log     ← 按时间切
app-2026-11-06.001.log ← 同一天内按大小切
app-2026-11-06.001.log.gz ← 压缩
```

| 维度 | 触发条件 | 适合 |
|------|----------|------|
| 按大小 | 超过 `MaxSize` MB | 流量波动大，防止单文件过大 |
| 按时间 | 每天/每小时 | 便于按日期归档、对齐业务 |
| 按数量 | 保留最近 N 个 | 简单，但时间跨度不可控 |
| 按年龄 | 超过 `MaxAge` 天 | 合规保留期 |

★ **关键认知**：轮转不是"把文件切小"，而是**同时解决写入、采集、保留、恢复四件事**。切分只是手段，真正要保证的是：采集端不丢行、旧文件能自动清理、磁盘有上限。

# 2. 轮转策略

## 2.1 按大小

```text
MaxSize = 100 MB
app.log 写到 100 MB → 改名 app-2026-11-06T10-00-00.000.log
                     → 新建 app.log 继续写
```

按大小的优点是**磁盘占用可预测**：最大占用 ≈ `MaxSize × (MaxBackups + 1)`。缺点是时间跨度不稳定：低峰期可能三天一个文件，高峰期一小时三个文件。

## 2.2 按时间

按时间轮转需要一个定时器或外部工具（如 `logrotate`）。Go 程序里常见做法是自建一个按天切分的 `WriteSyncer`，或直接用 Lumberjack 的 `MaxAge` 配合大小轮转。

```text
2026-11-05 00:00 → 关闭 app.log，改名 app-2026-11-05.log，新建 app.log
```

按时间的好处是**归档和查询对齐**：`app-2026-11-05.log` 一眼知道是哪天。

## 2.3 组合策略

生产上最常见的组合是：

```yaml
单文件上限：100 MB       # 防止单文件过大
最多保留：7 个备份       # 控制磁盘上限
最长保留：30 天          # 满足合规
压缩：开启               # 省 80% 空间
```

★ **关键认知**：**大小和时间必须同时设**。只设大小，低峰期文件可能跨越很多天，删除时不好按日期清理；只设时间，高峰期一天内可能写出几十 GB 的单文件，采集和打开都困难。

# 3. 压缩与保留

## 3.1 压缩的收益

日志是文本，压缩率通常很高：

| 类型 | 原始大小 | gzip 后 | 压缩率 |
|------|----------|---------|--------|
| JSON 日志 | 1 GB | 120-200 MB | 5-8 倍 |
| 纯文本访问日志 | 1 GB | 80-150 MB | 7-12 倍 |
| 已压缩的二进制 | 1 GB | 900 MB | 几乎无收益 |

```bash
# 手动验证压缩率
gzip -k app-2026-11-05.log
ls -lh app-2026-11-05.log app-2026-11-05.log.gz
```

⚠ **关键认知**：压缩会消耗 CPU，而且**压缩后的文件 Filebeat 不能直接按行读取**（需要先解压，或用支持 gzip 的采集方式）。所以"当前正在写的文件不压缩、已轮转的文件才压缩"是标准做法，Lumberjack 的 `Compress: true` 正是这个语义。

## 3.2 保留策略矩阵

| 场景 | 本地保留 | 归档 | 说明 |
|------|----------|------|------|
| 开发环境 | 3 天 / 500 MB | 无 | 够调试即可 |
| 一般业务 | 7-14 天 | 30-90 天到对象存储 | 覆盖一次发布周期 |
| 金融/医疗 | 30 天 | 1-3 年，WORM | 合规要求 |
| 审计日志 | 90 天 | 3 年以上，不可篡改 | 单独存储 |

保留策略要写进代码或配置，不能靠"记得手动删"：

```go
lumberjack.Logger{
    MaxBackups: 7,   // 最多 7 个备份
    MaxAge:     30,  // 超过 30 天的备份自动删除
    Compress:   true, // 备份压缩
}
```

★ **关键认知**：`MaxBackups` 和 `MaxAge` 是**或**的关系，任一条件满足就删除。所以两个都设时，实际保留时间可能比 `MaxAge` 更短。排查"日志怎么只剩三天"时，先看 `MaxBackups` 是不是太小。

# 4. 磁盘保护

## 4.1 磁盘水位

即使有轮转，也可能因为流量突增、日志级别被误调、或轮转配置错误而写满磁盘。要有兜底：

```go
// 启动时检查磁盘剩余空间，低于阈值直接拒绝启动或切到 Nop logger
func checkDisk(path string, minFreeGB float64) error {
    var stat syscall.Statfs_t
    if err := syscall.Statfs(path, &stat); err != nil {
        return err
    }
    freeGB := float64(stat.Bavail*uint64(stat.Bsize)) / (1 << 30)
    if freeGB < minFreeGB {
        return fmt.Errorf("disk free %.1fGB below threshold %.1fGB", freeGB, minFreeGB)
    }
    return nil
}
```

## 4.2 写入失败怎么办

磁盘满时，写日志会返回 `no space left on device`。此时有三种处理策略：

| 策略 | 行为 | 适合 |
|------|------|------|
| 丢弃并计数 | 继续服务，日志丢失 | 核心业务，不能因日志挂掉 |
| 降级到 Nop | 关闭日志，保证主流程 | 同上，但更彻底 |
| 阻塞重试 | 等磁盘恢复 | 审计场景，宁可慢不能丢 |

```go
// 用自定义 WriteSyncer 在写入失败时计数并降级
type safeSyncer struct {
    inner zapcore.WriteSyncer
    atom  *atomic.Bool
}

func (s *safeSyncer) Write(p []byte) (int, error) {
    if s.atom.Load() {
        return len(p), nil // 已降级，直接丢弃，不阻塞业务
    }
    n, err := s.inner.Write(p)
    if err != nil {
        s.atom.Store(true) // 标记降级，避免每次都尝试
        return len(p), nil // 对 Zap 报告成功，防止错误日志刷屏
    }
    return n, nil
}

func (s *safeSyncer) Sync() error { return s.inner.Sync() }
```

⚠ **关键认知**：日志写入失败时返回 `nil` 是一种**有意的取舍**，不是 bug。对 Zap 来说，`Write` 返回错误会被写到 `ErrorOutputPaths`，如果磁盘已满，这又会失败，可能形成刷屏。降级 + 计数 + 告警，比让日志拖垮业务更合理。

# 5. Lumberjack

## 5.1 最小示例

`lumberjack` 是 Zap 官方文档推荐的轮转库，只做一件事：按大小轮转、按数量/年龄清理、可选压缩。

```bash
go get gopkg.in/natefinch/lumberjack.v2
```

```go
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
    "gopkg.in/natefinch/lumberjack.v2"
)

func main() {
    // Lumberjack 实现了 io.Writer，用 AddSync 适配成 WriteSyncer
    w := &lumberjack.Logger{
        Filename:   "/var/log/order-api/app.log", // 当前写入文件
        MaxSize:    100,                          // 单文件 100 MB 后轮转
        MaxBackups: 10,                           // 最多保留 10 个备份
        MaxAge:     30,                           // 备份最多保留 30 天
        Compress:   true,                         // 备份 gzip 压缩
        LocalTime:  true,                         // 文件名用本地时间
    }

    cfg := zap.NewProductionEncoderConfig()
    cfg.EncodeTime = zapcore.ISO8601TimeEncoder

    core := zapcore.NewCore(
        zapcore.NewJSONEncoder(cfg),
        zapcore.AddSync(w), // Lumberjack 内部有锁，无需再包 Lock
        zap.InfoLevel,
    )

    logger := zap.New(core, zap.AddCaller())
    defer logger.Sync()

    logger.Info("app started", zap.String("addr", ":8080"))
}
```

## 5.2 参数详解

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `Filename` | string | 必填 | 当前写入的文件路径 |
| `MaxSize` | int | 100 | 单文件最大 MB |
| `MaxBackups` | int | 0 | 最多保留几个备份，0 表示不限 |
| `MaxAge` | int | 0 | 备份最多保留几天，0 表示不限 |
| `Compress` | bool | false | 是否 gzip 压缩备份 |
| `LocalTime` | bool | false | 备份文件名用本地时间还是 UTC |
| `RotateInterval` | time.Duration | 0 | 按时间轮转（v2.2+） |

轮转后的文件名格式是 `app-2026-11-06T10-00-00.000.log`，UTC 时间，精确到毫秒，避免同一秒内多次轮转冲突。

## 5.3 与 Zap 集成

```go
func NewFileLogger(path string) *zap.Logger {
    rotator := &lumberjack.Logger{
        Filename:   path,
        MaxSize:    200,
        MaxBackups: 14,
        MaxAge:     14,
        Compress:   true,
        LocalTime:  true,
    }

    core := zapcore.NewCore(
        zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
        zapcore.AddSync(rotator),
        zap.InfoLevel,
    )

    return zap.New(core, zap.AddCaller(), zap.AddStacktrace(zap.ErrorLevel))
}
```

## 5.4 已知限制

| 限制 | 说明 | 应对 |
|------|------|------|
| 不支持按天精确切分 | 只有 `RotateInterval` 近似 | 用外部 logrotate 或接受近似 |
| 轮转时短暂阻塞写 | `Write` 内部加锁 + 改名 | 高频写入时注意延迟 |
| 删除备份是尽力而为 | 失败只记录，不重试 | 监控磁盘，定期巡检 |
| 不清理"当前文件" | 只清理备份 | 当前文件由 `MaxSize` 控制 |
| Windows 上改名可能失败 | 文件被占用 | 避免多进程写同一文件 |

⚠ **关键认知**：多个进程（或容器的多个副本）写**同一个日志文件**时，Lumberjack 的锁只在进程内有效，跨进程仍会交错、轮转冲突。正确做法是每个实例写自己的文件（用 Pod 名或 PID 区分），由采集端汇总。

# 6. 归档与清理

## 6.1 本地归档

适合中小规模：轮转后的 `.gz` 文件保留在本地，定时同步到远端。

```bash
# 每天凌晨把前一天的日志同步到归档目录
find /var/log/order-api -name "app-*.log.gz" -mtime +1 -exec mv {} /archive/order-api/ \;

# 清理归档目录中超过 90 天的文件
find /archive/order-api -name "*.log.gz" -mtime +90 -delete
```

## 6.2 对象存储归档

规模大时，用 S3/OSS/MinIO 归档，成本远低于块存储：

```bash
# 用 rclone 同步到 S3，按日期分目录
rclone copy /var/log/order-api s3:logs-prod/order-api/2026/11/06/ \
  --include "*.gz" \
  --transfers 8
```

| 存储 | 成本 | 取回速度 | 适合 |
|------|------|----------|------|
| 本地 SSD | 高 | 极快 | 当前写入、最近 7 天 |
| 本地 HDD | 中 | 快 | 最近 30 天 |
| 对象存储标准 | 低 | 秒级 | 30-180 天 |
| 对象存储归档 | 极低 | 分钟-小时 | 合规长期保留 |

## 6.3 清理任务要可观测

清理脚本最怕"静默失败"：磁盘满了才发现根本没删。至少要做两件事：

```bash
#!/usr/bin/env bash
set -euo pipefail

DIR=/var/log/order-api
DAYS=30

deleted=$(find "$DIR" -name "*.log.gz" -mtime +$DAYS -print -delete | wc -l)
# 打点，方便告警"清理任务是否还在跑"
echo "log_cleanup_deleted_total{dir=\"$DIR\"} $deleted" >> /var/lib/node_exporter/textfile/log_cleanup.prom
```

★ **关键认知**：归档和清理是**两条独立的链路**。归档失败不代表可以停清理（会写满磁盘），清理失败也不代表可以停归档（会丢历史）。两者都要有监控，且**先保磁盘、再保历史**。

# 7. 常见错误

| 问题 | 现象 | 修复 |
|------|------|------|
| 只设 `MaxSize` 不设 `MaxBackups` | 磁盘无限增长 | 同时设数量和年龄 |
| `MaxBackups` 太小 | 日志只留几小时 | 按峰值估算，至少留 7 天 |
| 对当前文件压缩 | Filebeat 读不了 | 只压缩轮转后的备份 |
| 多副本写同一文件 | 行交错、轮转冲突 | 每实例独立文件 |
| 没设 `LocalTime` | 文件名时间与业务对不上 | 按团队时区统一 |
| 清理脚本无监控 | 静默失败，磁盘满 | 打点 + 告警 |
| 日志级别被调到 Debug 忘记改回 | 磁盘暴涨 | 调级接口带超时自动恢复 |

⚠ **关键认知**：**轮转不等于备份**。轮转后的文件仍在同一块磁盘上，磁盘损坏、误删、勒索软件都会一起带走。真正的备份必须在**另一台机器或对象存储**上，且要定期验证可恢复。

# 8. 练习与自检

1. 一个服务峰值每秒写 5 MB 日志，要求单文件不超过 200 MB、本地最多占 2 GB、保留 14 天。请给出 Lumberjack 的参数，并验算磁盘上限。
2. 为什么 `Compress: true` 不会影响正在写入的 `app.log`？请从 Lumberjack 的轮转流程解释。
3. 设计一个"磁盘剩余 5% 时自动降级日志"的方案，要求不阻塞业务、能告警、恢复后能自动回到正常。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 轮转 | 大小 + 时间组合，避免单文件过大和跨天 |
| 压缩 | 只压缩已轮转文件，省 80% 空间 |
| 保留 | `MaxBackups` 与 `MaxAge` 取先满足者 |
| 磁盘保护 | 水位检查 + 写入失败降级 + 告警 |
| Lumberjack | `io.Writer` 适配 `AddSync`，内部有锁 |
| 多实例 | 每实例独立文件，禁止跨进程共享 |
| 归档 | 本地 + 对象存储分层，清理任务要可观测 |
| 备份 | 轮转不是备份，必须异地 |

下一篇进入 ELK 架构，讲清 Elasticsearch 的索引与分片、Logstash 与 Filebeat 的分工、数据从产生到可查询的完整链路，以及容量规划的基本方法。
