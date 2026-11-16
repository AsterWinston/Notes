
---

# 1. upstream 基础配置

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

server {
    location /api/ {
        proxy_pass http://backend;
    }
}
```

---

# 2. 调度算法

| 算法 | 指令 | 说明 |
|------|------|------|
| **轮询（默认）** | — | 按顺序轮流分配 |
| **加权轮询** | `weight` | 按权重比例分配 |
| **IP Hash** | `ip_hash` | 同一 IP 固定到同一后端（会话保持） |
| **最小连接数** | `least_conn` | 连接数最少的优先 |
| **URL Hash** | `hash` | 按 URL 哈希分配（适合缓存场景） |
| **随机** | `random` | 随机选择后端 |

## 2.1 加权轮询

```nginx
upstream backend {
    server 192.168.1.10:8080 weight=3;    # 权重3，承担更多请求
    server 192.168.1.11:8080 weight=1;
    server 192.168.1.12:8080 weight=2;
}
```

## 2.2 IP Hash（会话保持）

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

> `ip_hash` 缺点：IP 变化会导致会话丢失；后端增删可能改变哈希分配。生产环境更推荐 **Redis/Token 实现会话共享**。

## 2.3 最小连接数

```nginx
upstream backend {
    least_conn;
    server 192.168.1.10:8080 weight=1;
    server 192.168.1.11:8080 weight=2;
}
```

## 2.4 URL Hash

```nginx
upstream backend {
    hash $request_uri consistent;   # consistent = 一致性哈希，减少后端变动时的影响
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

## 2.5 随机

```nginx
upstream backend {
    random two least_conn;          # 随机选两个，再选连接数少的（适用于多后端均匀分配）
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

---

# 3. 后端服务器参数

```nginx
upstream backend {
    server 192.168.1.10:8080 weight=3 max_fails=2 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=2 fail_timeout=30s down;
    server 192.168.1.12:8080 backup;       # 备用服务器（其他都不可用时才启用）
}
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `weight` | 1 | 权重（仅加权轮询和 least_conn 有效） |
| `max_fails` | 1 | 最大失败次数（超过后标记为不可用） |
| `fail_timeout` | 10s | 失败超时窗口 |
| `backup` | — | 备用服务器（主服务器全部不可用时启用） |
| `down` | — | 标记为永久不可用（维护模式） |
| `max_conns` | 0（无限制） | 最大并发连接数 |

---

# 4. 健康检查

## 4.1 被动健康检查（内置）

Nginx 默认对 upstream 进行**被动健康检查**：

```nginx
upstream backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;   # 30s 内失败3次则摘除30s
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
}
```

- 失败包括：连接超时、拒绝连接、返回错误（需配合 `proxy_next_upstream` 定义）
- 被摘除后等待 `fail_timeout` 后再尝试

## 4.2 定义"失败"标准

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    proxy_next_upstream_tries 2;        # 最多重试次数
    proxy_next_upstream_timeout 10s;    # 重试超时
}
```

## 4.3 主动健康检查（Nginx Plus / 第三方模块）

开源版 Nginx 不内置主动健康检查，可借助 **ngx_http_upstream_check_module**（Tengine 内置）或 nginx-upsync 等模块实现。

---

# 5. 长连接（Keepalive）到后端

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;

    keepalive 32;             # 保持到后端的空闲连接数
    keepalive_timeout 60s;    # 空闲连接超时
    keepalive_requests 1000;  # 单个连接最大请求数
}

location /api/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Connection "";     # 清除 Connection: close
}
```

---

# 6. Sticky Session（会话保持）

Nginx Plus 支持 `sticky` 指令，开源版可用第三方模块 `nginx-sticky-module` 实现基于 Cookie 的会话保持：

```nginx
# Nginx Plus 语法
upstream backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

---

# 7. 实践：完整负载均衡配置

```nginx
upstream app_backend {
    least_conn;
    server 192.168.1.10:8080 weight=3 max_fails=2 fail_timeout=30s;
    server 192.168.1.11:8080 weight=3 max_fails=2 fail_timeout=30s;
    server 192.168.1.12:8080 weight=2 max_fails=2 fail_timeout=30s backup;

    keepalive 64;
    keepalive_timeout 60s;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;

        proxy_next_upstream error timeout http_500 http_502 http_503;
        proxy_next_upstream_tries 2;
    }
}
```
