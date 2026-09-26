
---

# 1. 限流维度

| 限流类型 | 指令 | 说明 |
|----------|------|------|
| **请求频率** | `limit_req` | 限制每秒请求数（漏桶算法） |
| **并发连接数** | `limit_conn` | 限制同一 IP 的同时连接数 |
| **下载速度** | `limit_rate` | 限制单个连接的传输速率 |

---

# 2. limit_req（请求频率限制）

## 2.1 基本配置

```nginx
http {
    # 定义共享内存区域：按 IP 存储请求速率
    limit_req_zone $binary_remote_addr zone=req_zone:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=req_zone burst=20 nodelay;
            proxy_pass http://backend;
        }
    }
}
```

| 参数 | 说明 |
|------|------|
| `limit_req_zone` | 定义限流规则与共享内存 |
| `zone=req_zone:10m` | 共享内存名称和大小（10MB ≈ 16万 IP 地址） |
| `rate=10r/s` | 速率限制（每秒 10 个请求） |
| `burst=20` | 突发队列大小（超过 rate 但未超 burst 的排队等待） |
| `nodelay` | 不延迟处理 burst 队列中的请求（立即处理或拒绝） |

## 2.2 算法原理（漏桶算法）

```
请求 → ┌──────────────────┐
       │  ▼  ▼  ▼  ▼  ▼   │  rate = 10r/s（固定漏出速率）
       │  ▼  ▼  ▼  ▼  ▼   │  超过 rate 的进入 burst 等待
       │  ▼  ▼  ▼  ▼  ▼   │  超过 burst 的 → 503
       └──────────────────┘
```

## 2.3 按 URL 限流

```nginx
# 对登录接口严格限制
limit_req_zone $binary_remote_addr zone=login_rate:10m rate=1r/s;

location /api/login {
    limit_req zone=login_rate burst=3 nodelay;
    proxy_pass http://backend;
}
```

## 2.4 按域名 + IP 限流

```nginx
limit_req_zone $server_name$binary_remote_addr zone=per_domain:10m rate=20r/s;
```

## 2.5 返回自定义错误信息

```nginx
limit_req_status 429;    # 触发限流时返回 HTTP 429 Too Many Requests

error_page 429 /429.html;
location = /429.html {
    internal;
    default_type application/json;
    return 429 '{"error":"Too Many Requests","retry_after":1}';
}
```

---

# 3. limit_conn（并发连接数限制）

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn_zone:10m;

    server {
        location /download/ {
            limit_conn conn_zone 5;       # 同一 IP 最多 5 个并发连接
            limit_conn_status 503;
        }
    }
}
```

| 参数 | 说明 |
|------|------|
| `limit_conn_zone` | 定义连接数限制的共享内存 |
| `limit_conn` | 最大并发连接数 |
| `limit_conn_status` | 触发限制时的状态码（默认 503） |

```nginx
# 结合 server_name 区分
limit_conn_zone $server_name$binary_remote_addr zone=per_site:10m;
limit_conn per_site 20;
```

---

# 4. limit_rate（带宽限制）

```nginx
# 全局限制
location /download/ {
    limit_rate 500k;                  # 单个连接限速 500KB/s
    limit_rate_after 5m;              # 前 5MB 不限速，之后限速
}

# 按连接数动态分配
set $limit_rate 200k;                 # 可结合 map 动态设定
```

---

# 5. 完整限流策略示例

```nginx
http {
    # ── 定义限流规则 ──────────────
    limit_req_zone $binary_remote_addr zone=global:10m rate=30r/s;
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=20r/s;
    limit_conn_zone $binary_remote_addr zone=conn:10m;
    limit_conn_zone $server_name$binary_remote_addr zone=per_site_conn:10m;

    limit_req_status 429;
    limit_conn_status 503;

    server {
        listen 80;
        server_name www.example.com;

        # 全局频率限制
        limit_req zone=global burst=50 nodelay;
        limit_conn per_site_conn 20;
        limit_conn conn 10;

        # 登录接口——严格限制
        location /api/login {
            limit_req zone=login burst=3 nodelay;
            proxy_pass http://backend;
        }

        # API 接口——中等限制
        location /api/ {
            limit_req zone=api burst=30 nodelay;
            proxy_pass http://backend;
        }

        # 下载——带宽限制
        location /download/ {
            limit_rate 1m;
            limit_rate_after 10m;
            alias /data/downloads/;
        }

        # 自定义限流错误返回
        error_page 429 /429.json;
        location = /429.json {
            internal;
            default_type application/json;
            return 429 '{"code":429,"message":"请求过于频繁，请稍后再试"}';
        }
    }
}
```

---

# 6. 限流效果验证

```bash
# Apache Bench 压测
ab -n 1000 -c 100 http://example.com/api/test

# wrk 压测
wrk -t4 -c100 -d30s http://example.com/api/test

# 观察结果：超出限制的请求应返回 429 或 503
```
