
---

# 1. Nginx 缓存架构

```
客户端 → Nginx（缓存层）→ 命中缓存 → 直接返回（极快）
              ↓ 未命中
            上游服务器（后端）→ 响应 → 存入缓存 → 返回客户端
```

---

# 2. proxy_cache（反向代理缓存）

## 2.1 基本配置

```nginx
http {
    # ── 1. 定义缓存路径与参数 ──────────
    proxy_cache_path /data/nginx/cache levels=1:2
                     keys_zone=my_cache:10m
                     max_size=10g
                     inactive=60m
                     use_temp_path=off;

    # ── 2. 定义缓存 key ────────────────
    proxy_cache_key "$scheme$request_method$host$request_uri";

    server {
        location /api/ {
            proxy_pass http://backend;

            # ── 3. 启用缓存 ─────────────
            proxy_cache my_cache;
            proxy_cache_valid 200 302 10m;     # 200/302 缓存 10 分钟
            proxy_cache_valid 404      1m;     # 404 缓存 1 分钟
            proxy_cache_valid any      0;      # 其他状态码不缓存

            # ── 4. 缓存状态头 ───────────
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

## 2.2 proxy_cache_path 参数说明

| 参数 | 说明 |
|------|------|
| `levels` | 缓存目录层级（如 `1:2` = 一级 1 字符，二级 2 字符） |
| `keys_zone` | 共享内存区域名称和大小（存储缓存 key 元数据） |
| `max_size` | 缓存磁盘最大大小（超过后自动清理最少使用的） |
| `inactive` | 缓存过期时间（多久未访问则删除） |
| `use_temp_path` | 禁用临时路径，直接写入缓存路径（推荐 `off`） |

---

# 3. 缓存控制策略

## 3.1 跳过缓存的条件

```nginx
proxy_cache_bypass $http_pragma $http_authorization;     # 满足条件时跳过缓存（仍然更新缓存）
proxy_no_cache $http_pragma $http_authorization;         # 不缓存的条件
```

## 3.2 使用 Cookie 跳过缓存

```nginx
# 如果有 user_token Cookie，跳过缓存（登录用户看到实时数据）
proxy_cache_bypass $cookie_user_token;
proxy_no_cache $cookie_user_token;
```

## 3.3 精细化缓存控制

```nginx
# 根据响应时间决定缓存时间
proxy_cache_valid 200 302 10m;
proxy_cache_valid 301      1h;
proxy_cache_valid 404      1m;
proxy_cache_valid any      0;

# 缓存过期后使用旧缓存（后端宕机时保底）
proxy_cache_use_stale error timeout http_500 http_502 http_503;

# 多个请求同时访问同一个未缓存资源时，只发一个请求到后端
proxy_cache_lock on;
proxy_cache_lock_timeout 5s;
```

---

# 4. $upstream_cache_status 取值

| 值 | 说明 |
|----|------|
| `MISS` | 缓存未命中，从后端获取并已缓存 |
| `HIT` | 缓存命中 |
| `EXPIRED` | 缓存已过期，从后端获取新内容 |
| `STALE` | 使用过期旧缓存（后端不可用，配置了 `proxy_cache_use_stale`） |
| `UPDATING` | 缓存正在更新中（配置了 `proxy_cache_lock`） |
| `REVALIDATED` | 使用 `If-Modified-Since` 重新验证，304 响应 |
| `BYPASS` | 强制跳过缓存（触发了 `proxy_cache_bypass`） |

---

# 5. 缓存 purging（清理）

Nginx 开源版不内置缓存清理功能，可通过以下方案实现：

```nginx
# 方案1：使用 ngx_cache_purge 模块（需编译）
location ~ /purge(/.*) {
    allow 127.0.0.1;
    deny all;
    proxy_cache_purge my_cache $scheme$request_method$host$1;
}

# 方案2：脚本直接删除缓存文件
# rm -rf /data/nginx/cache/xxx
```

---

# 6. fastcgi_cache（PHP-FPM 缓存）

```nginx
http {
    fastcgi_cache_path /data/nginx/fastcgi_cache levels=1:2
                       keys_zone=fcgi_cache:10m max_size=1g inactive=60m;

    server {
        location ~ \.php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_cache fcgi_cache;
            fastcgi_cache_key "$scheme$request_method$host$request_uri";
            fastcgi_cache_valid 200 10m;
            fastcgi_cache_valid 404 1m;

            add_header X-FastCGI-Cache $upstream_cache_status;
        }
    }
}
```

---

# 7. 浏览器缓存（Expires / Cache-Control）

```nginx
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}

location ~* \.html$ {
    expires -1;
    add_header Cache-Control "no-cache, must-revalidate";
}
```

---

# 8. 缓存实践——分离静态与动态

```nginx
http {
    proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=api_cache:10m
                     max_size=1g inactive=30m use_temp_path=off;

    server {
        # 静态资源不经过后端，直接文件系统缓存
        location ~* \.(jpg|png|css|js|svg|woff2)$ {
            root /app/static;
            expires 7d;
        }

        # 动态 API 使用代理缓存
        location /api/ {
            proxy_pass http://backend;
            proxy_cache api_cache;
            proxy_cache_valid 200 1m;
            proxy_cache_key "$request_uri|$http_authorization";
            proxy_cache_bypass $http_authorization;          # 有认证头时跳过
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```
