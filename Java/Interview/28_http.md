本篇覆盖 HTTP 协议的高频面试题，按"主题块 → 详细分块 → 题目 → 解答"组织。五个主题块：协议全景与报文、连接传输与性能、状态与安全、缓存与代理、工程实践与演进。Web 容器侧的协议实现见 5_tomcat.md，Nginx 的代理与缓存配置见 12_nginx.md，网关层的路由与超时见 19_spring_cloud_gateway.md，长连接应用层协议见 36_websocket.md。

# 1. 协议全景与报文

## 1.1 分层与版本

### 1.1.1 HTTP 在网络分层里处于什么位置？一次请求完整经过哪些环节？

**分层定位**：

```text
应用层：HTTP（语义：方法/状态码/头/缓存）、DNS、WebSocket、gRPC
  ↕（TLS 在应用层与传输层之间：HTTPS = HTTP over TLS）
传输层：TCP（可靠字节流、有序、重传）/ UDP（QUIC 的底座）
网络层：IP（寻址与路由）
链路层/物理层：以太网/WiFi/光
```

**一次 `GET https://api.example.com/orders/1` 的完整链路（按序背，排障时就是按段定位）**：

```text
① DNS 解析（浏览器缓存 → 系统缓存 → 递归 DNS → 权威 DNS）：域名 → IP
② TCP 三次握手（SYN → SYN+ACK → ACK）：建立可靠连接
③ TLS 握手（TLS 1.3 为 1-RTT，会话复用可 0-RTT）：协商密钥、验证证书、ALPN 协商 h2
④ 发送 HTTP 请求（方法/路径/头/体）——HTTP/2 起是二进制帧而非文本行
⑤ 服务端处理（负载均衡 → 网关 → 应用 → DB/缓存），返回响应
⑥ 连接复用或关闭（Keep-Alive / 多路复用）
⑦ 浏览器渲染（HTML 解析、子资源请求、缓存命中）
```

**HTTPS 在分层里的准确说法**：TLS 位于**应用层与传输层之间**（不是"应用层的一部分"）——它给上层提供加密与身份认证的"安全通道"，HTTP 本身不知道 TLS 的存在（除 `https://` 的默认端口 443）。

**为什么先讲这条链路**：后面所有性能题（延迟、连接复用、HTTP/2/3）都能归到"这条链的哪一段被优化"——DNS 优化（HTTPDNS/预解析）、握手优化（TLS 1.3/0-RTT/QUIC）、传输优化（多路复用/无队头阻塞）、内容优化（压缩/缓存/CDN）。**答题时先定位到段，再讲机制**，层次立刻清楚。

**易错**：把 HTTPS 说成"HTTP + SSL 加密"就完（要说 TLS 版本、证书验证、ALPN）；把 DNS 当作"一次性的"（TTL 缓存、多级缓存、CDN 调度都在 DNS 层）；忽略**建连成本**（TCP+TLS 两个往返，是首屏延迟的大头，HTTP/3 的 0-RTT 就是冲它去的）。

### 1.1.2 HTTP/1.0、1.1、2、3 的演进主线是什么？各自解决什么问题？

**一条主线：每一代都在解决"上一代的阻塞与开销"**：

| 版本 | 关键变化 | 解决的问题 | 遗留问题 |
|------|---------|-----------|---------|
| **HTTP/1.0** | 每请求一个连接（默认短连接） | 简单 | **每次请求 TCP 握手**，性能极差 |
| **HTTP/1.1**（1997） | **持久连接 Keep-Alive**（默认）、管道化（理论）、Host 头（虚拟主机）、chunked、缓存头完善 | 连接复用，减少握手 | **应用层队头阻塞**（响应必须按序）、头部重复传输、多连接开销 |
| **HTTP/2**（2015） | **二进制分帧 + 多路复用**、HPACK 头压缩、流优先级、Server Push、单连接 | HTTP 层队头阻塞、头部冗余、连接数 | **TCP 层队头阻塞**（丢包阻塞所有流）、连接迁移难 |
| **HTTP/3**（2022，基于 QUIC） | **UDP + QUIC**、内置 TLS 1.3、0-RTT、连接迁移、流级独立重传 | TCP 队头阻塞、建连延迟、网络切换断连 | UDP 被部分网络封锁（需回退）、生态与调试工具仍在完善 |

**版本协商机制**：HTTP/1.1 用 `Upgrade` 头（h2c，少见）；HTTPS 下用 **ALPN**（TLS 握手时协商 `h2`/`http/1.1`）；HTTP/3 通过 **Alt-Svc** 响应头或 DNS HTTPS 记录宣告（浏览器先走 h2，发现支持再升级 h3）。

**三个常见误区（面试纠偏位）**：
1. **HTTP/2 并没有"消灭队头阻塞"**——它消灭的是 **HTTP 层的按序响应阻塞**，但底层 TCP 仍是单一有序字节流，**一个包丢失，所有流一起等重传**（TCP HOL）；HTTP/3 才把"流"下沉到传输层，做到**丢包只影响该流**；
2. **HTTP/2 的"单连接"不是银弹**——单连接上的 TCP 拥塞窗口共享，丢包对全部流的影响被放大（"一条连接承载一切"的代价）；实践中有"多连接/连接预热"的调优讨论；
3. **Server Push 已基本退场**——Chrome 106 起移除支持，规范层面也不再推荐（推送的资源无法被客户端精确控制、缓存命中差），现代替代是 **103 Early Hints + preload**——报得出这条即时效加分。

**收口**："HTTP 的版本演进是**把阻塞点和开销逐层下移**：1.1 解连接、2 解应用层队头、3 解传输层队头与建连——每一代都把问题推给下一层，直到 QUIC 把'流'和'连接'都重做了一遍。"

## 1.2 报文与方法

### 1.2.1 请求/响应报文的结构是什么？哪些头是"必知"的？

**报文结构（HTTP/1.x 文本格式，HTTP/2 改为二进制帧但语义不变）**：

```text
请求：  请求行（方法 路径 版本）\r\n
        请求头（Host/Content-Type/Authorization/Cookie...）\r\n
        空行 \r\n
        请求体（GET 通常无；POST/PUT 有）

响应：  状态行（版本 状态码 原因短语）\r\n
        响应头（Content-Type/Content-Length/Cache-Control/Set-Cookie...）\r\n
        空行 \r\n
        响应体
```

**必知头部（按用途分组，答题报"解决什么问题"而非背名字）**：

| 组 | 头 | 作用 |
|----|----|------|
| 目标 | **Host**（1.1 必带） | 虚拟主机路由（同一 IP 多域名）；HTTP/2 用 `:authority` 伪头 |
| 内容 | `Content-Type` / `Content-Length` / `Content-Encoding` | 体是什么格式 / 多长 / 是否压缩（gzip/br） |
| 传输 | `Transfer-Encoding: chunked` / `Connection: keep-alive` | 分块传输 / 连接复用 |
| 缓存 | `Cache-Control` / `ETag` / `Last-Modified` / `Vary` | 缓存策略与再验证（4.1） |
| 条件 | `If-None-Match` / `If-Modified-Since` / `If-Range` | 协商缓存、断点续传 |
| 认证 | `Authorization` / `WWW-Authenticate` / `Cookie` | 凭证与挑战（3.1） |
| 代理 | `X-Forwarded-For` / `X-Real-IP` / `Forwarded` | 真实客户端 IP 链（4.2） |
| 内容协商 | `Accept` / `Accept-Encoding` / `Accept-Language` | 客户端能接受什么 |
| 安全 | `Strict-Transport-Security` / `Content-Security-Policy` / `Referrer-Policy` | 传输与内容安全（3.2） |
| 追踪 | `traceparent`（W3C）/ `X-Request-Id` | 分布式链路（15_spring_cloud.md 4.2.1） |

**三个高频细节**：① **`Content-Length` 与 `Transfer-Encoding: chunked` 互斥**（同时出现是请求走私的经典入口，3.2.3）；② **`Host` 是路由的关键**——网关/反代按 Host 分流，改写了 Host 可能 403（S3 签名）或路由到错的服务（12_nginx.md）；③ **`Vary` 决定缓存键的维度**（如 `Vary: Accept-Encoding`，压缩与非压缩版本分开缓存）——漏配会导致"给不支持 gzip 的客户端发了 gzip"。

**易错**：GET 带请求体（协议不禁止但语义反常，中间层可能丢弃）；响应头里写 `Content-Length` 又用 chunked（代理行为不一致）；`Content-Type` 缺失时服务端/客户端按"猜测"处理（JSON 接口必须显式 `application/json; charset=utf-8`）。

### 1.2.2 方法的语义与幂等性？GET/POST/PUT/PATCH/DELETE 怎么正确使用？

**两个正交属性（背这张表即可）**：

| 方法 | 安全（Safe，不改服务端状态） | 幂等（Idempotent，重复执行结果一致） | 语义 |
|------|------------------------------|--------------------------------------|------|
| GET | ✅ | ✅ | 获取资源（可缓存、可重试） |
| HEAD | ✅ | ✅ | 只要头（探活、文件大小/修改时间） |
| OPTIONS | ✅ | ✅ | 能力探测/CORS 预检 |
| PUT | ❌ | ✅ | **整体替换**（客户端提供完整表示，不存在则创建） |
| DELETE | ❌ | ✅ | 删除（重复删同一资源结果一致，404 与否是语义选择） |
| POST | ❌ | ❌ | **创建/处理**（每次调用可能产生新资源，不可盲目重试） |
| PATCH | ❌ | ❌（**不保证**） | **局部更新**（取决于 patch 内容是否可重复应用） |

**工程口径（这题真正在考"重试安全"）**：
1. **只有幂等方法可以自动重试**（网络超时/5xx）——GET/PUT/DELETE 重试安全，**POST 重试要靠"幂等键"**：客户端生成唯一 `Idempotency-Key`，服务端去重表保证"同一键只处理一次"（支付/下单的标准做法，34_distributed_theory.md 的幂等家族）；这比"改成 PUT"更通用，因为"创建"语义天然不幂等；
2. **PUT 是"整体替换"**：漏传的字段会被置空/默认——"部分更新"用 PATCH 或 POST 专用动作接口；PATCH 的幂等性取决于实现（JSON Merge Patch 对同一补丁幂等；JSON Patch 的 `add` 数组元素不幂等）；
3. **REST 的资源语义 vs RPC 的动作语义**：`POST /orders/{id}/cancel` 这类"动作"接口在业务系统里合理存在——**别为了"纯 REST"把动作硬塞成 `PUT /orders/{id}` 的全量替换**（并发覆盖问题，5.1.1）。

**易错**：用 GET 做"删除/扣减"（浏览器预取/爬虫/缓存会误触发——安全方法必须无副作用）；POST 失败自动重试（重复下单，5.1.1 的幂等键）；DELETE 返回 200 带 body 还是 204 无 body（团队约定即可，但**一致性**重要）；把 405（方法不允许）答成 404。

### 1.2.3 状态码怎么分组？301/302/307/308、401/403、502/503/504 怎么区分？

**五组记忆**：**1xx 信息**（100 Continue、101 Switching Protocols、103 Early Hints）；**2xx 成功**（200 OK、201 Created、202 Accepted、204 No Content、206 Partial Content）；**3xx 重定向**（301/302/303/307/308、304 Not Modified）；**4xx 客户端**（400/401/403/404/405/408/409/413/415/422/429）；**5xx 服务端**（500/501/502/503/504）。

**四组"必考对"**：

| 对比 | 区别 |
|------|------|
| **301 vs 302** | 301 **永久**（可被缓存、搜索引擎改权重、实践上 POST 常被改成 GET）；302 **临时**（语义上方法应保持，但历史实现常改 GET）；**要严格保方法用 307/308** |
| **307/308 vs 303** | 307（临时）/308（永久）**保持原方法与请求体**重发；303 明确"用 GET 去取另一个资源"（POST 后重定向到结果页的标准姿势） |
| **401 vs 403** | 401 **未认证**（缺凭证/凭证无效，响应必须带 `WWW-Authenticate`，客户端应去登录）；403 **已认证但无权限**（别再试登录，找管理员）——"登录了还是 403"是权限配置问题 |
| **502/503/504** | 502 **网关收到上游的无效响应/连接失败**（上游挂了/协议错）；503 **服务自身不可用**（过载/维护，可带 `Retry-After`）；504 **网关等上游超时**（上游慢/卡死）——12_nginx.md 的 502/504 排障主线就是据此分叉 |

**其余高频**：201（创建成功，带 `Location`）、202（已接受、异步处理，返回任务 ID）、204（成功无内容，DELETE/PUT 常用）、206（部分内容，Range）、304（协商缓存未变，**不能带 body**）、400（请求本身有问题）、405（方法不允许，响应带 `Allow`）、409（冲突，如乐观锁版本不符）、413（体太大）、415（媒体类型不支持）、422（语义校验失败，REST API 常用）、429（限流，带 `Retry-After`，20_sentinel.md 的限流响应）。

**答题收口**："**状态码是接口契约的一部分**——前端按状态码分支（401 跳登录、403 提示权限、429 退避重试、5xx 报障），后端返回错码（异常一律 200 + code 字段）等于毁掉 HTTP 的通用语义，也让网关/监控/重试策略全部失灵。"

# 2. 连接、传输与性能

## 2.1 连接管理

### 2.1.1 Keep-Alive 与 HTTP/2 多路复用的区别？队头阻塞到底在哪一层？

**三个层次的"复用"（混着答必扣分）**：

| 机制 | 复用单位 | 并发形态 | 队头阻塞 |
|------|---------|---------|---------|
| HTTP/1.0 短连接 | 无（一请求一连接） | 串行/多连接 | 无（但握手成本巨大） |
| **HTTP/1.1 Keep-Alive** | **一条 TCP 连接串行跑多个请求** | 同一连接上**一次一个**（等响应完再下一个） | **HTTP 层**：前一个响应慢，后面全等 |
| HTTP/1.1 管道化（Pipelining） | 同连接上连续发多个请求 | 请求可连发，**响应必须按序** | 仍在（且代理兼容差，浏览器默认关闭） |
| **HTTP/2 多路复用** | **一条连接上多个 Stream 并发** | 帧交错、真正并行 | **HTTP 层消除**；**TCP 层仍在**（丢包阻塞所有流） |

**HTTP/1.1 的"多连接"是补丁不是方案**：浏览器对同一域名默认开 **6 个左右**并发连接——用"多开 TCP"近似并行，代价是握手/TLS/拥塞控制各自重来、服务器连接数与内存放大（5_tomcat.md 的连接账）；**域名分片（domain sharding）** 是 HTTP/1.1 时代的经典优化，到 HTTP/2 反而成了反模式（拆散多路复用）。

**"队头阻塞"的三层准确表述（面试核心）**：
1. **HTTP/1.1 应用层 HOL**：同一连接上响应按请求顺序返回，前一个慢就堵住后面；
2. **HTTP/2 的 TCP 层 HOL**：流在应用层独立了，但 TCP 是**单一有序字节流**——一个包丢失，内核要等它重传，**所有流的数据都排在它后面**；丢包率高的网络（移动/弱网）上，HTTP/2 的单连接可能比多连接的 HTTP/1.1 更慢（"一条连接承载全部"的放大器效应）；
3. **HTTP/3 的解法**：QUIC 把"流"下沉到传输层，每个流独立排序与重传——**丢一个包只阻塞该流**。

**Keep-Alive 的运维细节（追问题）**：`Connection: keep-alive`（1.1 默认）；服务端有 **idle timeout**（Nginx `keepalive_timeout`、Tomcat `keepAliveTimeout`）与 **max requests** 上限；**客户端连接池的空闲回收必须早于服务端**，否则拿到"对端已关"的死连接 → 偶发 502/连接重置（12_nginx.md 2.2.3、18_feign.md 3.1.2 的同一条账）。

**收口句**："**Keep-Alive 复用的是连接，多路复用复用的是'流'**——前者省握手，后者省连接数并解应用层排队；但只有 QUIC 才把排队问题从传输层拿掉。"

### 2.1.2 HTTP/2 的帧、流、HPACK、流控、Server Push 分别怎么工作？

**二进制分帧（Binary Framing）**：一切消息都拆成**帧**——9 字节帧头（24 位长度、8 位类型、8 位标志、1 位保留 + 31 位 **Stream ID**）+ 载荷：

| 帧类型 | 作用 |
|--------|------|
| **HEADERS** | 请求/响应头（含伪头 `:method/:path/:scheme/:authority`） |
| **DATA** | 消息体 |
| **SETTINGS** | 连接参数协商（并发流数、初始窗口、头表大小等） |
| **WINDOW_UPDATE** | 流控窗口增量 |
| **RST_STREAM** | 中止某个流（不影响连接） |
| **PING / GOAWAY** | 探活 / 优雅关闭连接（告知最后处理的流 ID） |
| PRIORITY / PUSH_PROMISE | 优先级（已废弃）/ 服务端推送（已退场） |
| CONTINUATION | 头部帧的续帧（大头部拆分） |

**流（Stream）= 一次请求/响应的逻辑通道**：客户端发起的流 ID 为奇数、服务端为偶数；同一连接上多个流**交错发送帧**，接收端按 Stream ID 重组——这就是"多路复用"的实现；并发流数受 `SETTINGS_MAX_CONCURRENT_STREAMS` 限制。

**HPACK 头压缩（省带宽的关键）**：**静态表**（61 个常见头/值的预定义索引，如 `:method: GET`）+ **动态表**（连接内已发送的头加入表，后续只发索引）+ **哈夫曼编码**（字面值压缩）。**为什么重要**：HTTP/1.1 每个请求都重复发 `Cookie`/`User-Agent`/`Authorization` 等大头部，移动端尤其浪费；HPACK 把"重复的头"变成"1 个索引"。**注意**：动态表是**每连接**的，所以 HTTP/2 的"单连接"也让压缩收益最大化（连接一换，表就重来）。

**流控（Flow Control）**：**每个流 + 整个连接**各有窗口（初始 65535 字节），接收方用 `WINDOW_UPDATE` 增量授权——防止"一个流把接收端内存打爆"（与 TCP 的接收窗口是两套独立机制，别混）。

**Server Push 的兴衰**：设计初衷是"服务器预判客户端要什么，主动推 CSS/JS"（`PUSH_PROMISE`）；实践中**命中率低、缓存与优先级难控制、浪费带宽**——Chrome 106 移除，规范不再推荐；现代替代：**`103 Early Hints` + `Link: rel=preload`**（先告诉浏览器"去预加载"，由浏览器决定）。

**部署细节**：浏览器只接受 **TLS 上的 h2**（ALPN 协商）；`h2c`（明文）仅内网/特殊场景；**h2 对后端**（Nginx→Tomcat）需显式开启（`proxy_http_version 1.1` 是 h1 的 keep-alive，h2 后端要 `grpc_pass`/`http2` 语义，12_nginx.md）；**HTTP/2 下的连接数不是越多越好**（单连接共享拥塞窗口）。

### 2.1.3 HTTP/3 与 QUIC 解决了什么？0-RTT 和连接迁移的代价是什么？

**QUIC 的三个立身之本**：
1. **没有 TCP 队头阻塞**：QUIC 在 UDP 之上实现**独立的流**（stream）——每个流有自己的序号与重传，**丢包只影响该流**（2.1.1 的第三层解法）；
2. **建连更快**：QUIC **把 TLS 1.3 内嵌进握手**（传输握手与加密握手合并）——新连接 **1-RTT**，会话复用 **0-RTT**（首个包就带加密应用数据）；
3. **连接迁移**：连接由 **Connection ID** 标识，而不是"源 IP+端口"四元组——手机从 WiFi 切 4G，**连接不断**（地址变了，Connection ID 不变）。

**三个"代价/边界"（这题的成熟度）**：
- **0-RTT 有重放风险**：0-RTT 数据可被攻击者截获重放（服务器无法区分"新请求"与"重放的旧请求"）——所以**只应对幂等请求启用**（GET），POST 等非幂等操作应等 1-RTT 完成；服务端还需**单次票据/反重放窗口**；
- **UDP 可能被封**：部分企业网络/防火墙禁 UDP 443——客户端要能**回退到 HTTP/2**（Alt-Svc 宣告 + 超时回退），"上了 h3 就一定更快"不成立；
- **生态与可观测性**：抓包（Wireshark 支持 QUIC/TLS 解密需密钥）、中间件（老防火墙/负载均衡不识别 UDP）、服务端 CPU（用户态协议栈）都还在成熟中。

**旧口径纠偏**：**"QUIC 有前向纠错（FEC）"是早期草案的特性，RFC 9000 的 QUIC 已移除 FEC**（靠快速重传与流级恢复），别再背"发冗余包恢复丢包"；QUIC 的拥塞控制是可插拔的（NewReno/CUBIC/BBR 等），"改进了拥塞控制"指的是**用户态实现与快速迭代**，不是某个固定算法。

**收口**："HTTP/3 不是'更快的 HTTP'，是**换了一套传输底座**：把可靠传输、加密、多路复用、连接标识全塞进 QUIC——代价是把复杂度从内核搬到了用户态与运维体系。"

## 2.2 传输细节

### 2.2.1 `Transfer-Encoding: chunked` 是什么？和 Content-Length 怎么选？

**问题背景**：HTTP/1.1 的报文边界靠"**Content-Length**（提前知道长度）"或"**chunked**（边生成边发，最后用 0 长度块收尾）"——动态内容（服务端边查边吐）与流式响应（SSE/大文件导出）**无法提前知道总长度**，就必须 chunked。

**格式（1.1 文本协议）**：

```text
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1a\r\n                      ← 块长度（十六进制 0x1a = 26 字节）
Hello, this is the data\r\n ← 块数据
0\r\n                       ← 结束块（长度 0）
\r\n                        ← 可选的 trailer 结束
```

**与 Content-Length 的关系（安全要点）**：**两者互斥**——同时出现（或 `Transfer-Encoding` 被中间层改写）是**请求走私（Request Smuggling）**的经典入口：前端代理按 CL 解析、后端按 TE 解析，多出来的字节被"夹带"成下一个请求（3.2.3）。**规范要求**：收到冲突的头应拒绝或按 TE 优先并以 400 关闭连接；**HTTP/2 没有 chunked**（DATA 帧 + `END_STREAM` 标志天然分帧，1.1 的 chunked 语义由帧机制取代）。

**工程使用**：SSE（`text/event-stream`）在 HTTP/1.1 下必须 chunked；大文件导出/日志流式输出用 chunked 省内存；**反向代理默认可能缓冲**（Nginx `proxy_buffering on`）——流式场景要关缓冲，否则"服务端在吐、客户端收不到"（12_nginx.md）。

**易错**：手动拼 chunked 编码（SDK/容器代劳，手拼极易错）；给 chunked 响应再设 Content-Length（非法）；以为 HTTP/2 也走 chunked（不是，帧机制）；trailer 头（chunk 后的元数据）多数客户端不支持，别依赖。

### 2.2.2 Range 断点续传怎么实现？206 与 416 的语义？

**协商流程**：

```text
① 首次请求 → 200 + Accept-Ranges: bytes（告知"支持按字节范围取"）
② 下载中断（网络/暂停）
③ 续传请求 → Range: bytes=1024-            （从 1024 到末尾）
              或 Range: bytes=0-1023,2048-4096（多段，响应为 multipart/byteranges）
④ 服务端 → 206 Partial Content + Content-Range: bytes 1024-2047/2048
⑤ 范围非法（超出文件大小/格式错）→ 416 Range Not Satisfiable + Content-Range: bytes */总长
```

**三个必讲细节**：
1. **`If-Range` 保证一致性**：续传时资源可能已更新——客户端带 `If-Range: <ETag 或 Last-Modified>`，服务端判断"资源没变才给 206，变了就返回 200 全量"（避免"旧片段 + 新文件"拼出损坏文件）；
2. **并发分片下载**：多线程各取一段（`bytes=0-...`）再按序拼接——下载器/视频播放器的实现基础；**服务端要防"大量 Range 请求"被滥用**（Nginx `max_ranges` 限制）；
3. **视频拖动进度条**：播放器发 `Range` 取关键帧附近数据，服务端必须支持 206 且**范围计算基于文件字节**（不是视频时间轴，时间→字节由播放器/索引换算）。

**易错**：服务端返回 200 却带了 `Content-Range`（矛盾）；206 的 `Content-Length` 应是**该片段长度**而非总长；对**动态生成**内容开 Range（内容每次不同，断点无意义）；CDN/对象存储的 Range 支持要实测（部分老网关会忽略 Range 返回全量，25_minio.md 的 `offset/length`）。

### 2.2.3 文件上传用 `multipart/form-data` 还是 JSON？

**四种上传形态**：

| 方式 | 结构 | 适用 | 代价 |
|------|------|------|------|
| `multipart/form-data` | boundary 分隔多个 part，每个 part 有 `Content-Disposition`/`Content-Type`，**文件以原始二进制放 part** | **表单 + 文件**（HTML 原生、Spring `MultipartFile`） | 解析复杂、边界处理；体大小受限（网关/容器上限） |
| `application/json` | 整个体是一个 JSON，二进制需 **Base64**（体积 +33%） | 小文件/结构化 API、需要与业务字段同构 | 内存放大（编解码）、不适合大文件 |
| `application/x-www-form-urlencoded` | `k=v&k2=v2`（URL 编码） | 纯文本小表单 | 不支持文件、体积膨胀 |
| **直传对象存储** | 后端发预签名 URL，客户端直传（25_minio.md 3.1.3） | 大文件/高并发上传 | 需要额外的鉴权/回调/清理链路 |

**multipart 的报文长这样（要能画）**：

```text
Content-Type: multipart/form-data; boundary=----Boundary123

------Boundary123
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<二进制数据>
------Boundary123
Content-Disposition: form-data; name="description"

A picture
------Boundary123--
```

**工程要点**：① **大小限制**贯穿容器/网关/框架（Tomcat `maxPostSize`、Spring `max-file-size`、Nginx `client_max_body_size`）——413 的排查从这三层看；② **流式解析**（别把整个 multipart 读进内存，Spring 配 `file-size-threshold` 落临时文件）；③ **文件名不可信**（路径穿越/同名覆盖，服务端重命名，25_minio.md 的 key 生成）；④ **Content-Type 可伪造**（按魔数/后缀校验，上传后扫描）；⑤ **boundary 冲突**（文件内容恰好含 boundary 概率极低，但生成 boundary 要随机且足够长）。

**选型句**："**表单带文件 → multipart；接口传小文件/结构化数据 → JSON（Base64）；大文件/高并发 → 直传对象存储**——把'文件穿过应用服务器'当成默认路径，是上传架构最常见的浪费。"

# 3. 状态与安全

## 3.1 状态管理

### 3.1.1 Cookie、Session、Token、JWT 是什么关系？怎么演进与选型？

**一条演进线：HTTP 无状态 → 需要"记住人" → 状态放哪、凭证长什么样**：

| 方案 | 状态存哪 | 凭证形态 | 优点 | 代价 |
|------|---------|---------|------|------|
| **Cookie** | 客户端（浏览器） | 任意 KV，随请求自动携带 | 简单、浏览器原生 | 容量小（4KB/条）、自动携带带来 CSRF 面、可被用户查看 |
| **Session** | **服务端**（内存/Redis） | 客户端只存 `JSESSIONID` | 服务端可控、易注销 | 服务端有状态，集群要共享（Redis，27_redis.md）；会话固定攻击 |
| **Token（不透明）** | 服务端（token→用户映射） | 随机串（放 Header 或 Cookie） | 可撤销、可控 | 每次校验要查存储（或缓存） |
| **JWT（自包含）** | **客户端**（服务端不存） | `header.payload.signature` 三段 Base64URL | **无状态**、天然水平扩展、可跨服务验签 | **注销难**（到期才失效）、载荷不可放敏感信息、体积大、签名算法要用对 |

**JWT 的结构与安全要点**：`header`（算法）、`payload`（claims：`sub`/`exp`/`iat`/自定义）、`signature`（对前两段签名）——**payload 只是 Base64URL 编码，不是加密**（谁都能解出内容，**别放密码/密钥**）；**验签必须固定算法**（拒绝 `alg: none` 与算法混淆攻击）；`exp` 短 + `refresh token` 是"可注销性"的折中（黑名单/版本号是另一条路，27_redis.md 的位图/集合）。

**选型口径（按部署形态）**：
- **单体/传统 Web**：Session（服务端可控、注销简单）+ Cookie（`HttpOnly`/`Secure`/`SameSite`，3.1.2）；
- **微服务/前后端分离**：JWT 或 Opaque Token 放 `Authorization: Bearer`——**别放 localStorage 裸奔**（XSS 可直接偷），更稳的是 **HttpOnly Cookie + CSRF 防护**（3.1.2）或"内存 + 刷新令牌"；
- **网关统一认证**：网关验 JWT、下游只信网关清洗后的身份头（19_spring_cloud_gateway.md 2.2.2、11_spring_security.md 4.3.1）——JWT 自包含的价值在此最大化。

**收口**："**Cookie 是'传输容器'，Session/Token 是'状态方案'，JWT 是'凭证格式'**——三者不是同一层的东西；面试把'状态放哪、凭证怎么带、怎么失效'三问答全，就超过了背名词的人。"

### 3.1.2 Cookie 的安全属性怎么配？CSRF 与 XSS 怎么防？

**Cookie 六个属性（逐条说"防什么"）**：

| 属性 | 作用 |
|------|------|
| `Domain` / `Path` | **作用域**——越小越好（`Domain=.example.com` 会让子域共享，扩大攻击面） |
| `Expires` / `Max-Age` | 持久化时长（会话 Cookie 关浏览器即失效） |
| **`Secure`** | 只走 HTTPS 发送（防明文嗅探） |
| **`HttpOnly`** | **JS 不可读**（`document.cookie` 拿不到）——防 XSS 偷会话 |
| **`SameSite`** | 跨站请求是否携带：`Strict`（完全不带）/ `Lax`（**现代浏览器默认**：顶层导航 GET 带，跨站 POST/iframe 不带）/ `None`（必须配 `Secure`，用于跨站嵌入场景） |
| `Partitioned`（CHIPS） | 按顶层站点分区存储，防第三方 Cookie 滥用（广告/嵌入场景的新标准） |

**CSRF（跨站请求伪造）的原理与防线**：攻击者站点让受害者浏览器**自动带上目标站 Cookie** 发请求（表单/img 都能跨站发）——**防线四层**：
1. **`SameSite=Lax/Strict`**（最省事的默认层，但跨站单点登录/嵌入场景要 `None`，此时其他层必须补上）；
2. **CSRF Token**：服务端发随机 token（存 Session 或双提交 Cookie），**表单/请求头带回**——攻击者跨站拿不到（同源策略保护读取）；
3. **校验 `Origin`/`Referer`**（对非 GET 的写操作）；
4. **要求自定义头**（如 `X-Requested-With`）——跨站表单发不了自定义头（简单请求限制，3.2.1）。

**XSS 与 CSRF 的分工（高频混淆）**：XSS 是"**在页面里执行攻击者的脚本**"（偷 Cookie/发请求/改页面），防御靠**输出编码、CSP、`HttpOnly`**；CSRF 是"**借用户的 Cookie 发请求**"（不需要读响应），防御靠 **SameSite/Token/Origin**。**关键认知**：**`HttpOnly` 防不了 CSRF**（请求照发），**CSRF Token 防不了 XSS**（脚本能读到页面里的 token）——两者是不同攻击面。（同时中 XSS 时，攻击者可以"以用户身份操作"，这已超出 CSRF 范畴。）

**易错**：`SameSite=None` 忘了配 `Secure`（浏览器直接拒收）；把所有 Cookie 都设 `HttpOnly`（前端要读的非敏感标记反而读不到）；用 CORS 当 CSRF 防护（CORS 限制的是"读响应"，不阻止"发请求"）；JWT 放 localStorage 还宣称"防 CSRF"（却开了 XSS 的门）。

## 3.2 Web 安全与跨域

### 3.2.1 同源策略是什么？CORS 预检的完整流程？

**同源 = 协议 + 主机 + 端口三者完全相同**（`https://a.com:443` 与 `http://a.com` 不同源，`https://a.com` 与 `https://b.a.com` 不同源）。**同源策略（SOP）限制的是"读取"**：跨源 `fetch/XHR` 默认**能发请求但读不到响应**；`<img>/<script>/<form>` 等"简单加载"可跨源发送（这也是 CSRF 的土壤）。

**CORS 的两种请求**：
1. **简单请求**（GET/HEAD/POST + 仅限 `text/plain`/`form-urlencoded`/`multipart/form-data` + 无自定义头）：直接发，浏览器看响应头 `Access-Control-Allow-Origin`（ACAO）决定是否放行读取；
2. **预检请求（Preflight）**：非简单请求（PUT/DELETE、`application/json`、自定义头）先发 **`OPTIONS`**：
   ```text
   OPTIONS /api/orders
   Origin: https://app.example.com
   Access-Control-Request-Method: PUT
   Access-Control-Request-Headers: authorization,content-type

   → 200
   Access-Control-Allow-Origin: https://app.example.com
   Access-Control-Allow-Methods: GET,POST,PUT,DELETE
   Access-Control-Allow-Headers: authorization,content-type
   Access-Control-Allow-Credentials: true
   Access-Control-Max-Age: 600        ← 预检结果缓存 10 分钟
   ```

**带凭证（Cookie/Authorization）的三条硬规则**：`Access-Control-Allow-Credentials: true` + **ACAO 必须是具体源**（不能 `*`）+ 前端 `credentials: 'include'`；跨站 Cookie 还要 `SameSite=None; Secure`（3.1.2）。

**四个高频事故**：① 响应出现**两个 ACAO**（网关与后端各加一遍）→ 浏览器报"multiple values"（19_spring_cloud_gateway.md 3.2.2 的 DedupeResponseHeader）；② **预检被鉴权拦成 401**——OPTIONS 不带 Authorization 是规范行为，白名单必须放行 OPTIONS；③ `Allow-Headers` 漏了自定义头（`X-Token`）→ 预检失败；④ 忘了 `Vary: Origin`（CDN/缓存把 A 源的响应发给 B 源）。

**替代与边界**：同源部署（Nginx 反代 `/api` 到后端）根本不触发 CORS（12_nginx.md 的首选解）；JSONP 是历史方案（只支持 GET、有 XSS 面）；`postMessage` 用于 iframe 跨源通信（要校验 `origin`）；**CORS 不是认证**——它只控制"浏览器是否允许读响应"，服务端该做的鉴权一个不能少（curl 可无视 CORS）。

### 3.2.2 HTTPS/TLS 握手怎么走？证书链与常见错误？

**HTTPS = HTTP over TLS**（默认 443）：TLS 提供**加密（机密性）、完整性、身份认证（证书）**。**TLS 1.2 握手（2-RTT）**：

```text
① ClientHello：客户端随机数 + 支持的密码套件 + SNI（要访问的域名）+ ALPN（h2/http1.1）
② ServerHello + Certificate + ServerKeyExchange：服务端随机数 + 证书链 + 密钥交换参数
③ 客户端验证证书链（是否受信 CA 签发、域名是否匹配、是否过期/吊销）
④ 双方用 ECDHE 协商出会话密钥（前向保密）→ Finished（加密握手完成）
⑤ 此后应用数据用对称加密传输
```

**TLS 1.3 的改进**：**1-RTT 完成握手**（把密钥交换与认证合并，删掉冗余往返）、**会话复用 0-RTT**（恢复连接首个包带数据，但有重放风险，只适合幂等请求）、**强制前向保密**（默认 ECDHE）、**移除老弱算法**。**证书相关**：证书链（叶证书→中间 CA→根 CA）、**SAN**（多域名）、**SNI**（一个 IP 多证书，握手时按域名选）、**OCSP Stapling**（服务端代取吊销状态，减少客户端查询）；**mTLS**（双向证书，服务间认证）。

**四种常见错误与处置**：`证书过期`（监控到期 + 自动续期，ACME/Let's Encrypt）；`域名不匹配`（证书 SAN 未含该域名，或反代改了 Host）；`证书链不全`（只配了叶证书，缺中间 CA——浏览器能补、Java/curl 常失败）；`PKIX path building failed`（客户端信任库没有根 CA——容器/distroless 镜像缺 `cacerts`，23_docker.md 5.1.1 的证书坑）。

**HSTS**：`Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`——告诉浏览器"以后这个域名只走 HTTPS"，防 SSL 剥离/降级；**上线前确认全站 HTTPS 可用**（HSTS 有"锁死"风险，`preload` 列表移除很慢）。

**易错**：把 TLS 1.3 的 0-RTT 说成"完全安全"（重放风险）；在反代层终结 TLS 后，后端不知道原始协议（要 `X-Forwarded-Proto` 还原，12_nginx.md）；自签证书用于生产（客户端不信任）；只测浏览器不测 Java/curl（信任库与协议版本差异）。

### 3.2.3 Web 常见攻击与 HTTP 层的防护头有哪些？请求走私是什么？

**攻击面速查（HTTP 层能防的部分）**：

| 攻击 | 原理 | HTTP 层防护 |
|------|------|------------|
| **XSS** | 注入脚本在受害者页面执行 | **CSP**（限制脚本源）、输出编码、`HttpOnly`、`X-Content-Type-Options: nosniff`（防 MIME 嗅探） |
| **CSRF** | 借用户 Cookie 发请求 | `SameSite`、CSRF Token、Origin 校验（3.1.2） |
| **点击劫持** | iframe 套页面诱导点击 | `X-Frame-Options: DENY/SAMEORIGIN` 或 CSP `frame-ancestors` |
| **开放重定向** | 跳转参数被换成恶意站点 | 跳转白名单、不直接回显参数 |
| **敏感信息泄露** | 响应头/错误页暴露版本 | 收敛 `Server`/`X-Powered-By`、统一错误页 |
| **降级/剥离** | 强制 HTTP 被劫持 | **HSTS**（3.2.2） |
| **中间人** | 明文传输被篡改 | 全站 HTTPS + `Secure` Cookie |

**一套基线安全响应头（背这组）**：

```text
Strict-Transport-Security: max-age=31536000; includeSubDomains
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example.com
X-Content-Type-Options: nosniff
X-Frame-Options: DENY                     # 或 CSP frame-ancestors 'none'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=()
```

**请求走私（Request Smuggling，进阶但高频）**：**前端代理与后端服务器对"请求边界"的解析不一致**——典型是 `Content-Length`（CL）与 `Transfer-Encoding: chunked`（TE）冲突：
- **CL.TE**：前端按 CL 解析、后端按 TE 解析 → 攻击者把第二个请求"藏"在体里，被后端当成新请求执行（绕过前端鉴权、污染缓存、劫持他人请求）；
- **TE.CL** / **TE.TE**（双 TE 头混淆）同理；
- **防线**：代理/服务端对冲突头**拒绝或统一规范化**（禁用 TE 或强制 CL）、**端到端 HTTP/2**（帧边界明确，天然免疫 CL/TE 歧义）、WAF 规则、及时升级组件（历史上 Nginx/Tomcat/各类代理都有过相关 CVE）。**答题口径**："**走私的根因不是某个漏洞，而是'两跳解析器对同一字节流理解不同'**——所以解药是统一边界语义，而不是补某个头。"

**收口**："HTTP 安全是**分层默认值**的工程：Cookie 属性、CORS、CSP、HSTS、帧边界——每层都给'不安全行为'设默认拒绝，攻击者才需要同时突破多层。"

# 4. 缓存与代理

## 4.1 缓存

### 4.1.1 强缓存与协商缓存的完整决策流程？

**两级缓存（浏览器与 CDN/代理都按这套逻辑）**：

```text
请求资源
  │
  ├─ 有缓存副本？──否──▶ 发请求 → 200 → 按响应头存缓存
  │
  └─ 是 → 判断"新鲜度"（Cache-Control: max-age / s-maxage / Expires）
        ├─ 新鲜（未过期）→ **强缓存命中**：直接用，不发请求（200 from disk/memory cache）
        └─ 过期 → **协商缓存**：带 If-None-Match(ETag) / If-Modified-Since
              ├─ 服务端 304 Not Modified → 用本地副本（更新缓存元数据）
              └─ 服务端 200 + 新内容 → 替换缓存
```

**强缓存（不发请求）的关键指令**：

| 指令 | 语义 |
|------|------|
| `max-age=3600` | 新鲜期 3600 秒（相对时间，优先于 `Expires`） |
| `s-maxage=600` | **共享缓存**（CDN/代理）专用，覆盖 max-age |
| `public` / `private` | 谁可缓存（`private` 只允许浏览器） |
| `no-cache` | **可以存但每次必须协商**（不是"不缓存"！） |
| `no-store` | **完全不存**（敏感数据） |
| `immutable` | 新鲜期内**连刷新都不重新验证**（带 hash 的静态资源） |
| `stale-while-revalidate=60` | 过期后先给旧内容、后台异步刷新（体验与一致性的折中） |

**协商缓存（发请求但可能只回 304）**：`ETag`（内容指纹，强/弱：`W/"..."`）配 `If-None-Match`；`Last-Modified`（秒级精度）配 `If-Modified-Since`——**两者同时存在时 `If-None-Match` 优先**；304 **不带 body**，只更新缓存新鲜度。

**工程策略（比背指令更重要）**：
1. **带 hash 的静态资源**：`Cache-Control: public, max-age=31536000, immutable`（一年 + immutable）——文件名变即新 URL，**永久缓存零风险**（前端构建产物 `app.3f2a1c.js`）；
2. **HTML 入口**：`no-cache`（每次协商）——保证发版后拿到新资源引用；
3. **API 响应**：按业务定（列表短缓存/详情不缓存），带用户态的响应**别让 CDN 缓存**（`private`/`no-store` + `Vary` 或直接不缓存）；
4. **`Vary` 是缓存键的一部分**：`Vary: Accept-Encoding, Origin`——漏配会导致"错发"（压缩/跨域响应串味）。

**易错**：把 `no-cache` 当 `no-store`（前者是"必须验证"，后者才是不存）；只配 `Expires` 不配 `Cache-Control`（HTTP/1.0 兼容语义，优先级低）；改资源内容但 URL 不变还设长缓存（用户永远看不到新版本——版本化 URL 是正解）；`304` 响应误带 `Content-Length` 或 body。

### 4.1.2 CDN 的调度与回源怎么工作？缓存键与刷新怎么设计？

**工作链路**：

```text
用户 → DNS/Anycast 调度（就近/按运营商/按健康度选边缘节点）
     → 边缘节点（PoP）查缓存
         ├─ 命中：直接返回（最快）
         └─ 未命中：回源（Origin 或上层 Shield 层）→ 拉取 → 缓存 → 返回
```

**调度技术**：**DNS 智能解析**（按地域/运营商返回不同节点 IP，TTL 短、可动态调整）与 **Anycast**（同一 IP 广播到多地，网络层就近）——面试报"DNS 调度 + Anycast"即到位；**边缘节点不止一层**（边缘 → 中间层/Origin Shield → 源站），Shield 层把"回源风暴"收敛成"一次回源"。

**缓存键（Cache Key）设计（这题的核心）**：默认是"**URL（host+path+query）**"，但业务常要加/减维度：
- **该加的**：影响内容的头/查询参数（`Vary`、版本参数）——但**加得越多，命中率越低**；
- **该减的**：无关的跟踪参数（`utm_*`）应在边缘**归一化/剔除**，否则同一资源被缓存成 N 份；
- **绝不能进缓存键的**：用户身份（会导致缓存碎片化/串号）——个性化内容应**不缓存**或走"边缘计算/ESI 片段"。

**刷新与失效**：`Purge`（按 URL/目录/标签清除边缘缓存）、**版本化 URL**（推荐：发版换文件名，无需 purge）、短 TTL + `stale-while-revalidate`（过期期间边缘先给旧内容，后台回源）——**"purge 全站"是核弹**（回源风暴打垮源站），能版本化就别 purge。

**CDN 的隐藏收益**：**DDoS 吸收**（边缘带宽与清洗）、**TLS 卸载**（证书在边缘，源站只处理内网明文）、**协议优化**（边缘到用户 h3、边缘到源站长连接/专线）、**动态加速**（路由优化，非缓存）。

**易错**：把带 `Set-Cookie` 的响应缓存了（用户 A 的 Cookie 发给 B——**灾难级事故**，源站必须 `Cache-Control: private` 或边缘剔除 `Set-Cookie`）；CDN 与浏览器**两层缓存**各自 TTL 打架（调了源站没刷 CDN）；**POST 被缓存**（默认不缓存，但某些配置/边缘规则会误伤）；跨域资源忘了 `Vary: Origin`（3.2.1）。

## 4.2 代理与链路

### 4.2.1 正向代理与反向代理有什么区别？X-Forwarded-For 和 X-Real-IP 怎么取真实 IP？

**正反向一句话**：**正向代理代表客户端**（客户端配置代理去访问外网：翻墙/企业出口/爬虫池）；**反向代理代表服务端**（客户端以为它就是服务器：Nginx/网关/SLB，做 TLS 终结、负载均衡、缓存、限流）。

**真实 IP 的两组头**：
- **`X-Real-IP`**：通常由**第一层反代**设置成"它看到的对端 IP"，只有一个值；
- **`X-Forwarded-For`（XFF）**：**链式追加**——`客户端IP, 代理1IP, 代理2IP`（每层把"自己的对端"append 到末尾）；标准化的头是 **`Forwarded`**（RFC 7239，`for=;proto=;host=`），但 XFF 仍是事实标准。

**取真实 IP 的正确姿势（安全题）**：**XFF 可被客户端伪造**（自己带一个假的最左值）——所以：
1. **只信任可控代理层追加的部分**：从右往左数，跳过已知代理 IP，**第一个非可信 IP 才是客户端**；
2. 应用侧配置"信任代理层级/网段"（Spring Boot `server.forward-headers-strategy=native/framework` + 可信代理配置；Nginx `set_real_ip_from` + `real_ip_header`）；
3. **别直接用 `XFF.split(",")[0]`**——攻击者可伪造最左值绕过 IP 限流/审计（20_sentinel.md 的 origin 解析同理）。

**Nginx 标准配置**：`proxy_set_header X-Real-IP $remote_addr; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Proto $scheme;`——**`X-Forwarded-Proto` 决定后端生成的重定向/绝对 URL 是 http 还是 https**（漏配会导致"登录后跳回 http"/混合内容）。

**易错**：多层代理只传 XFF 不传 X-Real-IP（后端取不到单值）；反向代理改了 `Host` 导致虚拟主机路由错/签名失败（S3、多租户）；把代理当"绝对安全边界"（代理后面的服务仍需鉴权）。

### 4.2.2 502、504、413、431 这些网关错误怎么排查？

**先按"谁超时/谁拒绝"分型**：

| 状态码 | 语义 | 常见根因 | 排查动作 |
|--------|------|---------|---------|
| **502 Bad Gateway** | 网关拿到**无效响应/连不上上游** | 上游进程挂了/端口不通；**keepalive 竞态**（拿到已关闭的连接）；上游 TLS/协议不匹配；上游提前断连（崩溃/被 OOM 杀）；DNS 解析失败 | 看网关 error log（`connect() failed`/`upstream prematurely closed`）；`curl` 直连上游；核对 keepalive 超时与上游一致（12_nginx.md 2.2.3） |
| **504 Gateway Timeout** | 网关**等上游超时** | 上游慢（DB/下游 RPC/GC 停顿）；线程池打满排队；网关 `proxy_read_timeout` 太短；**超时预算配反**（内层比外层长） | 看上游 RT 分位与线程栈；核对全链超时（15_spring_cloud.md 3.2.3）；扩容/限流 |
| **413 Payload Too Large** | 请求体超限 | `client_max_body_size`（Nginx）、`maxPostSize`（Tomcat）、Spring `max-file-size`、网关限制 | 三层逐一核对；大文件改直传（2.2.3） |
| **431 Request Header Fields Too Large** | 头部太大 | Cookie 过多/过大、JWT 放 Header 太长、代理 `large_client_header_buffers` 太小 | 收敛 Cookie、缩短 token、调大缓冲区（治标） |
| **499（Nginx 特有）** | **客户端主动断开** | 用户取消/前端超时先断、后端太慢 | 结合上游 RT 看，不是网关的错 |
| **408** | 请求超时（客户端没发完） | 慢网络/大上传/`client_body_timeout` | 调超时/直传 |

**排查顺序（通用）**：① 定位是哪一层返回的（响应头 `Server`、错误页样式、网关 access log 的 upstream 字段）；② 直连上游复现（绕过网关）；③ 对齐**超时链**与**连接复用**两个高频根因；④ 看资源（线程数/连接数/GC/磁盘）而不是只盯网络。

**易错**：把所有 5xx 都当"后端 bug"（502 常是连接层、504 常是超时预算）；只调大超时（掩盖慢查询/线程池问题，雪崩时更糟）；忽略"网关→上游"的 keepalive 与上游 `keepAliveTimeout` 的对齐（偶发 502 的经典来源）。

# 5. 工程实践与演进

## 5.1 API 设计

### 5.1.1 RESTful 接口怎么设计？幂等键与乐观并发怎么落地？

**资源化设计五条**：① **名词资源 + HTTP 方法表达动作**（`GET /orders/1`、`POST /orders`、`PUT /orders/1`、`DELETE /orders/1`）；② **状态码表达结果**（1.2.3，不要一律 200 + code）；③ **分页/过滤/排序用 query**（`?status=PAID&page=2&size=20`，深分页用游标）；④ **版本化**（URL `/v1` 或 Header，破坏性变更必须升版本）；⑤ **错误体结构化**（RFC 7807 `application/problem+json`：`type/title/status/detail/instance`），字段级错误给 `errors[]`。

**幂等键（Idempotency-Key）的标准实现（支付/下单必考）**：
1. 客户端为"一次业务意图"生成 UUID，放在 Header；
2. 服务端**唯一索引/Redis SETNX** 抢占该键：首次→执行业务并记录"键→结果"；重复→**直接返回首次结果**（同状态码/同响应体）；
3. 并发重复→第二个请求返回 **409**（处理中）或等待；键的 TTL 覆盖业务重试窗口；
4. **键与请求体指纹绑定**（同键不同体=客户端 bug，拒绝）——否则"复用了键"会返回错结果。

**乐观并发（防覆盖）**：资源带 `version`/`ETag`，更新时 `If-Match: "v3"`（或 body 带 version），不匹配返回 **412 Precondition Failed / 409**——比"最后写入赢"安全，比悲观锁轻（16_mysql.md 3.2.4）。

**长任务接口**：`POST /exports` → **202 Accepted + 任务 ID** → 客户端轮询 `GET /tasks/{id}` 或回调/webhook——**别让 HTTP 连接挂着等十分钟**（网关超时、线程占用，4.2.2）。

**易错**：`GET /deleteOrder?id=1`（安全方法做副作用，1.2.2）；分页用 `offset` 到百万级（深度分页，24_elastic_search.md 2.2.2 同款）；错误码与 HTTP 状态码"双轨制"且不一致；PUT 当 PATCH 用（全量替换把字段清空）。

### 5.1.2 WebSocket 与 SSE 怎么选？（101 升级与事件流）

**WebSocket（全双工）**：
```text
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: <base64 随机>
Sec-WebSocket-Version: 13

→ 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: base64(SHA1(key + 固定 GUID))
```
握手成功后**协议切换**（同一 TCP 连接上跑 WebSocket 帧：文本/二进制/ping/pong/close），**双向实时**；鉴权在握手阶段做（带 Cookie/Token），升级后不能再走 HTTP 语义；**代理必须支持 Upgrade**（Nginx `proxy_set_header Upgrade/Connection`、长连接超时调大），负载均衡要**会话粘滞或共享 pub/sub**（多实例广播）。

**SSE（Server-Sent Events，单向服务端推）**：`Content-Type: text/event-stream`，服务端持续写 `data:` 行，客户端 `EventSource` 自动重连（带 `Last-Event-ID` 续传）；**基于普通 HTTP**（天然过代理/网关，HTTP/2 下多路复用不占额外连接），实现简单；**只支持服务端→客户端**（客户端要发消息另发 HTTP 请求）。

**选型表**：

| 需求 | 选型 |
|------|------|
| 聊天/协同编辑/游戏/双向高频 | **WebSocket**（或 WebTransport/HTTP3） |
| 通知/进度/行情/日志流（单向） | **SSE**（简单、自动重连、易过网关） |
| 请求-响应为主、偶发推送 | 普通 HTTP + 轮询/长轮询 |
| 移动端弱网/多路复用要求高 | WebSocket over HTTP/2 或 WebTransport（生态评估） |

**易错**：把 WebSocket 当"普通 HTTP 接口"做鉴权（升级后无 Header 可用，握手时验）；SSE 忘了关代理缓冲（`proxy_buffering off`，否则"服务端在写、客户端不动"，2.2.1）；WebSocket 多实例不做广播（用户连到 A 实例、消息发到 B 实例丢失）；长连接不计入容量规划（连接数与内存，5_tomcat.md）。

## 5.2 调试与演进

### 5.2.1 HTTP 问题怎么高效定位？工具箱怎么用？

**四层工具（从协议到链路）**：
1. **curl**：`-v`（看请求/响应头与 TLS 握手）、`-i`（含响应头）、`--http2`/`--http3`（协议验证）、`--resolve host:443:ip`（绕 DNS 直连指定 IP，排查 DNS/证书）、`--cacert`（信任链）、`-w '%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %{time_total}'`（**分段计时**：DNS/建连/TLS/首字节/总耗时）；
2. **浏览器 DevTools**：Network 面板看**瀑布图**（排队/阻塞/连接/等待/下载）、请求头/响应头、Cookie、CORS 报错（控制台给的原因比猜准）、`Copy as cURL` 复现；Performance 看资源优先级；
3. **抓包**：`tcpdump`/Wireshark（TCP 重传、RST、TLS 握手失败）；HTTP/2/3 抓包需协议解码（h3 还要 TLS 密钥）；
4. **服务端日志**：Nginx access log 的 `$request_time`/`$upstream_response_time`/`$upstream_status`（区分"网关慢"还是"上游慢"，12_nginx.md 5.2.1）、应用日志的 traceId 串全链（15_spring_cloud.md 4.2.1）。

**排障心法**：**先定层再定因**——DNS 慢（`time_namelookup` 大）/建连慢（`time_connect` 大，网络或 backlog）/TLS 慢（`time_appconnect` 大，握手/证书/OCSP）/首字节慢（`time_starttransfer` 大，服务端处理）/下载慢（带宽/分块）。**"接口慢"先分清是"网络慢"还是"服务端慢"**，再进应用内部（DB/下游/GC）。

**易错**：只用浏览器测（缓存/CORS/代理差异掩盖问题）；不看上游耗时只看总耗时（502/504 定位错层）；抓包不看 TLS（以为明文能看懂）；在容器里 `curl localhost` 正常、外部不通（绑定地址/防火墙/发布端口，23_docker.md 2.2.2）。

### 5.2.2 HTTP 的未来演进方向是什么？

**四条主线（2026 视角）**：
1. **HTTP/3 普及**：CDN/大厂已大规模启用，浏览器默认协商——瓶颈从协议转向"企业网络禁 UDP"与中间件支持；**QUIC 生态**（用户态协议栈、可插拔拥塞控制）持续成熟；
2. **隐私增强**：**ECH（Encrypted Client Hello）** 加密 SNI（防"访问了谁"被中间网络看到），与 DNS over HTTPS/QUIC 一起构成"加密一切"的趋势；代价是网络治理/审计方式要变；
3. **双向流的新底座**：**WebTransport**（基于 HTTP/3 的多路双向流/数据报）在实时音视频、游戏、协同场景替代部分 WebSocket 需求；**MASQUE**（代理 over QUIC）用于 VPN/代理场景；
4. **语义稳定、传输演进**：HTTP 的**语义层（方法/状态码/缓存/头）保持稳定**——变的是传输与安全层。**"别追版本名词，追每一代解决的阻塞点"** 是答题的最高姿态（1.1.2 的主线）。

**一句话收口**："HTTP 的未来是**'语义不变、传输更快、隐私更强'**——业务开发者该关注的是缓存/连接/超时/安全这些语义层基本功，协议版本交给基础设施（CDN/网关/运行时）去演进。"