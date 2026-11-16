WebSocket 是浏览器里唯一被广泛支持的**全双工**通信方式，它把一个 HTTP 请求"升级"成一条长期存在的 TCP 连接，让服务端可以在任意时刻主动推送数据。本篇从 HTTP 的请求-响应模型讲起，说明 WebSocket 到底解决了什么问题，拆解 Upgrade 握手的每一个头部，并把 WebSocket 与 SSE、长轮询放在一起比较，最后给出连接生命周期与子协议协商的完整视图。

# 1. WebSocket 的定位

## 1.1 请求-响应模型的边界

HTTP 的基本假设是**客户端发起、服务端响应**。服务端永远处于被动位置，它没有办法在用户没有发请求时主动说一句话。这个模型适合读文章、查订单这类"拉取"场景，却在下面几类需求面前显得笨拙：

- **聊天与客服**：对方发来一条消息，服务端必须立刻送到你的浏览器。
- **协同编辑**：别人移动了光标、改了段落，本地要近乎实时地看到。
- **行情与监控**：价格、指标每秒都在变，客户端不该反复问"变了吗"。
- **游戏与弹幕**：延迟直接决定体验。

传统做法是让客户端不停发请求去"问"，也就是轮询。轮询能用，但代价是大量无意义的请求、延迟取决于轮询间隔、服务端连接数被迅速放大。

## 1.2 WebSocket 是什么

**WebSocket** 是 RFC 6455 定义的应用层协议，它复用 HTTP 的端口（`ws://` 默认 80，`wss://` 默认 443），在一条 TCP 连接上提供**双向、全双工、面向消息**的通道。三个关键词值得展开：

- **双向**：客户端和服务端都可以主动发数据，不再有"谁先开口"的限制。
- **全双工**：两个方向的数据可以同时流动，不需要排队等对方说完。
- **面向消息**：协议自带帧边界，应用层收到的是一条条完整消息，不必自己处理 TCP 的粘包与拆包。

握手阶段仍然是 HTTP，这就是它能穿过大多数企业防火墙、复用 443 端口和 TLS 证书的原因。握手一旦成功，后续通信就不再是 HTTP，而是 WebSocket 自己的帧协议。

> **行业实践**：生产环境几乎只用 `wss://`。明文 `ws://` 在浏览器地址栏是 HTTPS 时会被当作混合内容拦截，而且中间设备可以随意篡改帧内容。

## 1.3 什么时候不该用 WebSocket

WebSocket 不是"更高级的 HTTP"，它有自己的成本：连接要一直占着、需要心跳保活、需要处理重连与消息补偿、服务端扩容时还要解决跨实例广播。下面这些场景用普通 HTTP 更划算：

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 一次性查询、表单提交 | HTTP | 无持续通信需求 |
| 服务端单向推送、低频 | SSE | 实现简单，自带重连 |
| 大文件上传下载 | HTTP | WebSocket 不适合大块二进制搬运 |
| 请求-响应式 RPC | HTTP/gRPC | 语义清晰，易于治理 |
| 高频双向交互 | WebSocket | 正是它的主战场 |

**设计考量**：判断标准不是"要不要实时"，而是"**是否真的需要服务端主动、且双向频繁**"。如果只是服务端单向推送，SSE 往往能以更低的复杂度达到同样效果。

# 2. 与 SSE、长轮询的对比

## 2.1 短轮询与长轮询

**短轮询**是客户端每隔固定时间请求一次，服务端立即返回当前结果。实现最简单，但空请求多，实时性受间隔限制。

**长轮询**（long polling）是客户端发请求后，服务端把请求**挂住**，直到有新数据或超时才返回；客户端收到后立刻再发下一个请求。它把"空响应"压缩了，实时性也更好，但每条消息仍然要经历一次完整的 HTTP 请求，服务端需要为挂起的请求维持上下文。

```text
长轮询：
客户端                          服务端
  │ ─── GET /poll ────────────> │  请求挂起
  │                             │  等待新事件
  │ <── 200 新消息 ──────────── │
  │ ─── GET /poll ────────────> │  立刻再挂起
  │                             │
```

## 2.2 SSE

**SSE**（Server-Sent Events）基于普通 HTTP 响应，服务端用 `Content-Type: text/event-stream` 持续写出文本事件，浏览器通过 `EventSource` 接收。它天然支持自动重连和 `Last-Event-ID` 断点续传，但只有**服务端到客户端**一个方向，客户端要发消息还得另开 HTTP 请求。

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

id: 1024
event: price
data: {"symbol":"BTC","price":68000}

id: 1025
event: price
data: {"symbol":"ETH","price":3500}
```

## 2.3 横向对比

| 维度 | 短轮询 | 长轮询 | SSE | WebSocket |
|------|--------|--------|-----|-----------|
| 通信方向 | 单向拉取 | 单向拉取 | 服务端到客户端 | 双向 |
| 底层协议 | HTTP | HTTP | HTTP | HTTP 升级后自有协议 |
| 实时性 | 取决于间隔 | 较好 | 好 | 最好 |
| 消息类型 | 文本/二进制 | 文本/二进制 | 仅文本 | 文本/二进制 |
| 自动重连 | 无 | 需自行实现 | 浏览器内置 | 需自行实现 |
| 代理友好度 | 最好 | 好 | 好 | 需允许 Upgrade |
| 服务端连接成本 | 低 | 高 | 中 | 中 |
| 实现复杂度 | 低 | 中 | 低 | 高 |

> **行业实践**：行情推送、日志流、AI 回答流式输出这类单向场景，很多团队首选 SSE；聊天、协同、在线状态这类双向场景才用 WebSocket。二者可以共存，不必强行统一。

# 3. HTTP Upgrade 握手

## 3.1 握手请求

WebSocket 连接从一个标准 HTTP/1.1 请求开始，关键是 `Upgrade` 与 `Connection` 两个头部：

```http
GET /chat?room=42 HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat.v1, chat.v2
Origin: https://example.com
```

逐个解释：

| 头部 | 作用 | 注意点 |
|------|------|--------|
| `Upgrade: websocket` | 声明要升级的协议 | 值不区分大小写，但惯例小写 |
| `Connection: Upgrade` | 告诉代理这次连接要变 | 必须包含 `Upgrade` 令牌 |
| `Sec-WebSocket-Key` | 16 字节随机数的 Base64 | 每次连接都不同，防缓存与误判 |
| `Sec-WebSocket-Version` | 协议版本 | 现代实现固定为 13 |
| `Sec-WebSocket-Protocol` | 客户端支持的子协议 | 逗号分隔，服务端选一个回写 |
| `Origin` | 发起页面的来源 | 服务端应校验，防跨站劫持 |

这里没有 `Content-Length`，也没有请求体。握手请求本身就是一个"请把这条连接交给我"的协商。

## 3.2 握手响应

服务端同意升级时返回 `101 Switching Protocols`：

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat.v1
```

`101` 是唯一表示"协议切换"的状态码。响应发出后，这条 TCP 连接上的字节流就不再按 HTTP 解析，而是按 WebSocket 帧解析。如果服务端拒绝，可以返回 `400`、`403`、`426` 等普通状态码，连接按 HTTP 语义结束。

**注意**：握手响应里**不能**出现 `Content-Length` 或 `Transfer-Encoding`。有些反向代理会自作主张补上，导致客户端解析异常，排查时要看原始响应。

## 3.3 Sec-WebSocket-Accept 的推导

`Sec-WebSocket-Accept` 不是随机值，而是对客户端 `Sec-WebSocket-Key` 做固定运算的结果：

```text
accept = Base64( SHA-1( key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" ) )
```

那串 GUID 是 RFC 6455 规定的固定魔法字符串。它的作用不是加密，而是证明服务端**真的理解 WebSocket 协议**，而不是某个把 `Upgrade` 头原样转发的中间件。下面用 Java 复现这段计算：

```java
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.util.Base64;

public final class AcceptKey {
    private static final String GUID = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11";

    public static String compute(String clientKey) throws Exception {
        // 拼接固定 GUID，这是 RFC 6455 规定的握手校验算法
        String source = clientKey + GUID;
        MessageDigest sha1 = MessageDigest.getInstance("SHA-1");
        byte[] digest = sha1.digest(source.getBytes(StandardCharsets.US_ASCII));
        // 对 20 字节摘要做 Base64，得到响应头里的值
        return Base64.getEncoder().encodeToString(digest);
    }

    public static void main(String[] args) throws Exception {
        // RFC 6455 官方示例，结果应为 s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
        System.out.println(compute("dGhlIHNhbXBsZSBub25jZQ=="));
    }
}
```

## 3.4 观察一次真实握手

用 `curl` 可以手工发起握手，观察服务端是否返回 101：

```bash
# -i 打印响应头，-N 关闭缓冲，-H 逐条附加握手头
curl -i -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  "https://echo.websocket.org/.ws"
```

如果服务端正常，会看到 `HTTP/1.1 101 Switching Protocols`。之后 `curl` 会停在连接上等待字节，这正说明 HTTP 阶段已经结束。想看帧内容，需要用支持 WebSocket 的客户端，或者直接读下一章的帧格式。

> **注意**：很多网关默认只允许 `GET`/`POST`，并且会剥离 `Upgrade` 头。部署后握手失败，先确认 Nginx、SLB、Ingress 有没有显式放行 `Upgrade` 与 `Connection`。

# 4. 连接生命周期

## 4.1 四个状态

浏览器 `WebSocket` 对象暴露了 `readyState`，服务端实现也通常有对应的状态机：

```text
        new WebSocket(url)
                │
                ▼
        ┌──────────────┐   握手成功    ┌──────────────┐
        │  CONNECTING  │ ───────────> │     OPEN     │
        │     (0)      │              │     (1)      │
        └──────┬───────┘              └──────┬───────┘
               │ 握手失败                      │ close() 或对端关闭
               ▼                              ▼
        ┌──────────────┐              ┌──────────────┐
        │    CLOSED    │ <─────────── │   CLOSING    │
        │     (3)      │   关闭完成    │     (2)      │
        └──────────────┘              └──────────────┘
```

- **CONNECTING**：握手请求已发出，尚未收到 101。
- **OPEN**：可以双向收发消息。
- **CLOSING**：已发出关闭帧，等待对方回应。
- **CLOSED**：TCP 连接已断开。

只有 `OPEN` 状态可以调用 `send`。在 `CONNECTING` 阶段发消息，浏览器会抛 `InvalidStateError`；在 `CLOSING`/`CLOSED` 阶段发消息，数据会被静默丢弃或抛异常，取决于实现。

## 4.2 子协议协商

`Sec-WebSocket-Protocol` 用于在一条连接上约定"消息长什么样"。比如聊天业务可能同时存在 JSON 和 Protobuf 两种编码，客户端按优先级列出 `chat.protobuf, chat.json`，服务端选一个自己支持的写回响应头。协商失败时服务端应拒绝握手，而不是随便选一个。

**设计考量**：子协议是应用层约定，WebSocket 协议本身不关心内容。团队内部应把子协议名当成 API 契约管理，避免"同名不同格式"造成线上解析错误。

## 4.3 扩展协商

`Sec-WebSocket-Extensions` 用于协商压缩等扩展，最常见的是 `permessage-deflate`。它把每条消息独立压缩，能显著降低文本消息体积，但会增加 CPU 开销，也容易在某些代理上触发兼容问题。

```http
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

> **行业实践**：内网高吞吐场景可以开压缩；面向公网、消息本身很小的场景（如心跳、短指令）收益有限，反而增加排错难度，很多团队选择关闭。

# 5. 常见错误

| 错误 | 症状 | 正确做法 |
|------|------|----------|
| 用 `ws://` 部署在 HTTPS 页面 | 浏览器报混合内容被拦截 | 全站使用 `wss://` |
| 网关未放行 Upgrade | 握手返回 200 或 400 | 配置 `proxy_set_header Upgrade $http_upgrade` |
| 响应补了 `Content-Length` | 客户端解析帧失败 | 检查代理是否改写 101 响应 |
| 不校验 `Origin` | 任意站点可带用户 Cookie 建连 | 服务端白名单校验 Origin |
| 子协议写了但服务端不回 | 客户端认为协商失败 | 明确回写选中的协议 |
| 握手超时设置过短 | 弱网下频繁失败 | 握手超时单独配置，通常 5-10 秒 |

# 6. 动手练习

**练习一**：用 `curl` 对本地服务发起握手，分别去掉 `Sec-WebSocket-Key` 和 `Sec-WebSocket-Version`，观察服务端返回的状态码，并解释原因。

**练习二**：实现 3.3 节的 `AcceptKey`，用 RFC 6455 给出的示例 key 验证输出是否为 `s3pPLMBiTxaQ9kYGzzhZRbK+xOo=`。

**练习三**：画一张表，把你熟悉的三个业务分别归类到短轮询、SSE、WebSocket，并说明如果换成另一种方案会付出什么代价。

自检问题：WebSocket 握手为什么必须基于 HTTP？`Sec-WebSocket-Key` 的作用是加密吗？`101` 和 `200` 在语义上有什么区别？

# 7. 小结

| 主题 | 要点 |
|------|------|
| 定位 | 单 TCP 上的双向、全双工、面向消息通道 |
| 握手 | HTTP/1.1 Upgrade，服务端返回 101 |
| 关键头 | `Upgrade`、`Connection`、`Sec-WebSocket-Key`、`Sec-WebSocket-Version` |
| 校验 | Accept = Base64(SHA-1(key + 固定 GUID)) |
| 对比 | 双向高频选 WebSocket，单向推送可优先 SSE |
| 生命周期 | CONNECTING → OPEN → CLOSING → CLOSED |

握手只是把门打开，真正传输数据靠的是 WebSocket 帧。下一篇进入帧协议，逐字节拆解帧头、opcode、掩码、分片和关闭码。
