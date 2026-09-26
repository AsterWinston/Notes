# 1.MinIO 性能特点

## 1.1性能指标参考

| 指标 | 参考值 | 条件 |
|------|--------|------|
| GET 吞吐 | 10 GB/s+ | 单节点, NVMe, 100Gbps 网络, 大文件 |
| PUT 吞吐 | 5 GB/s+ | 单节点, NVMe, 100Gbps 网络, 大文件 |
| GET IOPS | 100K+ | 小文件 (< 1MB), NVMe |
| PUT IOPS | 50K+ | 小文件, NVMe |
| GET 延迟 (P99) | < 1ms | 内存缓存命中 |
| GET 延迟 (P99) | < 10ms | NVMe 磁盘, 局域网 |
| 集群线性扩展 | 近乎线性 | 16节点集群性能 ≈ 16 × 单节点性能 |

> 以上数据来自 MinIO 官方基准测试，实际性能取决于硬件配置、网络、文件大小分布和工作负载模式。

## 1.2影响性能的关键因素

```text
影响 MinIO 性能的因素（按重要程度排序）:

1. 网络带宽和延迟 ★★★★★
   - 节点间网络是最关键的瓶颈
   - 建议: 25Gbps+ 网卡，低延迟交换机
   
2. 磁盘 IOPS 和吞吐 ★★★★
   - NVMe >> SATA SSD >> HDD
   - 建议: 使用 NVMe 或企业级 SSD
   
3. 文件大小分布 ★★★★
   - 大文件（>100MB）: 吞吐密集，受网络/磁盘带宽限制
   - 小文件（<1MB）: IOPS 密集，受磁盘 IOPS 和元数据操作限制
   - 建议: 避免存海量极小文件（< 4KB）
   
4. EC 配置 ★★★
   - EC 条带数越多，读写需要参与的分片越多
   - 8+8 配置: 每次读取需要 8 个分片，写入需要 9 个成功确认
   - 建议: 对于性能敏感场景，可以降低校验比（如 12+4）

5. 客户端连接数和并发 ★★★
   - MinIO 使用 Go 协程，天然支持高并发
   - 建议: 客户端使用连接池，充分并发
   
6. 内核参数 ★★
   - 网络和文件系统相关的内核参数调优
   
7. CPU ★
   - Erasure Coding 编解码消耗 CPU（尤其写入时）
   - SHA-256 签名计算消耗 CPU
   - 建议: 至少 8 核，推荐 16 核+
```

---

# 2.基准测试工具

## 2.1使用 warp（MinIO 官方基准测试工具）

warp 是 MinIO 团队开发的专门针对 S3 兼容存储的基准测试工具，比传统 `s3bench` 更准确。

```bash
# 安装 warp
wget https://github.com/minio/warp/releases/latest/download/warp_linux_amd64
chmod +x warp_linux_amd64
sudo mv warp_linux_amd64 /usr/local/bin/warp

# 基本 PUT 基准测试
warp put \
  --access-key=minioadmin \
  --secret-key=minioadmin \
  --host=localhost:9000 \
  --bucket=warp-bench \
  --objects=10000 \
  --obj.size=10MB \
  --concurrent=20

# GET 基准测试
warp get \
  --access-key=minioadmin \
  --secret-key=minioadmin \
  --host=localhost:9000 \
  --bucket=warp-bench \
  --objects=10000 \
  --concurrent=50

# 混合负载基准测试
warp mixed \
  --access-key=minioadmin \
  --secret-key=minioadmin \
  --host=localhost:9000 \
  --bucket=warp-bench \
  --objects=1000 \
  --obj.size=1MB \
  --concurrent=100 \
  --get-distrib=60 \
  --put-distrib=30 \
  --delete-distrib=10 \
  --duration=5m
```

## 2.2warp 输出解读

```text
Operation: PUT
Duration: 5m0s
     Requests: 124,567
        Errors: 0
   Throughput: 41.5 objs/s, 415.0 MB/s
         Avg:  480ms
        P50:   450ms
        P90:   620ms
        P99:   850ms
   Slowest:    1.2s
```

### 关键指标

| 指标 | 关注重点 |
|------|----------|
| Throughput | 总吞吐量（对象数/秒、数据量/秒），反映整体性能 |
| P50 (中位数) | 典型延迟，代表普通用户的体验 |
| P99 | 长尾延迟，P99过高说明存在偶发性能问题 |
| Errors | 错误率必须为 0，否则说明系统存在稳定性问题 |

---

# 3.硬件选型建议

## 3.1CPU 选型

| 场景 | 推荐配置 |
|------|----------|
| 开发/测试 | 4 核 |
| 小规模生产 (<50TB) | 8 核 |
| 中等规模生产 (50TB~500TB) | 16 核 |
| 大规模生产 (>500TB) | 32-64 核 |

- 推荐 Intel Xeon 或 AMD EPYC
- 开启 CPU 性能模式（governor: performance），关闭 C-States
- 开启 Turbo Boost

## 3.2内存选型

| 场景 | 推荐配置 |
|------|----------|
| 开发/测试 | 8 GB |
| 生产环境 | 32-128 GB，Linux 自动将空闲内存用于 Page Cache |
| 高密度 | 256 GB+ |

> MinIO 本身内存占用低（Go 语言的 GC 高效），大量内存主要用于 OS 文件系统缓存，加速热数据访问。

## 3.3磁盘选型

| 类型 | IOPS | 吞吐 | 推荐场景 |
|------|------|------|----------|
| NVMe SSD | 100K-1M | 3-7 GB/s | 生产环境首选 |
| SATA SSD | 50K-100K | 500 MB/s | 小规模生产 |
| HDD (7200rpm) | 100-200 | 150 MB/s | 冷存储、归档 |
| NVMe over Fabrics | 1M+ | 10+ GB/s | 超高性能场景 |

**磁盘配置建议**：
- 每节点 8-16 块磁盘（最佳 EC 条带）
- 不要使用 RAID（MinIO 的 EC 替代了 RAID）
- 直接使用 JBOD 模式（各磁盘独立挂载）
- 使用一致的磁盘型号和容量

## 3.4网络选型

| 规模 | 推荐网络 |
|------|----------|
| 开发/测试 | 千兆 (1Gbps) |
| 小规模生产 | 万兆 (10Gbps) |
| 中大规模生产 | 25Gbps / 100Gbps |

---

# 4.操作系统与内核调优

## 4.1Linux 内核参数优化

```bash
# /etc/sysctl.conf 或 /etc/sysctl.d/99-minio.conf

# 网络调优
net.core.rmem_max = 134217728          # 最大接收缓冲区 (128MB)
net.core.wmem_max = 134217728          # 最大发送缓冲区 (128MB)
net.core.rmem_default = 134217728
net.core.wmem_default = 134217728
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.ipv4.tcp_tw_reuse = 1              # 复用 TIME_WAIT 连接
net.ipv4.tcp_fin_timeout = 30          # FIN 超时时间
net.ipv4.tcp_keepalive_time = 600      # KeepAlive 探测间隔

# 提高文件描述符限制
fs.file-max = 1000000
fs.nr_open = 1000000

# 减少脏页比率（适合 NVMe，避免大量脏页积压）
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5

# 禁用透明大页（THP 可能导致性能抖动）
vm.zone_reclaim_mode = 0

# SWAP 设置
vm.swappiness = 1                       # 尽量不使用 Swap（但保留 Swap 避免 OOM）

# 应用配置
sudo sysctl --system
```

## 4.2进程文件描述符限制

```bash
# /etc/security/limits.conf
minio-user  soft  nofile  1000000
minio-user  hard  nofile  1000000
minio-user  soft  nproc   65536
minio-user  hard  nproc   65536

# systemd 服务中设置（minio.service）
[Service]
LimitNOFILE=1000000
LimitNPROC=65536
```

## 4.3磁盘挂载优化

```bash
# /etc/fstab
# 关键挂载选项:
#   noatime   - 不更新访问时间（减少写入操作）
#   nodiratime - 不更新目录访问时间
#   largeio   - XFS 大 I/O 优化
#   inode64   - XFS 支持超过 2TB 的文件系统
#   swalloc   - XFS stripe width 对齐优化

LABEL=DISK1 /mnt/disk1 xfs defaults,noatime,nodiratime,largeio,inode64,swalloc 0 2
```

## 4.4CPU 性能模式设置

```bash
# 确保 CPU 工作在 performance 模式
for CPUFREQ in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance > $CPUFREQ
done

# 或者安装 cpupower 工具
sudo apt install linux-tools-common
sudo cpupower frequency-set -g performance
```

---

# 5.MinIO 本身配置优化

## 5.1MinIO 环境变量优化

```bash
# MinIO 高性能配置
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=<强密码>

# 增加 API 请求超时时间（处理大对象上传）
export MINIO_API_REQUESTS_MAX=0         # API 最大请求速率（0=不限）

# 增加扫描器并发度（加速后台 Heal 和元数据扫描）
export MINIO_SCANNER_SPEED=fast         # fast(default)/slow

# 增加最大连接数
# (通过 mc admin config set)

minio server /data --console-address ":9001"
```

## 5.2使用 mc 调优运行时配置

```bash
# 查看当前配置
mc admin config get myminio api

# 调整 API 配置
mc admin config set myminio api \
  requests_max=0              # 不限制请求速率
  cors_allow_origin="*"

# 调整 Heal 配置
mc admin config set myminio heal \
  max_sleep=1s               # Heal 操作最大等待间隔
  max_io=10                  # Heal 最大并发 I/O

# 调整 Scanner 配置
mc admin config set myminio scanner \
  speed=fast                  # 扫描速度: fast/default/slow

# 重启服务使配置生效
mc admin service restart myminio
```

---

# 6.客户端优化

## 6.1Java 客户端优化

```java
// 生产环境 MinioClient 配置
OkHttpClient httpClient = new OkHttpClient.Builder()
    // 连接超时
    .connectTimeout(10, TimeUnit.SECONDS)
    // 读写超时（大文件场景需要足够长）
    .writeTimeout(30, TimeUnit.MINUTES)
    .readTimeout(30, TimeUnit.MINUTES)
    // 连接池
    .connectionPool(new ConnectionPool(
        200,                              // 最大空闲连接数
        5, TimeUnit.MINUTES               // 空闲连接保活时间
    ))
    // 协议支持
    .protocols(Arrays.asList(Protocol.HTTP_2, Protocol.HTTP_1_1))
    // 启用连接失败重试
    .retryOnConnectionFailure(true)
    // 禁用请求缓冲（流式上传）
    // (minio SDK 内部已处理)
    .build();

MinioClient client = MinioClient.builder()
    .endpoint("https://minio.example.com:9000")
    .credentials(accessKey, secretKey)
    .httpClient(httpClient)
    .build();
```

## 6.2并发上传策略

```java
import java.util.concurrent.*;

// 并发上传多个文件
ExecutorService executor = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors() * 2
);

List<CompletableFuture<UploadResult>> futures = files.stream()
    .map(file -> CompletableFuture.supplyAsync(() -> {
        try {
            return fileService.uploadFile(file, generateObjectName(file), null);
        } catch (Exception e) {
            throw new CompletionException(e);
        }
    }, executor))
    .toList();

// 等待所有上传完成
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
```

---

# 7.Prometheus 监控

## 7.1配置 MinIO 暴露 Prometheus 指标

MinIO 自动在 `/minio/v2/metrics/cluster` 端点暴露 Prometheus 格式的指标。

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'minio'
    metrics_path: /minio/v2/metrics/cluster
    scheme: http
    static_configs:
      - targets: ['minio1.example.com:9000', 'minio2.example.com:9000']
    # 如果使用 HTTPS
    # scheme: https
    # tls_config:
    #   insecure_skip_verify: true
```

## 7.2关键 Prometheus 指标

### 系统级指标

```promql
# 集群可用容量（字节）
minio_cluster_capacity_usable_total_bytes

# 集群已用容量（字节）
minio_cluster_capacity_usable_free_bytes

# 节点在线状态
minio_node_disk_free_bytes

# 磁盘使用率
(minio_node_disk_total_bytes - minio_node_disk_free_bytes) / minio_node_disk_total_bytes * 100

# 磁盘健康状态（1=健康, 0=故障）
minio_node_disk_health_status
```

### S3 API 指标

```promql
# 请求总数（按 API 分组）
sum(rate(minio_s3_requests_total[5m])) by (api)

# 请求错误总数
sum(rate(minio_s3_requests_errors_total[5m])) by (api)

# 请求延迟（P99）
histogram_quantile(0.99, sum(rate(minio_s3_time_seconds_bucket[5m])) by (le, api))

# S3 流量（发送）
rate(minio_s3_traffic_sent_bytes[5m])

# S3 流量（接收）
rate(minio_s3_traffic_received_bytes[5m])
```

### 网络指标

```promql
# 节点间网络流量
rate(minio_inter_node_traffic_sent_bytes[5m])
rate(minio_inter_node_traffic_received_bytes[5m])
```

### 缓存指标

```promql
# 总缓存命中次数
minio_node_cache_hits_total

# 总缓存未命中次数
minio_node_cache_misses_total

# 缓存命中率
minio_node_cache_hits_total / (minio_node_cache_hits_total + minio_node_cache_misses_total)
```

### Heal 指标

```promql
# Heal 对象总数
minio_heal_objects_total

# Heal 字节总数
minio_heal_bytes_total

# Heal 失败次数
minio_heal_errors_total
```

---

# 8.Grafana 监控面板

## 8.1导入官方 Dashboard

MinIO 官方提供了预制的 Grafana Dashboard（ID: 13502）：

```bash
# 通过 Grafana UI 导入
# Dashboards → Import → 输入 13502
# 或通过 Grafana API
curl -X POST http://grafana:3000/api/dashboards/import \
  -H "Content-Type: application/json" \
  -d '{"dashboard": {...}, "overwrite": true}'
```

## 8.2关键告警规则

```yaml
# Prometheus 告警规则 (minio-alerts.yml)
groups:
  - name: minio_alerts
    rules:
      # 磁盘使用率 > 80%
      - alert: MinioDiskUsageHigh
        expr: |
          (1 - minio_node_disk_free_bytes / minio_node_disk_total_bytes) > 0.80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 磁盘使用率过高 ({{ $value | humanizePercentage }})"
          description: "节点 {{ $labels.node }}, 磁盘 {{ $labels.disk }}"

      # 磁盘使用率 > 90%
      - alert: MinioDiskUsageCritical
        expr: |
          (1 - minio_node_disk_free_bytes / minio_node_disk_total_bytes) > 0.90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 磁盘使用率严重过高"

      # 磁盘故障
      - alert: MinioDiskDown
        expr: minio_node_disk_health_status == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 磁盘故障 (节点: {{ $labels.node }})"

      # 节点离线
      - alert: MinioNodeOffline
        expr: minio_cluster_nodes_offline_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MinIO 节点离线 (数量: {{ $value }})"

      # 请求错误率 > 5%
      - alert: MinioHighErrorRate
        expr: |
          sum(rate(minio_s3_requests_errors_total[5m])) by (api)
          /
          sum(rate(minio_s3_requests_total[5m])) by (api)
          > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MinIO 请求错误率过高 ({{ $value | humanizePercentage }})"
```

---

# 9.日志管理

## 9.1日志格式配置

```bash
# 查看日志配置
mc admin config get myminio logger

# 配置日志输出为 JSON 格式（便于日志采集系统解析）
mc admin config set myminio logger_webhook:primary \
  endpoint="http://logstash:8080" \
  auth_token="token123"

# 重新加载配置
mc admin service restart myminio
```

## 9.2日志采集方案

使用 Filebeat 采集 MinIO Console 日志：

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/minio/minio.log
    json.keys_under_root: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
```

---

# 10.健康检查

## 10.1健康检查端点

```bash
# 存活检查（Liveness Probe）
curl http://localhost:9000/minio/health/live
# 返回 200 OK: 服务正在运行

# 就绪检查（Readiness Probe）
curl http://localhost:9000/minio/health/ready
# 返回 200 OK: 服务已就绪可接收请求
# 返回 503: 集群正在初始化/修复中

# 集群健康检查
curl http://localhost:9000/minio/health/cluster
# 返回 drd/stored_data(已修复/总数据)信息
```

## 10.2K8s 健康检查配置

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: minio
      image: minio/minio:latest
      livenessProbe:
        httpGet:
          path: /minio/health/live
          port: 9000
        initialDelaySeconds: 30
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3

      readinessProbe:
        httpGet:
          path: /minio/health/ready
          port: 9000
        initialDelaySeconds: 10
        periodSeconds: 10
        timeoutSeconds: 5
        successThreshold: 1
        failureThreshold: 3
```

---

# 11.性能问题排查

## 11.1常见性能问题

| 问题 | 可能原因 | 排查方法 |
|------|----------|----------|
| PUT 吞吐低 | 网络带宽不足、磁盘IO瓶颈、EC编码开销 | 检查网络使用率、iostat、CPU使用率 |
| GET 延迟高 | 磁盘读取慢、网络延迟、缺少缓存 | iostat 查看 await 指标、ping 节点间延迟 |
| 小文件PUT慢 | 元数据操作过多、IOPS瓶颈 | 减少小文件、合并上传、优化 EC 配置 |
| 集群不稳定 | 节点间时间不同步、磁盘性能不一致 | ntpq -p 检查时间偏差、iostat 检查各盘性能 |

## 11.2性能诊断工具

```bash
# 磁盘 I/O 性能
iostat -x 1                    # 查看每块磁盘的 await, util, r/s, w/s

# 网络性能
iperf3 -s                      # 服务器端
iperf3 -c <server_ip> -t 30   # 客户端，测试 30 秒

# 查看 MinIO 内部性能
mc admin info myminio

# 查看正在进行的分片上传（可能阻塞资源）
mc admin trace myminio         # 实时跟踪请求

# 查看慢请求
mc admin top locks myminio     # 查看当前锁争用
```
