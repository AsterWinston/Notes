
---

# 1. 静态站点配置

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /data/www/static;

    index index.html;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;

    location / {
        try_files $uri $uri/ =404;
    }

    # 静态资源长期缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

---

# 2. SPA（单页面应用）配置

```nginx
server {
    listen 80;
    server_name app.example.com;
    root /data/www/spa;

    index index.html;

    location / {
        try_files $uri $uri/ /index.html;      # 前端路由回退到 index.html
    }

    location /api/ {
        proxy_pass http://backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

# 3. Spring Boot 前后端分离配置

```nginx
upstream app_server {
    server 127.0.0.1:8080;
    keepalive 32;
}

server {
    listen 80;
    server_name www.example.com;

    # 前端静态文件
    root /data/www/frontend;
    index index.html;

    # API 反向代理到 Spring Boot
    location /api/ {
        proxy_pass http://app_server/;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 60s;
        client_max_body_size 50m;              # 文件上传大小
    }

    # 前端路由回退
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源不记录日志
    location ~* \.(js|css|png|jpg|gif|ico)$ {
        access_log off;
        expires 30d;
    }
}
```

---

# 4. HTTPS + HTTP/2 完整站点

```nginx
server {
    listen 80;
    server_name www.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.example.com;

    ssl_certificate     /etc/nginx/ssl/example.com.pem;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    root /data/www/site;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

# 5. API 网关配置（多服务聚合）

```nginx
# 各后端服务 upstream
upstream user_service { server 192.168.1.10:8080; keepalive 32; }
upstream order_service { server 192.168.1.11:8080; keepalive 32; }
upstream product_service { server 192.168.1.12:8080; keepalive 32; }

server {
    listen 80;
    server_name api.example.com;

    # 通用代理头
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 限流
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=20r/s;
    limit_req zone=api_limit burst=30 nodelay;

    location /api/user/ {
        proxy_pass http://user_service/;
    }

    location /api/order/ {
        proxy_pass http://order_service/;
    }

    location /api/product/ {
        proxy_pass http://product_service/;
    }

    # 健康检查
    location /health {
        access_log off;
        return 200 "OK";
    }
}
```

---

# 6. WebSocket 代理配置

```nginx
upstream ws_server {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 80;
    server_name ws.example.com;

    location /ws/ {
        proxy_pass http://ws_server;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        proxy_read_timeout 86400s;
    }
}
```

---

# 7. 图片服务器配置

```nginx
server {
    listen 80;
    server_name img.example.com;
    root /data/images;

    # 防盗链
    location ~* \.(jpg|jpeg|png|gif|webp)$ {
        valid_referers none blocked *.example.com;
        if ($invalid_referer) {
            return 403;
        }
        expires 7d;
    }
}
```

---

# 8. 文件下载服务器

```nginx
server {
    listen 80;
    server_name files.example.com;

    location /downloads/ {
        alias /data/files/;
        autoindex on;
        autoindex_exact_size off;

        # 下载限速
        limit_rate 1m;
        limit_rate_after 10m;

        # 并发限制
        limit_conn_zone $binary_remote_addr zone=download_conn:10m;
        limit_conn download_conn 3;
    }
}
```

---

# 9. 多域名 HTTPS（SNI / 单 IP）

```nginx
server {
    listen 443 ssl http2;
    server_name www.example.com;
    ssl_certificate /etc/nginx/ssl/example.com.pem;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
    # ...
}

server {
    listen 443 ssl http2;
    server_name www.another.com;
    ssl_certificate /etc/nginx/ssl/another.com.pem;
    ssl_certificate_key /etc/nginx/ssl/another.com.key;
    # ...
}
```

---

# 10. 灰度发布（按 Header / Cookie 分流）

```nginx
upstream stable { server 192.168.1.10:8080; }
upstream canary { server 192.168.1.11:8080; }

split_clients "${remote_addr}${http_user_agent}" $variant {
    10% canary;
    *   stable;
}

server {
    location /api/ {
        proxy_pass http://$variant;
    }
}
```

```nginx
# 按 Header 灰度
map $http_x_canary $backend {
    default stable;
    "true" canary;
}
```

---

# 11. 全局通用 nginx.conf 模板

```nginx
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
pid /var/run/nginx.pid;

events {
    use epoll;
    worker_connections 10240;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format json escape=json '{'
        '"timestamp":"$time_iso8601",'
        '"remote_addr":"$remote_addr",'
        '"method":"$request_method",'
        '"uri":"$uri",'
        '"status":$status,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time"'
    '}';

    access_log /var/log/nginx/access.log json buffer=64k flush=30s;
    error_log  /var/log/nginx/error.log warn;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    server_tokens off;
    client_max_body_size 20m;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;

    include /etc/nginx/conf.d/*.conf;
}
```
