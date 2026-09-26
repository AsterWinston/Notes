
# 1. 什么是 Grafana

Grafana 是一个开源的**数据可视化和监控分析平台**，支持 Prometheus、InfluxDB、Elasticsearch、Loki 等数十种数据源，是 Prometheus 生态中最常用的可视化工具。

# 2. 安装 Grafana

## 2.1 二进制安装

```bash
# Ubuntu/Debian
sudo apt-get install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt-get update
sudo apt-get install grafana
sudo systemctl start grafana-server
```

## 2.2 Docker

```bash
docker run -d \
  --name grafana \
  -p 3000:3000 \
  -v grafana_data:/var/lib/grafana \
  grafana/grafana:latest
```

# 3. 配置 Prometheus 数据源

## 3.1 通过 UI 配置

1. 登录 Grafana（默认 `admin/admin`）
2. 进入 **Configuration > Data Sources > Add data source**
3. 选择 **Prometheus**
4. 填写 URL：`http://prometheus:9090`
5. 点击 **Save & Test**

## 3.2 通过配置文件

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    version: 1
    editable: true
    jsonData:
      httpMethod: POST
      prometheusType: Prometheus
      prometheusVersion: 2.53.0
      timeInterval: 15s
```

# 4. Dashboard 设计

## 4.1 创建 Dashboard

```json
{
  "title": "Node Exporter Dashboard",
  "panels": [
    {
      "title": "CPU Usage",
      "type": "timeseries",
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "100 - (avg(rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) by (instance) * 100)",
          "legendFormat": "{{ instance }}"
        }
      ]
    }
  ]
}
```

## 4.2 使用模板变量

模板变量让 Dashboard 具有交互性，用户可动态切换过滤条件。

### 4.2.1 定义变量

| 变量名 | 类型 | 查询 |
|--------|------|------|
| `$instance` | Query | `label_values(up{job="node"}, instance)` |
| `$job` | Query | `label_values(up, job)` |
| `$cpu` | Query | `label_values(node_cpu_seconds_total, cpu)` |
| `$mountpoint` | Query | `label_values(node_filesystem_size_bytes, mountpoint)` |
| `$datasource` | Datasource | Prometheus |

### 4.2.2 级联变量

```yaml
# 变量1: job
name: job
query: label_values(up, job)

# 变量2: instance（依赖 job）
name: instance
query: label_values(up{job="$job"}, instance)
```

### 4.2.3 在查询中使用变量

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle", instance="$instance"}[5m])) by (cpu) * 100)
```

## 4.3 Dashboard 设计原则

1. **保持简洁**：每个 Dashboard 聚焦一个关注领域（如 Node、Kubernetes、应用）
2. **使用模板变量**：使 Dashboard 可复用
3. **合理的时间范围**：使用适合的聚合窗口
4. **添加描述与注释**：方便团队理解
5. **使用行（Row）分隔**：按逻辑分组 Panel

# 5. 常用 Dashboard 推荐

| Dashboard | 说明 | 来源 |
|-----------|------|------|
| Node Exporter Full | 主机监控 | 1860 (grafana.com) |
| Prometheus 2.0 Overview | Prometheus 自身监控 | 3662 |
| Kubernetes Cluster | K8s 集群监控 | 315 |
| Spring Boot Statistics | Spring Boot 应用监控 | 6756 |
| Redis Dashboard | Redis 监控 | 11835 |

```bash
# 导入 Dashboard（通过 API）
curl -X POST \
  -H "Authorization: Bearer <api_key>" \
  -H "Content-Type: application/json" \
  -d '{"dashboard": {"id": null, "title": "Node Exporter", ...}, "overwrite": true}' \
  http://grafana:3000/api/dashboards/db
```

# 6. Grafana Alerting

Grafana 自带告警引擎，可作为 Alertmanager 的补充或替代。

```yaml
# grafana.ini
[unified_alerting]
enabled = true
```

## 6.1 创建告警规则

在 Panel 中创建告警：

```promql
# 条件：CPU 使用率 > 80% 持续 5 分钟
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 80
```

## 6.2 告警渠道

```yaml
# /etc/grafana/provisioning/alerting/contact_points.yml
apiVersion: 1
contactPoints:
  - name: Slack
    receivers:
      - uid: slack
        type: slack
        settings:
          url: https://hooks.slack.com/services/...
          title: "Grafana Alert"
```

# 7. 预配（Provisioning）

Grafana 支持通过配置文件自动创建数据源、Dashboard 和告警渠道。

## 7.1 自动导入 Dashboard

```yaml
# /etc/grafana/provisioning/dashboards/prometheus.yml
apiVersion: 1
providers:
  - name: 'Prometheus'
    orgId: 1
    folder: 'Prometheus'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /var/lib/grafana/dashboards
```

# 8. 嵌入与分享

## 8.1 嵌入 Dashboard Panel

```html
<iframe src="http://grafana:3000/d-solo/abc123/panel?orgId=1&from=now-1h&to=now&panelId=1"
  width="800" height="400" frameborder="0"></iframe>
```

## 8.2 分享 Snapshot

Grafana 支持创建 Dashboard Snapshot，生成可分享的静态链接（实时数据的快照）。

# 9. 安全配置

## 9.1 反向代理

```nginx
server {
    listen 443 ssl;
    server_name grafana.example.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 9.2 认证集成

```ini
[auth.ldap]
enabled = true
config_file = /etc/grafana/ldap.toml

[auth.oauth]
enabled = true
```

# 10. 性能优化

```ini
# grafana.ini
[server]
# 限制最大并发
http_max_conns_per_route = 100

[metrics]
# 开启内部监控
enabled = true

[panels]
# 禁用不必要的渲染
disable_sanitize_html = false
```
