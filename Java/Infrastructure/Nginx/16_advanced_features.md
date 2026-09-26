
---

# 1. Stream 模块（TCP/UDP 四层代理）

## 1.1 基本概念

Stream 模块工作于 **OSI 第四层（传输层）**，不解析 HTTP，直接转发 TCP/UDP 数据包。

```
客户端 ── TCP 连接 ──→ Nginx（Stream）──→ 上游 MySQL/Redis/Kafka
```

## 1.2 TCP 代理

```nginx
stream {
    upstream mysql_backend {
        server 192.168.1.10:3306 weight=1;
        server 192.168.1.11:3306 weight=1 backup;
    }

    server {
        listen 3306;
        proxy_pass mysql_backend;
        proxy_connect_timeout 5s;
        proxy_timeout 60s;
    }
}
```

## 1.3 SSL 四层代理（TLS Termination）

```nginx
stream {
    server {
        listen 443 ssl;
        proxy_pass 192.168.1.10:8443;

        ssl_certificate /etc/nginx/ssl/example.com.pem;
        ssl_certificate_key /etc/nginx/ssl/example.com.key;
        ssl_protocols TLSv1.2 TLSv1.3;
    }
}
```

## 1.4 按域名转发（SNI）

```nginx
stream {
    map $ssl_preread_server_name $backend {
        www.example.com  backend1;
        api.example.com  backend2;
        default          backend_default;
    }

    upstream backend1 { server 10.0.0.1:8443; }
    upstream backend2 { server 10.0.0.2:8443; }
    upstream backend_default { server 10.0.0.3:8443; }

    server {
        listen 443;
        ssl_preread on;                     # 预读 TLS SNI，不解密
        proxy_pass $backend;
    }
}
```

| 常见 stream 代理场景 | 端口 |
|----------------------|------|
| MySQL | 3306 |
| Redis | 6379 |
| Kafka | 9092 |
| PostgreSQL | 5432 |
| SSH | 22 |

---

# 2. WebSocket 代理

```nginx
http {
    upstream ws_backend {
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;
    }

    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        location /ws/ {
            proxy_pass http://ws_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            proxy_read_timeout 3600s;
            proxy_send_timeout 3600s;
        }
    }
}
```

| 关键配置 | 说明 |
|----------|------|
| `proxy_http_version 1.1` | WebSocket 需要 HTTP/1.1 |
| `Upgrade` / `Connection` | WebSocket 协议升级头 |
| `proxy_read_timeout` | 需设置为较大值（WS 长连接） |

---

# 3. HTTP/3（QUIC）

```nginx
server {
    listen 443 quic reuseport;      # UDP 监听
    listen 443 ssl http2;           # TCP 回退

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_certificate /etc/nginx/ssl/example.com.pem;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # 告诉客户端支持 HTTP/3
    add_header Alt-Svc 'h3=":443"; ma=86400';
}
```

> HTTP/3 基于 UDP（QUIC 协议），需要 Nginx 1.25+ 编译时带 `--with-http_v3_module`，且 SSL 库需支持 QUIC（如 BoringSSL、LibreSSL、OpenSSL 3.2+）。

---

# 4. GeoIP（地理位置分流）

```nginx
http {
    geoip_country /usr/share/GeoIP/GeoIP.dat;
    geoip_city /usr/share/GeoIP/GeoLiteCity.dat;

    map $geoip_country_code $allowed_country {
        default no;
        CN yes;
        US yes;
        JP yes;
    }

    server {
        if ($allowed_country = no) {
            return 403;
        }
    }
}
```

---

# 5. 图片处理（image_filter）

```nginx
location /img/ {
    # 裁剪为 200×200
    image_filter crop 200 200;
    image_filter_jpeg_quality 80;
    image_filter_buffer 10M;
}
```

---

# 6. 动态模块加载（Nginx 1.9.11+）

```bash
# 编译动态模块
./configure --with-http_image_filter_module=dynamic
make modules

# 加载动态模块
load_module modules/ngx_http_image_filter_module.so;
```

---

# 7. Lua 扩展（OpenResty）

OpenResty 在 Nginx 中嵌入 **LuaJIT**，实现复杂的业务逻辑：

```nginx
# OpenResty 配置
location /api/ {
    access_by_lua_block {
        -- 自定义鉴权逻辑
        local jwt = require "resty.jwt"
        local token = ngx.req.get_headers()["Authorization"]
        -- ...
    }

    content_by_lua_block {
        ngx.say("Hello from Lua!")
    }
}
```

| 场景 | 说明 |
|------|------|
| 动态路由 | 从 Redis/DB 读取路由规则 |
| 自定义鉴权 | JWT 校验、OAuth2 令牌验证 |
| API 聚合 | 合并多个后端响应 |
| 限流/降级/熔断 | 比内置模块更灵活的控制逻辑 |

---

# 8. GRPC 代理

```nginx
server {
    listen 443 ssl http2;
    server_name grpc.example.com;

    location / {
        grpc_pass grpc://grpc_backend;
    }
}

upstream grpc_backend {
    server 192.168.1.10:50051;
    server 192.168.1.11:50051;
}
```

---

# 9. 透明代理（ngx_http_proxy_protocol）

```nginx
# 上游支持 Proxy Protocol 时，穿透四层代理获取真实客户端 IP
server {
    listen 80 proxy_protocol;
    set_real_ip_from 10.0.0.0/8;
    real_ip_header proxy_protocol;
}
```
