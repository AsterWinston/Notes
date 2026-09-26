# 1. 知识点
- **OSI七层模型**: 应用层(HTTP/DNS/FTP)、表示层(加密/压缩)、会话层(Session)、传输层(TCP/UDP)、网络层(IP)、数据链路层(MAC)、物理层
- **TCP/IP四层**: 应用层(HTTP)、传输层(TCP)、网络层(IP)、网络接口层。HTTP工作在TCP之上，HTTPS中TLS插入在应用层和传输层之间
- **HTTP请求报文格式**: 请求行(方法 URL 协议版本) + 请求头(Host/Cookie/User-Agent/Content-Type等) + 空行 + 请求体
- **HTTP响应报文格式**: 状态行(协议版本 状态码 状态描述) + 响应头(Content-Type/Content-Length/Set-Cookie等) + 空行 + 响应体
- **请求头Host必须**: HTTP/1.1强制要求Host头，用于虚拟主机(同一IP承载多域名)。HTTP/1.0可选
- **Cookie机制**: 服务端通过Set-Cookie响应头设置，客户端后续请求携带Cookie请求头。解决HTTP无状态问题(登录态维持)
- **User-Agent**: 标识客户端信息(浏览器类型/版本/操作系统)，用于兼容性判断和统计分析
- **HTTP无状态**: 每个请求独立，服务端不保存客户端上下文。通过Cookie/Session/JWT/Token补充状态
- **HTTP/1.1 Keep-Alive**: TCP连接复用，默认开启(Connection: keep-alive)。同一连接可发送多个HTTP请求/响应，减少TCP握手开销
- **HTTP/1.1管道化(Pipelining)**: 客户端在收到上一个响应前可连续发送多个请求(不等待响应)。但响应必须按请求顺序返回(队头阻塞Head-of-Line Blocking)，实际使用有限
- **HTTP/1.1队头阻塞**: 管道化要求响应按序返回，第一个请求慢会影响后续所有请求的响应。浏览器默认关闭管道化，改用多连接(6-8个并发)缓解
- **HTTP/2多路复用(Multiplexing)**: 同一TCP连接上同时传输多个Stream(流)，帧交错发送，接收方根据Stream ID重新组装，彻底解决队头阻塞
- **HTTP/2帧格式**: 二进制编码(HTTP/1.1是文本)，帧头9字节(长度/类型/标志/Stream ID)，数据载荷。帧类型：HEADERS帧、DATA帧、PRIORITY帧、RST_STREAM帧、SETTINGS帧、PUSH_PROMISE帧、PING帧、GOAWAY帧、WINDOW_UPDATE帧、CONTINUATION帧
- **HTTP/2 HEADER帧**: HTTP头部二进制编码压缩后封装发送
- **HTTP/2 DATA帧**: HTTP请求体/响应体数据封装发送
- **HTTP/2 Server Push(服务器推送)**: 服务器在客户端请求前主动推送资源。如客户端请求index.html，服务器同时推送style.css和app.js。用PUSH_PROMISE帧实现
- **HTTP/2 HPACK头部压缩**: 静态表(预定义的61个常见头部)和动态表(连接内动态添加)、哈夫曼编码。大幅减少重复头部传输(如Cookie、User-Agent每次相同)
- **HTTP/3(QUIC)**: 基于UDP实现的传输协议，解决TCP队头阻塞(TCP丢包整个窗口阻塞)。0-RTT建连(已连接过)、多路复用无队头阻塞、连接迁移(WiFi切4G不断连)
- **QUIC特性**: 基于UDP、前向安全(Forward Error Correction)、0-RTT/1-RTT握手、连接ID标识连接(支持网络切换)、内置TLS 1.3加密
- **HTTP/3优势**: TCP队头阻塞修复(单个流丢包不影响其他流)、0-RTT降低延迟、移动网络切换友好
- **Content-Type(Media Type/MIME)**: 标识请求/响应体的数据格式。常见：`application/json`、`application/x-www-form-urlencoded`、`multipart/form-data`、`text/html`、`image/png`
- **multipart/form-data**: 文件上传格式。boundary分隔符分隔不同表单字段，每个字段包含独立的Content-Disposition和Content-Type头
- **Range断点续传**: 客户端请求部分内容，`Range: bytes=0-1023`。服务端返回206 Partial Content，`Content-Range: bytes 0-1023/2048`。不满足返回416 Range Not Satisfiable
- **缓存-Cache-Control**: 强缓存控制。`max-age=3600`(缓存3600秒)、`no-cache`(需协商)、`no-store`(不缓存)、`public`(可被任何缓存缓存)、`private`(仅浏览器缓存)、`immutable`(资源不变)
- **缓存-ETag/If-None-Match**: 协商缓存。服务端返回资源的ETag(版本标识)，客户端下次请求带`If-None-Match: <ETag>`，资源未变返回304 Not Modified
- **缓存-Last-Modified/If-Modified-Since**: 协商缓存。服务端返回资源的Last-Modified时间，客户端下次请求带`If-Modified-Since: <time>`，资源未变返回304
- **强缓存 vs 协商缓存**: 强缓存(Cache-Control)命中时不发请求直接使用缓存(200 from disk cache)。协商缓存(ETag/Last-Modified)需要发请求，服务端返回304表示可用缓存
- **CDN缓存原理**: 用户→边缘节点(Edge Node)→源站(Origin)。边缘节点缓存静态资源，命中直接返回，未命中回源获取并缓存
- **常用请求头**: `X-Forwarded-For`(记录经过的代理IP链)、`X-Real-IP`(客户端真实IP，由代理设置)、`Accept`(期望的响应格式)、`Content-Encoding: gzip`(压缩算法)、`Connection: keep-alive`(持久连接)、`Transfer-Encoding: chunked`(分块传输编码)、`Authorization`(认证信息)
- **状态码**: 1xx信息(101 Switching Protocols WebSocket升级)、2xx成功(200 OK、201 Created、204 No Content、206 Partial Content)、3xx重定向(301永久、302临时、304 Not Modified、307临时不改变方法)、4xx客户端错误(400 Bad Request、401 Unauthorized、403 Forbidden、404 Not Found、405 Method Not Allowed、429 Too Many Requests)、5xx服务端错误(500 Internal Server Error、502 Bad Gateway、503 Service Unavailable、504 Gateway Timeout)

# 2. 面试题

## 2.1 HTTP/1.1的Keep-Alive和HTTP/2的多路复用有什么区别？
| 维度 | HTTP/1.1 Keep-Alive | HTTP/2 多路复用 |
|------|---------------------|-----------------|
| 连接复用方式 | 串行，一个请求完成才能发下一个(同一连接) | 并行，多个请求/响应同时传输(Stream交错) |
| 队头阻塞 | 存在(HOL Blocking) | HTTP层面解决(TCP仍有队头阻塞) |
| 并发连接数 | 浏览器限制6-8个并发连接 | 1个连接承载所有请求 |
| 文本/二进制 | 文本协议(明文请求行+头部) | 二进制帧(Binary Framing) |
| 头部压缩 | 无(每次发送完整头部) | HPACK压缩 |
| 服务器推送 | 不支持 | 支持(PUSH_PROMISE帧) |
| 连接开销 | 每个资源可能独占连接 | 全部复用1个连接 |

核心差异：HTTP/2通过帧(Stream ID)实现同连接的请求/响应交错，真正并行传输；HTTP/1.1 Keep-Alive只能串行，浏览器用多连接(6-8个)来近似并行。

## 2.2 HTTP/2解决了HTTP/1.1的哪些问题？还留有什么问题？
**解决的问题**：
1) 多路复用：消除HTTP/1.1管道化的队头阻塞(HOL Blocking)
2) 头部压缩：HPACK压缩节省带宽(尤其是Cookie等大请求头)
3) 二进制分帧：解析高效，减少文本解析开销
4) Server Push：主动推送资源减少往返
5) 流优先级：客户端可指定资源加载优先级
6) 单连接：减少TCP连接开销(TLS握手、拥塞控制启动慢)

**遗留问题(TCP层面)**：
1) TCP队头阻塞：TCP是可靠字节流，某个包丢失，所有后续包(包括其他Stream的)都需要等待重传
2) TCP连接迁移：切换网络(WiFi→4G)需要重新建立TCP连接
HTTP/3(QUIC)基于UDP解决了这些问题。

## 2.3 HTTP/3(QUIC)相比HTTP/2有哪些提升？
1) **UDP替代TCP**：QUIC基于UDP，彻底解决TCP队头阻塞。丢包只影响丢失的Stream，其他Stream不受影响
2) **0-RTT握手**：与已访问过的服务器重建连接，加密数据随第一个包直接发送，延迟极低
3) **连接迁移**：QUIC用Connection ID标识连接而非(IP+端口的四元组)，网络切换时不需要重建连接(移动端优势明显)
4) **前向纠错(FEC)**：发送冗余编码数据，丢失的包可以从冗余数据恢复，无需重传
5) **内置TLS 1.3**：加密是QUIC的强制组成部分，没有明文QUIC
6) **改进的拥塞控制**：基于应用层的灵活拥塞控制算法

## 2.4 缓存相关的HTTP头有哪些？工作原理是什么？
**强缓存**(优先级高)：
- `Cache-Control: max-age=3600` — 缓存3600秒，期间不发送请求
- `Cache-Control: no-cache` — 需要协商验证(非"不缓存"，Nginx中Same)
- `Cache-Control: no-store` — 真正不缓存
- `Cache-Control: private` — 仅浏览器缓存，中间代理不缓存
- `Cache-Control: public` — 任何中间节点都可缓存
- `Expires: Thu, 01 Dec 2022 16:00:00 GMT` — HTTP/1.0绝对时间(已被Cache-Control取代)

**协商缓存**(强缓存失效后)：
- `ETag` / `If-None-Match` — ETag是资源指纹(Hash)，精确但计算开销
- `Last-Modified` / `If-Modified-Since` — 资源最后修改时间，精度为秒

**工作流程**：请求资源→检查强缓存，命中直接用→未命中，检查协商缓存条件→发请求带If-None-Match/If-Modified-Since→服务端比较，未变返回304→客户端用缓存

## 2.5 什么是CDN？其缓存原理是什么？
CDN(Content Delivery Network，内容分发网络)是将静态资源部署到全球各地的边缘服务器，用户访问最近的节点获取资源。

**工作流程**：
1) 用户访问域名，DNS解析到CDN的智能调度系统
2) 调度系统根据用户IP、网络质量返回最优的边缘节点IP
3) 请求到达边缘节点，命中缓存直接返回(最快)
4) 未命中，边缘节点回源(Origin)拉取资源并缓存
5) 资源按缓存策略(Cache-Control/ETag)设置过期时间
6) 源站更新资源时刷新CDN缓存(Purge/接口调用)

**优点**：加速访问(就近访问)、减少源站压力、防护DDoS、跨运营商访问优化。

## 2.6 X-Forwarded-For和X-Real-IP有什么区别？
`X-Real-IP`：记录最原始的客户端IP，通常由第一层反向代理(Nginx直连用户的那一层)设置，只有一个IP值。

`X-Forwarded-For`：记录请求经过的所有代理IP链，格式为`客户端IP, 代理1 IP, 代理2 IP`。每经过一个代理，代理把自己看到的客户端IP追加到后面。可伪造(客户端可伪造)，需信任上游代理。

实际使用：Nginx配置`proxy_set_header X-Real-IP $remote_addr; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`。取真实IP时优先取经过信任代理过滤的X-Forwarded-For最左边IP。

## 2.7 Range断点续传是怎么实现的？
**流程**：
1) 客户端第一次请求资源，服务端返回200，响应头`Accept-Ranges: bytes`(告知支持断点续传)
2) 下载中断(如网络断开)
3) 客户端恢复下载时发送请求：`Range: bytes=1024-`(从1024字节开始到结束) 或 `bytes=0-1023,2048-4096`(请求多段)
4) 服务端响应206 Partial Content，响应头`Content-Range: bytes 1024-2047/2048`(当前范围/总大小)，响应体为该范围的字节
5) 客户端按序拼接所有片段，完成文件下载
如果Range不满足(如超出文件大小)，返回416 Range Not Satisfiable。

## 2.8 文件上传时multipart/form-data和application/json有什么区别？
`application/json`：整个请求体为一个JSON字符串。适合固定结构的小数据，不适合文件上传(需Base64编码，体积+33%)。

`multipart/form-data`：请求体按boundary分割为多个部分(Part)，每个Part有独立的Content-Disposition和Content-Type。Part可以是文本字段和文件字段。适合表单提交(含文件上传)。

示例请求体：
```
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg
[二进制文件数据]
------WebKitFormBoundary
Content-Disposition: form-data; name="description"
A picture
------WebKitFormBoundary--
```

## 2.9 Transfer-Encoding: chunked是什么？
分块传输编码，允许服务端将响应分块发送(动态内容生成时无需提前知道Content-Length)。HTTP/1.1仅支持（HTTP/2使用DATA帧，不需要chunked）。

格式：
```
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1A\r\n        # 16进制块大小(26字节)
Hello, this is the data\r\n
0\r\n          # 结束块(大小为0)
\r\n           # 尾随头和空行
```

用途：动态生成内容(如流式输出)、大文件传输(逐块发送减少内存占用)、反向代理(上游分块时无需缓冲完整响应)。在HTTP/2中data帧已支持流式传输，chunked仅用于HTTP/1.1。
