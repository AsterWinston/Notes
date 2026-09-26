
---

# 1. nginx.conf 结构总览

```nginx
# ── 全局块（Global Context）───────────────────
user  nginx;                        # Worker 进程运行用户
worker_processes  auto;             # Worker 进程数
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

# ── events 块 ───────────────────────────────
events {
    worker_connections  1024;
    use epoll;
}

# ── http 块 ──────────────────────────────────
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 全局日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    keepalive_timeout  65;

    # ── upstream 块 ─────────────────────────
    upstream backend {
        server 192.168.1.10:8080 weight=1;
        server 192.168.1.11:8080 weight=2;
    }

    # ── server 块（虚拟主机）────────────────
    server {
        listen       80;
        server_name  www.example.com;

        # ── location 块 ────────────────────
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }

        location /api/ {
            proxy_pass http://backend;
        }
    }
}
```

---

# 2. 配置块层级关系

```
main（全局块）
├── events         —— 连接处理相关
├── http           —— HTTP 协议相关
│   ├── upstream   —— 上游服务器组
│   ├── server     —— 虚拟主机
│   │   └── location —— URI 路由匹配
│   └── server ...
├── stream         —— TCP/UDP 四层代理
│   ├── upstream
│   └── server
│       └── location ...
└── mail           —— 邮件代理
```

---

# 3. 核心指令详解

## 3.1 全局块指令

| 指令 | 说明 | 示例 |
|------|------|------|
| `user` | Worker 进程运行用户 | `user nginx;` |
| `worker_processes` | Worker 进程数 | `auto` / `4` |
| `worker_rlimit_nofile` | Worker 最大打开文件数 | `worker_rlimit_nofile 65535;` |
| `error_log` | 错误日志路径和级别 | `error_log logs/error.log warn;` |
| `pid` | Master 进程 PID 文件路径 | `pid /run/nginx.pid;` |

## 3.2 events 块指令

| 指令 | 说明 | 示例 |
|------|------|------|
| `worker_connections` | 单个 Worker 最大连接数 | `worker_connections 10240;` |
| `use` | 事件模型选择 | `use epoll;` |
| `multi_accept` | 一次接受所有新连接 | `multi_accept on;` |
| `accept_mutex` | Worker 互斥锁 | `accept_mutex on;` |

## 3.3 http/server 块指令

| 指令 | 说明 | 示例 |
|------|------|------|
| `sendfile` | 零拷贝传输文件 | `sendfile on;` |
| `tcp_nopush` | TCP_CORK 选项，在 sendfile 开启时使用 | `tcp_nopush on;` |
| `tcp_nodelay` | TCP_NODELAY 选项，禁用 Nagle 算法 | `tcp_nodelay on;` |
| `keepalive_timeout` | 长连接超时时间（秒） | `keepalive_timeout 65;` |
| `client_max_body_size` | 客户端请求体最大大小 | `client_max_body_size 10m;` |
| `client_header_buffer_size` | 请求头缓冲区大小 | `client_header_buffer_size 1k;` |
| `server_tokens` | 是否在响应头中显示版本号 | `server_tokens off;` |

---

# 4. 虚拟主机（server_name）

Nginx 通过 `server_name` 区分不同站点：

## 4.1 匹配优先级

```
1. 精确匹配        server_name www.example.com;
2. 前通配符        server_name *.example.com;
3. 后通配符        server_name www.example.*;
4. 正则表达式      server_name ~^wwww\d+\.example\.com$;
5. 默认服务器      listen 80 default_server;
```

```nginx
server {
    listen 80 default_server;       # 默认虚拟主机（没有匹配到其他时使用）
    server_name _;                  # _ 表示不关心主机名
}
```

## 4.2 多域名配置

```nginx
server {
    listen 80;
    server_name example.com www.example.com *.api.example.com;
}
```

---

# 5. Nginx 变量体系

## 5.1 内置变量（部分常用）

| 变量 | 含义 | 示例值 |
|------|------|--------|
| `$remote_addr` | 客户端 IP | `192.168.1.100` |
| `$remote_port` | 客户端端口 | `54321` |
| `$server_addr` | 服务器 IP | `10.0.0.1` |
| `$server_port` | 服务器端口 | `80` |
| `$server_name` | 虚拟主机名 | `www.example.com` |
| `$host` | 请求头 Host 字段 | `www.example.com` |
| `$uri` | 当前 URI（不含参数） | `/index.html` |
| `$args` / `$query_string` | URL 参数 | `?page=1&size=10` |
| `$request_method` | 请求方法 | `GET` / `POST` |
| `$request_uri` | 完整原始请求 URI | `/index.html?page=1` |
| `$scheme` | 协议 | `http` / `https` |
| `$http_<name>` | 任意请求头（小写，横线转下划线） | `$http_user_agent` |
| `$upstream_addr` | 上游服务器地址（反向代理后） | `192.168.1.10:8080` |
| `$upstream_status` | 上游响应状态码 | `200` |
| `$upstream_response_time` | 上游响应时间（秒） | `0.032` |
| `$request_time` | 请求总处理时间（秒） | `0.051` |
| `$status` | 响应状态码 | `200` |
| `$body_bytes_sent` | 响应体字节数 | `1234` |

## 5.2 自定义变量

```nginx
set $my_var "hello";
set $mobile_flag "";

if ($http_user_agent ~* (mobile|android|iphone)) {
    set $mobile_flag "1";
}
```

---

# 6. include 与模块化配置

```nginx
http {
    include /etc/nginx/conf.d/*.conf;          # 批量引入站点配置
    include /etc/nginx/upstreams/*.conf;       # 批量引入 upstream 配置
}
```

典型的模块化目录结构：

```
/etc/nginx/
├── nginx.conf                    # 主配置
├── conf.d/                       # 站点配置
│   ├── www.example.com.conf
│   └── api.example.com.conf
├── upstreams/                    # upstream 配置
│   └── backend.conf
└── includes/                     # 公共片段
    ├── ssl.conf
    ├── security.conf
    └── gzip.conf
```
