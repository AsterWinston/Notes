
---

# 1. 什么是反向代理

```
客户端 ──→ Nginx（反向代理） ──→ 后端服务器1
                  ├───────────→ 后端服务器2
                  └───────────→ 后端服务器3
```

- **正向代理**：代理客户端，服务端不知道真实客户端（如 VPN）
- **反向代理**：代理服务端，客户端不知道真实后端（如 Nginx 在 Web 服务前）

---

# 2. proxy_pass 指令

```nginx
location /api/ {
    proxy_pass http://192.168.1.10:8080;
}
```

## 2.1 URI 拼接规则（重要）

**`proxy_pass` 末尾有无 `/` 影响行为**：

| 配置 | 请求 `/api/user/list` 转发为 | 说明 |
|------|------------------------------|------|
| `proxy_pass http://backend;` | `http://backend/api/user/list` | **不带路径**：完整 URI 拼接 |
| `proxy_pass http://backend/;` | `http://backend/user/list` | **带路径**：替换 location 前缀 |
| `proxy_pass http://backend/api;` | `http://backend/apiuser/list` | 直接拼接（不推荐） |
| `proxy_pass http://backend/api/;` | `http://backend/api/user/list` | 指定路径 + 拼接剩余部分 |
| `proxy_pass http://backend/v1/;` | `http://backend/v1/user/list` | 路径替换 |

> 规则总结：`proxy_pass` 的 URL 中**包含路径部分**时，location 的匹配前缀会被替换；**不含路径**时，完整 URI 直接拼接。

---

# 3. 请求头传递

Nginx 反向代理时默认会重置一些头信息，需手动配置透传：

```nginx
location /api/ {
    proxy_pass http://backend;

    # ── 基本头传递 ─────────────────────
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
}
```

| 头字段 | 含义 |
|--------|------|
| `Host` | 原始请求的主机名 |
| `X-Real-IP` | 客户端真实 IP（单个 IP） |
| `X-Forwarded-For` | 代理链 IP 列表（逗号分隔） |
| `X-Forwarded-Proto` | 原始请求协议（http/https） |
| `X-Forwarded-Host` | 原始 Host 头 |
| `X-Forwarded-Port` | 原始请求端口 |

---

# 4. 代理 Buffer 配置

```nginx
location /api/ {
    proxy_pass http://backend;

    # ── 代理缓冲区 ─────────────────────
    proxy_buffering on;                       # 开启代理缓冲（默认开启）
    proxy_buffer_size 4k;                     # 响应头缓冲区大小
    proxy_buffers 8 8k;                       # 响应体缓冲区（数量 × 大小）
    proxy_busy_buffers_size 16k;              # 允许忙碌缓冲区的最大值
    proxy_max_temp_file_size 1024m;           # 临时文件最大大小

    # ── 上传缓冲区 ─────────────────────
    client_body_buffer_size 128k;             # 客户端请求体缓冲区
}
```

---

# 5. 超时配置

```nginx
location /api/ {
    proxy_pass http://backend;

    proxy_connect_timeout 60s;        # 连接后端超时（默认 60s）
    proxy_send_timeout 60s;           # 发送请求到后端超时
    proxy_read_timeout 60s;           # 从后端读取响应超时
}
```

---

# 6. 常用代理场景

## 6.1 API 代理

```nginx
location /api/ {
    proxy_pass http://backend_api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 去除 /api 前缀再转发
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass http://backend_api;
}
```

## 6.2 WebSocket 代理

```nginx
location /ws/ {
    proxy_pass http://backend_ws;
    proxy_http_version 1.1;           # WebSocket 需要 HTTP/1.1
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;

    proxy_read_timeout 3600s;         # WebSocket 长连接，超时设长
}
```

## 6.3 直接返回状态码

```nginx
# 屏蔽特定路径
location /admin/ {
    return 403;
}

# 通用 return 语法
location /old-path {
    return 301 https://example.com/new-path$request_uri;
}
```
