
---

# 1. CORS 跨域配置

## 1.1 基本 CORS

```nginx
location /api/ {
    proxy_pass http://backend;

    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
    add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-Requested-With";
    add_header Access-Control-Allow-Credentials false;
    add_header Access-Control-Max-Age 86400;          # 预检请求缓存 24 小时

    # OPTIONS 预检请求直接返回 204
    if ($request_method = OPTIONS) {
        return 204;
    }
}
```

## 1.2 带凭证的 CORS

```nginx
server {
    # 白名单域名（通过变量控制）
    set $cors_origin "";

    if ($http_origin ~* "https?://(www\.)?example\.com$") {
        set $cors_origin $http_origin;
    }

    location /api/ {
        add_header Access-Control-Allow-Origin $cors_origin;
        add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
        add_header Access-Control-Allow-Headers "Content-Type, Authorization";
        add_header Access-Control-Allow-Credentials true;
        add_header Access-Control-Max-Age 86400;

        if ($request_method = OPTIONS) {
            return 204;
        }
    }
}
```

> `Access-Control-Allow-Origin` 使用 `*` 时，`Allow-Credentials` 必须为 `false`，否则浏览器会拒绝。

## 1.3 map 方式实现多域名 CORS

```nginx
map $http_origin $cors_origin {
    default "";
    "~^https?://(www\.)?example\.com$" "$http_origin";
    "~^https?://localhost(:\d+)?$" "$http_origin";
}

server {
    location /api/ {
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials true always;
        ...
    }
}
```

---

# 2. Basic Auth（基本认证）

```bash
# 安装 httpd-tools 工具
yum install -y httpd-tools

# 创建密码文件（添加用户）
htpasswd -c /etc/nginx/.htpasswd admin
# 追加用户（不加 -c）
htpasswd /etc/nginx/.htpasswd user1
```

```nginx
location /admin/ {
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

---

# 3. IP 访问控制

```nginx
location /admin/ {
    allow 192.168.1.0/24;      # 允许内网
    allow 10.0.0.0/8;
    deny all;                  # 拒绝其他所有
}

location /internal/ {
    deny 192.168.1.100;        # 封禁特定 IP
    allow all;
}
```

---

# 4. 安全响应头

```nginx
server {
    # 隐藏 Nginx 版本
    server_tokens off;

    # XSS 防护（浏览器内置反射型 XSS 过滤）
    add_header X-XSS-Protection "1; mode=block" always;

    # 禁止 MIME 类型嗅探
    add_header X-Content-Type-Options "nosniff" always;

    # 控制 iframe 嵌套（防点击劫持）
    add_header X-Frame-Options "SAMEORIGIN" always;

    # 内容安全策略（CSP）
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data:;" always;

    # 引荐策略
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # 权限策略
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
}
```

---

# 5. 隐藏后端信息

```nginx
http {
    server_tokens off;              # 隐藏版本号（Server 头显示 "nginx"）
    # server_tokens build;          # 显示自定义字符串（商业版）

    # 隐藏后端响应中的 Server 头
    proxy_hide_header X-Powered-By;
    proxy_hide_header X-Application-Version;
    proxy_hide_header Server;

    # 自定义 Server 头（需安装 headers-more 模块）
    # more_set_headers "Server: cloud-platform";
}
```

---

# 6. 防止 DDoS 攻击的基础配置

```nginx
http {
    # 限制请求体大小（防大文件上传攻击）
    client_max_body_size 10m;

    # 限制请求头大小
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;

    # 限制连接数（见第14章限流详解）
    limit_conn_zone $binary_remote_addr zone=conn:10m;
    limit_conn conn 10;
}
```

---

# 7. SSL 安全加固

```nginx
server {
    listen 443 ssl http2;
    server_tokens off;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5:!RC4;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}
```

---

# 8. njs 脚本安全（Web 防火墙基础）

```nginx
# Nginx App Protect / njs 实现 WAF 规则示例（需加载 njs 模块）
js_import /etc/nginx/waf.js;

server {
    location / {
        js_content waf.check;
    }
}
```

> 开源版可用 **OpenResty + lua-resty-waf** 实现更完整的 WAF 功能；生产环境可选用 **ModSecurity + Nginx** 或 **Nginx Plus WAF**。
