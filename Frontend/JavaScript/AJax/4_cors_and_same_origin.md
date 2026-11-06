跨域是前端调试时最常见的红字来源。很多人第一反应是"后端加个 `*` 就好了"，但 `*` 为什么有时无效、预检为什么多出一次 OPTIONS、带了 Cookie 为什么反而失败，都需要回到同源策略和 CORS 的规则本身。本篇把浏览器这套"允许发、不一定允许读"的机制讲清楚。

# 1. 同源策略

## 1.1 什么是"源"

**源(origin)** 由三部分组成：协议、主机、端口。三者完全相同才算同源：

| 页面地址 | 请求地址 | 是否同源 | 原因 |
|----------|----------|----------|------|
| `https://a.com/page` | `https://a.com/api` | 是 | 协议、主机、端口都相同 |
| `https://a.com` | `http://a.com` | 否 | 协议不同 |
| `https://a.com` | `https://api.a.com` | 否 | 主机不同（子域也算不同） |
| `https://a.com` | `https://a.com:8443` | 否 | 端口不同 |
| `http://localhost:3000` | `http://localhost:8080` | 否 | 端口不同 |

★ **关键认知**：`http://a.com` 与 `https://a.com` 默认端口不同(80 与 443)，所以**升级 HTTPS 也会造成跨域**。⚠ `localhost` 与 `127.0.0.1` 在浏览器看来也是两个不同的主机，开发时混用会莫名跨域。

## 1.2 同源策略限制什么

同源策略是浏览器的安全边界，主要限制三类行为：

| 行为 | 是否受限制 | 说明 |
|------|------------|------|
| 读取跨域响应内容 | 是 | AJAX/fetch 拿不到正文 |
| 读取跨域 Cookie/Storage | 是 | 按源隔离 |
| 操作跨域 iframe DOM | 是 | 不能读对方文档 |
| 发送跨域请求 | 否 | 请求能发出，只是响应可能读不到 |
| 加载图片、脚本、样式 | 否 | 标签本身可跨域加载 |

★ **关键认知**：CORS 不是"阻止请求发出"，而是**阻止脚本读取响应**。⚠ 简单请求的场景下，服务器可能已经处理了业务(比如真的删了一条数据)，只是浏览器不把结果交给 JS。所以排查跨域时不要以为"请求没到后端"。

## 1.3 哪些标签天然可跨域

```html
<img src="https://cdn.example.com/a.png">
<script src="https://cdn.example.com/a.js"></script>
<link rel="stylesheet" href="https://cdn.example.com/a.css">
```

这些标签在引入资源时不受同源策略限制，但脚本**读不到**它们返回的原始内容(图片像素、脚本源码)，这正是 JSONP 能存在、又只能传"可执行 JS"的原因。

# 2. 跨域请求的三种类型

## 2.1 简单请求

满足以下所有条件的请求是"简单请求"，浏览器直接发出，不预检：

| 条件 | 允许值 |
|------|--------|
| 方法 | `GET`、`HEAD`、`POST` |
| 请求头 | 只含 CORS 安全列表头，如 `Accept`、`Accept-Language`、`Content-Language`、`Range` |
| `Content-Type` | `application/x-www-form-urlencoded`、`multipart/form-data`、`text/plain` |
| 其他 | 没有上传事件监听、body 不是 `ReadableStream` |

```javascript
// 简单请求：表单编码 + POST
fetch("https://api.example.com/login", {
  method: "POST",
  headers: { "Content-Type": "application/x-www-form-urlencoded" },
  body: "name=zhangsan&password=123",
});
```

⚠ **关键认知**：`Content-Type: application/json` 会**触发预检**，因为它不在简单请求的白名单里。很多人以为"POST 就是简单请求"，结果因为发了 JSON 多出一次 OPTIONS。

## 2.2 预检请求

不满足简单请求条件时，浏览器先发一个 `OPTIONS` 请求问服务器"允不允许"：

```text
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: content-type, authorization
```

服务器需要用响应头回答：

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

预检通过后，浏览器才发送真正的请求。`Access-Control-Max-Age` 告诉浏览器可以缓存这次预检结果多少秒，避免每个请求都多一次往返。

★ **关键认知**：预检是**浏览器自动发起**的，业务代码无法手动控制，也看不到它的完整细节(Network 面板里能看到)。⚠ 预检请求**不携带 Cookie**，所以它不需要 `Access-Control-Allow-Credentials`；真正需要凭证的是预检之后的那次实际请求。

## 2.3 带凭证请求

```javascript
fetch("https://api.example.com/me", {
  credentials: "include", // 跨域也带上 Cookie
});
```

服务器必须同时满足：

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

★ **关键认知**：一旦带凭证，`Access-Control-Allow-Origin` **不能是 `*`**，必须是具体的源。⚠ 这是"我明明配了 `*` 还是跨域失败"的头号原因，浏览器会直接拒绝这种组合。

# 3. CORS 响应头详解

## 3.1 Access-Control-Allow-Origin

```http
Access-Control-Allow-Origin: https://app.example.com
```

只允许一个源，或者 `*`。需要允许多个源时，服务器要根据请求的 `Origin` 动态回写，并配合 `Vary: Origin` 避免缓存串源：

```javascript
// Node/Express 伪代码
const allowed = ["https://a.com", "https://b.com"];
const origin = req.headers.origin;
if (allowed.includes(origin)) {
  res.setHeader("Access-Control-Allow-Origin", origin);
  res.setHeader("Vary", "Origin"); // 告诉缓存按 Origin 区分
}
```

## 3.2 Allow-Methods 与 Allow-Headers

```http
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization, X-Trace-Id
```

`Allow-Headers` 必须覆盖实际请求里所有**非安全列表**的请求头。漏写 `Authorization` 是最常见的预检失败原因。

## 3.3 Expose-Headers

默认情况下，跨域响应里 JS 只能读到少数安全列表头(`Content-Type`、`Content-Length`、`Cache-Control` 等)。要读自定义头，服务器必须显式暴露：

```http
Access-Control-Expose-Headers: X-Total-Count, X-Request-Id
```

```javascript
const response = await fetch("https://api.example.com/items");
console.log(response.headers.get("X-Total-Count")); // 未暴露时为 null
```

## 3.4 Allow-Credentials

```http
Access-Control-Allow-Credentials: true
```

它只对"带凭证的实际请求"有意义。⚠ 把它设成 `true` 却继续返回 `Access-Control-Allow-Origin: *`，浏览器会判定配置非法，等于没配。

## 3.5 Max-Age

```http
Access-Control-Max-Age: 600
```

单位是秒。值越大，预检缓存越久，请求越快；但权限策略变更后生效也越慢。开发环境建议设小一点，避免改了配置浏览器还拿旧缓存。

# 4. 预检的完整流程

```text
浏览器                                  服务器
   │                                      │
   │ 1. OPTIONS /api/users                │
   │    Origin: https://app.example.com   │
   │    Access-Control-Request-Method: PUT│
   │    Access-Control-Request-Headers:   │
   │      content-type, authorization     │
   │─────────────────────────────────────▶│
   │                                      │
   │ 2. 204 + CORS 响应头                 │
   │◀─────────────────────────────────────│
   │                                      │
   │ 3. 浏览器检查响应头是否放行           │
   │    通过 ──▶ 发真正的 PUT 请求 ──────▶│
   │    失败 ──▶ 控制台报 CORS 错误        │
   │                                      │
   │ 4. 实际响应 + CORS 头                │
   │◀─────────────────────────────────────│
   │                                      │
   │ 5. 再次检查 Allow-Origin 等           │
   │    通过 ──▶ 把响应交给 JS             │
```

★ **关键认知**：预检通过**不代表**实际请求一定放行，浏览器会对实际响应再做一次校验。⚠ 所以排查时要同时看两次请求的响应头，只看 OPTIONS 的 204 容易误判。

# 5. 常见报错与排查

## 5.1 No 'Access-Control-Allow-Origin' header

```text
Access to fetch at 'https://api.example.com' from origin
'https://app.example.com' has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

含义：实际响应里没有 `Allow-Origin`，或值与当前源不匹配。

排查顺序：

1. Network 面板确认请求 URL、`Origin` 是否正确。
2. 看响应头是否真的带了 `Access-Control-Allow-Origin`。
3. 检查是否被重定向到了另一个域，最终响应缺头。
4. 检查网关、Nginx、CDN 是否覆盖或删除了该头。

## 5.2 Preflight 失败

```text
Response to preflight request doesn't pass access control check:
It does not have HTTP ok status.
```

常见原因：

| 原因 | 处理 |
|------|------|
| 服务器没处理 `OPTIONS` | 为路由注册 OPTIONS 响应 |
| `Allow-Methods` 少了实际方法 | 补上 `PUT`/`PATCH` 等 |
| `Allow-Headers` 少了自定义头 | 补上 `Authorization` 等 |
| 预检返回 3xx/4xx | 预检必须返回 2xx |

## 5.3 带凭证时的通配符冲突

```text
The value of the 'Access-Control-Allow-Origin' header
in the response must not be the wildcard '*'
when the request's credentials mode is 'include'.
```

处理：把 `*` 换成请求的具体 `Origin`，并加 `Access-Control-Allow-Credentials: true`。

## 5.4 简单请求也失败

即使没有预检，浏览器仍会检查实际响应。⚠ 典型误区是"我没发预检，所以服务器不用配 CORS"——简单请求同样需要 `Allow-Origin`，只是少了 OPTIONS 那一步。

# 6. 开发环境怎么办

## 6.1 用开发服务器代理

前端开发服务器把 `/api` 转发到后端，浏览器看到的是同源请求：

```javascript
// Vite 配置示例
export default {
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:8080",
        changeOrigin: true, // 把 Host 头改成目标服务器
        rewrite: (path) => path.replace(/^\/api/, ""),
      },
    },
  },
};
```

★ **关键认知**：代理能解决开发期跨域，是因为**请求由开发服务器转发**，浏览器只和同源的开发服务器通信。⚠ 生产环境没有这个代理，要么后端正确配置 CORS，要么用同域部署/网关转发。

## 6.2 不要依赖关闭浏览器安全策略

用 `--disable-web-security` 启动浏览器只能骗过自己，线上依旧失败，而且会掩盖真正的配置问题。

# 7. 练习

1. 列出三个地址，判断它们与 `https://shop.example.com` 是否同源，并说明原因。
2. 给一个使用 `PUT + application/json + Authorization` 的请求画出预检流程，写出服务器应返回的关键响应头。
3. 模拟"带 Cookie 跨域"场景，解释为什么 `Allow-Origin: *` 会失败，并给出两种修复方案。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 同源 | 协议、主机、端口三者全同 |
| 同源策略 | 限制读响应，不阻止请求发出 |
| 简单请求 | 方法、头、Content-Type 都在白名单内 |
| 预检 | 非简单请求先发 OPTIONS，需 2xx 与正确响应头 |
| 凭证 | `credentials: include` 时 Origin 不能为 `*` |
| 暴露头 | 自定义响应头需 `Expose-Headers` |
| 开发期 | 优先用代理，不要关闭浏览器安全策略 |

下一篇进入 JSON 与表单，讲清楚请求体到底该用什么格式、`Content-Type` 如何影响服务端解析，以及 `FormData` 上传文件的细节。
