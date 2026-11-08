
---

Nginx 站在流量入口，天然是整个链路"第一现场"的观测点：**它知道每个请求花了多久（`$request_time`）、后端花了多久（`$upstream_response_time`）、返回了什么状态码、以及每台后端的健康状况**。本篇覆盖三层：Nginx 自监控（stub_status）、访问日志定制与采集（ELK/Prometheus）、负载均衡视角的健康检查（主动 vs 被动）与后端长连接复用对 Tomcat 的影响。

# 1. stub_status（基础状态监控）

`stub_status` 由 `ngx_http_stub_status_module` 提供（源码编译需 `--with-http_stub_status_module`；发行版包一般已编入，`nginx -V` 确认），暴露一组进程级计数器：

```nginx
location /nginx_status {
    stub_status;
    allow 127.0.0.1;         # 仅允许本机访问
    deny all;
}
```

访问 `curl http://localhost/nginx_status` 返回：

```text
Active connections: 291
server accepts handled requests
 16630948 16630948 31070465
Reading: 6 Writing: 179 Waiting: 106
```

| 字段 | 含义 |
|------|------|
| `Active connections` | 当前活跃连接数 |
| `accepts` | 累计接受的连接数 |
| `handled` | 累计处理的连接数 |
| `requests` | 累计处理的请求数 |
| `Reading` | 正在读取请求头的连接数 |
| `Writing` | 正在发送响应的连接数 |
| `Waiting` | 保持连接（keep-alive）等待新请求的连接数 |

**指标解读的正确姿势是做差分**：所有计数都是启动以来的累计值，直接看没意义，`rate()` 后才变成 QPS 与连接速率。三个高信号组合：其一 **`requests / handled` 得出平均每连接请求数**，健康站点应明显大于 1（keep-alive 生效），≈1 说明客户端都在用完即关，连接开销白白翻倍；其二 `Reading` 持续偏高意味着 Nginx 在**读客户端请求头**上卡住（慢客户端、body 未发完、CC 攻击特征），`Writing` 高是发响应慢（带宽或慢客户端），与后端无关——**后端慢体现在 access_log 的 upstream_response_time，不体现在这三个计数上**。

> **注意**：stub_status 的计数从 1.3 起即为**全局聚合**，无需担心多 worker 求和问题；但该端点本身必须收紧访问——它泄漏流量规模，公网可达等于给对手免费仪表盘。生产至少 `allow` 内网网段 + `deny all`，或只在管理端口暴露。

# 2. 自定义监控指标

不想用 exporter 的轻量场景，可用内置变量把状态拼成 JSON 直接给采集脚本：

```nginx
server {
    location /status {
        access_log off;
        default_type application/json;
        return 200 '{
            "connections_active": $connections_active,
            "connections_reading": $connections_reading,
            "connections_writing": $connections_writing,
            "connections_waiting": $connections_waiting
        }';
    }
}
```

$connections_* 系列变量的数字**不带引号恰好构成合法 JSON**。按 upstream 维度的活跃/空闲连接、队列深度，开源版只有 Plus 状态 API 才给全，社区普遍用 access_log 反推（见下节）。

# 3. Prometheus + nginx-exporter

标准方案是官方 exporter 抓取 stub_status 转成 Prometheus 指标：

```bash
# 部署 nginx-prometheus-exporter
docker run -d --name nginx-exporter \
    -p 9113:9113 \
    nginx/nginx-prometheus-exporter:latest \
    -nginx.scrape-uri http://nginx/nginx_status
```

**Grafana Dashboard**：导入官方模板（ID: 11199 / 12708）

exporter 把 `nginx_connections_active`、`nginx_http_requests_total` 等指标暴露在 9113，Prometheus scrape 后配 Alertmanager 告警，典型规则如 `rate(nginx_http_requests_total[5m]) < 10`（流量断崖，可能入口故障）。

**比 stub_status 更细的指标要从 access_log 来**：用日志解析型 exporter 得到**按状态码、按 location、按 upstream** 的 QPS 与延迟直方图——P99、5xx 比例这类 SLO 指标只能这么算。

# 4. access_log 自定义格式：链路延迟的真相

监控反向代理链路，最核心的两个变量是 `$request_time`（Nginx 收到第一个字节到发完最后一个字节的**端到端总时长**）与 `$upstream_response_time`（Nginx 与后端建连到收完响应的时长，多次重试时会以逗号并列多个值）。定制格式一次采齐：

```nginx
# 定义在 http 块
log_format perf escape=json
    '$time_iso8601'
    ' "$request_method $request_uri" '
    '$status '
    '$body_bytes_sent '
    '"$http_x_forwarded_for" '
    '$request_time '                    # 端到端耗时（含客户端网络时间）
    '$upstream_response_time '          # 后端耗时（含重试多值）
    '$upstream_addr '                   # 实际打到的后端
    '$upstream_status '                 # 后端返回的状态
    '$scheme "$http_user_agent"';

access_log /var/log/nginx/access.log perf;
```

**四个衍生判据**：

| 指标组合 | 健康形态 | 异常含义 |
|----------|---------|---------|
| `request_time ≈ upstream_response_time` | 正常，Nginx 层无开销 | 差值大 → 客户端接收慢 / 带宽打满 / 缓冲配置不当 |
| `upstream_response_time` 只有一个值 | 单跳成功 | 逗号多个 → 发生了重试，第一条后端有问题 |
| `upstream_status = 502/504` | — | 后端进程挂/超时，对照 error.log 时间戳定位 |
| 5xx 占比与 `upstream_addr` 交叉 | 均摊 | 集中在某一台 → 该实例故障，摘除 |

`escape=json` 是 1.19+ 的选项，保证 URL 里的引号不破坏后续解析（没有它时含 `"` 的 UA 会让整行字段错位，是 ELK 解析最常见的脏数据来源）。

**日志采集链路**按规模分两档：中小规模直接 Filebeat/Vector tail `/var/log/nginx/*.json` → Elasticsearch/Kafka → Kibana/Grafana Loki；云原生环境用 fluent-bit DaemonSet 采容器 stdout。一个易踩的运维细节是**轮转与采集的竞争**：logrotate 的 `create` 模式先 mv 再建新文件，Filebeat 可能丢 tail 句柄，改 `copytruncate` 或让采集器监听 rename 事件；日志文件里 `location` 级 `access_log off`（静态资源）会让 QPS 统计"偏低"，报表口径要写清楚。

> **行业实践**：告警阈值定在**比例和分位数**上而不是绝对值——"5xx 占比 > 1% 持续 5 分钟"比"错误数 > 100"稳定；延迟用 P99 而非均值。ELK 里为 Nginx 建 index template，`request_time`/`upstream_response_time` 显式映射为 float，别让 ES dynamic mapping 把 `"-, -, 0.031"`（重试三跳的 upstream 时间）识别成 keyword 导致聚合失效。

# 5. 健康检查：主动 vs 被动

## 5.1 被动健康检查（开源版默认）

开源 Nginx 的摘除机制是**被动**的——不做任何主动探测，只在**真实请求打过去失败时**记账：

```nginx
upstream backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=60s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=60s;
}

location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    proxy_next_upstream_tries 2;
    proxy_next_upstream_timeout 10s;
}
```

语义是：`fail_timeout` 窗口内失败达到 `max_fails` 次，把该 server 标记不可用**同样的时长**（窗口与冷却合一，这是常见误解——fail_timeout 不是"探测间隔"而是"失败计数窗口 + 摘除时长"）。什么算"失败"由 `proxy_next_upstream` 定义；默认不含 http_500，写进去要谨慎（见 6_reverse_proxy 篇：500 换台机器通常也白搭）。

**被动检查的三个盲区**：其一，**必须有人踩雷才知道雷在**——第一个请求要么报错要么超时后故障机才被摘除，发现滞后一个真实用户；其二，**后端恢复同样要真流量试**，没有慢启动，刚活过来的 JVM 预热期会被瞬间打满；其三，重试与用户请求耦合，拉长尾部延迟。

## 5.2 主动健康检查

**主动检查**由独立探测流周期性打健康检查端点，与业务流量无关：

```nginx
# Nginx Plus（商业版）
upstream backend {
    zone backend_zone 64k;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;

    health_check interval=10s fails=3 passes=2;
    match server_alive {
        status=200-399;
    }
}
```

```nginx
# Tengine / ngx_http_upstream_check_module（开源补丁方案）
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;

    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}
```

```lua
-- OpenResty 方案：lua 脚本定期探测后端，动态更新 upstream
local check = require "resty.upstream.healthcheck"
```

```text
主动 vs 被动时序对比：

被动：  请求─▶ 后端A(已挂) ─超时─▶ 记1次失败 ─▶ 重试B成功 ─▶ 累计max_fails后A才摘除
        （每个"第一批撞上"的请求都要付一次超时代价）

主动：  探测─▶ 后端A /health 超时 ─▶ fall次后标记down ─▶ 业务请求根本不去A
        └─── 每interval秒一次，独立于业务流量，rise次后回池
```

三种落地路线的取舍：

| 方案 | 获取方式 | 优点 | 代价 |
|------|---------|------|------|
| Nginx Plus `health_check` | 商业订阅 | 官方支持、配合动态 API | License 费用 |
| Tengine / check 模块 | 换发行版或重编译 | 开源免费、久经考验 | 非主线代码，升级绑定分支 |
| OpenResty lua 探测 | OpenResty 发行版 | 灵活（可查依赖后再判健康） | 自研维护成本 |

**现代架构的第三种答案**：K8s 的 readinessProbe + Service、或注册中心（Nacos/Consul）+ 动态 upstream 同步，把健康检查交给平台层，Nginx 只消费"当前可用列表"。自建 VM 无平台层时，才值得上主动检查补丁。

> **注意**：`/health` 端点的语义要设计——Spring Boot Actuator 的 `/actuator/health` 默认聚合 DB/Redis/MQ 依赖，**依赖抖动会让全部实例同时"不健康"从而雪崩式摘除**；给 LB 用的端点应只反映"本进程能否接流量"，深诊断走独立通道；探测频率也要与容量匹配，故障时高频探测会放大成 DDoS。

# 6. keepalive 到后端：对 Tomcat 的影响

开源版 upstream 到后端的连接复用需三件套（细节见 7_load_balancing）：

```nginx
upstream backend {
    server 192.168.1.10:8080;
    keepalive 32;                      # 每个 worker 缓存的空闲长连接数
}

location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;            # 必须 HTTP/1.1
    proxy_set_header Connection "";    # 必须清掉默认的 Connection: close
}
```

对 **Tomcat 的影响是双向的**。正面：省掉 TCP 三次握手与（HTTPS 后端时）TLS 握手，高 QPS 下显著降低后端 **TIME_WAIT/端口耗尽**压力；Nginx→Tomcat 的内网连接本是极短命的，不复用时每秒几千新建连接会让 Tomcat 的 accept 线程与 GC（每个连接关联对象）持续承压。负面：Tomcat `maxKeepAliveRequests` **默认 100**，超过就服务端主动关闭——Nginx 侧表现为随机 `upstream prematurely closed connection`（502）。生产必调：

```text
# server.xml / Spring Boot application.yml
maxConnections=8192  keepAliveTimeout=65000  maxKeepAliveRequests=-1(或≥1000)
```

参数要**分层对齐**：Tomcat `keepAliveTimeout`（默认 60s）应与 Nginx `keepalive_timeout`（upstream 空闲连接回收，默认 60s）错开——**后端关连接的时间要晚于 Nginx**，否则 Nginx 复用一条后端刚 RST 的连接就踩 502（竞态）。`keepalive 32` 是**每 worker** 的缓存数（不是到每台后端 32），worker=8 时理论上限 256 条空闲连接；突发过池后会新建，稳态复用率看监控里 TIME_WAIT 数量验证。

> **设计考量**：开了 upstream keepalive 后，`proxy_next_upstream` 的"连接复用失败重试"会掩盖部分后端问题——第一次用死连接失败、Nginx 自动换新连接重试成功，access_log 里 `upstream_response_time` 出现两个值但用户无感。看到大量这种"隐形重试"日志，应该去查后端连接回收参数而不是继续调 Nginx。

# 7. 错误日志监控

```bash
# 监控错误率
tail -f error.log | grep -E "error|crit|alert" | wc -l

# 配合 filebeat 采集到 ELK
```

error.log 的级别（debug/info/notice/warn/error/crit/alert/emerg）默认 `warn`，**生产不建议 info 以下**。按级别路由采集是最省事的治理：`error` 及以上的行直接进告警队列，`warn`（upstream 慢、SSL 握手失败）做趋势看板。

> **注意**：error.log 出现 `worker_connections are not enough`、`Too many open files` 属于**容量类告警**，出现即代表正在拒服务，优先级等同 5xx 突增；`upstream timed out` 与 `connect() failed` 分别对应后端慢（504 方向）与后端挂（502 方向），定位路径完全不同。

# 8. Nginx Amplify（官方监控 SaaS）

Nginx Amplify 提供图形化监控，包括：
- 请求数 / 响应时间 / 状态码分布
- CPU / 内存 / 磁盘 / 网络
- Nginx 配置分析报告
- 异常告警

```bash
curl -sSL https://nginx.org/amplify | sh
```

一行脚本装 agent 即得开箱看板，适合无自研监控栈的中小团队；但数据需上传官方云端，合规敏感环境不适用。

# 9. 监控指标速查

| 关键指标 | 获取方式 | 告警阈值参考 |
|----------|----------|-------------|
| 502/504 错误率 | access_log 统计 | > 1% |
| 活跃连接数 | stub_status | > worker_connections × 80% |
| 上游响应时间 | `$upstream_response_time` | P99 > 1s |
| 请求处理时间 | `$request_time` | P99 > 2s |
| Nginx 进程 | `ps -ef \| grep nginx` | 进程不存在 |
| 磁盘使用率 | `df -h` | > 80%（日志/缓存） |

补充两条经验值：`requests/handled`（每连接平均请求数）**< 1.5 说明 keep-alive 基本失效**；危险信号是 Reading/Writing 单边持续高企（有人拖连接），Waiting 攀升通常是客户端数量自然增长。

# 10. 小结

| 层次 | 手段 | 回答的问题 |
|------|------|-----------|
| 进程/连接层 | stub_status + exporter | 容量够不够、keep-alive 生效没有 |
| 请求层 | perf 格式 access_log（request_time vs upstream_response_time） | 慢在链路哪一段、哪台后端 |
| 后端健康 | 被动 max_fails / 主动 health_check / 平台层探针 | 故障机何时摘除、何时回池 |
| 连接复用 | upstream keepalive + Tomcat 参数对齐 | 502 "prematurely closed" 与端口耗尽 |

- **差分思维**：stub_status 全是累计值，rate 之后才是指标；重试会让 upstream_* 变量变逗号多值，日志字段建模要预留。
- **被动检查的本质**是"拿真实请求当探针"，代价是第一批用户；要主动就得 Plus/Tengine/OpenResty 或干脆交给 K8s/Nacos 平台层。
- **健康检查端点要"浅"**：只答"能否接流量"，别把依赖抖动放大成全实例摘除。
- **Nginx/Tomcat 长连接参数成对调**：后端空闲超时要比 Nginx 的晚关，否则偶发 502 找不到原因。

下一篇（18_high_availability）讲 keepalived + VIP 与双机热备，把单点入口从架构图上消掉。
