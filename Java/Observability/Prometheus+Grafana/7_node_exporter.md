
# 1. 概述

Node Exporter 是 Prometheus 官方推荐的**主机监控利器**，用于采集 Linux/Windows/macOS 系统的各类指标数据，运行在 9100 端口。

# 2. 安装与部署

## 2.1 二进制安装

```bash
# Linux
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz
tar xvf node_exporter-1.8.0.linux-amd64.tar.gz
cd node_exporter-1.8.0.linux-amd64

# 启动
./node_exporter
```

## 2.2 Docker 部署

```bash
docker run -d \
  --name node_exporter \
  --restart always \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  prom/node-exporter:v1.8.0 \
  --path.rootfs=/host
```

## 2.3 Systemd 服务

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

# 3. 核心采集器（Collectors）

Node Exporter 通过**采集器（Collector）** 机制获取不同类别的系统指标。默认启用所有 Collector。

| Collector | 默认 | 指标示例 |
|-----------|------|----------|
| cpu | 启用 | `node_cpu_seconds_total` |
| diskstats | 启用 | `node_disk_reads_completed_total` |
| filesystem | 启用 | `node_filesystem_avail_bytes` |
| loadavg | 启用 | `node_load1`, `node_load5`, `node_load15` |
| meminfo | 启用 | `node_memory_MemTotal_bytes` |
| netdev | 启用 | `node_network_receive_bytes_total` |
| stat | 启用 | `node_context_switches_total` |
| textfile | 启用 | 用户自定义文本指标 |
| uname | 启用 | `node_uname_info` |
| systemd | 可选 | `node_systemd_unit_state` |
| nvme | 可选 | `node_nvme_info` |
| hwmon | 可选 | `node_hwmon_temp_celsius` |
| infiniband | 可选 | `node_infiniband_port_data_transmitted_bytes_total` |

## 3.1 启用/禁用 Collector

```bash
# 只启用特定 collector
./node_exporter --collector.disable-defaults \
  --collector.cpu \
  --collector.meminfo \
  --collector.filesystem

# 禁用特定 collector
./node_exporter --no-collector.dmi \
  --no-collector.zfs

# 启用可选 collector
./node_exporter --collector.systemd \
  --collector.processes
```

# 4. CPU 指标

```promql
# CPU 使用率（排除 idle）
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# 按 CPU 核心查看使用率
100 - (avg by (instance, cpu) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# CPU 各模式时间占比
rate(node_cpu_seconds_total[5m])

# CPU Load（1/5/15 分钟平均）
node_load1
node_load5
node_load15

# CPU 上下文切换
rate(node_context_switches_total[5m])
```

## 4.1 CPU Mode 说明

| Mode | 说明 |
|------|------|
| user | 用户态程序执行时间 |
| system | 内核态系统调用时间 |
| idle | CPU 空闲时间 |
| iowait | 等待 IO 完成 |
| irq | 处理硬件中断 |
| softirq | 处理软件中断 |
| steal | 虚拟机偷取时间 |
| guest | 运行 Guest OS |

# 5. 内存指标

```promql
# 总内存
node_memory_MemTotal_bytes

# 可用内存（包含 buffer/cache）
node_memory_MemAvailable_bytes

# 已用内存
node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 页缓存 + Buffer
node_memory_Buffers_bytes + node_memory_Cached_bytes

# Swap 使用率
(node_memory_SwapTotal_bytes > 0)
  and (node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes)
  / node_memory_SwapTotal_bytes * 100

# NUMA 内存分配
node_memory_NumaAlloc
```

## 5.1 内存指标对应关系

```
+-------------------+-------------------------------+
| /proc/meminfo     | Prometheus 指标               |
+-------------------+-------------------------------+
| MemTotal          | node_memory_MemTotal_bytes    |
| MemFree           | node_memory_MemFree_bytes     |
| MemAvailable      | node_memory_MemAvailable_bytes|
| Buffers           | node_memory_Buffers_bytes     |
| Cached            | node_memory_Cached_bytes      |
| SwapTotal         | node_memory_SwapTotal_bytes   |
| SwapFree          | node_memory_SwapFree_bytes    |
| Dirty             | node_memory_Dirty_bytes       |
| Writeback         | node_memory_Writeback_bytes   |
+-------------------+-------------------------------+
```

# 6. 磁盘指标

```promql
# 磁盘使用率
(1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes) * 100

# 根据 fstype 过滤
node_filesystem_size_bytes{fstype="ext4"}
node_filesystem_avail_bytes{mountpoint="/data"}

# 磁盘 IO 吞吐量
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])

# 磁盘 IOPS
rate(node_disk_reads_completed_total[5m])
rate(node_disk_writes_completed_total[5m])

# 磁盘 IO 延迟
rate(node_disk_read_time_seconds_total[5m])
  / rate(node_disk_reads_completed_total[5m])
  * 1000  # 转为毫秒

# 磁盘当前队列深度
node_disk_io_now

# 磁盘使用时间百分比
rate(node_disk_io_time_seconds_total[5m])
```

## 6.1 磁盘 IO 监控最佳实践

```promql
# 排除不关心的分区
node_filesystem_avail_bytes{
  mountpoint!~"/(boot|dev|proc|sys|run).*"
}

# 关注特定磁盘
rate(node_disk_read_bytes_total{device=~"sd[a-z]"}[5m])
```

# 7. 网络指标

```promql
# 网络吞吐量
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])

# 网络错误率
rate(node_network_receive_errors_total[5m])
rate(node_network_transmit_errors_total[5m])

# 网络丢包率
rate(node_network_receive_drop_total[5m])
rate(node_network_transmit_drop_total[5m])

# TCP 连接数
node_netstat_Tcp_CurrEstab

# 排除 loopback 接口
rate(node_network_receive_bytes_total{device!="lo"}[5m])
```

# 8. Textfile Collector

用于采集 Node Exporter 无法直接收集的指标，通过运行脚本生成文本文件供 Node Exporter 读取。

## 8.1 配置方式

```bash
# 启动时指定目录
./node_exporter --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

## 8.2 脚本示例

```bash
#!/bin/bash
# /var/lib/node_exporter/textfile_collector/custom_metrics.prom

# 采集自定义指标
echo "# HELP custom_app_health Custom application health status"
echo "# TYPE custom_app_health gauge"
echo "custom_app_health{app=\"myapp\"} $(curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/health)"

# 采集当前登录用户数
echo "# HELP node_login_users Current login users"
echo "# TYPE node_login_users gauge"
echo "node_login_users $(who | wc -l)"

# 采集系统自定义信息
echo "# HELP node_custom_info Custom system info"
echo "# TYPE node_custom_info gauge"
echo 'node_custom_info{env="production",region="us-east-1"} 1'
```

## 8.3 注意事项

- 文件后缀必须为 `.prom`
- Node Exporter 使用 `filemodtime` 辅助指标跟踪文件修改时间
- 脚本执行应放在 cron 中，而非每次采集执行
- 建议采集间隔为 30s - 60s

```bash
# cron 配置：每 60 秒生成一次
* * * * * /usr/local/bin/custom_metrics.sh
```

# 9. 高基数指标警告

Node Exporter 默认采集的指标中，某些可能导致高基数：

- `node_arp_entries`：每个邻居表项一条（网络设备多时可能膨胀）
- `node_network_addresses`：每个 IP 地址一条（容器环境下多个虚拟接口时）
- `node_systemd_unit_state`：每个 systemd unit 一条

建议在生产环境评估是否需要禁用某些 Collector：

```bash
# 禁用在容器环境下可能导致高基数的 collector
./node_exporter \
  --no-collector.arp \
  --no-collector.softnet \
  --no-collector.systemd
```

# 10. Prometheus 采集配置

```yaml
scrape_configs:
  - job_name: 'node'
    scrape_interval: 30s
    scrape_timeout: 5s
    static_configs:
      - targets:
        - '10.0.0.1:9100'
        - '10.0.0.2:9100'
        - '10.0.0.3:9100'
      labels:
        env: 'production'
        region: 'us-east-1'
```
