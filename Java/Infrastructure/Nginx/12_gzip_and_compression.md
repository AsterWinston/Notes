
---

# 1. gzip 压缩配置

```nginx
http {
    gzip on;                          # 开启 gzip 压缩
    gzip_vary on;                     # 响应头添加 Vary: Accept-Encoding
    gzip_min_length 1024;             # 最小压缩字节（小于此值不压缩）
    gzip_comp_level 6;                # 压缩等级 1~9（6 是性能与压缩率的平衡点）
    gzip_types text/plain text/css text/javascript
               application/javascript application/json
               application/xml application/rss+xml
               image/svg+xml;         # 压缩的 MIME 类型
    gzip_disable "msie6";             # 禁用对 IE6 的压缩
    gzip_proxied any;                 # 作为反向代理时的压缩策略
    gzip_buffers 16 8k;               # 压缩缓冲区
    gzip_http_version 1.1;            # 启用压缩的最低 HTTP 版本
}
```

| 指令 | 说明 |
|------|------|
| `gzip` | 开关 |
| `gzip_min_length` | 只压缩大于此值的响应（防止小文件压缩后更大） |
| `gzip_comp_level` | 压缩级别 1~9（1 最快压缩率最低，9 最慢压缩率最高） |
| `gzip_types` | 声明压缩的 MIME 类型（**注意：不设置时默认只压缩 `text/html`**） |
| `gzip_vary` | 添加 `Vary: Accept-Encoding` 头，帮助 CDN/代理正确缓存 |
| `gzip_proxied` | 作为代理时对上游响应的压缩策略 |
| `gzip_disable` | 按 UA 禁用压缩（正则匹配） |
| `gzip_buffers` | 压缩缓冲区（数量 × 大小） |

---

# 2. gzip_proxied 取值

| 值 | 说明 |
|----|------|
| `off` | 不压缩上游响应（默认） |
| `any` | 无条件压缩所有代理响应 |
| `expired` | 上游响应包含 `Expires` 头且过期时压缩 |
| `no-cache` | 上游响应包含 `Cache-Control: no-cache` 时压缩 |
| `no-store` | 上游响应包含 `Cache-Control: no-store` 时压缩 |
| `private` | 上游响应包含 `Cache-Control: private` 时压缩 |
| `auth` | 上游响应包含 `Authorization` 头时压缩 |

---

# 3. gzip_static（预压缩文件）

```nginx
http {
    gzip_static on;       # 优先查找 .gz 预压缩文件，无需实时压缩
    gzip_static always;   # 始终使用预压缩文件（忽略浏览器是否支持）
}
```

**前置条件**：源码编译时带 `--with-http_gzip_static_module`。

**使用方式**：

```bash
# 预先压缩静态资源
gzip -9 -k style.css        # 生成 style.css.gz
gzip -9 -k app.js           # 生成 app.js.gz
```

**优势对比**：

| 方式 | CPU | 响应速度 | 磁盘 |
|------|-----|----------|------|
| 实时 gzip | 高（每次请求都压缩） | 慢 | 省 |
| gzip_static | 零（直接发送） | 快 | 多一份 .gz 文件 |

---

# 4. Brotli 压缩（更高级）

Brotli 是 Google 开发的压缩算法，相比 gzip 压缩率更高（约 20% 提升），但压缩更慢。

```nginx
# 需要编译 ngx_brotli 模块（官方不内置）
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css text/javascript
             application/javascript application/json
             application/xml image/svg+xml;
brotli_static on;          # 预压缩 .br 文件
```

| 对比 | gzip | brotli |
|------|------|--------|
| 压缩率 | 基准 | 约高 20% |
| 压缩速度 | 快 | 慢（尤其高压缩级别） |
| 解压速度 | 快 | 相当 |
| 浏览器支持 | 所有 | 现代浏览器（Chrome/Firefox/Edge/Safari 11+） |
| Nginx 支持 | 内置 | 需第三方模块 |

---

# 5. sendfile / tcp_nodelay / tcp_nopush 组合

```nginx
http {
    sendfile on;            # 零拷贝传输文件
    tcp_nopush on;          # sendfile 开启时生效，数据装满一个包再发送
    tcp_nodelay on;         # 对于 keepalive 连接，小的数据包立即发送（不等待）
}
```

**三者关系**：

```
sendfile on  → 减少内核-用户空间拷贝
tcp_nopush  → 配合 sendfile，数据积累后一次性推送（大文件友好）
tcp_nodelay → keepalive 下小包即时发送（低延迟场景）
```

---

# 6. HTTP/2 推送（Server Push）

```nginx
server {
    listen 443 ssl http2;

    location / {
        http2_push /css/style.css;
        http2_push /js/app.js;
        http2_push_preload on;    # 自动推送 Link preload 的资源
    }
}
```

> ⚠️ HTTP/2 Push 可能浪费带宽（推送客户端已缓存的内容），Chrome 106+ 已移除此功能，建议用 `<link rel="preload">` 替代。

---

# 7. 完整传输优化配置

```nginx
http {
    # ── 基础传输优化 ──────────────
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # ── gzip 压缩 ────────────────
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/javascript
               application/javascript application/json
               application/xml application/rss+xml
               image/svg+xml font/woff2;
    gzip_proxied any;

    # ── gzip_static 预压缩 ───────
    gzip_static on;

    # ── keepalive ────────────────
    keepalive_timeout 65;
    keepalive_requests 100;
}
```
