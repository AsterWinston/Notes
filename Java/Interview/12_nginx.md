# 1. 知识点
- **高性能HTTP反向代理**：接收客户端请求后转发给后端服务器，隐藏后端真实地址，提供安全性和负载均衡能力
- **IMAP/POP3邮件代理**：Nginx 也可作为邮件代理服务器，支持 IMAP、POP3、SMTP 协议
- **TCP/UDP负载均衡**：Nginx Stream 模块支持四层负载均衡，对 TCP/UDP 流量进行代理和转发
- **高并发事件驱动**：采用异步非阻塞事件驱动模型，单机可支持数万甚至数十万并发连接
- **epoll模型**：Linux 下使用 epoll 多路复用，高效管理大量连接，事件通知机制避免轮询
- **multi-process架构**：Master 进程 + 多个 Worker 进程，Worker 之间独立运行互不影响
- **Master主进程**：负责管理和监控 Worker 进程、读取和解析配置、绑定端口、接收信号
- **Worker工作进程**：实际处理客户端请求，每个 Worker 是单线程的，一个 Worker 可同时处理多个连接
- **一个Worker一个进程单线程**：Worker 进程内部是单线程模型，通过事件驱动处理多个连接，避免线程切换开销
- **worker_processes auto**：设置 Worker 进程数为 CPU 核心数，auto 自动检测
- **CPU亲和性(worker_cpu_affinity)**：将每个 Worker 进程绑定到指定 CPU 核心，减少 CPU 切换提升缓存命中率
- **worker_connections**：单个 Worker 进程能同时打开的最大连接数，需配合操作系统 ulimit 调整
- **multi_accept**：一次接受所有新连接的开关，开启后 Worker 会一次尽可能多地接收连接
- **最大连接数公式**：最大连接数 = worker_processes × worker_connections，反向代理时要除以2（客户端+后端各占一个）
- **proxy_pass**：反向代理核心指令，将请求转发到后端服务器，可指定 URL 路径
- **upstream**：定义上游服务器组，可配置多个后端服务器及负载均衡策略
- **轮询(round-robin)**：默认负载均衡算法，按顺序将请求依次分配给服务器
- **加权(weight)**：为每个后端设置权重，权重越高分配越多请求，适合服务器性能不均的场景
- **ip_hash**：根据客户端 IP 生成哈希值，同一 IP 始终路由到同一台后端，解决 Session 不一致问题
- **least_conn**：将请求转发到当前活跃连接数最少的服务器
- **least_time**：将请求转发到响应时间最短的服务器（需商业版 Nginx Plus）
- **url_hash**：按请求 URL 哈希分配，同一 URL 始终到同一台服务器，用于缓存命中优化
- **fair**：第三方负载均衡算法，根据服务器响应时间动态分配请求
- **动静分离**：通过 location 规则区分静态/动态请求，静态请求直接读取本地文件返回
- **try_files**：按顺序检查文件是否存在，常用于前端 SPA 应用的路由回退
- **location匹配规则(=精确匹配/^~前缀匹配/~正则区分大小写/~*正则不区分大小写/通用前缀匹配)**：location 有多种匹配方式，优先级为 = > ^~ > ~/~* > 前缀匹配
- **Session共享问题**：分布式环境下 Nginx 反向代理分配的服务器可能不同，导致 Session 丢失
- **ip_hash粘性会话**：使用 ip_hash 算法让同一客户端始终访问同一后端
- **Redis集中存储Session**：将 Session 统一存储在 Redis 中，所有服务器共享
- **JWT Token**：使用无状态 Token 认证，完全不需要服务端存储 Session
- **跨域配置**：通过 add_header Access-Control-Allow-Origin 等指令解决跨域访问问题
- **HTTPS配置**：使用 ssl_certificate 和 ssl_certificate_key 配置 SSL 证书和私钥，ssl_protocols 指定 TLS 版本
- **ssl_certificate**：指定 SSL 证书文件路径
- **ssl_certificate_key**：指定 SSL 私钥文件路径
- **ssl_protocols**：指定支持的 TLS 协议版本，如 TLSv1.2 TLSv1.3
- **rewrite重写**：URL 重写指令，flag 有 last（继续匹配）、break（停止匹配）、redirect（302临时重定向）、permanent（301永久重定向）
- **缓存(proxy_cache)**：proxy_cache_path 定义缓存路径/层级/内存大小，proxy_cache_valid 设置不同状态码缓存时间
- **proxy_cache_path**：定义缓存文件存储路径、缓存层级深度(levels)、内存中键存储区(keys_zone)
- **gzip压缩**：gzip on 开启压缩、gzip_types 指定压缩的 MIME 类型、gzip_comp_level 压缩级别 1-9、gzip_min_length 最小压缩长度
- **限流(limit_req_zone)**：基于令牌桶算法，定义限流区域和速率，limit_req 引用限流规则
- **rate=10r/s**：速率设置，每秒允许10个请求
- **burst峰值**：允许的突发请求上限，超过 burst 的请求直接拒绝或延迟处理
- **日志格式(log_format)**：定义自定义日志格式，通过 $remote_addr $request_time 等变量记录详细信息
- **$request_time**：Nginx 处理请求的总时间，即从读取客户端第一个字节到发送完最后一个字节的时间
- **$upstream_response_time**：后端服务器响应时间，用于排查后端性能问题，两者的差值反映 Nginx 自身的处理耗时
- **健康检查**：在 upstream 内设置 max_fails 失败次数和 fail_timeout 超时时间，被动检查后端健康状态；也可使用第三方模块 ngx_http_upstream_check_module 主动健康检查
- **max_fails**：在 fail_timeout 时间内最大失败次数，超过后标记后端不可用
- **fail_timeout**：后端标记不可用的持续时间，之后重试
- **性能优化**：调整 worker_processes、worker_connections、启用 sendfile on 零拷贝、tcp_nodelay 禁用 Nagle 算法、keepalive_timeout 合理设置
- **tcp_nodelay**：禁用 TCP 延迟确认，小数据包立即发送，减少延迟
- **sendfile on**：开启零拷贝文件传输，文件数据直接从磁盘到网络不经过用户态，大幅提升静态文件传输效率
- **Nginx vs Apache**：Nginx 异步非阻塞适合高并发静态和反向代理，Apache 同步多进程/线程模型适合动态处理
- **OpenResty**：基于 Nginx + LuaJIT 的高性能 Web 平台，可在 Nginx 中编写 Lua 脚本实现动态逻辑

# 2. 面试题

## 2.1 Nginx 的进程模型是怎样的？
Nginx 采用 Master-Worker 多进程架构。Master 进程只有一个，负责管理和监控 Worker 进程、加载配置、绑定端口和接收信号。Worker 进程数量由 worker_processes 指令配置（通常设为 CPU 核心数），每个 Worker 是单线程独立的进程，通过事件驱动模型（epoll）异步非阻塞地处理多个客户端连接。Worker 之间是平等的，通过共享内存来竞争 accept 锁获取新连接。当 Master 收到 reload 信号时，会先创建新的 Worker，待新 Worker 就绪后优雅关闭旧 Worker，实现热加载。

## 2.2 Nginx 如何处理高并发？为什么比 Apache 性能好？
Nginx 采用异步非阻塞事件驱动模型，使用 epoll（Linux）等 I/O 多路复用技术，一个 Worker 单线程即可同时处理数万个连接。请求处理中无阻塞操作，所有 I/O 都注册到事件循环中。而 Apache 传统采用 prefork（多进程）或 worker（多线程）模型，每个连接占用一个进程/线程，大量并发时内存和 CPU 上下文切换开销巨大。Nginx 在处理静态文件、反向代理场景下性能远超 Apache。

## 2.3 location 匹配的优先级是什么？
location 匹配优先级从高到低：
1. `= /path` — 精确匹配，最高优先级
2. `^~ /prefix` — 前缀匹配，如果匹配成功则不再进行正则匹配
3. `~ pattern` — 区分大小写的正则匹配
4. `~* pattern` — 不区分大小写的正则匹配（3 和 4 按配置文件中的书写顺序匹配）
5. `/prefix` — 普通前缀匹配，匹配最长前缀

## 2.4 rewrite 指令中 last 和 break 有什么区别？
两者都不会改变浏览器 URL 地址栏，区别在于重写后的行为：
- `last`：重写后立即停止当前 rewrite 指令集，用新的 URI 重新在 server 块中进行新一轮 location 匹配
- `break`：重写后停止当前 rewrite 指令集，不再进行后续的 rewrite，但不会重新触发 location 匹配，直接在当前 location 中处理

`redirect`（302）和 `permanent`（301）则会返回 HTTP 重定向给客户端，浏览器地址栏会改变。

## 2.5 什么是正向代理和反向代理？Nginx 主要做什么代理？
正向代理是代理客户端，客户端知道代理的存在和目标服务器，常见场景如翻墙 VPN、公司内网代理上网。反向代理是代理服务器端，客户端不知道后端真实服务器地址，以为是和代理直接通信，常见场景如负载均衡、安全防护、SSL 终结。Nginx 主要做反向代理（也可配置为正向代理）。

## 2.6 Nginx 的负载均衡算法有哪些？
- **轮询(round-robin)**：默认，按顺序依次分配
- **权重(weight)**：根据服务器性能分配不同权重
- **ip_hash**：按客户端 IP 哈希，同一 IP 始终打同一台后端
- **least_conn**：分配到当前活跃连接最少的后端
- **least_time**：分配到响应时间最短的后端（Nginx Plus 商业版）
- **url_hash**：按请求 URL 哈希分配
- **fair**：第三方模块，按响应时间动态分配
配置方式是在 upstream 块中设置对应指令，如 `least_conn;`、`ip_hash;`。

## 2.7 如何配置 Nginx 的 HTTPS？
```nginx
server {
    listen 443 ssl;
    server_name example.com;
    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    location / {
        proxy_pass http://backend;
    }
}
```
同时配置 80 端口跳转到 443：
```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

## 2.8 分布式环境下 Session 共享问题怎么解决？
三种方案：
1. **ip_hash**：Nginx 使用 ip_hash 负载均衡，同一客户端 IP 始终路由到同一台服务器，简单但服务器宕机时 Session 丢失
2. **Redis 集中存储**：所有服务器共享同一个 Redis/Session 集群，Spring Session 等框架支持，推荐方案
3. **JWT Token**：使用无状态 Token 认证，Token 包含用户信息，每次请求携带 Token 验证，完全不需要服务端存储 Session

## 2.9 Nginx 如何配置动静分离？
```nginx
server {
    listen 80;
    server_name example.com;
    # 静态资源
    location /static/ {
        root /data/www;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    # 动态请求
    location /api/ {
        proxy_pass http://backend_server;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```
通过 location 匹配规则，静态资源直接从本地文件系统响应，可以配合 expires 设置浏览器缓存；动态请求通过 proxy_pass 转发到后端应用服务器。

## 2.10 如何配置 Nginx 限流？
```nginx
# http 块中定义限流区域（基于客户端 IP，令牌桶速率 10r/s）
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    location /api/ {
        limit_req zone=mylimit burst=20 nodelay;
        proxy_pass http://backend;
    }
}
```
使用 `limit_req_zone` 定义限流区域及速率，`limit_req` 引用限流规则，`burst` 设置突发容量，`nodelay` 表示超过 rate 但小于 burst 的请求也立即处理。

## 2.11 Nginx 缓存如何配置？
```nginx
# http 块
proxy_cache_path /data/nginx/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60m use_temp_path=off;

server {
    location / {
        proxy_cache my_cache;
        proxy_cache_valid 200 304 12h;
        proxy_cache_valid 301 302 1h;
        proxy_cache_valid any 1m;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        add_header X-Cache-Status $upstream_cache_status;
        proxy_pass http://backend;
    }
}
```
通过 `proxy_cache_path` 定义缓存路径，`proxy_cache_valid` 设置不同状态码的缓存时间，`$upstream_cache_status` 变量可查看缓存命中状态（HIT/MISS/EXPIRED）。

## 2.12 Nginx 性能优化的措施有哪些？
- `worker_processes auto;` — Worker 进程数等于 CPU 核心数
- `worker_connections 65535;` — 单个 Worker 最大连接数，结合 OS ulimit 调整
- `sendfile on;` — 启用零拷贝文件传输
- `tcp_nopush on;` — 配合 sendfile，数据包累积到一定大小再发送
- `tcp_nodelay on;` — 适用于 keepalive 连接，小包不延迟
- `keepalive_timeout 65;` — 合理设置长连接超时
- `gzip on;` — 启用压缩减少传输数据
- `proxy_cache` — 启用代理缓存减少后端压力
- 日志适当分级（减少不必要的日志级别），使用异步日志

## 2.13 什么是 OpenResty？有什么应用场景？
OpenResty 是基于 Nginx + LuaJIT 的高性能 Web 平台，它将 Lua 脚本引擎嵌入 Nginx，开发者可以在 Nginx 的每个请求阶段（如 access、content、log 等阶段）执行 Lua 代码。典型应用场景包括：API 网关（Kong 就是基于 OpenResty 构建）、动态限流、WAF 防火墙、AB 测试分流、实时业务规则处理等。通过 Lua 脚本可以实现传统 Nginx 配置无法实现的复杂动态逻辑。

## 2.14 如何配置 Nginx 跨域 CORS？
```nginx
location /api/ {
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS, PUT, DELETE';
    add_header Access-Control-Allow-Headers 'Content-Type, Authorization, X-Requested-With';
    add_header Access-Control-Max-Age 86400;
    if ($request_method = 'OPTIONS') {
        return 204;
    }
    proxy_pass http://backend;
}
```
OPTIONS 预检请求直接返回 204，不需要转发到后端。

## 2.15 Nginx 如何实现灰度发布/金丝雀发布？
可以通过 split_clients 模块根据客户端特征（IP、Header、Cookie 等）将部分流量路由到新版本服务：
```nginx
split_clients "${remote_addr}${http_user_agent}" $variant {
    10% backend_v2;
    *   backend_v1;
}
upstream backend_v1 { server v1:8080; }
upstream backend_v2 { server v2:8080; }
server {
    location / {
        proxy_pass http://$variant;
    }
}
```
或者通过 map 指令基于 Cookie 或 Header 做流量分发。
