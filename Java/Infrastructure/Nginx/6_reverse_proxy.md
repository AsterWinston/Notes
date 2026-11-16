
---

反向代理是 Nginx 在 Java 架构中最核心的用法：应用（Tomcat / Spring Boot / Go / Node）藏在 Nginx 后面，客户端只与 Nginx 通信。看似一句 `proxy_pass` 就完事，但生产事故几乎都埋在细节里——**URI 拼接规则、转发后丢失的原始信息（Host / 真实 IP / 协议）、超时参数、缓冲策略、WebSocket 握手**。

# 1. 什么是反向代理

```text
客户端 ──→ Nginx（反向代理） ──→ 后端服务器1
                  ├───────────→ 后端服务器2
                  └───────────→ 后端服务器3
```

- **正向代理**：代理客户端，服务端不知道真实客户端（如 VPN）
- **反向代理**：代理服务端，客户端不知道真实后端（如 Nginx 在 Web 服务前）

一句话记牢方向：**正向代理替"客户端"出头，隐藏的是客户端；反向代理替"服务端"接客，隐藏的是后端拓扑**。对 Java 后端而言反向代理的价值：SSL 终结集中在入口、把慢客户端的网络收发从应用线程剥离、负载均衡与故障转移、横切能力（缓存/限流/鉴权）统一实现。

> **注意**：反向代理会**断开 TCP 层**——客户端到 Nginx 是一条连接，Nginx 到后端是另一条独立连接，两段的超时、缓冲、keep-alive 各自独立。这解释了为什么后端看到的源 IP 永远是 Nginx 的 IP（真实 IP 要靠请求头透传），也解释了为什么 WebSocket、SSE 这类长连接要专门配置。

# 2. proxy_pass 指令

最基础的写法，把 `/api/` 前缀的请求转给后端：

```nginx
location /api/ {
    proxy_pass http://192.168.1.10:8080;
}
```

`proxy_pass` 的目标可以是 IP:port、域名、也可以是 upstream 组名（`http://backend`，见负载均衡篇）。**是否带 URI 路径才是关键**。

## 2.1 URI 拼接规则（重要）

| 配置 | 请求 `/api/user/list` 转发为 | 说明 |
|------|------------------------------|------|
| `proxy_pass http://backend;` | `http://backend/api/user/list` | **不带路径**：完整 URI 拼接 |
| `proxy_pass http://backend/;` | `http://backend/user/list` | **带路径**：替换 location 前缀 |
| `proxy_pass http://backend/api;` | `http://backend/apiuser/list` | 直接拼接（不推荐） |
| `proxy_pass http://backend/api/;` | `http://backend/api/user/list` | 指定路径 + 拼接剩余部分 |
| `proxy_pass http://backend/v1/;` | `http://backend/v1/user/list` | 路径替换 |

> 规则总结：`proxy_pass` 的 URL 中**包含路径部分**时，location 的匹配前缀会被替换；**不含路径**时，完整 URI 直接拼接。

**这是反向代理第一大坑**，成因要回到 Nginx 处理 URI 的两条路径。当 `proxy_pass` 不带 URI 时，Nginx 把客户端**原始 URI** 整段透传；一旦带了 URI（哪怕只是一个 `/`），Nginx 就改用"location 前缀替换"逻辑，把匹配到的 location 部分换成 proxy_pass 的路径。于是：

- `location /api/ { proxy_pass http://backend; }` → `/api/x` 变 `http://backend/api/x`（后端也要有 `/api` 前缀才接得住）。
- `location /api/ { proxy_pass http://backend/; }` → `/api/x` 变 `http://backend/x`（**剥掉了 `/api` 前缀**）。

两者只有一字（`/`）之差，行为完全不同。Spring Boot 应用如果 context-path 配成 `/api`，就用**不带斜杠**那行；应用根路径是 `/`、Nginx 只是用 `/api` 做路由区分，就用**带斜杠**那行剥前缀。**记忆口诀：想保留前缀不写斜杠、想剥前缀写斜杠**。

> **设计考量**：`proxy_pass http://backend/api;`（带路径但不带尾斜杠）会直接字符串拼接产出 `/apiuser/list` 这种畸形路径，几乎总是 bug。团队约定应统一：要么 `http://backend`（无 URI），要么 `http://backend/`，避免模糊写法。

# 3. 请求头传递

反向代理断开 TCP 后，后端拿不到客户端原始信息。而且 Nginx 默认转发时**不会带上请求里的 Host**，而是用 `proxy_pass` 目标主机名（或 upstream 名）作为 Host，导致后端虚拟主机、跳转 URL、应用日志全部错乱：

```nginx
location /api/ {
    proxy_pass http://backend;

    # ── 基本头传递 ─────────────────────
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;
}
```

| 头字段 | 含义 |
|--------|------|
| `Host` | 原始请求的主机名 |
| `X-Real-IP` | 客户端真实 IP（单个 IP） |
| `X-Forwarded-For` | 代理链 IP 列表（逗号分隔） |
| `X-Forwarded-Proto` | 原始请求协议（http/https） |
| `X-Forwarded-Host` / `X-Forwarded-Port` | 原始 Host / 端口 |

**`X-Real-IP` 与 `X-Forwarded-For` 的区别**：前者是**单个**客户端 IP，`$remote_addr` 直取；后者是**代理链列表**，`$proxy_add_x_forwarded_for` 会把"客户端已有 XFF + 本机 remote_addr"追加成 `client, proxy1, proxy2` 链。**信任边界很重要**：客户端可伪造 XFF 第一个值，后端**绝不能直接信 XFF[0] 做安全判断**，应只信任最靠近自己的那一跳。Spring Boot 侧 `X-Forwarded-Proto` 直接关系重定向 URL 与 Secure Cookie，需开 `server.forward-headers-strategy=NATIVE` 让应用消费这些头。

> **行业实践**：多层代理（CDN → Nginx → 应用）时，真实 IP 要么用 `real_ip` 模块配 `set_real_ip_from <可信代理网段>` + `real_ip_header X-Forwarded-For`，把 `$remote_addr` 直接改写成真实客户端 IP（日志和限流都受益）；要么约定"从右往左取第一个非内网 IP"。切忌无条件采信 XFF。

# 4. 代理 Buffer 配置

`proxy_buffering` 决定后端响应是"先收进缓冲区再发"还是"边收边发"，直接影响后端连接被占用的时长：

```nginx
location /api/ {
    proxy_pass http://backend;

    # ── 代理缓冲区 ─────────────────────
    proxy_buffering on;                       # 开启代理缓冲（默认开启）
    proxy_buffer_size 4k;                     # 响应头缓冲区大小
    proxy_buffers 8 8k;                       # 响应体缓冲区（数量 × 大小）
    proxy_busy_buffers_size 16k;              # 允许忙碌缓冲区的最大值
    proxy_max_temp_file_size 1024m;           # 临时文件最大大小

    # ── 上传缓冲区 ─────────────────────
    client_body_buffer_size 128k;             # 客户端请求体缓冲区
}
```

**取舍逻辑**：`proxy_buffering on`（默认）时，Nginx 尽快把后端响应读进内存缓冲、**随即释放后端连接**，再用缓冲慢慢喂给（可能很慢的）客户端——这对保护**稀缺的 Tomcat 线程**极其关键，代价是大响应溢出到 `proxy_temp_file` 磁盘文件。关闭缓冲（`proxy_buffering off`）变成"边收边发"，后端连接被占用到客户端取完为止，只适合**流式响应**（SSE、长轮询），首字节延迟最低但后端资源消耗最大。

> **注意**：`proxy_max_temp_file_size` 设 0 表示禁用临时文件，缓冲装不下时后端连接就一直被占。流式接口除了关 proxy 缓冲，还要让后端回 `X-Accel-Buffering: no`（Nginx 会识别）或该 location 显式 `proxy_buffering off`，否则 SSE 事件会被攒在缓冲里迟迟不下发，表现为"消息延迟一批批到达"。

# 5. 超时配置

反向代理涉及连接后端、向后端发请求、等响应三段超时，加上客户端侧构成参数族：

```nginx
location /api/ {
    proxy_pass http://backend;

    proxy_connect_timeout 60s;        # 连接后端超时（默认 60s）
    proxy_send_timeout 60s;           # 发送请求到后端超时
    proxy_read_timeout 60s;           # 从后端读取响应超时
}
```

| 参数 | 计时对象 | 何时触发 | 调优建议 |
|------|---------|---------|---------|
| `proxy_connect_timeout` | Nginx ↔ 后端建连 | 后端宕机/网络不通（上限 75s） | 内网 3-5s 足够，快速失败 |
| `proxy_send_timeout` | Nginx → 后端上传请求体 | 后端收得慢 | 上传接口适当加大 |
| `proxy_read_timeout` | Nginx → 后端等首字节/响应 | **后端处理慢/长查询** | 最常被调到；普通 API 10-30s，报表/大查询更长 |
| `send_timeout` | Nginx → 客户端发响应 | 客户端接收慢 | 与后端无关 |
| `client_header/body_timeout` | 客户端 → Nginx 收请求 | 慢客户端/CC 攻击 | 入口防护，短（10-15s） |

**排障口诀：502 看 connect（连不上后端），504 看 read（后端处理超时）**。`proxy_read_timeout` 是日常最常调的：报表导出、慢 SQL、大模型推理这类长耗时接口要单独 location 放大该值，不要全局调大（会拖延正常请求的故障判定）。注意 `proxy_connect_timeout` 文档上限 75s，超过会被取整。

> **设计考量**：超时链路要**对齐递减**。若 Nginx 的 `proxy_read_timeout=30s` 而后端 Spring 的异步/DB 超时是 60s，Nginx 会先 504 掐断，后端却还在白跑；合理做法是入口超时 ≤ 应用超时，并配合 `proxy_next_upstream timeout` 让超时请求换一台后端重试（写接口慎用，见第 7 节）。

# 6. WebSocket 代理

WebSocket 建立在 HTTP/1.1 的 `Upgrade` 握手之上。Nginx 默认用 **HTTP/1.0** 连后端且不透传 Upgrade 头，导致握手失败连接被立刻关闭。正确配置：

```nginx
location /ws/ {
    proxy_pass http://backend_ws;
    proxy_http_version 1.1;           # WebSocket 需要 HTTP/1.1
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;

    proxy_read_timeout 3600s;         # WebSocket 长连接，超时设长
}
```

四个要点缺一不可：`proxy_http_version 1.1`（默认 1.0 不支持 Upgrade）、`Upgrade` 与 `Connection` 头透传、以及**放大 `proxy_read_timeout`**——WebSocket 空闲时长时间无数据，默认 60s 读超时会把静默连接掐断，表现为"每隔一分钟掉线一次"。

> **注意**：更优雅的写法是用 `map $http_upgrade $connection_upgrade { default upgrade; '' close; }` 生成 `$connection_upgrade` 赋给 Connection 头——普通 HTTP 请求（无 Upgrade）会正确设成 `close`，同一 location 即可同时代理 ws 与普通流量。负载均衡多实例时 WebSocket 有状态，务必配合 `ip_hash` 或 sticky，否则消息丢失。

# 7. 常用代理场景

## 7.1 API 代理与重试

屏蔽路径与旧地址跳转都用 `return`：

```nginx
location /admin/ {
    return 403;                                      # 屏蔽特定路径
}

location /old-path {
    return 301 https://example.com/new-path$request_uri;   # 通用 return 语法
}
```

带故障转移的 API 代理：

```nginx
location /api/ {
    proxy_pass http://backend;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_connect_timeout 3s;
    proxy_read_timeout 30s;
    proxy_next_upstream error timeout http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
}
```

`proxy_next_upstream` 定义"什么情况下换一台后端重试"。**关键陷阱：默认只含 `error timeout`，且对非幂等请求重试可能造成重复下单/重复写入**。生产上更保守的配置应加 `non_idempotent` 才允许重试 POST，或干脆只对 GET 类安全方法开重试；`http_500` 一般**不放进去**（500 是后端逻辑错误，换一台也一样，重试只会放大故障）。剥前缀除了 proxy_pass 尾斜杠，也可用 rewrite 显式改写：`rewrite ^/api/(.*)$ /$1 break;` 后再 `proxy_pass http://backend_api;`，效果等价但多一次正则开销。

> **行业实践**：把"公共头透传、超时、缓冲、重试"抽成 `includes/proxy_common.conf`，各 location `include` 复用，避免每处 copy-paste 后走样：

```nginx
# includes/proxy_common.conf
proxy_http_version 1.1;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_connect_timeout 5s;
proxy_send_timeout 30s;
proxy_read_timeout 30s;
proxy_buffering on;
```

## 7.2 upstream 与负载均衡策略

反向代理的目标一旦从"一台 IP:port"换成"一组后端"，就需要 `upstream` 块声明服务器组并选择分发算法（深入内容见 7_load_balancing）：

| 策略 | 指令 | 分发依据 | 典型场景 | 主要风险 |
|------|------|---------|---------|---------|
| 加权轮询（默认） | 无 / `weight` | 顺序 + 权重 | 后端配置一致、短请求 | 处理时间差异大时慢机器堆积 |
| 最小连接数 | `least_conn` | 当前活跃连接最少 | 请求耗时差异大的业务 | 未考虑机器性能差异 |
| IP 哈希 | `ip_hash` | 客户端 IPv4 前三段哈希 | HttpSession 未共享的老系统 | NAT/LB 后 IP 集中导致倾斜 |
| URL/自定义哈希 | `hash $request_uri consistent` | 键的哈希（一致性） | 后端本地缓存命中优先 | 键分布不均 |

**选择倾向**：会话仍写本地 HttpSession 又一时改不动 → `ip_hash`；请求耗时天然不齐 → `least_conn`；无状态微服务 → 默认加权轮询即可。**任何情况下都不如把会话外置到 Redis**，让策略选择自由起来。

## 7.3 完整反向代理配置

```nginx
upstream backend_api {
    server 192.168.1.10:8080 max_fails=2 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=2 fail_timeout=30s;
    keepalive 64;
}

server {
    listen 80;
    server_name api.example.com;

    location /api/ {
        proxy_pass http://backend_api/;         # 带斜杠：剥掉 /api 前缀
        include includes/proxy_common.conf;
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 2;
    }

    location /ws/ {
        proxy_pass http://backend_api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;   # 来自 map
        proxy_read_timeout 3600s;
    }
}
```

# 8. 小结

- **proxy_pass 尾斜杠**：不带 URI 透传原始路径、带 `/` 剥 location 前缀——保留前缀不写斜杠、剥前缀写斜杠。
- **头透传**：`Host` 必设；真实 IP 用 `X-Real-IP`（单值）+ `X-Forwarded-For`（链），后端只信最近一跳。
- **超时族**：`connect`(502) / `read`(504) / `send`，入口超时 ≤ 应用超时，长耗时接口单独放大 `proxy_read_timeout`。
- **缓冲**：`proxy_buffering on` 释放后端连接保护应用线程；流式/SSE 关缓冲并用 `X-Accel-Buffering: no`。
- **WebSocket**：HTTP/1.1 + Upgrade/Connection 头 + 长读超时，多实例配会话保持。
- **重试**：`proxy_next_upstream` 慎对非幂等请求，500 不重试。

下一篇（7_load_balancing）展开 upstream 的负载均衡算法与后端健康摘除。
