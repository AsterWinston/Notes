
---

# 1. 基础站点配置

```nginx
server {
    listen 80;
    server_name www.example.com;
    root /usr/share/nginx/html;       # 站点根目录
    index index.html index.htm;       # 默认首页文件
}
```

**`root` vs `alias`**：

| 指令 | 行为 | 示例 |
|------|------|------|
| `root` | 拼接路径：`root路径 + URI` | `root /www;` → `/www/images/logo.png` |
| `alias` | 替换路径：`alias路径替换location前缀` | `alias /www/;` → `/www/logo.png` |

```nginx
# root 示例
location /images/ {
    root /var/www;              # 请求 /images/logo.png → /var/www/images/logo.png
}

# alias 示例（注意路径末尾的 /）
location /static/ {
    alias /var/www/static/;     # 请求 /static/logo.png → /var/www/static/logo.png
}
```

> `alias` 替换的是 location 匹配的前缀部分，不是拼接。路径末尾的 `/` 需要保持一致。

---

# 2. MIME 类型配置

```nginx
http {
    include /etc/nginx/mime.types;      # 标准 MIME 映射表
    default_type application/octet-stream;  # 未匹配时的默认类型
}
```

常用 MIME 类型片段（`mime.types`）：

```nginx
types {
    text/html                    html htm shtml;
    text/css                     css;
    text/javascript              js;
    image/png                    png;
    image/jpeg                   jpeg jpg;
    image/gif                    gif;
    application/json             json;
    application/pdf              pdf;
    application/xml              xml;
    font/woff2                   woff2;
}
```

---

# 3. 目录浏览（autoindex）

```nginx
location /downloads/ {
    alias /data/files/;
    autoindex on;                # 开启目录列表
    autoindex_exact_size off;    # 显示可读的文件大小（KB/MB/GB）
    autoindex_localtime on;      # 显示本地时间而非 UTC
    autoindex_format html;       # 格式：html | json | xml | jsonp
}
```

> **生产环境谨慎开启** autoindex，可能暴露目录结构。

---

# 4. 文件访问控制

```nginx
# 禁止访问隐藏文件（.开头的文件）
location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

# 禁止访问指定文件类型
location ~* \.(sql|log|md|yml|yaml)$ {
    deny all;
}

# 只允许特定请求方法
location / {
    limit_except GET HEAD POST {
        deny all;
    }
}
```

---

# 5. 静态资源缓存（Expires）

```nginx
# 图片、字体等长期缓存
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    expires 30d;                                 # 缓存 30 天
    add_header Cache-Control "public, immutable";
}

# HTML 不缓存（或短期缓存）
location ~* \.html$ {
    expires -1;                                  # 不缓存
    add_header Cache-Control "no-cache, must-revalidate";
}
```

| `expires` 值 | 说明 |
|--------------|------|
| `expires 1d;` | 1 天后过期 |
| `expires -1;` | 不缓存（等同于 `Cache-Control: no-cache`） |
| `expires off;` | 不添加 Expires/Cache-Control 头 |
| `expires max;` | 2037 年过期（等同于永久缓存） |

---

# 6. sendfile 与 tcp_nopush

```nginx
http {
    sendfile on;          # 零拷贝文件传输（绕过用户空间，直接内核态传输）
    tcp_nopush on;        # 配合 sendfile，将数据包装满后一次性发送
}
```

**sendfile 原理对比**：

```
传统方式：
  磁盘 → 内核缓冲区 → 用户缓冲区 → 内核Socket缓冲区 → 网卡

sendfile：
  磁盘 → 内核缓冲区（直接）→ 内核Socket缓冲区 → 网卡
  省去了一次"内核 → 用户 → 内核"的拷贝
```

---

# 7. try_files 指令

常用于前端 SPA 或优雅的 404 处理：

```nginx
location / {
    try_files $uri $uri/ /index.html;   # 找不到文件时返回 index.html（SPA 历史模式）
}
```

```nginx
# 更复杂的使用方式
location / {
    try_files $uri $uri/ @backend;
}

location @backend {
    proxy_pass http://127.0.0.1:8080;
}
```

`try_files` 的 `=code` 变体：

```nginx
location /images/ {
    try_files $uri /default.png =404;
}
```

---

# 8. 静态资源优化实践

```nginx
server {
    listen 80;
    server_name static.example.com;

    root /data/static;

    # 防盗链
    location ~* \.(jpg|jpeg|png|gif|webp)$ {
        valid_referers none blocked example.com *.example.com;
        if ($invalid_referer) {
            return 403;
        }
    }

    # 不记录静态资源日志
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        access_log off;
        expires 30d;
    }

    # SPA 回退
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
