
---

# 1. 知识体系回顾

```
┌─────────────────────────────────────────────────┐
│                 Nginx 知识体系                     │
├────────────┬──────────────┬──────────────────────┤
│  核心概念    │   配置实操     │    运维与架构          │
├────────────┼──────────────┼──────────────────────┤
│ Master/Worker│ nginx.conf  │ 安装部署               │
│ 事件驱动     │ 静态资源服务  │ SSL 证书               │
│ 11阶段处理   │ 反向代理     │ 日志管理               │
│ 变量体系     │ 负载均衡     │ 健康检查               │
│              │ Rewrite     │ 高可用（Keepalived）    │
│              │ 缓存体系     │ 平滑升级               │
│              │ 压缩优化     │ 监控（Prometheus）     │
│              │ 安全/CORS   │ 压测调优               │
│              │ 限流        │ 内核参数优化            │
│              │ Stream模块   │                       │
│              │ WebSocket   │                       │
└────────────┴──────────────┴──────────────────────┘
```

---

# 2. 高频面试题

## 2.1 Q1: Nginx 的 Master-Worker 进程模型是如何工作的？

```
Master 进程负责：
  - 读取和验证配置
  - 管理 Worker 进程的创建与销毁
  - 接收外部信号（reload、stop 等）

Worker 进程负责：
  - 处理所有客户端连接与请求
  - 通过共享锁（accept_mutex）竞争新连接
  - 单线程事件驱动，异步非阻塞 I/O

优势：
  - 热部署（Master 信号控制，Worker 平滑重启）
  - 进程隔离（Worker 崩溃不影响其他 Worker）

补充：
  - 优雅的进程架构使得 Nginx 可同时处理数万并发连接
  - 每个 Worker 处理大量连接而不阻塞，得益于异步 IO 多路复用模型
```

## 2.2 Q2: Nginx 如何处理请求？11 个阶段是什么？

```
请求处理依次经过 11 个阶段：

1.  POST_READ        → 读取请求头后，realip 模块
2.  SERVER_REWRITE   → server 块内的 rewrite
3.  FIND_CONFIG      → location 匹配
4.  REWRITE          → location 块内的 rewrite
5.  POST_REWRITE     → 防止 rewrite 死循环
6.  PREACCESS        → 限流（limit_req, limit_conn）
7.  ACCESS           → 访问控制（allow/deny, auth_basic）
8.  POST_ACCESS      → satisfy 指令
9.  PRECONTENT       → try_files
10. CONTENT          → 内容生成（static, proxy_pass, fastcgi 等）
11. LOG              → 记录 access_log

关键点：同一阶段的模块按配置顺序执行，不同阶段严格按阶段顺序执行。
```

## 2.3 Q3: Nginx 负载均衡算法有哪些？各自优缺点？

| 算法 | 指令 | 适用场景 |
|------|------|----------|
| 轮询（默认） | — | 后端性能一致 |
| 加权轮询 | `weight` | 后端性能不均 |
| IP Hash | `ip_hash` | 需要会话保持（但有局限性） |
| 最小连接 | `least_conn` | 长连接/请求时间不一 |
| URL Hash | `hash $request_uri` | 缓存友好 |
| 随机 | `random` | 流量均匀分布 |

> 补充：Nginx Plus 还有 `least_time`（最小响应时间）；hash 配合 `consistent` 实现一致性哈希，减少节点变动时的影响。

## 2.4 Q4: root 和 alias 的区别？

```
root：拼接完整路径
  root /var/www;
  请求 /images/logo.png → /var/www/images/logo.png

alias：路径替换
  alias /var/www/static/;
  请求 /static/logo.png  → /var/www/static/logo.png

alias 仅替换 location 匹配的前缀部分
```

## 2.5 Q5: proxy_pass 末尾 `/` 的影响？

```
location /api/ {
    proxy_pass http://backend;       → /api/user → http://backend/api/user
    proxy_pass http://backend/;      → /api/user → http://backend/user
}
不含路径 → 完整 URI 拼接
含有路径 → 替换 location 匹配前缀
```

## 2.6 Q6: Nginx 如何实现高可用？

```
1. Keepalived + VIP 漂移：
   - 主备两台 Nginx，Keepalived 管理虚拟 IP
   - 主节点故障 → VIP 自动漂移到备节点
   - 配合检测脚本实现 Nginx 进程监控

2. DNS 多活：多节点同时对外，DNS 轮询分发

3. 云平台 SLB/CLB/ALB + Nginx 集群

4. 平滑升级（USR2 + WINCH）实现零停机版本更新
```

## 2.7 Q7: Nginx 限流是如何实现的？有哪些方式？

```
1. limit_req（请求频率）：漏桶算法，限制每秒请求数
   limit_req_zone $binary_remote_addr zone=req:10m rate=10r/s;
   limit_req zone=req burst=20 nodelay;

2. limit_conn（并发连接数）：限制同一 IP 的连接数
   limit_conn_zone $binary_remote_addr zone=conn:10m;
   limit_conn conn 5;

3. limit_rate（带宽限制）：限制单连接传输速率
   limit_rate 500k;
   limit_rate_after 5m;   发送 5MB 后限速
```

## 2.8 Q8: Nginx 如何优化性能？

| 层面 | 优化措施 |
|------|----------|
| Worker | `worker_processes auto` + CPU 亲和性 |
| 连接 | `worker_connections` 调大 + `multi_accept on` |
| I/O | `sendfile on` (零拷贝) + `tcp_nopush` / `tcp_nodelay` |
| 压缩 | gzip level 5-6，静态资源使用 `gzip_static` |
| 连接池 | upstream `keepalive` |
| 缓存 | `proxy_cache` / `fastcgi_cache` |
| 日志 | 静态资源 `access_log off` + 日志缓冲 |
| 静态化 | 动静分离，CDN 加速 |
| 内核 | somaxconn / tcp_tw_reuse / 文件描述符限制 |

## 2.9 Q9: Nginx 常见状态码含义？

| 状态码 | 含义 |
|--------|------|
| 200 | 正常响应 |
| 301 | 永久重定向 |
| 302 | 临时重定向 |
| 304 | 缓存未修改 |
| 400 | 请求错误 |
| 403 | 禁止访问（权限/防盗链/限流） |
| 404 | 资源未找到 |
| 405 | 请求方法不允许 |
| 429 | 请求过于频繁（限流触发） |
| 499 | 客户端主动断开连接 |
| 502 | 上游服务器返回无效响应 |
| 503 | 服务暂时不可用（过载/维护） |
| 504 | 上游服务器响应超时 |

## 2.10 Q10: access_log 出现大量 502 如何排查？

```
1. 检查 upstream 后端服务是否正常运行
   - 直接 curl 后端地址
   - 查看后端应用日志

2. 检查 Nginx 与后端的网络连通性
   - telnet/ping 后端 IP 端口

3. 检查代理超时配置
   - proxy_connect_timeout / proxy_read_timeout 是否太短

4. 检查后端是否有防火墙/安全组限制

5. 检查 Worker 连接数是否达到上限
   - worker_connections 是否足够
```

---

# 3. 配置文件命名规范检查清单

| 检查项 | 说明 |
|--------|------|
| `nginx -t` | 配置语法检查（每次修改后必做） |
| 变量名拼写 | `$remote_addr` 不是 `$remote_addr` |
| 指令作用域 | `proxy_pass` 只在 location 内，`upstream` 只在 http 块 |
| location 优先级 | 注意精确匹配、前缀匹配和正则匹配的优先级 |
| return 后无其他指令 | `return` 执行后不再执行后续指令 |

---

# 4. 常用配置速查

```bash
# 检查配置
nginx -t

# 重载配置（平滑）
nginx -s reload

# 测试 URL
curl -I http://localhost

# 查看编译参数
nginx -V

# 查看当前连接情况
curl http://localhost/nginx_status
```

---

# 5. 进阶学习方向

```
Nginx 基础
  ├── OpenResty（Lua 开发）
  │     ├── API 网关开发（动态路由、鉴权、熔断）
  │     └── Redis/Lua 共享内存
  ├── Kong / APISIX（API 网关平台）
  ├── Tengine（阿里定制版）
  ├── Nginx Ingress Controller（K8s）
  └── CDN 原理与实现（基于 Nginx）
```
