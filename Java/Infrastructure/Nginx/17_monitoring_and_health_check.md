
---

# 1. stub_status（基础状态监控）

```nginx
location /nginx_status {
    stub_status;
    allow 127.0.0.1;         # 仅允许本机访问
    deny all;
}
```

访问 `curl http://localhost/nginx_status` 返回：

```
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

| 字段 | 含义 |
|------|------|
| `Active connections` | 当前活跃连接数 |
| `accepts` | 累计接受的连接数 |
| `handled` | 累计处理的连接数 |
| `requests` | 累计处理的请求数 |
| `Reading` | 正在读取请求头的连接数 |
| `Writing` | 正在发送响应的连接数 |
| `Waiting` | 保持连接（keep-alive）等待新请求的连接数 |

---

# 2. 自定义监控指标

```nginx
server {
    location /status {
        access_log off;
        default_type application/json;
        return 200 '{
            "connections_active": $connections_active,
            "connections_reading": $connections_reading,
            "connections_writing": $connections_writing,
            "connections_waiting": $connections_waiting
        }';
    }
}
```

---

# 3. Prometheus + nginx-exporter

```bash
# 部署 nginx-prometheus-exporter
docker run -d --name nginx-exporter \
    -p 9113:9113 \
    nginx/nginx-prometheus-exporter:latest \
    -nginx.scrape-uri http://nginx/nginx_status
```

**Grafana Dashboard**：导入官方模板（ID: 11199 / 12708）

---

# 4. 被动健康检查

```nginx
upstream backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=60s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=60s;
}

location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
    proxy_next_upstream_timeout 10s;
}
```

---

# 5. 主动健康检查

## 5.1 Nginx Plus（商业版）

```nginx
upstream backend {
    zone backend_zone 64k;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;

    health_check interval=10s fails=3 passes=2;
    health_check uri=/health;
}
```

## 5.2 Tengine / ngx_http_upstream_check_module

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;

    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

## 5.3 OpenResty 方案

```lua
-- lua 脚本定期探测后端，动态更新 upstream
local check = require "resty.upstream.healthcheck"
-- ...
```

---

# 6. access_log 监控分析

```bash
# QPS（每秒请求数）
tail -f access.log | pv -l -i 1 -r > /dev/null

# 按分钟统计请求量
awk -F: '{print $2":"$3}' access.log | awk -F' ' '{print $1}' | uniq -c

# 慢请求分析（request_time > 1s）
awk '$NF > 1 {print $0}' access.log
```

---

# 7. 错误日志监控

```bash
# 监控错误率
tail -f error.log | grep -E "error|crit|alert" | wc -l

# 配合 filebeat 采集到 ELK
```

---

# 8. Nginx Amplify（官方监控 SaaS）

Nginx Amplify 提供图形化监控，包括：
- 请求数 / 响应时间 / 状态码分布
- CPU / 内存 / 磁盘 / 网络
- Nginx 配置分析报告
- 异常告警

```bash
curl -sSL https://nginx.org/amplify | sh
```

---

# 9. 监控指标速查

| 关键指标 | 获取方式 | 告警阈值参考 |
|----------|----------|-------------|
| 502/504 错误率 | access_log 统计 | > 1% |
| 活跃连接数 | stub_status | > worker_connections × 80% |
| 上游响应时间 | `$upstream_response_time` | P99 > 1s |
| 请求处理时间 | `$request_time` | P99 > 2s |
| Nginx 进程 | `ps -ef \| grep nginx` | 进程不存在 |
| 磁盘使用率 | `df -h` | > 80%（日志/缓存） |
