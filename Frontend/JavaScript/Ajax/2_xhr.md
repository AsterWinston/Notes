`XMLHttpRequest`(简称 XHR)是 AJAX 的元老级实现。今天新代码大多直接用 `fetch`，但大量老项目、上传进度场景和面试题仍然绕不开它。本篇按"创建 → 打开 → 发送 → 收响应"的生命周期，把 `readyState`、`status`、事件模型和经典坑一次讲清。

# 1. XMLHttpRequest 概览

## 1.1 创建与四个步骤

```javascript
const xhr = new XMLHttpRequest();          // 1. 创建对象
xhr.open("GET", "/api/user", true);        // 2. 配置方法、地址、是否异步
xhr.onload = () => { /* 处理响应 */ };     // 3. 注册回调
xhr.send();                                // 4. 真正发出请求
```

★ **关键认知**：`open` 不发送任何网络数据，它只是**初始化请求参数**；真正的网络动作发生在 `send`。⚠ 因此"在 `open` 之前设置 `responseType`"会抛错，而"在 `send` 之后再改请求头"则完全无效。

## 1.2 最小可用示例

```javascript
function getUser(id) {
  const xhr = new XMLHttpRequest();
  xhr.open("GET", `/api/users/${id}`, true);
  xhr.responseType = "json"; // 必须在 open 之后、send 之前设置

  xhr.onload = () => {
    // onload 触发时，响应已完整到达
    if (xhr.status >= 200 && xhr.status < 300) {
      console.log("用户：", xhr.response); // responseType=json 时是对象
    } else {
      console.error("HTTP 错误：", xhr.status);
    }
  };

  xhr.onerror = () => console.error("网络错误"); // 断网、DNS 失败
  xhr.ontimeout = () => console.error("请求超时");
  xhr.timeout = 5000; // 毫秒

  xhr.send();
}
```

★ **关键认知**：XHR 有**两条独立的"成功"判断线**——`readyState === 4` 只说明"请求流程结束"，`status` 才说明"服务器怎么回应"。⚠ 只判断 `readyState` 不判断 `status`，会把 404、500 都当成成功，这是最经典的 XHR 事故。

# 2. readyState 与 status

## 2.1 readyState 五个状态

| 值 | 常量 | 含义 |
|----|------|------|
| 0 | `UNSENT` | 对象已创建，`open` 未调用 |
| 1 | `OPENED` | `open` 已调用，尚未 `send` |
| 2 | `HEADERS_RECEIVED` | 已收到响应头，正文还没开始 |
| 3 | `LOADING` | 正在接收响应体 |
| 4 | `DONE` | 响应接收完毕 |

```text
0 UNSENT
   │ open()
   ▼
1 OPENED
   │ send()
   ▼
2 HEADERS_RECEIVED
   │ 数据陆续到达
   ▼
3 LOADING
   │ 接收完毕
   ▼
4 DONE
```

★ **关键认知**：`readyState` 描述的是**请求生命周期**，和 HTTP 状态码无关。⚠ 状态 2 和 3 会触发多次 `readystatechange`，如果在这里反复处理数据，很容易重复渲染；要"等完整响应"就判断 `readyState === 4`，或直接用 `load` 事件。

## 2.2 status 与响应内容

```javascript
xhr.onreadystatechange = () => {
  if (xhr.readyState !== 4) return; // 只关心最终态
  console.log(xhr.status);          // 200、404、500……
  console.log(xhr.statusText);      // "OK"、"Not Found"
  console.log(xhr.responseURL);     // 最终请求地址（可能被重定向）
  console.log(xhr.responseText);    // 文本形式的响应体
};
```

常见状态码与含义：

| 状态码 | 含义 | 前端常见处理 |
|--------|------|--------------|
| 200 | 成功 | 读取数据 |
| 201 | 已创建 | 读取新资源地址 |
| 204 | 无内容 | 不要尝试解析正文 |
| 301/302 | 重定向 | 浏览器通常自动跟随 |
| 400 | 请求参数错误 | 提示用户检查输入 |
| 401/403 | 未认证/无权限 | 跳登录或提示无权限 |
| 404 | 资源不存在 | 提示或回退 |
| 500 | 服务器错误 | 提示稍后重试 |

### 2.2.1 responseType

`responseType` 决定 `xhr.response` 的类型，必须在 `open` 之后、`send` 之前设置：

```javascript
xhr.responseType = "";        // 默认，等同 "text"
xhr.responseType = "text";    // 字符串
xhr.responseType = "json";    // 自动 JSON.parse，解析失败则为 null
xhr.responseType = "blob";    // 二进制大对象，适合图片、文件
xhr.responseType = "arraybuffer"; // 原始字节，适合音视频处理
```

★ **关键认知**：设置 `responseType = "json"` 后应读 `xhr.response`，而不是 `xhr.responseText`。⚠ 当 JSON 非法时，`response` 会是 `null`，不会抛异常；如果业务依赖解析错误提示，用默认的 `text` 再手动 `JSON.parse` 反而更可控。

## 2.3 三个响应属性

| 属性 | 内容 | 使用场景 |
|------|------|----------|
| `response` | 按 `responseType` 解析后的结果 | 统一读取入口 |
| `responseText` | 纯文本，非 text 类型下访问会抛错 | 调试、手动解析 |
| `responseXML` | 解析后的 XML Document | 老接口，现已少见 |

# 3. 事件模型

## 3.1 readystatechange 与 load

```javascript
// 老写法：任何 readyState 变化都会触发
xhr.onreadystatechange = () => {
  if (xhr.readyState === 4 && xhr.status === 200) {
    console.log(xhr.responseText);
  }
};

// 新写法：只在成功完成时触发一次，语义更清晰
xhr.onload = () => {
  console.log(xhr.status, xhr.responseText);
};
```

★ **关键认知**：`load` 只在**响应完整接收**时触发，等价于"`readyState` 变成 4 且没有网络层错误"，但仍然**不管 HTTP 状态码**。⚠ `load` 里拿到 500 是正常现象，所以 `status` 检查不能省。

## 3.2 error、abort、timeout

```javascript
xhr.onerror = () => console.error("网络层失败"); // 断网、跨域被拒等
xhr.onabort = () => console.warn("请求被取消");   // 调用 xhr.abort()
xhr.ontimeout = () => console.warn("超时");       // 超过 xhr.timeout
xhr.onloadend = () => console.log("无论成败都会执行"); // 适合收尾
```

| 事件 | 触发条件 | 是否代表 HTTP 失败 |
|------|----------|--------------------|
| `load` | 响应完整到达 | 否，需自查 `status` |
| `error` | 网络层错误 | 是 |
| `abort` | 主动调用 `abort()` | 是 |
| `timeout` | 超过 `timeout` 设置 | 是 |
| `loadend` | 请求结束(成功或失败) | 不判断 |

★ **关键认知**：`error`、`abort`、`timeout` 三者互斥，且都不会触发 `load`。⚠ 取消请求时 `status` 可能是 0，不能拿它当"服务器返回 0"来理解。

## 3.3 upload 事件

上传文件时，`xhr.upload` 是一个独立的事件目标，可以监听上传进度：

```javascript
xhr.upload.onprogress = (event) => {
  if (event.lengthComputable) {
    const percent = (event.loaded / event.total) * 100;
    console.log(`已上传 ${percent.toFixed(1)}%`);
  }
};
```

这正是 `fetch` 目前**做不到**的能力，第 6 篇会展开讲上传进度与取消。

# 4. 同步与异步

## 4.1 async 参数

`open(method, url, async)` 的第三个参数默认是 `true`(异步)。传 `false` 会变成同步请求：

```javascript
xhr.open("GET", "/api/data", false); // 同步：send 会阻塞
xhr.send();
console.log(xhr.responseText);        // 下一行就能拿到结果
```

## 4.2 为什么几乎不该用同步

| 影响 | 说明 |
|------|------|
| 主线程冻结 | 用户点击、滚动、动画全部暂停 |
| 跨域被禁止 | 现代浏览器直接抛 `InvalidAccessError` |
| 无法设置 timeout | 同步请求下 `timeout` 不生效 |
| 调试困难 | 卡顿表现为"页面假死"，难定位 |

⚠ **关键认知**：同步 XHR 只适合极少数 Worker 场景或历史遗留代码。页面主线程里请一律使用异步，并用 Promise 封装它。

# 5. 配置请求

## 5.1 open 的完整参数

```javascript
xhr.open(
  "POST",                 // HTTP 方法
  "/api/users",           // URL
  true,                   // 是否异步
  "user",                 // 可选：用户名
  "password"              // 可选：密码
);
```

★ **关键认知**：URL 里的查询参数必须自己 `encodeURIComponent`，否则 `&`、`#`、中文会破坏 URL 结构：

```javascript
const keyword = "张三 & 李四";
const url = `/api/search?q=${encodeURIComponent(keyword)}`;
```

## 5.2 setRequestHeader

```javascript
xhr.open("POST", "/api/users", true);
xhr.setRequestHeader("Content-Type", "application/json"); // 告诉服务器正文格式
xhr.setRequestHeader("X-Requested-With", "XMLHttpRequest");
xhr.send(JSON.stringify({ name: "张三" }));
```

⚠ **关键认知**：`setRequestHeader` 必须在 `open` 之后、`send` 之前调用；同名头会**追加合并**而不是覆盖。受浏览器限制，`Cookie`、`Host`、`Content-Length` 等头不能由脚本设置。

## 5.3 timeout 与 withCredentials

```javascript
xhr.timeout = 8000;          // 8 秒未完成就触发 timeout 事件
xhr.withCredentials = true;  // 跨域时携带 Cookie 等凭证
```

★ **关键认知**：`withCredentials` 只影响**跨域**请求；同源请求本来就会带 Cookie。⚠ 开启后，服务器不能再用 `Access-Control-Allow-Origin: *`，必须返回具体源，否则浏览器会拦截响应(详见第 4 篇)。

# 6. 常见坑

## 6.1 属性设置顺序错误

```javascript
const xhr = new XMLHttpRequest();
xhr.responseType = "json"; // 错误：open 之前设置会抛 InvalidStateError
xhr.open("GET", "/api/data", true);
```

正确顺序是：`new` → `open` → 设置 `responseType`/请求头 → 注册事件 → `send`。

## 6.2 status 为 0

`status === 0` 通常意味着**请求根本没走完 HTTP 流程**：断网、跨域被拦截、请求被取消、或用了 `file://` 协议。⚠ 不要把它映射成"服务器错误 0"，它更像"连接层面没拿到响应"。

## 6.3 重复 send

一个 XHR 实例在 `readyState === 4` 之前再次 `send` 会抛 `InvalidStateError`。要复用实例，先 `abort()` 再 `open()`：

```javascript
xhr.abort();                    // 取消上一次
xhr.open("GET", "/api/next", true);
xhr.send();
```

## 6.4 忘记处理非 2xx

```javascript
xhr.onload = () => {
  if (!xhr.status) return;                 // 网络异常时可能是 0
  if (xhr.status >= 200 && xhr.status < 300) {
    render(xhr.response);
  } else {
    showError(`请求失败：${xhr.status}`);   // 404/500 走这里
  }
};
```

# 7. 练习

1. 用 XHR 写一个 `request(url)` 函数，返回 Promise，要求覆盖成功、HTTP 错误、网络错误、超时四种情况。
2. 实现一个"输入框停止输入 500ms 后自动搜索"的功能，要求用 `abort()` 取消上一次未完成的请求。
3. 用 `xhr.upload.onprogress` 做一个文件上传进度条，显示百分比并在完成时隐藏。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 生命周期 | `new` → `open` → 设置属性 → `send` → 事件回调 |
| readyState | 0-4 描述请求流程，不等于 HTTP 结果 |
| status | 2xx 才是业务成功，必须显式判断 |
| 事件 | `load`/`error`/`abort`/`timeout`/`loadend` 分工明确 |
| 同步请求 | 阻塞主线程，已被现代浏览器限制，不要使用 |
| 常见坑 | 属性顺序、status 0、重复 send、URL 编码 |

下一篇进入 `fetch`，看它如何用 Promise 和 `Response` 对象重写这套流程，以及为什么它"更现代"却也带来了新的错误处理陷阱。
