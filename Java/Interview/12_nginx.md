本篇覆盖 Nginx 的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：定位与高性能架构、路由与代理配置、负载均衡与流量治理、HTTPS 与安全、性能调优与可观测。入口层与会话/跨域的上下游配合见 5_tomcat.md、11_spring_security.md，网关层流量治理见 19_spring_cloud_gateway.md。

# 1. 定位与高性能架构

## 1.1 定位与进程模型

### 1.1.1 Nginx 是什么？能干什么？正向代理和反向代理怎么区分？

**定位**：Apache 开源的高性能 **HTTP 反向代理 / 负载均衡器 / Web 服务器**，附带四层（stream 模块）TCP/UDP 代理与邮件代理（IMAP/POP3/SMTP）能力——现代部署里它站在最入口：终结 TLS、静态直出、按路径分流、负载均衡、限流防护，把动态请求转给后端应用（Java 侧通常是 Tomcat/Spring Boot，5_tomcat.md）。

**正向代理 vs 反向代理**（一句话辨位：**代理谁，谁就是"看不见"的那方**）：

| 维度 | 正向代理 | 反向代理 |
|------|---------|---------|
| 部署位置 | 客户端侧（代理"我"出去） | 服务端侧（代理"它"对外） |
| 服务端感知 | 知道真实客户端？取决于配置；知道自己在跟代理说话 | 只认识代理（真实 IP 靠转发头，4.2.2/2.2.1） |
| 典型场景 | 公司出口网关、跨境加速、缓存上网内容 | 负载均衡、SSL 终结、隐藏后端拓扑、动静分离 |
| 谁配置 | 客户端主动指向 | DNS 直接把域名解析到它，客户端无感 |

Nginx 两者都能做（`resolver` + `proxy_pass $uri` 可当正向代理），但面试与生产的重心都在反向代理。

**四层 vs 七层**：stream 模块只看 IP/端口（四层，可代理 MySQL/Redis 流量）；http 模块解析 HTTP 语义（能按 URL/Header 路由、缓存、改头）——"Nginx 是四层还是七层负载"的准确答法：**都能，默认身份是七层**。

### 1.1.2 Nginx 的进程模型是怎样的？Master 和 Worker 各干什么？reload 为什么不中断服务？

**模型**：1 Master + N Worker **多进程**，Worker 之间**对等、各跑各的事件循环**，进程内**单线程**（一个 Worker 管成千上万连接全靠事件驱动，1.1.3）。

- **Master**：不处理请求——读/校验配置（`nginx -t`）、绑定监听端口、按 `worker_processes`（auto=核数）孵化并监护 Worker（挂了自动拉起）、接收信号（SIGHUP reload / SIGUSR1 重开日志 / SIGTERM 快速退出 / SIGUSR2 热升级二进制）；
- **Worker**：epoll 事件循环里完成读请求 → 路由 → 转发/直出 → 写响应全部工作；彼此隔离——一个 Worker 崩溃不影响其他（多进程比多线程"故障域小"的设计红利）；同机多 Worker 不需要负载均衡它们（OS 调度 + 监听共享）。

**reload 热加载流程**（追问高频）：Master 收到 SIGHUP → 校验新配置 → **用新配置孵化一批新 Worker** → 给旧 Worker 发"退休"指令：不再 accept 新连接、处理完手头请求（或到 `worker_shutdown_timeout` 强制）后退出——全程新旧交替、服务不断。版本升级则用 SIGUSR2 双 Master 并存的**在线二进制升级**（旧 Master 驻留，验证新二进制后逐个放走）。

**配套细节**：`worker_cpu_affinity auto` 把 Worker 绑核（减少上下文切换与缓存失效）；`multi_accept on` 一次事件尽可能收完所有新连接；历史上多 Worker 争抢 accept 有惊群/accept_mutex 话题，现代内核（epoll Exclusive wake-up）与 `reuseport`（每 Worker 独立监听队列）已基本消解——被问到"连接怎么分给各 Worker"答这两层即可。

### 1.1.3 为什么 Nginx 能扛高并发？和 Apache 的差距在哪？

**结论**：根因是 **I/O 模型代差**——"每连接一线程"与"事件驱动 + 非阻塞"的对比，不是语言或调参差距。

**Apache 传统模型**：prefork（每连接一进程）/worker 模式（每连接一线程）——同步阻塞：连接 1 万 = 常驻 1 万线程/进程，每线程 8MB 栈起步、上下文切换与调度成本随并发平方级恶化，"C10K"死在这。

**Nginx 模型**：Worker 单线程事件循环——epoll 把"谁就绪了"变成**回调通知**（不是轮询）：accept、read、write、上游响应到达全是注册事件；连接在内存里只是一条几十字节的会话结构 + 收发缓冲，**并发数与线程数解耦**，单机 10 万长连接（keep-alive）是工程常态。

**加分项（把"为什么快"讲成体系）**：
1. 异步非阻塞 + epoll（LT 模式为主，高版本 accept 用 ET 减少 wakeup；对比 select/poll 的 FD_SET 轮询：epoll 红黑树 + 就绪链表，O(ready) 不是 O(N)）；
2. 内存治理：池化分配（每请求一 pool 整块回收，几乎无碎片）、连接对象复用缓存；
3. 静态路径短：sendfile 零拷贝直达内核（5.1.1）、`open_file_cache` 缓存 fd 与元数据；
4. 配置简单无锁：一进程一事件循环天然免多线程锁竞争。

**边界要说**（"Nginx 一定比 Apache 快？"）：动态内容两边都不是终点（Apache mod_php/CGI 模型时代已远）；Nginx 的快在**静态与代理**场景，后端应用才是动态瓶颈；Apache 的 .htaccess 分布式配置、模块成熟度在部分托管场景仍有拥趸——答"分场景"比背"吊打"稳。

## 1.2 连接容量模型

### 1.2.1 Nginx 最大连接数怎么算？worker_connections 设 65535 就真能到吗？

**账面公式**：`最大并发 ≈ worker_processes × worker_connections`；**反向代理场景除以 2**——一个客户端请求在 Nginx 里占用**两条连接**（客户端↔Nginx、Nginx↔upstream，尤其未启用 upstream keepalive 时每条都要新建，3.1.1 关联）：4 核 × 65535 ÷ 2 ≈ 13 万。

**账外四道天花板**（"设了 65535 却到不了"的排查清单，这题的深度全在这里）：
1. **文件描述符**：连接=fd——OS `ulimit -n` 与 `worker_rlimit_nofile`（Nginx 自己抬高 Worker 上限，不配则继承启动 shell 的 1024，经典瓶颈；"Too many open files" 直接对应）；
2. **内存**：每连接缓冲（`proxy_buffers` 等）+ 应用层对象，10 万连接吃掉数 GB 很正常；
3. **端口与连接表**：作为客户端连 upstream，出向 ephemeral port 有 6 万级上限（HTTP/1.0 式短连接打满就撞）——复用 keepalive 是解法；nf_conntrack 表满丢包也是隐形墙；
4. **带宽与后端**：入口容量再大，后端 RT 一慢，连接全在 proxy_read 等待里堆积——入口容量永远和后端线程池一起算账（5_tomcat.md 4.1.2 同一道题的两面）。

`worker_connections` 同时包含客户端与 upstream 连接，不是"纯客户端数"；HTTPS 还要算握手 CPU——容量评估四件套（fd/内存/端口/后端）答齐即高分。

# 2. 路由与代理配置

## 2.1 location 与请求匹配

### 2.1.1 location 有哪几种匹配？优先级如何？

**五类修饰**：

```nginx
location = /api/login      { }   # 精确匹配，命中即定
location ^~ /static/       { }   # 前缀命中后"短路"，跳过正则
location ~ \.php$          { }   # 正则（区分大小写），按书写顺序抢
location ~* \.(jpg|png)$   { }   # 正则（不区分大小写）
location /app/             { }   # 普通前缀，取最长者
```

**真实匹配算法**（背优先级之前先懂过程）：先扫**全部前缀型**记下"最长匹配"——若最长者是 `^~` 直接用它（正则不跑）；否则拿 URI 按**配置文件书写顺序**逐条测正则，**第一条命中的正则赢**；正则全落空 → 回到那个"最长前缀"。`=` 是特例：存在即最快命中。

**优先级口诀**（= > ^~ > ~ / ~* > 普通前缀）的三个易错：
1. 多个正则**只看书写顺序**、不看"更长更具体"——把宽泛正则写在前面会截胡后面的精确正则；
2. `^~` 的价值：高频前缀（/static/）用 `^~` 免跑正则链，省匹配开销；
3. location 匹配的是 **URI 路径**，不含查询串；大小写敏感还受 `server` 编码归一（`$uri` 已解码归一，`$request_uri` 保持原样）影响——正则里拿哪个变量，行为差一截。

### 2.1.2 rewrite 的 last、break、redirect、permanent 有什么区别？

**先分组**：`last/break` = **服务端重写**（URI 变了、浏览器地址栏不变）；`redirect/permanent` = **客户端重定向**（302/301，地址栏变）。

| flag | 行为 |
|------|------|
| `last` | 重写立即生效，**用新 URI 重新跑一轮 server 级 location 匹配**——适合"server 级改写、交给别处的 location 处理"；改写后还落回同一 rewrite 且结果相同 → 循环，500 |
| `break` | 重写后停止本 location 的 rewrite 链，**不再重新匹配 location**，就地处理——适合"location 内改路径然后 proxy_pass/static 直出" |
| `redirect` | 返回 302（临时），浏览器重访仍走原 URL——SEO 不迁权重 |
| `permanent` | 返回 301（永久），浏览器/搜索引擎缓存新地址——站点迁移用，**上了 301 反悔极难**（客户端强缓存），上线灰度期常用 302 试探 |

**last/break 的经典对照**（面试官最爱）：

```nginx
location /old/ { rewrite ^/old/(.*)$ /new/$1 last;  }   # 跳出去找 /new/ 的 location
location /img/ { rewrite ^/img/(.*)$ /data/$1 break; proxy_pass http://backend; } # 改完就地转发
```

补充语义差异：三者都在**rewrite 阶段**执行，先于 location 内容处理；`if` + rewrite 是"最熟悉的陌生坑"——`if` 块里的指令优先级特殊（"if is evil"），复杂条件改用 map（2.2.1/3.3.1 的 map 就是为绕开它而生）。

### 2.1.3 root 和 alias 有什么区别？（高频配置坑）

同一段 URI，两种映射公式：

```nginx
location /images/ { root /data/www; }    # 拼全路径：/data/www + /images/ + 文件名
location /images/ { alias /data/www/; }  # 换前缀：  /data/www/ + 去掉 /images/ 的剩余
```

**root** 把 location 前缀**也**拼进磁盘路径（适合"目录结构与 URI 天然对齐"的静态站）；**alias** 把 location 前段**替换**掉（适合部署在任意目录的独立资源，如前端产物）。

**三个坑**：① alias 的路径**结尾斜杠要与 location 对齐**（location 带 /、alias 也必须带 /），否则文件名粘连出不来（404 大户）；② alias 不支持带捕获组的正则 location（官方限制）；③ 权限与 `autoindex` 开合只在 root 场景常见——目录列不出、403 先查这三处。`try_files` 里两类混用也按各自公式解析（2.1.4）。

### 2.1.4 前端 SPA（Vue/React 打包产物）如何部署到 Nginx？try_files 是干什么的？

**部署三件套**：静态产物目录 + history 路由回退 + 资源缓存策略：

```nginx
server {
    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;    # 核心行
        location ~* \.(js|css|png|woff2)$ { expires 30d; add_header Cache-Control "public, immutable"; }
    }
    location /api/ { proxy_pass http://backend; }   # 同源代理，顺带消灭跨域（4.2.2）
}
```

**try_files 语义**：按序探测候选（`$uri` 文件 → `$uri/` 目录），**全落空才走最后一项**（内部重定向到 index.html）——SPA 的 history 路由里 `/user/123` 磁盘上不存在，回退 index.html 让前端 Router 接管，否则刷新 404（hash 模式 #/ 不需要这条）。

**配套细节**：index.html 本身要 **no-cache**（`location = /index.html { add_header Cache-Control "no-cache"; }`，否则发版后用户拿到旧入口——"JS 带 hash 长缓存 + html 不缓存"是前端发布的标准组合，hash 文件名让 immutable 成立）；4.2.1 的 `expires` 写进静态 location；`try_files` 末项也可以不是回退而是 404/后端接口，通用"探测+兜底"模式。

## 2.2 反向代理与动静分离

### 2.2.1 proxy_pass 如何转发请求？为什么必须配 proxy_set_header？

**基本形态与那个著名的斜杠差异**：

```nginx
location /api/ { proxy_pass http://backend; }          # 原样透传：/api/user → backend/api/user
location /api/ { proxy_pass http://backend/; }         # 带 URI 前缀：替换 location 段 → backend/user
location /api/ { proxy_pass http://backend/v1/; }      # /api/user → backend/v1/user
```

**proxy_pass 不带 URI（无路径）→ 全 URI 透传；带 URI → 把 location 前缀替换成该 URI**——"后端 404 说路径多了一段"的根因。动态变量场景（`proxy_pass http://$upstream_host;`）行为又不同（URI 需重写补全），属进阶追问。

**必配头清单**（"为什么后端拿不到真实信息"逐条解释）：

```nginx
proxy_set_header Host              $host;              # 默认会送 upstream 名——虚拟主机/Tomcat Host 匹配全乱（5_tomcat.md 1.1.3）
proxy_set_header X-Real-IP         $remote_addr;       # 真实客户端 IP
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;  # 追加式 IP 链（多级代理串起来）
proxy_set_header X-Forwarded-Proto $scheme;            # 终结 TLS 后告诉后端"外面是 https"（回调地址/安全重定向用）
```

不配这些的后果：后端 access log 全是 127.0.0.1、限流按代理 IP 全打一个桶、`request.getRemoteAddr()` 失真、HTTPS 跳回死循环。后端接收侧：Tomcat `remoteip valve`/Boot `forward-headers-strategy`（10_springboot.md 4.1.1、5_tomcat.md 4.2.2）——**两头要一起配**，只配一边等于没配。

**响应侧两个补丁**：`proxy_redirect` 修正后端 302 Location 里的内网地址；HTTP/1.1 化 + 清 Connection 头才能和 upstream keepalive 配合（3.1.1）。

### 2.2.2 动静分离怎么用 Nginx 落地？各侧的活怎么分？

**动机**：静态文件（JS/CSS/图/视频）根本不该劳应用容器——Nginx 直出走 sendfile 零拷贝（5.1.1），Tomcat 线程全留给动态业务（容量立刻松一档，5_tomcat.md 4.1.2 的"入口分流"红利）。

**落地模板**：

```nginx
location /static/  { root /data/www; expires 30d; add_header Cache-Control "public, immutable"; }  # 带内容 hash 的产物
location /upload/  { alias /data/upload/; }                     # 用户上传：本地盘或对象存储回源（25_minio.md）
location /api/     { proxy_pass http://backend; }               # 动态转应用
```

**三层缓存职责**：浏览器（expires/max-age + hash 文件名 immutable）→ CDN（静态上云加速，回源 Nginx）→ Nginx 本地盘直读。**发布纪律**：改文件名 hash、只缓存不可变资源、HTML 短缓存——2.1.4 的组合在动静分离下同样成立。

**易错与追问**：① 动态接口别图省事 `proxy_cache` 缓存（失效一致性难，见 3.2.2）；② 大文件下载限速用 `limit_rate`（会话级带宽封顶，防单用户吃光出口）；③ "上传走哪"——入口 `client_max_body_size` 先放开（5.2.2 的 413），大文件直传对象存储更好（25_minio.md）。

### 2.2.3 proxy_*_timeout 和 upstream keepalive 分别怎么配？配错会出什么症状？

**代理超时三件套**：`proxy_connect_timeout`（与后端建 TCP 的等待，默认 60s）、`proxy_read_timeout`（建连后**两次读到的间隔**——不是后端总耗时！后端 25s 才吐出第一个字节而 read 设 10s → 504，而流式接口只要持续有数据就不断）、`proxy_send_timeout`（向上游写请求体）。keep-alive 侧：客户端空闲连接 `keepalive_timeout 65s`。

**症状对照**（面试把超时和报错配对就赢）：后端线程池满排队 → 建连快但迟迟不吐数据 → `read timeout` 504；后端进程死/未启动 → `connect refused` → **502**（5.2.2）。**与后端超时链对齐**：网关/负载均衡超时必须 > 应用服务端超时 > 业务内部 RPC 超时，否则"上游先掐线、下游还在跑"——重复执行与半响应事故（5_tomcat.md 4.1.3 的 keepAlive 对齐是同一原则的另一半）。

**upstream 长连接**（默认 Nginx→后端是**短连接**——每请求新建 TCP，高并发下后端 TIME_WAIT 爆炸、TLS 握手重做，Tomcat 侧 accept 压力陡增）：

```nginx
upstream backend { server 10.0.0.1:8080; keepalive 64; }        # 每 Worker 驻留的空闲连接池
location /api/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;                                      # 三件套缺一不可
    proxy_set_header Connection "";                              # 清掉默认的 close
}
```

`keepalive N`（空闲池上限）+ `keepalive_requests`（单连接复用请求数）+ `keepalive_timeout`（池内空闲保活）共同决定复用率；后端（Tomcat 默认 keep-alive 100 请求，5_tomcat.md 4.1.3）要大于等于前端池配置，否则连接被后端先关、前端拿到死连接报偶发 502——**这条链路对齐是 502 玄学第一排查点**。

# 3. 负载均衡与流量治理

## 3.1 算法与健康检查

### 3.1.1 upstream 负载均衡算法有哪些？各适用什么场景？

**开源版开箱四种 + 场景对照**：

| 算法 | 配置 | 机制 | 适用 |
|------|------|------|------|
| 轮询（默认） | 不写即轮询 | 逐请求依次循环，可配 `weight=3` 加权（性能不齐时按权重分） | 无状态服务、机器同构 |
| `ip_hash` | 指令一行 | 客户端 IPv4 **前 1~3 段**做哈希（不是完整 IP）→ 同一网段用户固定落同一后端 | 会话粘滞保底（3.1.3）；注意 IPv6 按 /64 整段，粒度粗 |
| `least_conn` | 指令一行 | 新请求派给"**活跃连接数/权重**"最小的节点 | 请求耗时参差的长任务（报表/上传），轮询会把慢节点越堆越深 |
| `hash $var consistent` / `random least_conn` | hash 指令 | 一致性哈希环（key 可变：`$uri`、自定义头）；random 带两次取优 | 分片缓存、按租户路由等定制亲和 |

**开源 vs Plus 边界（追问必考）**：`least_time`（按平均响应时间+在途综合打分）**只有 Nginx Plus（商业版）有**；第三方模块（nginx_upstream_fair、cookie-hash）能补齐一部分但非官方——简历写"熟悉 least_time"会被拆穿口径。一致性哈希在开源 hash 指令里支持，但**节点增减仍会重排部分 key**（虚拟节点缓解），与 22_sharding_sphere 数据分片的重排问题是同一道题。

**权重与健康联动**：`max_fails=3 fail_timeout=30s` 参与选节点判定——被标失败的节点在 fail_timeout 窗口内**从算法候选里摘除**（3.1.2），算法与被动健康检查是叠加关系不是二选一。

### 3.1.2 Nginx 如何判断后端健康？主动和被动健康检查的区别？

**被动（开源内置，`server` 指令参数）**：平时不探测，**真实流量出错才记账**——连续 `max_fails` 次失败（超时、连接被拒等，HTTP 状态码默认不算失败，用 `http_502|http_503=...` 之类需 Plus；开源可用 `proxy_next_upstream` 变相控制）→ 该节点标记失败 `fail_timeout` 时长，窗口内不派新请求，窗口结束恢复试探。特点：**零额外流量、判定时机晚**（第一批用户已经当小白鼠）。

**主动（Nginx Plus 的 `health_check` 指令 / 开源第三方 `nginx_upstream_check_module`）**：独立定时器按配置间隔发**专用探测请求**（`GET /health` 期望 200，可配升降阈值）——不等业务流量，后端一挂秒级摘除。自建体系要主动检查：要么上 Plus/云 ALB，要么编译带 check 模块的 Tengine/OpenResty 发行版（5.3.1）。

**语义配套两条**：
1. **恢复也是渐进的**——被摘节点重新参选后流量瞬间全量打回（刚启动的 JVM 还没 JIT 预热）容易二次雪崩；配合应用侧预热（10_springboot.md 优雅停机同理）或多级发布；
2. **`proxy_next_upstream`**：请求失败**自动换下一台重试**（默认 error/timeout 触发；`non_idempotent` 才允许重试 POST——防止重复下单，这个默认值是这题的安全考点）。重试只对"还没发出 body/可安全重放"的请求成立，配超时链（2.2.3）一起算总耗时。

**应用侧接法**：后端提供真 `/actuator/health`（探到 DB/Redis 依赖），探测频率与超时独立于业务流量；网关侧健康视图见 19_spring_cloud_gateway.md——**多层代理各自摘除、各自恢复**，链路排障时要分清是谁把节点踢了（5.2.2）。

### 3.1.3 负载均衡下 Session 会话如何保持？

三条路线按工程优先级（会话本体机制见 5_tomcat.md 3.1/3.2）：

1. **无状态化（正解）**：token 自包含（JWT，11_spring_security.md 4.1）或集中会话（Spring Session + Redis，27_redis.md）——会话与节点解耦后**任何算法都能用**，扩缩容发布无感；
2. **ip_hash / 一致性哈希（粘滞）**：同 IP 恒定同节点——代价：后端节点增减整段重排、NAT 后大公司全员挤一台、节点故障会话清零；只作存量系统的过渡；
3. **会话复制**：Tomcat 集群 DeltaManager（5_tomcat.md 3.1.3）——复制流量大、规模上不去，现代新项目不用。

**进阶追问：cookie 粘滞**。Nginx 开源没有原生 sticky 指令（Plus 的 `sticky cookie` 或第三方模块）；土办法：`map $http_cookie $backend { ... }` 按 JSESSIONID 里的 jvmRoute 后缀路由（5_tomcat.md 3.1.1 的 `id.node1` 就是为此设计），或后端 Set-Cookie 写 `route=节点名` + map 查表。答案收口仍是"**为什么不用它**——粘滞把架构复杂度从存储挪回了调度，不如无状态一劳永逸"。

## 3.2 限流与缓存

### 3.2.1 怎么做限流？limit_req 的 rate、burst、nodelay 各是什么？

**两把刀**：`limit_req_zone` 按 key（IP、URI、header）限**请求速率**；`limit_conn_zone` 限**并发连接数**（长连接防护）。

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;   # 10MB ≈ 16 万 IP 的状态表
server { location /api/ { limit_req zone=api burst=50 nodelay; } }
```

**令牌桶模型拆解**（三参数讲成一句话才及格）：rate 是令牌发放速率；burst 是桶容量——超出 rate 的突发请求**入队等令牌**；`nodelay` 改变语义：**立即放行**桶里已有的额度（不排队），超出部分才 503；不加 nodelay 则 burst 内请求按 rate 匀速"延迟放行"（削峰但拉长 RT）。`limit_req_status 429`（默认 503，改成语义正确的 429 是细节加分项）；超限**响应体兜底** `error_page 429 = @throttled` 自定义 JSON。

**多级粒度**：key 换成 `$host$uri`、用户 ID（`$http_x_user_id`）做**按租户限流**；全局阈值与单 IP 阈值叠两条 zone（总量防雪崩、个体防恶意）；`$binary_remote_addr` 比 `$remote_addr` 省内存（4/16 字节 vs 字符串）。

**边界与分工**（这题的深度落点）：Nginx 限流是**单实例视角**——多节点部署各自计数，全局实际阈值 = N × rate；要全局精确限流下沉到网关/中间件（Sentinel 集群限流，20_sentinel.md；Redis+Lua 方案，27_redis.md）。**入口粗限 + 应用细熔断**是常态组合，Nginx 挡的是爬虫、撞库、瞬时洪峰这类"无差别流量"。

### 3.2.2 proxy_cache 如何缓存后端响应？缓存 key、过期、失效怎么设计？

**最小闭环**：

```nginx
proxy_cache_path /data/nginx_cache levels=1:2 keys_zone=api:100m max_size=10g inactive=60m use_temp_path=off;
location /api/products/ {
    proxy_pass http://backend;
    proxy_cache api;
    proxy_cache_key "$scheme$host$uri$is_args$args";     # 默认 key 即此式，Vary 头参与
    proxy_cache_valid 200 5m;  404 1m;                   # 按状态码设 TTL
    proxy_cache_use_stale error timeout updating http_502 http_503;  # 后端挂了继续供旧缓存 = 降级
    add_header X-Cache $upstream_cache_status;           # HIT/MISS/STALE/EXPIRED 观测
    proxy_cache_bypass $http_cache_control;              # 尊重客户端 no-cache
}
```

**机制四要点**：① `keys_zone` 是共享内存里的索引（key→元数据），实体文件在 `proxy_cache_path` 目录（levels=1:2 散列成两级目录防单目录爆量）；② **`inactive` 不是 TTL**——"多久没被访问就清走"，`proxy_cache_valid` 才是响应有效期；两者都过才删；③ **只缓存安全方法**（GET/HEAD 默认；POST 需 `proxy_cache_methods` 强行 + 想清楚副作用）；④ 响应含 `Set-Cookie`/`Cache-Control: no-store/private` 默认**不缓存**——后端忘了关头就整层失效（排障先看 `$upstream_cache_status`）。

**进阶三件**：`proxy_cache_lock on` 防缓存击穿（同 key 并发只放一个请求去后端，其余等它的结果——对应 27_redis.md 互斥方案）；`proxy_cache_background_update` 先供旧同时后台刷新；**主动失效**靠 `X-Accel-Redirect` 内部接口（应用发 purge 请求走该 location 删缓存）——**没有内建全局失效总线的现实要讲出来**：所以动态业务慎用 Nginx 缓存，缓存该放静态（2.2.2）与 CDN；真正"接口级缓存"的正解在应用侧（Spring Cache + Redis，见 10_springboot.md、27_redis.md），Nginx 这层适合"读多写少 + 容忍分钟级过期"的只读接口。

### 3.2.3 如何做灰度发布/按流量比例切分？

**三把刀组合**：

1. **split_clients 按权重分流**（哈希桶比例切分，随机且同人稳定）：

```nginx
split_clients "${remote_addr}AAA" $variant {
    5%      backend_v2;     # 5% 流量进新版本
    *       backend_v1;
}
location / { proxy_pass http://$variant; }   # 变量 upstream 需配合 resolver/无 URI 形式
```

2. **map 按特征分流**：版本头/白名单用户/地域——`map $http_x_gray { default backend_v1; "on" backend_v2; }`，内部员工带 `X-Gray: on` 全量先试；
3. **加权 upstream 常驻**：`server 10.0.0.1:8080 weight=95; server 10.0.0.2:8080 weight=5;` 新节点长期 5% 承接观察，扩权到 100 再摘旧——**蓝绿=整组切换、灰度=比例渐进**，upstream 分组（`@canary` 多 location 复用 include）即可管理。

**闭环要点**（与纯配置题拉开差距）：分流维度要**可追踪**（响应头打 `$variant` 版本号，日志按变量记录，5.2.1）；比例调整与**会话亲和**冲突时优先用用户 ID 而非 IP 做 hash key（登录用户跨页不一致=事故）；发布系统的自动放量 + 指标门禁（错误率/RT 超阈值自动回滚）在平台层实现——Nginx 只提供切流原语，网关层等价能力见 19_spring_cloud_gateway.md，K8s 场景 Ingress/Gateway API 同理（32_kubernetes.md）。

# 4. HTTPS 与安全

## 4.1 TLS 终结

### 4.1.1 HTTPS 握手过程？Nginx 侧如何配置与优化 TLS？

**握手主线**（TLS1.2 → 点出 1.3 差异即可）：ClientHello（支持的版本/密码套件/SNI 域名/随机数）→ ServerHello（选定套件 + 证书链 + key_share）→ 客户端**验链**（信任锚、有效期、域名匹配、吊销检查 OCSP）→ 密钥协商（ECDHE 临时密钥 = **前向保密**，私钥日后泄露也解不开历史流量——RSA 密钥交换的缺陷，套件清单必须 ECDHE 开头）→ Finished 互验 → 对称加密传输。**TLS 1.3 剪枝**：1-RTT 完成（会话恢复 0-RTT）、废弃 RSA 密钥交换与弱套件——2026 口径只应开 TLS1.2/1.3。

**Nginx 配置基线**：

```nginx
server {
    listen 443 ssl;             # http2 直接并入 listen（旧 http2 on 独立指令，1.25+）
    ssl_certificate     /etc/nginx/certs/example.com.pem;      # 全链证书（含中间证书，漏装=部分安卓/老客户端验证失败）
    ssl_certificate_key /etc/nginx/certs/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;  # 现代套件优先
    ssl_prefer_server_ciphers on;      # TLS1.3 下意义不大，1.2 保留
    ssl_session_cache   shared:SSL:10m;   # 会话复用：命中走 PSK/Ticket 免全握手
    ssl_session_timeout 1d;
    ssl_stapling on;  ssl_stapling_verify on;   # OCSP 装订：替客户端省一次吊销查询外呼
}
server { listen 80; return 301 https://$host$request_uri; }    # http 强跳
```

**性能账**：握手 CPU（RSA-2048 签名最贵 → ECDSA 证书省一大半）、会话复用率（`$ssl_session_reused`）、**OCSP 装订**、**证书链精简**、HTTP/2 多路复用摊薄连接成本——"一域名一握手"是基准，多域名靠 **SNI**（一个 IP 一张证书按 ClientHello 里的域名选）：`server { listen 443 ssl; server_name a.com; ssl_certificate a.pem; }` 多 server 块即多证书，**默认 server 块放兜底证书**否则老客户端不带 SNI 直接握手失败。

**证书运维**：Let's Encrypt/acme.sh 自动续期（cron + reload）、过期监控告警——"证书过期事故"年年有，答出自动化即超出背题型。

## 4.2 安全加固与跨域

### 4.2.1 Nginx 生产安全加固清单？

**按暴露面从外到内**：

1. **信息面**：`server_tokens off;`（响应头与错误页不露版本——只是抬高扫描成本，配合 5_tomcat.md 后端侧同样收敛）；默认站点/example 页删除；
2. **传输面**：仅 TLS1.2+/强套件（4.1.1）；HSTS `add_header Strict-Transport-Security "max-age=31536000" always;`（**上生产用低 max-age 试跑**，一旦下发浏览器强制 https 反悔周期长，含子域更要 `includeSubDomains` 前核对全子域证书就绪）；HTTP→HTTPS 全局 301；
3. **内容面**：`X-Content-Type-Options: nosniff`、`X-Frame-Options: SAMEORIGIN`（或 CSP frame-ancestors）、`Referrer-Policy`、CSP 按需；`add_header` **继承陷阱**：子级一个 add_header 会清空父级全部——统一放 http 级或用 include 片段；
4. **访问面**：管理端点（/admin、状态页）`allow 内网; deny all;` 或直接不在公网 server 暴露；`limit_req` 撞库接口（登录/验证码）前置（3.2.1）；`client_max_body_size` 收敛（5.2.2）；
5. **请求面**：非法方法 `if ($request_method !~ ^(GET|HEAD|POST)$) { return 405; }`（或 default_type 收敛 OPTIONS）；路径穿越类特征 map 拦截；`$request_uri` 长度限制（`large_client_header_buffers` 是参数层兜底）；
6. **主机面**：以非 root 低权限运行（unprivileged nginx 镜像/专用账号）、配置文件权限与 `nginx -t` 纳入发布校验、access/error log 轮转 + 进 SIEM（5.2.1）。

**加分判断**：加固清单要和"业务可用"对表——上例 HSTS 误发全域、`deny all` 挡住回调 IP、405 卡住 PATCH 都是事故源；**灰度下发 + 错误率观测**再固化，答出这句说明真做过。

### 4.2.2 CORS 跨域在 Nginx 层怎么配？和后端配置怎么分工？

**协议基础**（简单请求 vs 预检的完整语义见 28_http.md，应用侧配置见 11_spring_security.md 5.1.2、8_spring_mvc.md 4.2.2）：跨源带自定义头/凭证的请求，浏览器先 `OPTIONS` 预检，按响应头三件套放行——`Access-Control-Allow-Origin`（**单值**，带凭证禁 `*`）、`Allow-Credentials`、`Allow-Methods/Headers`；`Vary: Origin` 必带否则 CDN/代理缓存把 A 源的响应喂给 B 源。

**Nginx 写法**：

```nginx
location /api/ {
    if ($http_origin ~* ^https://([a-z0-9-]+\.)?example\.com$) {      # 白名单正则，别写 *
        add_header Access-Control-Allow-Origin $http_origin always;   # always：4xx/5xx 也带，前端才能读错误体
        add_header Access-Control-Allow-Credentials true always;
        add_header Access-Control-Allow-Methods 'GET,POST,PUT,DELETE,OPTIONS' always;
        add_header Access-Control-Allow-Headers 'Authorization,Content-Type,X-XSRF-TOKEN' always;
        add_header Vary Origin always;
    }
    if ($request_method = OPTIONS) { return 204; }    # 预检在入口层短路，不进后端
    proxy_pass http://backend;
}
```

注意 `add_header` 不继承的陷阱（4.2.1 第 3 条）与 `if` 的维护性（复杂策略上 OpenResty，5.3.1）。

**分工判断（这题真正考点）**：**入口统一管 CORS 是更优解**——一处配置、后端全免、预检流量在入口消化；但**网关/Spring 侧配置时 Nginx 就不要再加**，两层都加响应头重复（浏览器按多值报错 "not a single origin"，排障现场很痛）。**同源部署**（2.1.4 的反代 `/api/` 同域名）从根上消灭 CORS——前端架构上这是首选，跨域配置是"多域名体系"的补救而非目标。

# 5. 性能调优与可观测

## 5.1 性能参数

### 5.1.1 worker_processes、sendfile、缓冲区这些性能参数怎么选？

**分四组记**（每组报"默认 + 什么时候动 + 动错的后果"）：

1. **进程与连接**：`worker_processes auto`（绑核数，超线程场景先压测再决定按逻辑核还是物理核）；`worker_cpu_affinity auto`；`worker_rlimit_nofile` 抬 fd 上限（与 1.2.1 第一道天花板对齐——**不改 ulimit 光调 worker_connections 是新手标配错误**）；`worker_shutdown_timeout` 给 reload 退休 Worker 兜底时限；
2. **传输路径**：`sendfile on`（静态文件零拷贝：不在用户态中转，磁盘→socket 内核直送）+ `tcp_nopush on`（攒满一个 MSS 再发，配合 sendfile 才有意义）+ `tcp_nodelay on`（禁用 Nagle，小包即时发，keep-alive 连接上默认生效）——三件套语义是"**大块推送攒包、交互小包即时**"，说反即外行；`output_buffers`/`sendfile_max_chunk`（单事件循环内 sendfile 搬运上限，防一个大文件下载饿死同 Worker 其他连接——限速敏感度高的场景调这个）；
3. **请求体与缓冲**：`client_header/body_timeout`（Slowloris 防御，比默认更短）、`client_body_buffer_size`（大 POST 超过它落临时文件）、`large_client_header_buffers`（URI/Cookie 超长 414/400）、`client_max_body_size`（默认 1m，上传 413 的开关，5.2.2）；**代理侧缓冲默认开**——`proxy_buffering on`：后端快速吐完、Nginx 按 `proxy_buffers`+`proxy_buffer_size` 慢慢喂慢客户端，**后端线程早释放**（这是 Nginx 保护 Tomcat 的隐形价值，5_tomcat.md 4.1.2 的"排队"压力被入口缓冲吸收一大块）；流式/SSE/WebSocket 必须 `proxy_buffering off`（36_websocket.md）；
4. **文件与缓存元数据**：`open_file_cache max=10000 inactive=20s`（缓存 fd/大小/查找失败，静态站命中率立竿见影）、`open_file_cache_valid/expire`。

**姿态收尾**：Nginx 默认值对普通站点已接近最优——**先观测（5.2.1 的 $request_time 拆解）再动参数**；"改 sendfile=off 排查 weird 静态问题"这类动作要能讲出因果（文件系统/网络驱动对零拷贝支持异常的老坑），不是玄学。

### 5.1.2 gzip 如何配置？还有哪些压缩/体积优化手段？

```nginx
gzip on;
gzip_comp_level 4;                     # 1~9：收益边际递减，4~5 是吞吐/体积折中
gzip_min_length 1k;                    # 小响应压缩不划算（还破坏分片小包）
gzip_types text/plain text/css application/json application/javascript image/svg+xml;  # text/html 恒含不必写
gzip_vary on;                          # Vary: Accept-Encoding——CDN 缓存分变体，漏配出"压缩内容当明文发"事故
gzip_proxied any;                      # 经代理来的请求也压（默认 many）
```

**机制与坑**：压缩在响应**输出阶段**做，CPU 与带宽的交换比；**已压缩内容（jpg/png/zip/woff2）重复压纯烧 CPU**——所以 `gzip_types` 别图省事 `*`；动态接口每次现压浪费（同一 JSON 千人访问压千次）→ **`gzip_static on`**：配合构建期预压 `.gz` 产物（前端打包器/CI 生成），Nginx 直接送现成文件，效果同 CDN 边缘压缩。**现代升级路线**：Brotli（`brotli_comp_level` 等，第三方模块编译或换发行版；浏览器 `Accept-Encoding: br` 协商）压缩率高 15~20% 且慢——**离线预压场景（gzip_static 同款思路）最搭**；HTTP/2/3 下压缩仍有效但**别再手工分文件小包装**（HPACK + 多路复用已解决队头）。

**体积优化全景**（压缩只是其一）：文本资源 minify（构建期）、图片现代格式 WebP/AVIF、JS 按路由分包、字体子集化 + `preconnect`/`dns-prefetch`、CDN 承接静态（2.2.2）——**Nginx 是执行位不是决策位，体积治理主战场在前端构建链**（33_cicd.md），答题带出这句显出架构视角。

## 5.2 日志与排障

### 5.2.1 access log 怎么定制？慢请求如何定位到具体环节？

**格式定制**：

```nginx
log_format main '$remote_addr - [$time_local] "$request" $status '
                '$request_time $upstream_response_time $upstream_status '
                '$upstream_addr $body_bytes_sent "$http_referer" "$http_user_agent" $http_x_request_id';
access_log /var/log/nginx/access.log main buffer=32k flush=1s;   # 缓冲写防 IO 放大；flush 保证排障实时性
error_log  /var/log/nginx/error.log warn;                         # 级别：debug 需 --with-debug 编译且流量场景禁用
```

**两个耗时变量的拆解是本题灵魂**：`$request_time` = 客户端视角总耗时（**含上传请求体时间 + Nginx 处理 + 等待响应 + 发响应给客户端的时间**）；`$upstream_response_time` = 后端视角（建连 + 后端处理 + 传输，多次重试/内部跳转时是逗号分隔的**多值与累加** `$upstream_response_time` 配合 `$upstream_addr` 一起看）。**症状对照**：`$request_time` 大而 `upstream` 小 → 慢在**客户端上传**（大 body 慢网络，接口被传文件）或响应下行；`upstream` 大 → 后端慢（去应用侧链路：Tomcat 线程排队 5_tomcat.md、慢 SQL 16_mysql.md、下游 RPC）；`upstream_status` 出现多值 → 触发过 next_upstream 重试（3.1.2）。

**日志工程**：按 `$upstream_addr`/`$status` 建索引进 ELK/Loki（24_elastic_search.md）；**`$request_id`** 内置变量透传给后端（`proxy_set_header X-Request-Id $request_id;`）做**全链路关联**——一次 5xx 从入口日志追到应用日志只靠这一个 ID，可观测性最低成本最高收益的一步（trace 体系见 15_spring_cloud.md）；`map $status $log_not_found` + `access_log ... if=$log_not_found` 过滤 404 扫描噪音；滚动策略 `logrotate` + `SIGUSR1`（1.1.2 reload 家族信号）。

### 5.2.2 502、504、413 分别什么原因？怎么排查？

**先给判定树**（"看到码先想谁产生的"）：**5xx 里 502/504 是 Nginx 生成的"对客户端的交代"，根因全在后端或链路；413 是 Nginx 自己拦的。**

- **502 Bad Gateway**（建连失败或拿到非法响应）：后端进程没起/端口不对（`connect() failed (111: Connection refused)`）；后端崩溃/重启窗口（发布期偶发 502——优雅停机没接住，10_springboot.md 4.1.2）；**keepalive 竞态**（后端先关连接、Nginx 复用池里拿到死连接——症状"偶发、重试就好"，对齐方案见 2.2.3 与 5_tomcat.md 4.1.3，502 玄学头号主角）；upstream 快速响应畸形（如 HTTP/1.0 半截）；`prematurely closed connection`（后端处理超时被自身或容器杀死）。**排查**：error_log 的 errno 文案直接分类（refused/timeout/reset 三种指向完全不同），再看 `$upstream_addr` 确认打到谁。
- **504 Gateway Timeout**（等后端超时）：`proxy_read_timeout` 内后端没吐首字节（3.2.1 线程池满排队、慢 SQL、下游 RPC 卡死、GC 停顿——**症状是成片的、RT 高但连接在**，与 502 的"根本没连上"区分）；慢接口要**单独 location 放宽超时**而不是全局调大（全局放大=故障时雪崩等死，快速失败+熔断归网关 20_sentinel.md）。
- **413 Request Entity Too Large**：`client_max_body_size`（默认 1m）拦截上传——放开前先想"该不该由 Nginx 收这个文件"（大文件直传对象存储，25_minio.md）；相关码还有 **414**（URI 过长，`large_client_header_buffers`）、**400**（Cookie 过大同理）。

**通用五步**：① error_log 定位 errno/消息；② access_log 看 `$upstream_status/$upstream_response_time` 还原链路；③ 复现请求带 `$request_id` 串后端日志；④ 直连后端端口绕过 Nginx 二分（后端本体 or 链路？）；⑤ 压测基线对比——"发布后才出现"查配置漂移（`nginx -t` + 版本 diff），"偶发"查 keepalive/超时对齐，"周期性"查后端 cron/GC。

## 5.3 扩展生态

### 5.3.1 什么是 OpenResty？和 Nginx、Kong 什么关系？

**OpenResty = Nginx + LuaJIT + 一堆 Lua 库**（ngx_lua 模块为核心）——把可编程性塞进 Nginx 事件循环：每个阶段（rewrite/access/content/header_filter/log）都能挂 Lua，且 **API 全程非阻塞**（coroutine 包装，`ngx.socket.tcp`/resty.mysql/resty.redis 在请求内异步取数据不卡 Worker）——性能姿态：C 模块的能力、脚本的开发速度，10 倍 QPS 场景下比"转发去后端查再回来"省一整跳。

**典型用例**（都能对上前面某题的"配置层做不到"）：动态限流（Redis 计数替代单实例 limit_req，3.2.1 的全局精确问题）、JWT 验签在入口做完（网关卸载，11_spring_security.md 4.1）、灰度按用户特征路由（map 表达不了的多表决策，3.2.3）、WAF（lua-resty-waf/ Coraza）、响应改写、`balancer_by_lua` 自定义负载均衡算法（least_time 的开源替代，3.1.1 边界）。

**Kong = OpenResty 之上的 API 网关产品**：插件模型（认证/限流/转换/日志）+ etcd/Postgres 存配置 + Dashboard——**"Nginx 是引擎、OpenResty 是脚本化的引擎、Kong 是整车"**；Java 技术栈里与 Spring Cloud Gateway 的分工常见口径：Kong/Ingress 管**南北向**（对外 API 治理），SCG 管**东西向/业务语义路由**（与注册中心联动，15_spring_cloud.md、19_spring_cloud_gateway.md）——两者可共存，答"选一个"反而露怯。

**同类生态**：Tengine（阿里，主动健康检查 check 模块、consistent_hash 等"官方没有"的补齐，3.1.2）、APISIX（etcd 动态化、插件热更新）、Caddy（自动 HTTPS 省心但生态浅）——"生产直接写 Lua 的时机"判断收尾：入口层**小逻辑**（验签/打标/路由决策）用 OpenResty 划算，**复杂策略**（多源数据、审计、流控联动）交给专业网关，别把 Nginx 写成没有事务与测试基建的微服务。