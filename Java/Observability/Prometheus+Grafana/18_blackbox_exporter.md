
# 1. 概述

Blackbox Exporter 是 Prometheus 官方提供的**网络探测 Exporter**，通过执行 HTTP/HTTPS/TCP/ICMP/DNS 等协议的探测任务，检查目标服务的可用性和响应性能。常用于**外部监控**和**黑盒测试**场景。

## 1.1 典型应用场景

- 网站可用性探测（HTTP/HTTPS）
- SSL 证书过期监控
- API 端点健康检查
- TCP 端口连通性检测
- ICMP Ping 延迟监控
- DNS 解析质量监控
- 数据库端口可达性检测

# 2. 安装与部署

```bash
# 二进制
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz
tar xvf blackbox_exporter-0.25.0.linux-amd64.tar.gz
cd blackbox_exporter-0.25.0.linux-amd64
./blackbox_exporter --config.file=blackbox.yml
```

```bash
# Docker
docker run -d \
  --name blackbox \
  -p 9115:9115 \
  -v /path/to/blackbox.yml:/etc/blackbox_exporter/config.yml \
  prom/blackbox-exporter:v0.25.0
```

# 3. 配置文件

## 3.1 模块配置（blackbox.yml）

```yaml
modules:
  # HTTP 探测
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200, 201, 204]
      method: GET
      follow_redirects: true
      preferred_ip_protocol: ip4
      tls_config:
        insecure_skip_verify: false
      headers:
        User-Agent: "Prometheus/Blackbox-Exporter"

  # HTTP Post 探测
  http_post_2xx:
    prober: http
    http:
      method: POST
      headers:
        Content-Type: application/json
      body: '{"key": "value"}'
      preferred_ip_protocol: ip4

  # HTTPS + SSL 证书检查
  http_ssl:
    prober: http
    http:
      valid_status_codes: []
      method: GET
      fail_if_ssl: true
      tls_config:
        insecure_skip_verify: false
      preferred_ip_protocol: ip4

  # TCP 端口探测
  tcp_connect:
    prober: tcp
    timeout: 5s

  # ICMP Ping
  icmp:
    prober: icmp
    timeout: 30s
    icmp:
      preferred_ip_protocol: ip4

  # DNS 探测
  dns_query:
    prober: dns
    dns:
      query_name: "example.com"
      query_type: "A"
      valid_rcodes:
        - NOERROR
      preferred_ip_protocol: ip4

  # 带认证的 HTTP
  http_basic_auth:
    prober: http
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      valid_status_codes: []
      method: GET
      basic_auth:
        username: "admin"
        password: "password"

  # 自定义 HTTP 头
  http_custom_headers:
    prober: http
    http:
      method: GET
      headers:
        X-API-Key: "my-api-key"
        Accept: "application/json"
```

# 4. Prometheus 采集配置

通过 `params` 传递探测目标和模块：

```yaml
scrape_configs:
  # HTTP 探测
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://example.com
        - https://api.example.com/health
        - http://internal-service:8080
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
      - source_labels: [__param_module]
        target_label: module

  # TCP 端口探测
  - job_name: 'blackbox-tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
        - 'mysql-server:3306'
        - 'redis-server:6379'
        - 'kafka-server:9092'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter:9115

  # SSL 证书检查
  - job_name: 'blackbox-ssl'
    metrics_path: /probe
    params:
      module: [http_ssl]
    static_configs:
      - targets:
        - https://example.com
        - https://api.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter:9115
      - source_labels: [__param_target]
        target_label: instance
```

# 5. 核心指标

## 5.1 探测状态

```promql
# 探测是否成功（1=成功，0=失败）
probe_success

# 探测持续时间
probe_duration_seconds

# 重定向次数
probe_http_redirects
```

## 5.2 HTTP 专属指标

```promql
# HTTP 状态码
probe_http_status_code

# HTTP 版本
probe_http_version

# 内容长度
probe_http_content_length

# 是否使用 SSL
probe_http_ssl

# SSL 证书过期时间（Unix 时间戳）
probe_ssl_earliest_cert_expiry

# SSL 证书信息
probe_ssl_last_chain_expiry_timestamp_seconds
```

## 5.3 TCP 专属指标

```promql
# TCP 连接建立时间
probe_tcp_duration_seconds{phase="connect"}
probe_tcp_duration_seconds{phase="setup"}
```

## 5.4 ICMP 专属指标

```promql
# Ping 延迟
probe_icmp_duration_seconds{phase="rtt"}

# 丢包率
probe_icmp_duration_seconds{phase="loss"}
```

## 5.5 DNS 专属指标

```promql
# DNS 查询延迟
probe_dns_duration_seconds

# DNS 查询结果数
probe_dns_additional_rr
probe_dns_answer_rr
probe_dns_authority_rr
```

# 6. SSL 证书监控

## 6.1 证书过期告警

```yaml
groups:
  - name: ssl_alerts
    rules:
      - alert: SSLCertExpiringSoon
        expr: probe_ssl_earliest_cert_expiry - time() < 86400 * 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL certificate for {{ $labels.instance }} expires in {{ $value | humanizeDuration }}"

      - alert: SSLCertExpired
        expr: probe_ssl_earliest_cert_expiry - time() < 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "SSL certificate for {{ $labels.instance }} has expired"
```

## 6.2 SSL 证书信息查询

```promql
# 证书剩余天数
(probe_ssl_earliest_cert_expiry - time()) / 86400

# 证书颁发者
probe_ssl_last_chain_issuer

# 证书 SAN 列表
probe_ssl_last_chain_san
```

# 7. 常见告警规则

```yaml
groups:
  - name: blackbox_alerts
    rules:
      - alert: EndpointDown
        expr: probe_success == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint {{ $labels.instance }} is down"
          description: "Module: {{ $labels.module }}"

      - alert: HighLatency
        expr: probe_duration_seconds > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency for {{ $labels.instance }}"
          description: "Duration: {{ $value }}s"

      - alert: DNSResolutionFailed
        expr: probe_success{module="dns_query"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "DNS resolution failed for {{ $labels.instance }}"
```

# 8. 高级用法

## 8.1 多阶段探测

```yaml
modules:
  http_oauth2:
    prober: http
    http:
      method: GET
      valid_status_codes: [200]
      oauth2:
        client_id: "client"
        client_secret: "secret"
        token_url: "https://auth.example.com/token"
      preferred_ip_protocol: ip4
```

## 8.2 自定义有效状态码

```yaml
modules:
  http_custom:
    prober: http
    http:
      valid_status_codes:
        - 200
        - 201
        - 204
        - 301
        - 302
      fail_if_body_not_matches_regexp:
        - ".*healthy.*"
      fail_if_header_matches:
        - header: "X-Cache"
          regexp: "MISS"
          allow_missing: true
```

## 8.3 Kubernetes ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: blackbox-exporter
  namespace: monitoring
spec:
  jobLabel: job
  selector:
    matchLabels:
      app.kubernetes.io/name: blackbox-exporter
  endpoints:
    - port: http
      path: /probe
      params:
        module: [http_2xx]
        target: ["https://example.com"]
      interval: 30s
      relabelings:
        - sourceLabels: [__param_target]
          targetLabel: instance
```
