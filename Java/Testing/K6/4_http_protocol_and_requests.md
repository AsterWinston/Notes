# 1. HTTP 模块基础

## 1.1 常见请求方法
k6 提供 `http.get`、`http.post`、`http.put`、`http.del` 等方法：

```javascript
import http from "k6/http";

http.get("https://example.com/api");
http.post("https://example.com/api", JSON.stringify({ name: "n" }), {
  headers: { "Content-Type": "application/json" }
});
```

## 1.2 params 参数
常用参数包括 `headers`、`tags`、`cookies`、`redirects`、`timeout` 等：

```javascript
http.get("https://example.com", {
  headers: { "X-Trace-Id": "abc" },
  tags: { endpoint: "home" },
  timeout: "5s",
  redirects: 3
});
```

## 1.3 批量请求与并发
`http.batch` 可同时发起多个请求：

```javascript
http.batch([
  ["GET", "https://example.com/a"],
  ["GET", "https://example.com/b"],
  ["POST", "https://example.com/c", JSON.stringify({ id: 1 })]
]);
```

# 2. 会话与认证

## 2.1 Cookie 管理
- 默认每个 VU 有独立的 Cookie Jar
- 可通过 `http.cookieJar()` 自定义读写

```javascript
const jar = http.cookieJar();
jar.set("https://example.com", "sid", "abc");
```

## 2.2 认证方式
- Basic/Auth 或 Token 认证通常通过请求头实现
- 对短期 token 建议在 `setup` 获取并传入 VU 执行

```javascript
http.get("https://example.com/api", {
  headers: { Authorization: "Bearer token" }
});
```

# 3. 连接与协议

## 3.1 Keep-Alive 与连接复用
- 默认支持连接复用，减少握手开销
- 长连接可显著降低高并发压测的系统开销

## 3.2 HTTP/2
- 目标服务支持 HTTP/2 时可减少连接数
- 需结合服务端配置与链路支持情况

## 3.3 TLS 与双向认证
- 证书问题是常见失败原因
- 双向认证可通过脚本选项配置客户端证书

# 4. 响应处理

## 4.1 常用响应字段
- `status`：HTTP 状态码
- `body`：响应体
- `json()`：解析 JSON

```javascript
const res = http.get("https://example.com/api");
const data = res.json();
```

## 4.2 大响应与内存
- 不需要响应体时，可设置 `discardResponseBodies` 降低内存占用
- 仅对关键字段做解析，避免全量处理

## 4.3 响应时间构成
常见耗时包含 DNS 解析、连接建立、TLS 握手、等待首字节与内容接收等阶段，可通过内置指标分拆分析。

# 5. 性能与稳定性要点

## 5.1 超时与重试
- k6 不内置重试，需要脚本显式实现
- 超时设置过小会放大网络波动

## 5.2 DNS 与连接错误
- DNS 故障会导致整体延迟激增
- 连接错误需区分网络问题与服务端拒绝

## 5.3 资源与瓶颈
- 客户端本身也可能成为瓶颈（CPU、带宽）
- 观察 `http_req_blocked`、`http_req_connecting` 等指标判断瓶颈位置
