
# 1. 监控体系架构

```
        Nacos 监控运维体系

  ┌────────────────────────────────────────────┐
  │           数据采集层                        │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │ JMX      │  │ Prometheus│  │ 日志聚合  │ │
  │  │ Metrics  │  │ Endpoint │  │ ELK/SLS  │ │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
  ├───────┼─────────────┼─────────────┼────────┤
  │       └─────────────┼─────────────┘        │
  │                     ▼                       │
  │          ┌─────────────────┐               │
  │          │   存储与可视化   │               │
  │          │  Prometheus     │               │
  │          │  + Grafana      │               │
  │          └─────────────────┘               │
  ├────────────────────────────────────────────┤
  │           告警层                            │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
  │  │ AlertManager│ 钉钉/企微  │ 邮件/短信  │ │
  │  └──────────┘  └──────────┘  └──────────┘ │
  └────────────────────────────────────────────┘
```

# 2. Nacos 内置监控

## 2.1 内置监控面板

Nacos 控制台提供了基础监控：

```
控制台 → 监控中心 → 服务监控
控制台 → 监控中心 → 配置监控
```

## 2.2 Nacos 暴露的 Metrics 端点

Nacos 基于 Spring Boot Actuator 暴露了丰富的内部指标：

```bash
# Prometheus 格式指标
curl http://127.0.0.1:8848/nacos/actuator/prometheus
```

# 3. Prometheus + Grafana 集成

## 3.1 Nacos 服务端 Prometheus 配置

```properties
# conf/application.properties
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.metrics.export.prometheus.enabled=true
management.metrics.export.prometheus.rsocket.enabled=false
management.metrics.export.prometheus.pushgateway.enabled=false
```

## 3.2 Prometheus 配置

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'nacos'
    metrics_path: '/nacos/actuator/prometheus'
    static_configs:
      - targets:
        - 'nacos-1:8848'
        - 'nacos-2:8848'
        - 'nacos-3:8848'
        labels:
          cluster: 'nacos-cluster'
          env: 'production'
```

## 3.3 关键 Prometheus 指标

**系统指标**：

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `jvm_memory_used_bytes` | Gauge | JVM 内存使用量 |
| `jvm_memory_max_bytes` | Gauge | JVM 最大内存 |
| `jvm_gc_pause_seconds` | Summary | GC 暂停时间 |
| `process_cpu_usage` | Gauge | 进程 CPU 使用率 |
| `system_cpu_usage` | Gauge | 系统 CPU 使用率 |
| `jvm_threads_live_threads` | Gauge | 活跃线程数 |

**HTTP 请求指标**：

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `http_server_requests_seconds_count` | Counter | HTTP 请求总数 |
| `http_server_requests_seconds_sum` | Summary | HTTP 请求耗时 |
| `http_server_requests_seconds_max` | Gauge | HTTP 最大响应时间 |

**Nacos 业务指标**：

| 指标名 | 类型 | 说明 |
|--------|------|------|
| `nacos_naming_subscriber_count` | Gauge | 服务订阅者数量 |
| `nacos_naming_publisher_count` | Gauge | 服务发布数量 |
| `nacos_naming_dom_count` | Gauge | 域（服务）数量 |
| `nacos_naming_ip_count` | Gauge | IP 数量 |
| `nacos_config_configs_size` | Gauge | 配置数量 |
| `nacos_config_notify_tasks` | Gauge | 配置通知任务数 |
| `nacos_consistent_protocol_latency` | Timer | 一致性协议延迟 |

## 3.4 Grafana 仪表盘配置

```json
// 推荐 Grafana Dashboard 模板
// Nacos 官方 Dashboard ID: 13221 (Nacos monitoring)
// 导入地址: https://grafana.com/grafana/dashboards/13221

// 自定义 Dashboard 面板示例
{
  "panels": [
    {
      "title": "Nacos HTTP Request Rate",
      "targets": [
        {
          "expr": "rate(http_server_requests_seconds_count{application=\"nacos\"}[1m])"
        }
      ]
    },
    {
      "title": "Nacos GC Pause Time",
      "targets": [
        {
          "expr": "rate(jvm_gc_pause_seconds_sum{application=\"nacos\"}[5m])"
        }
      ]
    },
    {
      "title": "Nacos Service Count",
      "targets": [
        {
          "expr": "nacos_naming_dom_count"
        }
      ]
    },
    {
      "title": "Nacos Instance Count",
      "targets": [
        {
          "expr": "nacos_naming_ip_count"
        }
      ]
    },
    {
      "title": "Config Notify Tasks",
      "targets": [
        {
          "expr": "nacos_config_notify_tasks"
        }
      ]
    }
  ]
}
```

# 4. 日志监控

## 4.1 日志配置

```properties
# conf/application.properties

# 日志级别
nacos.naming.log.level=INFO
nacos.config.log.level=INFO

# 日志保留天数
nacos.naming.log.retain.days=30
nacos.config.log.retain.days=30
```

## 4.2 关键日志文件

| 日志文件 | 内容 | 关注点 |
|---------|------|--------|
| `nacos.log` | 应用主日志 | ERROR/WARN 日志、异常堆栈 |
| `nacos-cluster.log` | 集群日志 | 节点上下线、Leader 选举 |
| `nacos-naming.log` | 服务发现日志 | 服务注册、心跳、推送 |
| `nacos-config.log` | 配置管理日志 | 配置变更、长轮询 |
| `nacos-remote.log` | RPC 通信日志 | gRPC 连接、超时 |
| `gc.log` | JVM GC 日志 | GC 频率、暂停时间 |

## 4.3 日志分析关键模式

```bash
# 统计 ERROR 日志
grep "ERROR" logs/nacos.log | wc -l

# 查看最近的服务注册异常
grep "register" logs/nacos-naming.log | grep "ERROR"

# 查看集群节点变化
grep "member change" logs/nacos-cluster.log

# 查看配置变更记录
grep "LocalDataChangeEvent" logs/nacos-config.log

# 监控 Raft 选举
grep "leader elected\|vote for" logs/nacos.log
```

# 5. MySQL 监控

## 5.1 数据库性能监控

```sql
-- Nacos 数据库连接数
SELECT count(*) FROM information_schema.processlist WHERE db='nacos';

-- 配置表数据量
SELECT count(*) FROM config_info;

-- 历史配置表大小
SELECT count(*) FROM his_config_info;

-- 检查慢查询
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;

-- 数据库表碎片率
SELECT TABLE_NAME, 
       ROUND((DATA_LENGTH + INDEX_LENGTH) / 1024 / 1024, 2) AS SIZE_MB,
       ROUND(DATA_FREE / 1024 / 1024, 2) AS FREE_MB
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'nacos';
```

# 6. 告警规则

## 6.1 Prometheus 告警规则

```yaml
# nacos-alerts.yml
groups:
  - name: nacos_alerts
    rules:
      # Nacos 节点宕机
      - alert: NacosNodeDown
        expr: up{job="nacos"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nacos node {{ $labels.instance }} is down"
          description: "Nacos node has been down for more than 1 minute."

      # Nacos 集群节点数不足
      - alert: NacosClusterSizeInsufficient
        expr: count(up{job="nacos"}) < 3
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Nacos cluster has fewer than 3 nodes"
          description: "Current node count: {{ $value }}"

      # 内存使用率过高
      - alert: NacosMemoryHigh
        expr: jvm_memory_used_bytes{area="heap", application="nacos"} / jvm_memory_max_bytes{area="heap", application="nacos"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Nacos memory usage > 85%"
          description: "Heap memory usage: {{ $value | humanizePercentage }}"

      # GC 频率过高
      - alert: NacosHighGCFrequency
        expr: rate(jvm_gc_pause_seconds_count{application="nacos"}[10m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Nacos GC frequency too high"

      # HTTP 请求延迟过高
      - alert: NacosHighRequestLatency
        expr: histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{application="nacos"}[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Nacos P99 request latency > 1s"
          description: "P99 latency: {{ $value }}s"

      # 数据库连接失败
      - alert: NacosDatabaseConnectionFailure
        expr: nacos_db_health != 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Nacos cannot connect to database"
```

## 6.2 钉钉告警通知

```yaml
# alertmanager.yml
route:
  receiver: 'dingtalk-nacos'
  routes:
    - match:
        severity: critical
      receiver: 'dingtalk-nacos'

receivers:
  - name: 'dingtalk-nacos'
    webhook_configs:
      - url: 'http://dingtalk-webhook:8060/dingtalk/webhook1/send'
        send_resolved: true
```

# 7. 运维操作

## 7.1 日常巡检脚本

```bash
#!/bin/bash
# nacos-health-check.sh

NACOS_URL="http://127.0.0.1:8848"
LOG_FILE="/var/log/nacos-health.log"

echo "===== Nacos Health Check $(date) =====" >> $LOG_FILE

# 1. 健康检查
echo "1. Liveness Check:" >> $LOG_FILE
curl -s -o /dev/null -w "%{http_code}" ${NACOS_URL}/nacos/v1/console/health/liveness >> $LOG_FILE
echo "" >> $LOG_FILE

# 2. 集群状态
echo "2. Cluster Nodes:" >> $LOG_FILE
curl -s ${NACOS_URL}/nacos/v1/core/cluster/nodes | jq '.data[].state' >> $LOG_FILE

# 3. 服务数量
echo "3. Service Count:" >> $LOG_FILE
curl -s "${NACOS_URL}/nacos/v1/ns/service/list?pageNo=1&pageSize=1" | jq '.count' >> $LOG_FILE

# 4. 配置数量
echo "4. Config Count:" >> $LOG_FILE
# 需要 accessToken
echo "See Nacos Console" >> $LOG_FILE

# 5. 数据库连接
echo "5. DB Connection:" >> $LOG_FILE
# 自定义检查
echo "Done" >> $LOG_FILE
```

## 7.2 数据备份

```bash
#!/bin/bash
# nacos-backup.sh

BACKUP_DIR="/data/backup/nacos"
MYSQL_USER="root"
MYSQL_PASS="root"
MYSQL_DB="nacos"

DATE=$(date +%Y%m%d_%H%M%S)

# 1. 备份 MySQL 数据库
mysqldump -u${MYSQL_USER} -p${MYSQL_PASS} ${MYSQL_DB} \
  > ${BACKUP_DIR}/nacos_db_${DATE}.sql

# 2. 备份配置文件
cp -r /opt/nacos/conf ${BACKUP_DIR}/conf_${DATE}/

# 3. 清理 7 天前的备份
find ${BACKUP_DIR} -type f -mtime +7 -delete

echo "Backup completed: ${DATE}"
```

## 7.3 扩容操作

```bash
# Nacos 集群扩容步骤

# 1. 修改所有节点（包括新节点）的 cluster.conf
echo "10.0.0.1:8848
10.0.0.2:8848
10.0.0.3:8848
10.0.0.4:8848" > /opt/nacos/conf/cluster.conf

# 2. 启动新节点
sh /opt/nacos/bin/startup.sh

# 3. 等待新节点加入集群（约 30s）
sleep 30

# 4. 验证集群状态
curl http://10.0.0.4:8848/nacos/v1/core/cluster/nodes

# 5. 更新负载均衡器配置
# - Nginx upstream 添加新节点
# - 重载 Nginx: nginx -s reload
```

# 8. 性能调优

## 8.1 JVM 调优

```bash
# 推荐 JVM 参数（8C16G）
-Xms8g
-Xmx8g
-Xmn4g
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:ParallelGCThreads=8
-XX:ConcGCThreads=4
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/nacos/logs/
```

## 8.2 系统参数调优

```bash
# 文件描述符限制
echo "* soft nofile 655350" >> /etc/security/limits.conf
echo "* hard nofile 655350" >> /etc/security/limits.conf

# 内核参数
cat >> /etc/sysctl.conf << EOF
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
EOF
sysctl -p
```

## 8.3 MySQL 调优

```ini
[mysqld]
# 连接配置
max_connections = 500
max_connect_errors = 1000

# InnoDB 配置
innodb_buffer_pool_size = 4G
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2

# 查询缓存（MySQL 8.0+ 已废弃）
# query_cache_type = 0
```
