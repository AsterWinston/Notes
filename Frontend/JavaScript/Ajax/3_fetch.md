`fetch` 是现代浏览器原生的请求 API：它把一次 HTTP 交互拆成"拿到响应"和"读取正文"两个 Promise，配合 `async/await` 写起来接近同步代码。但它并不是 XHR 的简单替代——**HTTP 404 不会让 fetch 失败**、正文只能读一次、上传进度无处监听。本篇把这些语义一条条拆开。

# 1. fetch 的基本模型

## 1.1 最小示例

```javascript
async function loadUser(id) {
  const response = await fetch(`/api/users/${id}`); // 第一段：拿到响应头
  if (!response.ok) {                               // 4xx/5xx 必须自己判断
    throw new Error(`HTTP ${response.status}`);
  }
  const user = await response.json();               // 第二段：读取并解析正文
  return user;
}
```

★ **关键认知**：`fetch` 返回的 Promise 在**响应头到达**时就 resolve，此时正文可能还没下载完；`response.json()` 返回的是**第二个 Promise**。⚠ 这就是为什么总要写两次 `await`，也是"为什么我只 await 了一次却拿不到数据"的根源。

## 1.2 两段式 Promise 的意义

```text
fetch(url)
   │  发出请求
   ▼
Promise<Response>        ← 收到状态行和响应头就 resolve
   │  response.json() / text() / blob()
   ▼
Promise<数据>            ← 正文全部下载并解析后 resolve
```

这种设计让"先看状态码再决定要不要下载正文"成为可能，也为流式读取留出了入口。代价是调用方必须理解这两个阶段。

★ **关键认知**：`fetch` 的 resolve 只代表"服务器给了响应"，**不代表请求成功**。判断业务成功要看 `response.ok` 或 `response.status`。⚠ 这是从 XHR 迁移过来时最容易出错的地方——XHR 至少还会让你写 `if (status === 200)`。

# 2. Response 对象

## 2.1 状态与元信息

```javascript
const response = await fetch("/api/users");

response.status;      // 200
response.statusText;  // "OK"
response.ok;          // status 在 200-299 之间为 true
response.url;         // 最终请求地址（考虑重定向后）
response.redirected;  // 是否发生过重定向
response.type;        // "basic"、"cors"、"opaque" 等
response.headers.get("content-type"); // 读取响应头
```

| 属性/方法 | 说明 |
|-----------|------|
| `status` | HTTP 状态码 |
| `ok` | `status >= 200 && status < 300` |
| `headers` | `Headers` 对象，大小写不敏感 |
| `url` | 最终 URL |
| `body` | `ReadableStream` 或 `null` |
| `bodyUsed` | 正文是否已被读取 |

## 2.2 读取正文的五种方法

```javascript
await response.text();        // 字符串
await response.json();        // 解析 JSON，非法时抛 SyntaxError
await response.blob();        // Blob，适合图片、文件下载
await response.arrayBuffer(); // ArrayBuffer，适合二进制处理
await response.formData();    // 解析 multipart 表单
```

★ **关键认知**：响应正文是**一次性流**，上面五个方法只能调用其中一个，且**只能调用一次**。⚠ 第二次调用会抛 `TypeError: body stream already read`，这不是 bug，而是流被消费完了。

## 2.3 bodyUsed 与 clone

```javascript
const response = await fetch("/api/data");

if (!response.bodyUsed) {
  const copy = response.clone(); // 先克隆，两份各自读
  const text = await response.text();
  const data = await copy.json();
}
```

`clone()` 会把尚未读取的流分成两支，代价是浏览器需要缓冲内容。⚠ 如果正文很大，克隆会占用额外内存；更好的做法是**读一次原始文本，再决定怎么解析**：

```javascript
const raw = await response.text();
try {
  const data = JSON.parse(raw);
} catch {
  console.error("返回的不是 JSON：", raw.slice(0, 200));
}
```

# 3. 请求配置

## 3.1 常用字段

```javascript
const response = await fetch("/api/users", {
  method: "POST",                       // 默认 GET
  headers: {
    "Content-Type": "application/json", // 声明正文格式
    "Authorization": "Bearer token",    // 认证信息
  },
  body: JSON.stringify({ name: "张三" }), // GET/HEAD 不能带 body
  mode: "cors",                         // 跨域模式
  credentials: "same-origin",           // Cookie 策略
  cache: "no-cache",                    // 缓存策略
  redirect: "follow",                   // 重定向策略
  signal: controller.signal,            // 取消信号
});
```

| 字段 | 常用值 | 说明 |
|------|--------|------|
| `method` | GET/POST/PUT/PATCH/DELETE | 默认 GET |
| `headers` | 对象或 `Headers` | 自定义请求头 |
| `body` | 字符串/FormData/Blob/... | GET、HEAD 不允许 |
| `mode` | `cors`/`same-origin`/`no-cors` | 控制跨域行为 |
| `credentials` | `omit`/`same-origin`/`include` | 是否携带 Cookie |
| `cache` | `default`/`no-store`/`no-cache`/`force-cache` | 缓存策略 |
| `signal` | `AbortSignal` | 取消请求 |

## 3.2 body 能放什么

```javascript
// 字符串（通常配合 application/json）
body: JSON.stringify(payload)

// URL 编码表单，自动设置 Content-Type
body: new URLSearchParams({ name: "张三", age: "20" })

// 文件上传，自动生成 multipart 边界
const form = new FormData();
form.append("avatar", fileInput.files[0]);
body: form

// 二进制
body: new Blob([bytes], { type: "image/png" })
body: new Uint8Array(buffer)
```

★ **关键认知**：`fetch` **不会根据 body 自动猜 Content-Type**，除了 `FormData`、`URLSearchParams`、`Blob` 这类自带类型的对象。⚠ 发 JSON 时如果忘了写 `Content-Type: application/json`，服务器很可能按纯文本处理，`request.getParameter()` 一个字段都拿不到。

## 3.3 credentials 与跨域

```javascript
// 不携带任何凭证
fetch(url, { credentials: "omit" });

// 只对同源请求携带 Cookie（默认值）
fetch(url, { credentials: "same-origin" });

// 跨域也携带 Cookie
fetch(url, { credentials: "include" });
```

★ **关键认知**：跨域携带 Cookie 时，服务器必须返回 `Access-Control-Allow-Credentials: true`，且 `Access-Control-Allow-Origin` 不能是 `*`。⚠ 这两个条件缺一，浏览器就会拦截响应，控制台报 CORS 错误，但 Network 面板里能看到服务器其实返回了 200。

# 4. 流式读取

## 4.1 用 reader 逐块读取

```javascript
const response = await fetch("/api/large-text");
const reader = response.body.getReader();
const decoder = new TextDecoder("utf-8");
let result = "";

while (true) {
  const { done, value } = await reader.read(); // value 是 Uint8Array
  if (done) break;
  result += decoder.decode(value, { stream: true }); // 允许跨块字符
}
result += decoder.decode(); // 冲掉解码器里的残余字节
console.log(result);
```

★ **关键认知**：`response.body` 是 `ReadableStream`，适合处理大文件、SSE、AI 流式输出。⚠ `TextDecoder` 必须用 `{ stream: true }`，否则一个中文字符被切在两个字块之间时会解码成乱码。

## 4.2 下载进度

流式读取的副作用是能顺便算出进度：

```javascript
async function downloadWithProgress(url, onProgress) {
  const response = await fetch(url);
  const total = Number(response.headers.get("content-length")) || 0;
  const reader = response.body.getReader();
  const chunks = [];
  let loaded = 0;

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    chunks.push(value);
    loaded += value.length;
    if (total) onProgress(loaded / total);
  }

  return new Blob(chunks);
}
```

⚠ 服务器如果没有返回 `Content-Length`(例如用了 gzip 分块传输)，`total` 会是 0，此时只能显示"已下载多少字节"，无法算百分比。

# 5. 错误语义

## 5.1 fetch 只拒绝网络错误

```javascript
try {
  const response = await fetch("/api/not-exist"); // 服务器返回 404
  console.log(response.status);                  // 404，没有抛错
} catch (error) {
  console.log("只有网络层失败才会到这里");        // 不会执行
}
```

`fetch` 的 Promise 在以下情况才 reject：

| 情况 | 错误类型 |
|------|----------|
| DNS 解析失败、断网 | `TypeError` |
| 连接被拒绝、TLS 失败 | `TypeError` |
| 被 CORS 拦截 | `TypeError` |
| 请求被 `AbortController` 取消 | `DOMException`，`name === "AbortError"` |
| 响应体读取中断 | `TypeError` |

## 5.2 正确检查业务成功

```javascript
async function requestJson(url, init) {
  const response = await fetch(url, init);
  if (!response.ok) {
    const detail = await response.text(); // 错误正文也可能是 JSON
    throw new HttpError(response.status, detail);
  }
  return response.json();
}
```

★ **关键认知**：把"网络是否通"和"业务是否成功"分成两层处理后，代码才可靠。⚠ 不要写 `fetch(url).then(res => res.json())` 就完事——404 页面返回的 HTML 会让 `json()` 抛 `SyntaxError`，错误信息与真实原因完全对不上。

## 5.3 识别取消

```javascript
try {
  await fetch("/api/slow", { signal: controller.signal });
} catch (error) {
  if (error.name === "AbortError") {
    console.log("用户主动取消，不必提示错误");
  } else {
    console.error("真正的请求失败", error);
  }
}
```

# 6. fetch 与 XHR 对比

| 维度 | XMLHttpRequest | fetch |
|------|----------------|-------|
| 编程模型 | 事件回调 | Promise / `async-await` |
| 响应时机 | `load` 时正文已完成 | resolve 时只收到响应头 |
| HTTP 错误 | 需自查 `status` | 同样需自查 `ok` |
| 上传进度 | `xhr.upload.onprogress` | 不支持 |
| 下载进度 | `onprogress` | 需手动读流 |
| 取消 | `xhr.abort()` | `AbortController` |
| Cookie | `withCredentials` | `credentials` |
| 流式处理 | 不支持 | `response.body` |
| 默认超时 | 可设 `timeout` | 无，需自己用 `AbortSignal.timeout` |

★ **关键认知**：`fetch` 的优势是**组合能力**——Promise、流、标准 Request/Response 对象，让它容易封装和测试；XHR 的优势是**上传进度和成熟度**。⚠ 做带进度条的大文件上传时，很多团队仍会退回 XHR 或使用封装库，这不是倒退，而是按场景选工具。

# 7. 练习

1. 写一个 `postJson(url, data)`，自动设置 `Content-Type`，非 2xx 时抛出包含状态码和响应正文的错误对象。
2. 用 `response.body.getReader()` 读取一段较大的文本，统计下载字节数并每 100ms 更新一次页面。
3. 对比实验：请求一个返回 404 的地址，分别用 XHR 和 fetch 处理，记录各自在什么条件下进入"成功"分支。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 两段式 | 先 resolve 响应头，再读正文 |
| Response | `ok` 判断业务成功，正文只能读一次 |
| 请求配置 | `method`/`headers`/`body`/`credentials`/`signal` |
| 流式 | `response.body.getReader()` 可做下载进度 |
| 错误 | 只对网络错误 reject，4xx/5xx 需手动判断 |
| 取舍 | 上传进度用 XHR，组合与流式用 fetch |

下一篇进入同源策略与 CORS，解释"为什么浏览器允许请求发出却不让脚本读响应"，以及预检请求、凭证和常见报错到底该怎么排查。
