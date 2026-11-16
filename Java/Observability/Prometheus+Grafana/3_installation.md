
# 1. 安装方式对比

| 方式 | 适用场景 | 复杂度 |
|------|---------|--------|
| 二进制部署 | 测试环境、小规模生产 | 低 |
| Docker | 单机容器环境 | 低 |
| Docker Compose | 本地开发测试、小规模部署 | 低 |
| Helm (Kubernetes) | 生产环境 K8s 集群 | 中 |
| Operator (K8s) | 大规模生产 K8s 集群 | 高 |

# 2. 二进制安装

## 2.1 Linux

```bash
# 下载最新版本
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz

# 解压
tar xvf prometheus-2.53.0.linux-amd64.tar.gz
cd prometheus-2.53.0.linux-amd64

# 创建数据目录
mkdir -p /data/prometheus

# 启动
./prometheus --config.file=prometheus.yml --storage.tsdb.path=/data/prometheus
```

## 2.2 Windows

```powershell
# 下载并解压后运行
.\prometheus.exe --config.file=prometheus.yml --storage.tsdb.path=data
```

## 2.3 macOS

```bash
brew install prometheus
brew services start prometheus
```

# 3. Docker 部署

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v /path/to/data:/prometheus \
  prom/prometheus:v2.53.0
```

## 3.1 Docker Compose

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:v2.53.0
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'

volumes:
  prometheus_data:
```

# 4. 配置文件详解

## 4.1 全局配置

```yaml
global:
  # 默认采集频率
  scrape_interval: 15s
  # 默认告警规则评估频率
  evaluation_interval: 15s
  # 采集超时时间
  scrape_timeout: 10s
  # 与外部系统通信时添加的标签
  external_labels:
    cluster: 'prod'
    region: 'us-east-1'
```

## 4.2 采集配置

```yaml
scrape_configs:
  # 采集 Prometheus 自身指标
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 采集 Node Exporter
  - job_name: 'node'
    scrape_interval: 30s
    scrape_timeout: 5s
    static_configs:
      - targets:
        - '192.168.1.10:9100'
        - '192.168.1.11:9100'
      labels:
        env: 'production'

  # 通过 Consul 服务发现采集
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'localhost:8500'
        services: []
    relabel_configs:
      - source_labels: [__meta_consul_service]
        regex: '.*-exporter'
        action: keep
```

## 4.3 规则配置

```yaml
rule_files:
  - "rules/*.yml"
  - "alerts/*.yml"
```

## 4.4 告警配置

```yaml
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'localhost:9093'
```

## 4.5 远程存储配置

```yaml
remote_write:
  - url: "http://localhost:9201/write"
    queue_config:
      max_samples_per_send: 1000
      capacity: 2500

remote_read:
  - url: "http://localhost:9201/read"
    read_recent: true
```

# 5. 启动参数详解

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--config.file` | `prometheus.yml` | 配置文件路径 |
| `--storage.tsdb.path` | `data/` | TSDB 数据存储路径 |
| `--storage.tsdb.retention.time` | `15d` | 数据保留时间 |
| `--storage.tsdb.retention.size` | `0` (不限) | 数据保留大小 |
| `--web.listen-address` | `0.0.0.0:9090` | Web 监听地址 |
| `--web.enable-lifecycle` | `false` | 启用热加载配置（HTTP POST /-/reload） |
| `--web.enable-admin-api` | `false` | 启用管理 API（删除指标、快照等） |
| `--log.level` | `info` | 日志级别（debug/info/warn/error） |
| `--query.timeout` | `2m` | 查询超时时间 |
| `--query.max-concurrency` | `20` | 最大并发查询数 |
| `--storage.tsdb.min-block-duration` | `2h` | 最小 Block 时长 |
| `--storage.tsdb.max-block-duration` | `36h` | 最大 Block 时长 |

# 6. 验证与测试

## 6.1 检查服务状态

```bash
# 查看配置是否加载成功
curl http://localhost:9090/api/v1/status/config

# 查看所有已配置的采集目标
curl http://localhost:9090/api/v1/targets

# 查看通过 /-/reload 热加载配置（需开启 lifecycle）
curl -X POST http://localhost:9090/-/reload
```

## 6.2 访问 Web UI

打开浏览器访问 `http://localhost:9090`：

- **Graph**：PromQL 查询界面
- **Alerts**：查看告警规则状态
- **Status > Targets**：查看所有采集目标是否正常
- **Status > Configuration**：查看当前加载的配置
- **Status > Flags**：查看启动参数
- **Status > TSDB Status**：查看 TSDB 运行状态

# 7. 生产环境部署建议

```bash
# 创建专用系统用户
useradd --no-create-home --shell /sbin/nologin prometheus

# 创建数据目录
mkdir -p /data/prometheus
chown -R prometheus:prometheus /data/prometheus

# systemd service
cat > /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data/prometheus \
  --storage.tsdb.retention.time=30d \
  --web.enable-lifecycle \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.console.templates=/etc/prometheus/consoles

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable prometheus
systemctl start prometheus
systemctl status prometheus
```

# 8. 安全配置

## 8.1 启用 Basic Auth

通过 Nginx 反向代理实现：

```nginx
server {
    listen 9090 ssl;
    server_name prometheus.example.com;

    ssl_certificate /etc/ssl/certs/prometheus.crt;
    ssl_certificate_key /etc/ssl/private/prometheus.key;

    location / {
        auth_basic "Prometheus";
        auth_basic_user_file /etc/nginx/.htpasswd;
        proxy_pass http://127.0.0.1:9090;
    }
}
```

## 8.2 启用 HTTPS

Prometheus 自身不支持 HTTPS，推荐使用反向代理（Nginx / Envoy / Traefik）终止 TLS。

## 8.3 网络隔离

- 防火墙只允许受信任的监控网络访问 9090 端口
- 使用 `--web.listen-address` 绑定到内网 IP 而非 `0.0.0.0`
- 使用网络策略（K8s NetworkPolicy）限制 Prometheus Server 与 Target 的双向网络
