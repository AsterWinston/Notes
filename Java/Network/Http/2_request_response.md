理解 HTTP 报文结构，是排查接口问题、设计 API 和阅读框架源码的基础。本篇拆解请求行、状态行、头部和消息体，讲清常用方法与状态码的语义，并补充内容协商和分块传输等容易忽略的细节。

# 1. HTTP 报文结构

## 1.1 请求行与状态行

请求报文的第一行是请求行：

```http
GET /users/42?verbose=true HTTP/1.1
```

它由三部分组成：

- **方法**：`GET`，表示动作。
- **请求目标**：`/users/42?verbose=true`，包含路径和查询参数。
- **协议版本**：`HTTP/1.1`。

响应报文的第一行是状态行：

```http
HTTP/1.1 200 OK
```

- **协议版本**：`HTTP/1.1`。
- **状态码**：`200`，三位数字。
- **原因短语**：`OK`，给人阅读，程序不应依赖它做判断。

## 1.2 头部字段

头部是 `key: value` 形式，按用途大致分类：

| 类别 | 常见字段 | 作用 |
|------|----------|------|
| 通用头 | `Connection`、`Cache-Control`、`Date` | 请求和响应都可能出现 |
| 请求头 | `Host`、`Authorization`、`Accept`、`User-Agent` | 描述请求 |
| 响应头 | `Server`、`Set-Cookie`、`Location` | 描述响应 |
| 实体头 | `Content-Type`、`Content-Length`、`Content-Encoding` | 描述消息体 |

示例：

```http
GET /api/users HTTP/1.1
Host: example.com
Accept: application/json
Authorization: Bearer eyJhbGciOi...
User-Agent: curl/8.0
```

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 27
Cache-Control: no-store

{"id":42,"name":"alice"}
```

⚠ 头部字段名不区分大小写，但值通常区分。不要假设头部顺序固定，也不要把自定义头部的值当成可信输入。

## 1.3 消息体

消息体不是每个请求或响应都有：

- GET 通常没有请求体。
- POST、PUT、PATCH 通常有请求体。
- 204、304 响应没有消息体。
- HEAD 响应没有消息体，但头部与 GET 相同。

消息体长度有两种表达方式：

| 方式 | 含义 | 特点 |
|------|------|------|
| `Content-Length` | 消息体固定长度 | 简单，但要求提前知道总长度 |
| `Transfer-Encoding: chunked` | 分块传输 | 适合流式生成，不能同时使用 `Content-Length` |

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked

1a
{"message":"hello"}
0
```

`1a` 是十六进制长度，`0` 表示结束。分块传输让服务端可以在生成数据的同时发送，不必等全部内容准备好。

# 2. 方法与语义

## 2.1 GET、POST、PUT、PATCH、DELETE

| 方法 | 用途 | 请求体 | 幂等 | 安全 | 典型状态码 |
|------|------|--------|------|------|------------|
| GET | 查询资源 | 通常无 | 是 | 是 | 200、304、404 |
| POST | 创建资源或执行操作 | 通常有 | 否 | 否 | 201、200、400 |
| PUT | 整体替换资源 | 通常有 | 是 | 否 | 200、204、404 |
| PATCH | 部分更新资源 | 通常有 | 不一定 | 否 | 200、204、400 |
| DELETE | 删除资源 | 通常无 | 是 | 否 | 204、404 |

这里的"安全"不是指加密，而是指**不改变服务端状态**。GET 和 HEAD 是安全方法。

## 2.2 幂等性与安全性

**幂等**是指同一个请求执行一次和执行多次，对服务端状态的最终效果相同。

- `GET /users/42` 执行多次，结果一样，幂等。
- `DELETE /users/42` 第一次删除，第二次返回 404，但最终状态都是"用户不存在"，通常认为幂等。
- `POST /orders` 执行多次可能创建多个订单，不幂等。

★ **关键认知**：客户端超时后是否重试，核心判断是请求是否幂等。对非幂等请求盲目重试，可能造成重复下单、重复扣款。必要时使用幂等键（idempotency key）让服务端识别重复请求。

# 3. 状态码

## 3.1 2xx 与 3xx

| 状态码 | 含义 | 典型场景 |
|--------|------|----------|
| 200 OK | 成功 | 查询、更新成功 |
| 201 Created | 已创建 | POST 创建资源，配合 `Location` 头 |
| 204 No Content | 成功但无内容 | DELETE 成功 |
| 301 Moved Permanently | 永久重定向 | 域名迁移、HTTPS 跳转 |
| 302 Found | 临时重定向 | 登录跳转、临时地址 |
| 304 Not Modified | 资源未修改 | 协商缓存命中 |

301 会被浏览器和搜索引擎长期缓存，迁移域名时要谨慎。302 是临时跳转，后续请求仍访问原地址。

## 3.2 4xx 与 5xx

| 状态码 | 含义 | 常见原因 |
|--------|------|----------|
| 400 Bad Request | 请求格式错误 | JSON 解析失败、参数校验失败 |
| 401 Unauthorized | 未认证 | 缺少或无效的 Token |
| 403 Forbidden | 无权限 | 已登录但不能访问该资源 |
| 404 Not Found | 资源不存在 | ID 不存在、路由错误 |
| 409 Conflict | 资源冲突 | 用户名已存在、版本冲突 |
| 429 Too Many Requests | 请求过多 | 限流 |
| 500 Internal Server Error | 服务端错误 | 未捕获异常 |
| 502 Bad Gateway | 网关收到无效响应 | 上游服务崩溃 |
| 503 Service Unavailable | 服务不可用 | 过载、维护 |
| 504 Gateway Timeout | 网关等待超时 | 上游服务响应过慢 |

★ **关键认知**：401 表示"你还没证明你是谁"，403 表示"我知道你是谁，但你不能做这件事"。把两者混用会让客户端无法决定是跳登录页还是提示无权限。

# 4. 内容协商与编码

## 4.1 Content-Type

`Content-Type` 告诉接收方消息体是什么格式：

```http
Content-Type: application/json; charset=utf-8
Content-Type: application/x-www-form-urlencoded
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary
Content-Type: text/html; charset=utf-8
```

常见 MIME 类型：

| 类型 | 用途 |
|------|------|
| `application/json` | JSON 数据 |
| `application/xml` | XML 数据 |
| `application/x-www-form-urlencoded` | 普通表单 |
| `multipart/form-data` | 文件上传 |
| `text/plain` | 纯文本 |
| `image/png` | PNG 图片 |

客户端还可以用 `Accept` 表示希望接收的格式，服务端用 `Content-Type` 返回实际格式。服务端应根据 `Accept` 做内容协商，而不是永远返回同一种格式。

## 4.2 压缩与分块传输

`Content-Encoding: gzip` 表示消息体使用了 gzip 压缩。客户端需要解压后再解析。压缩可以显著减少文本响应体积，但会增加 CPU 开销。

```http
HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Length: 153
```

⚠ 使用压缩时，`Content-Length` 是压缩后的长度，不是原始长度。不要在压缩响应上手写原始长度。分块传输和 `Content-Length` 不能同时使用。

# 5. 常见错误

| 错误 | 问题 | 正确做法 |
|------|------|----------|
| 401 和 403 混用 | 客户端无法区分认证和授权 | 未认证 401，无权限 403 |
| GET 带请求体 | 语义不清晰，部分代理会丢弃 | 查询用 Query，复杂查询用 POST |
| Content-Length 不匹配 | 请求截断或挂起 | 由框架/客户端自动计算 |
| 所有响应都 200 | 监控和网关无法识别失败 | 使用语义正确的状态码 |
| 304 响应带 body | 违反协议，可能被客户端忽略 | 304 只返回头部 |
| Content-Type 缺失 | 接收方无法正确解析 | 明确设置类型和字符集 |

# 6. 动手练习

用 curl 完成以下练习，并解释返回结果：

```bash
# 1. 查看完整请求和响应头
curl -v https://httpbin.org/get

# 2. 发送 JSON POST
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"name":"alice"}'

# 3. 查看重定向
curl -I https://httpbin.org/redirect/1

# 4. 请求不存在的资源
curl -i https://httpbin.org/status/404
```

要求：写出每个请求的方法、路径、Content-Type、状态码和响应体类型。

# 7. 小结

| 主题 | 要点 |
|------|------|
| 报文结构 | 起始行 + 头部 + 空行 + 消息体 |
| 方法语义 | 区分安全与幂等，重试前先判断幂等性 |
| 状态码 | 2xx 成功、3xx 重定向、4xx 客户端错误、5xx 服务端错误 |
| Content-Type | 明确消息体格式和字符集 |
| 传输方式 | Content-Length 与 chunked 二选一 |
| 常见错误 | 401/403 混用、全 200、GET 带体、304 带体 |

下一篇进入连接管理，讲清三次握手、Keep-Alive、连接池、超时和重试。
