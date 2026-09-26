
---

# 1. Worker 进程优化

```nginx
# ── worker 数量 ────────────────
worker_processes auto;                    # 自动匹配 CPU 核数（推荐）
# worker_processes 4;                    # 手动指定

# ── CPU 亲和性（绑定 CPU核心）──
worker_cpu_affinity auto;                # Nginx 1.9+ 自动分配
# worker_cpu_affinity 0001 0010 0100 1000;  # 4核手动绑定

# ── Worker 优先级 ──────────────
worker_priority -5;                      # 范围 -20 ~ 19（越小越优先）

# ── 最大文件描述符 ────────────
worker_rlimit_nofile 65535;              # Worker 进程最大打开文件数
```

---

# 2. 连接与事件优化

```nginx
events {
    use epoll;                            # Linux 下最优事件模型
    worker_connections 10240;             # 单 Worker 最大并发连接数
    multi_accept on;                      # 一次接受所有新连接
    accept_mutex off;                     # Linux 3.9+ 内核已解决惊群，可关闭
}
```

**最大连接数**：
```
连接数 = worker_processes × worker_connections

系统级限制需同步调整：
echo 65535 > /proc/sys/net/core/somaxconn          # 全连接队列
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse             # TIME_WAIT 复用
echo 1024 65535 > /proc/sys/net/ipv4/ip_local_port_range  # 可用端口范围
```

---

# 3. HTTP 层面优化

```nginx
http {
    sendfile on;                          # 零拷贝
    tcp_nopush on;                        # 累积发送
    tcp_nodelay on;                       # 即时发送小包

    keepalive_timeout 65;                 # 长连接超时
    keepalive_requests 100;               # 单连接最大请求数

    client_max_body_size 10m;             # 请求体最大限制
    client_body_buffer_size 128k;         # 请求体缓冲区

    client_header_buffer_size 1k;         # 请求头缓冲区
    large_client_header_buffers 4 8k;     # 大型请求头缓冲区

    # 不记录静态资源访问日志
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        access_log off;
    }
}
```

---

# 4. 连接池（Keepalive）优化

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    keepalive 100;                        # 空闲长连接池大小
    keepalive_timeout 60s;                # 空闲连接超时
    keepalive_requests 1000;              # 单连接最大请求数（Nginx 1.19+）
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;           # 强制 HTTP/1.1
        proxy_set_header Connection "";   # 清除 Connection: close
    }
}
```

---

# 5. 缓冲区调优

```nginx
http {
    # ── 代理缓冲区 ──────────────
    proxy_buffer_size 4k;
    proxy_buffers 8 16k;
    proxy_busy_buffers_size 32k;
    proxy_max_temp_file_size 256m;

    # ── 响应式内容可选关闭缓冲 ─
    # proxy_buffering off;       # 关闭缓冲降低延迟（长连接/大文件慎用）
}
```

---

# 6. Gzip 压缩参数优化

```nginx
gzip on;
gzip_comp_level 5;            # 过高浪费 CPU 且收益递减（5~6 为最佳平衡）
gzip_min_length 512;
gzip_buffers 16 8k;
gzip_vary on;
```

---

# 7. 操作系统内核参数调优

```bash
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535

net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 3

net.ipv4.ip_local_port_range = 1024 65000

# 文件描述符限制
echo "* soft nofile 65535" >> /etc/security/limits.conf
echo "* hard nofile 65535" >> /etc/security/limits.conf

# 立即生效
sysctl -p
```

---

# 8. 静态资源分离架构

```
                         ┌─→ Nginx 静态（static.example.com）→ 静态文件
用户 → CDN → Nginx 主入口─┤
                         └─→ Nginx 代理 → upstream → Spring Boot（api.example.com）
```

```nginx
# 主入口
server {
    listen 80;
    location /api/ {
        proxy_pass http://app_backend;
    }
    location / {
        # 静态资源 CNAME 到 CDN / 独立静态域名
        return 301 $scheme://static.example.com$request_uri;
    }
}
```

---

# 9. 性能监控与测试

```bash
# 压测工具
ab -n 10000 -c 100 http://localhost/
wrk -t4 -c100 -d30s --latency http://localhost/

# 查看 Nginx 连接状态
curl http://localhost/nginx_status      # 需要 stub_status 模块

# 实时连接数查看
ss -ant | grep :80 | wc -l
```

**压测目标参考**：

| 场景 | 静态文件 QPS | 动态代理 QPS |
|------|-------------|-------------|
| 单机（4C8G） | 50,000+ | 5,000~15,000 |
| 开启日志 | 略降 5~10% | 略降 5~10% |
| 开启 HTTPS | 约为 HTTP 的 30~50% | 同左 |
