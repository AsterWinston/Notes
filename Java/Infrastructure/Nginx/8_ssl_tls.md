
---

# 1. HTTPS 基本配置

```nginx
server {
    listen 443 ssl;
    server_name www.example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.pem;      # 证书文件（公钥）
    ssl_certificate_key /etc/nginx/ssl/example.com.key;      # 私钥文件

    # 可选：中间证书链
    # ssl_trusted_certificate /etc/nginx/ssl/chain.pem;
}
```

---

# 2. 证书相关指令

| 指令 | 说明 |
|------|------|
| `ssl_certificate` | 服务器证书（PEM 格式） |
| `ssl_certificate_key` | 私钥文件 |
| `ssl_trusted_certificate` | 中间证书 / CA 根证书链 |
| `ssl_password_file` | 私钥密码文件 |
| `ssl_protocols` | 启用的 SSL/TLS 协议版本 |
| `ssl_ciphers` | 加密套件 |
| `ssl_prefer_server_ciphers` | 优先使用服务端加密套件 |
| `ssl_session_cache` | SSL 会话缓存 |
| `ssl_session_timeout` | SSL 会话超时时间 |

---

# 3. 安全推荐配置

```nginx
server {
    listen 443 ssl http2;                     # 启用 HTTP/2
    server_name www.example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.pem;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # ── 协议与加密套件 ─────────────────
    ssl_protocols TLSv1.2 TLSv1.3;            # 禁用 TLSv1.0 / TLSv1.1（不安全）
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # ── SSL 会话缓存 ──────────────────
    ssl_session_cache shared:SSL:10m;         # 共享缓存，10MB ≈ 40000 会话
    ssl_session_timeout 1d;                   # 会话超时1天
    ssl_session_tickets off;                  # 关闭 Session Ticket（安全性考虑）

    # ── DH 参数（提升安全性）───────────
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;
}
```

> **生成 DH 参数**：`openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048`

---

# 4. HTTP 强制跳转 HTTPS

```nginx
# 方案1：单独 server 块做 301 重定向
server {
    listen 80;
    server_name www.example.com;
    return 301 https://$host$request_uri;
}

# 方案2：同 server 块内判断
server {
    listen 80;
    listen 443 ssl http2;
    server_name www.example.com;

    if ($scheme = http) {
        return 301 https://$host$request_uri;
    }
}
```

> 推荐**方案1**（两个独立的 server 块），更清晰且性能更好。

---

# 5. HSTS（HTTP Strict Transport Security）

```nginx
server {
    listen 443 ssl http2;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}
```

| 参数 | 含义 |
|------|------|
| `max-age` | HSTS 生效时长（秒），推荐 63072000（2年） |
| `includeSubDomains` | 子域名同样强制 HTTPS |
| `preload` | 允许加入浏览器 HSTS Preload 列表 |

---

# 6. SSL 性能优化

```nginx
# 开启 SSL Session 缓存
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;

# 开启 OCSP Stapling（减少客户端验证时间）
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
```

---

# 7. 多域名 / 泛域名证书

```nginx
# 多域名证书（证书包含多个 SAN）
server {
    listen 443 ssl;
    server_name www.example.com api.example.com;
    ssl_certificate /etc/nginx/ssl/multi.pem;
    ssl_certificate_key /etc/nginx/ssl/multi.key;
}

# 泛域名证书
server {
    listen 443 ssl;
    server_name ~^(www|api|cdn)\.example\.com$;
    ssl_certificate /etc/nginx/ssl/wildcard.pem;
    ssl_certificate_key /etc/nginx/ssl/wildcard.key;
}
```

---

# 8. Let's Encrypt 免费证书 + 自动续签

```bash
# 安装 certbot
yum install -y certbot python3-certbot-nginx

# 获取证书（自动配置 Nginx）
certbot --nginx -d example.com -d www.example.com

# 仅获取证书（手动配置）
certbot certonly --nginx -d example.com

# 自动续签（添加到 crontab）
0 3 * * * certbot renew --quiet --post-hook "nginx -s reload"
```

---

# 9. 自签证书（测试环境）

```bash
# 生成自签证书
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/nginx/ssl/selfsigned.key \
    -out /etc/nginx/ssl/selfsigned.crt \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=Test/CN=localhost"
```

---

# 10. SSL 测试工具

```bash
# 在线检测（Qualys SSL Labs）
# https://www.ssllabs.com/ssltest/

# 命令行检测
openssl s_client -connect example.com:443 -tls1_2
nmap --script ssl-enum-ciphers -p 443 example.com
```
