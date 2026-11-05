本篇把 HTTP 模块的知识串成一张地图，用自检问题和排查工具帮助复习。重点不是背状态码，而是能在接口超时、跨域失败、证书报错时快速定位问题所在层。

# 1. 知识地图

```text
┌──────────────────────────────────────────────────────┐
│                      HTTP 模块                       │
├──────────────┬──────────────┬────────────────────────┤
│ 协议与报文   │ 连接与性能   │ 安全与浏览器           │
│ 请求/响应    │ Keep-Alive   │ HTTPS / TLS            │
│ 方法/状态码  │ 连接池       │ 缓存 / Cookie          │
│ HTTP/1.1/2/3 │ 超时 / 重试  │ CORS / XSS / CSRF      │
├──────────────┴──────────────┴────────────────────────┤
│               Java HTTP 客户端实践                    │
│      HttpClient / OkHttp / Apache HttpClient         │
└──────────────────────────────────────────────────────┘
```

学习顺序建议：

1. 先理解请求和响应长什么样。
2. 再理解连接如何建立、复用和超时。
3. 然后理解 HTTPS 如何保证安全。
4. 接着理解浏览器缓存、Cookie 和跨域。
5. 最后把知识落到 Java 客户端封装。

# 2. 高频问题自检

## 2.1 协议基础

1. HTTP 请求报文由哪几部分组成？
2. GET 和 POST 的语义差异是什么？GET 一定不能有请求体吗？
3. 401 和 403 的区别是什么？
4. 301 和 302 的区别是什么？
5. `Content-Length` 和 `Transfer-Encoding: chunked` 能同时使用吗？
6. 什么是幂等？哪些方法通常幂等？

参考答案要点：

- 起始行、头部、空行、消息体。
- GET 语义是获取资源，通常无体；协议不禁止，但代理和缓存可能不友好。
- 401 未认证，403 已认证但无权限。
- 301 永久重定向会被缓存，302 临时重定向。
- 不能同时使用，二者都是消息体边界表达方式。
- 同一请求执行多次，最终状态效果相同；GET、PUT、DELETE 通常幂等。

## 2.2 连接与性能

1. 为什么 HTTP/1.1 需要多个连接？
2. Keep-Alive 复用了什么？它和 Session 是一回事吗？
3. 连接超时和读取超时的区别是什么？
4. HTTP/2 的多路复用解决了什么问题？没有解决什么问题？
5. 什么情况下可以重试？什么情况下不能重试？
6. 连接池越大越好吗？

参考答案要点：

- 应用层队头阻塞和浏览器并发连接限制。
- 复用 TCP 连接，不是会话状态。
- 连接超时覆盖握手，读取超时覆盖等待响应数据。
- 解决应用层队头阻塞，未解决 TCP 丢包导致的底层阻塞。
- 幂等或明确可恢复的瞬时错误可以重试；非幂等写操作不能盲目重试。
- 不是，应匹配下游容量。

## 2.3 HTTPS 与安全

1. HTTPS 提供了哪三个安全目标？
2. 证书链是如何验证的？
3. keystore 和 truststore 分别存什么？
4. CORS 预检请求什么时候触发？
5. CORS 能防 CSRF 吗？
6. XSS、CSRF、SSRF 的防御重点分别是什么？

参考答案要点：

- 机密性、完整性、身份认证。
- 服务器证书 → 中间 CA → 根 CA，逐级验证签名。
- keystore 存自己的私钥和证书，truststore 存信任的 CA。
- 非简单请求，例如 PUT、DELETE、自定义头、JSON Content-Type。
- 不能，CORS 限制读取响应，CSRF 只要求请求被发送。
- XSS 靠输出编码和 CSP，CSRF 靠 SameSite/Token，SSRF 靠 URL 白名单和内网拦截。

# 3. 排查工具

## 3.1 curl

```bash
# 查看请求和响应头
curl -v https://example.com

# 只看响应头
curl -I https://example.com

# 发送 JSON
curl -X POST https://example.com/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"alice"}'

# 查看耗时分解
curl -w "@curl-format.txt" -o /dev/null -s https://example.com

# 查看 TLS 证书
openssl s_client -connect example.com:443 -servername example.com </dev/null
```

`curl-format.txt` 示例：

```text
time_namelookup:  %{time_namelookup}\n
time_connect:     %{time_connect}\n
time_appconnect:  %{time_appconnect}\n
time_starttransfer: %{time_starttransfer}\n
time_total:       %{time_total}\n
```

## 3.2 浏览器 DevTools

Network 面板重点看：

- **Headers**：请求方法、URL、状态码、请求头、响应头。
- **Payload**：请求体内容。
- **Response**：响应体。
- **Timing**：DNS、连接、TLS、等待、下载各阶段耗时。
- **Protocol**：HTTP/1.1、h2、h3。
- **Cookies**：请求和响应中的 Cookie。

排查跨域时，先看 Console 的 CORS 报错，再看 Network 中 OPTIONS 请求的状态码和响应头。

## 3.3 Wireshark

常用过滤条件：

```text
tcp.port == 443
tls.handshake
http.request
http.response
ip.addr == 192.168.1.10
```

Wireshark 适合分析：

- TCP 重传、乱序、RST。
- TLS 握手失败。
- 明文 HTTP 请求内容。

但 HTTPS 内容默认加密，看不到业务报文，除非有会话密钥。生产环境抓包要注意合规和隐私。

# 4. 实践清单

接口开发：

- 方法和状态码语义正确。
- Content-Type 和字符集明确。
- 错误响应结构统一。
- 参数校验在入口完成。
- 超时、重试、幂等策略明确。

客户端调用：

- 连接超时和请求超时都设置。
- 客户端实例复用。
- 响应体正确关闭。
- 只对幂等请求重试。
- 日志脱敏。

安全配置：

- 全站 HTTPS，证书自动续期。
- Cookie 设置 HttpOnly、Secure、SameSite。
- CORS 来源明确，不使用 `*` + 凭证。
- 输出编码防 XSS。
- 敏感 URL 做 SSRF 白名单。
- 安全响应头统一配置。

# 5. 小结

| 主题 | 核心记忆 |
|------|----------|
| 协议 | 报文、方法、状态码、内容协商 |
| 连接 | 三次握手、Keep-Alive、连接池、超时 |
| 性能 | HTTP/2 多路复用，HTTP/3 基于 QUIC |
| 安全 | HTTPS、CORS、XSS、CSRF、SSRF |
| 客户端 | 复用实例、设置超时、统一异常、谨慎重试 |
| 排查 | curl 看链路，DevTools 看浏览器，Wireshark 看底层 |

HTTP 模块到此结束。下一步可以进入 `Java/Network/TCP` 学习传输层细节，或进入 Spring MVC 学习服务端框架如何封装 HTTP。
