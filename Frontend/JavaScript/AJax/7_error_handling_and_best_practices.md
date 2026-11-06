请求失败不可怕，可怕的是失败后代码以为成功了。`fetch` 不因 404 reject、`response.json()` 可能抛解析错误、用户切页面时请求还在跑——错误处理要做的，是把这些情况**分层识别、统一收口、该重试的重试、该放弃的放弃**。本篇是 AJAX 模块的收尾，也给出可直接落地的封装与安全清单。

# 1. 错误分类

## 1.1 网络错误

请求根本没拿到有效响应：断网、DNS 失败、连接被拒、TLS 证书错误、跨域被拦截、被取消。`fetch` 会以 `TypeError` 或 `AbortError` 拒绝。

## 1.2 HTTP 错误

服务器收到了请求，但返回 4xx/5xx。**fetch 不会 reject**，必须检查 `response.ok` 或 `response.status`。

## 1.3 解析错误

响应格式与预期不符，比如服务器返回 HTML 错误页，前端却调用 `response.json()`，此时抛 `SyntaxError`。

## 1.4 业务错误

HTTP 200，但响应体里的业务码表示失败，例如 `{ "code": 40001, "message": "余额不足" }`。前端要把业务码也纳入成功判断。

## 1.5 五类错误对比

| 类型 | 典型来源 | 是否可重试 | 用户提示 |
|------|----------|------------|----------|
| 网络错误 | 断网、DNS、CORS | 通常可以 | "网络异常，请检查连接" |
| HTTP 5xx | 服务器故障 | 可以，需退避 | "服务繁忙，请稍后重试" |
| HTTP 4xx | 参数/权限/不存在 | 一般不可以 | 按状态码给具体提示 |
| 解析错误 | 返回格式不符 | 重试通常无效 | "数据格式异常" |
| 业务错误 | 业务规则拒绝 | 看规则 | 直接展示服务端 message |
| 取消 | 用户/路由切换 | 不需要 | 静默处理 |

★ **关键认知**：这几类错误**必须分开处理**。⚠ 把它们都塞进一个 `catch` 里，用户会看到"网络错误"却不知道其实是余额不足；日志里也只有一句 `Failed to fetch`，排障时毫无信息量。

# 2. 统一错误对象

## 2.1 定义错误类型

```javascript
class RequestError extends Error {
  constructor(kind, message, extra = {}) {
    super(message);
    this.name = "RequestError";
    this.kind = kind; // network | http | parse | business | abort
    Object.assign(this, extra); // 可带 status、body、url、cause
  }
}
```

有了 `kind` 和 `status`，上层就能做判断，而不是靠字符串匹配错误信息。

## 2.2 一个健壮的请求函数

```javascript
async function requestJson(url, init = {}) {
  let response;

  try {
    response = await fetch(url, init);
  } catch (error) {
    if (error.name === "AbortError") throw new RequestError("abort", "请求已取消");
    if (error.name === "TimeoutError") throw new RequestError("network", "请求超时");
    throw new RequestError("network", "网络异常", { cause: error });
  }

  if (!response.ok) {
    const body = await response.text(); // 错误正文可能不是 JSON
    throw new RequestError("http", `HTTP ${response.status}`, {
      status: response.status, body, url: response.url,
    });
  }

  if (response.status === 204) return null; // 无正文，别硬解析

  try {
    return await response.json();
  } catch (error) {
    throw new RequestError("parse", "响应不是合法 JSON", { cause: error });
  }
}
```

★ **关键认知**：错误正文也要"尽力读取"，但**不要再假设它是 JSON**。⚠ 网关返回的 502 页面往往是 HTML，直接 `response.json()` 会抛 `SyntaxError`，把真正的 502 掩盖掉。

## 2.3 错误分发

```javascript
function handleError(error) {
  if (error.kind === "abort") return; // 用户主动取消，不打扰
  if (error.kind === "network") return toast("网络异常，请检查连接");
  if (error.status === 401) return redirectToLogin();
  if (error.status === 403) return toast("没有权限");
  if (error.status === 404) return toast("内容不存在");
  if (error.status >= 500) return toast("服务繁忙，请稍后重试");
  toast(error.message || "请求失败");
}
```

★ **关键认知**：错误提示要**分级**——能用行内提示解决的就不要弹全局 toast，能静默恢复的就不要打断用户。⚠ 把所有错误都 `alert` 一遍，是体验最差的实现之一。

# 3. 重试

## 3.1 哪些错误值得重试

| 错误 | 是否重试 | 原因 |
|------|----------|------|
| 网络抖动 | 是 | 换一次可能就通 |
| 408 请求超时 | 是 | 服务端明确表示超时 |
| 429 限流 | 是，需更长等待 | 读 `Retry-After` |
| 500/502/503/504 | 是，有限次 | 多为临时故障 |
| 400/401/403/404 | 否 | 重试结果不会变 |
| 解析错误 | 否 | 返回格式不会自己变 |
| 用户取消 | 否 | 重试违背用户意图 |

## 3.2 指数退避与抖动

```javascript
function sleep(ms, signal) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(resolve, ms);
    signal?.addEventListener("abort", () => (clearTimeout(timer), reject(signal.reason)), { once: true });
  });
}

async function withRetry(task, { retries = 3, base = 300, signal } = {}) {
  let attempt = 0;
  while (true) {
    try {
      return await task();
    } catch (error) {
      const retryable = !["abort", "business", "parse"].includes(error.kind) &&
        (error.kind === "network" || error.status === 408 || error.status === 429 || error.status >= 500);
      if (!retryable || attempt >= retries) throw error;
      const backoff = base * 2 ** attempt;                 // 300、600、1200……
      await sleep(backoff + Math.random() * base, signal); // 加抖动，打散重试
      attempt += 1;
    }
  }
}
```

★ **关键认知**：指数退避解决"越失败越猛冲"，**抖动(jitter)** 解决"很多客户端同一时刻一起重试把服务器再次打垮"。⚠ 没有抖动的重试在服务恢复瞬间容易形成新的流量尖峰。

## 3.3 幂等性

只有**幂等**的操作才能安全重试：

| 方法 | 幂等 | 说明 |
|------|------|------|
| GET | 是 | 只读 |
| HEAD | 是 | 只读 |
| PUT | 是 | 整体替换，多次结果一致 |
| DELETE | 是 | 删除不存在也视为成功 |
| POST | 通常否 | 可能重复下单、重复扣款 |

POST 需要重试时，让服务端支持**幂等键**：同一笔业务重试时复用同一个 `Idempotency-Key` 请求头，服务端据此去重。

⚠ **关键认知**：前端自动重试 POST 是危险动作。⚠ 用户看到"失败"又点了一次提交，如果两次都成功，就会产生两笔订单。重试策略必须和后端幂等设计配套。

# 4. 取消与超时

## 4.1 把取消当成正常流程

```javascript
let current; // 保存上一次请求的 controller

async function search(keyword) {
  current?.abort(); // 取消上一次
  current = new AbortController();
  try {
    const response = await fetch(`/api/search?q=${encodeURIComponent(keyword)}`, {
      signal: current.signal,
    });
    render(await response.json());
  } catch (error) {
    if (error.name !== "AbortError") throw error;
  }
}
```

## 4.2 超时是错误，取消不是

```javascript
const signal = AbortSignal.any([userController.signal, AbortSignal.timeout(8000)]);

fetch(url, { signal }).catch((error) => {
  if (error.name === "TimeoutError") showRetry("请求超时，是否重试？");
  else if (error.name !== "AbortError") showError(error); // 取消则静默
});
```

★ **关键认知**：**超时值得提示并重试，取消不该提示**。⚠ 两者都表现为"Promise 被拒绝"，如果不区分错误名，用户每次切换页面都会看到"请求失败"。

# 5. 请求封装

## 5.1 基础封装

```javascript
const BASE_URL = "/api";

async function http(path, options = {}) {
  const { timeout = 10000, retries = 0, signal, headers, ...init } = options;
  const timeoutSignal = AbortSignal.timeout(timeout);
  const combined = signal ? AbortSignal.any([signal, timeoutSignal]) : timeoutSignal;

  const run = () => requestJson(`${BASE_URL}${path}`, {
    ...init, headers: { Accept: "application/json", ...headers }, signal: combined,
  });

  return withRetry(run, { retries, signal: combined });
}

const users = await http("/users", { retries: 2 });
```

## 5.2 拦截器思路

封装到一定规模后，可以把"统一加 token、统一解包、统一报错"抽成拦截器：请求拦截器在 `fetch` 前修改 config，响应拦截器在返回后统一处理状态码与错误。拦截器适合放**横切关注点**，不要把业务逻辑塞进去，否则某个接口一旦要特殊处理就会变成新的意大利面。

## 5.3 配置分层

| 层级 | 放什么 |
|------|--------|
| 全局 | baseURL、超时、默认头、认证 |
| 模块 | 某类接口的公共参数、错误映射 |
| 单次请求 | 该接口特有的 header、body、重试次数 |

# 6. 安全

## 6.1 XSS

AJAX 拿到的数据是**不可信输入**，直接拼进 HTML 就是 XSS 入口。用 `textContent` 写入数据，避免 `innerHTML` 把服务端内容当 HTML 执行。

★ **关键认知**：`textContent` 会把内容当纯文本，`innerHTML` 会当 HTML 执行。⚠ 必须渲染富文本时，先用可信的白名单清洗库处理，并配合 CSP 限制脚本来源。

## 6.2 CSRF

浏览器会自动带上同站 Cookie，攻击者可以诱导用户向已登录的站点发请求。常见防御：

| 手段 | 说明 |
|------|------|
| `SameSite=Lax/Strict` | 跨站请求不带 Cookie |
| CSRF Token | 请求头或表单里带一次性令牌，服务端校验 |
| 校验 `Origin`/`Referer` | 拒绝来源不明的写请求 |
| 自定义请求头 | 跨域时会触发预检，简单表单无法伪造 |

## 6.3 凭证与密钥

| 做法 | 风险 |
|------|------|
| 把 API Key 写在前端代码 | 打包后人人可见 |
| Token 存 `localStorage` | XSS 可直接读取 |
| Token 存 `HttpOnly` Cookie | JS 读不到，但仍需防 CSRF |
| 在 URL 里传 token | 会进日志、Referer、浏览器历史 |

★ **关键认知**：**前端没有秘密**。任何写进 JS 的密钥都等于公开，真正的鉴权必须在服务端完成。⚠ CORS 也不是访问控制——它只约束浏览器，攻击者用 curl 可以完全绕过，接口自身必须有认证和权限校验。

# 7. 体验与性能

## 7.1 加载状态

请求期间禁用按钮防重复提交，用 `try/finally` 保证无论成败都恢复状态，失败时交给统一的 `handleError`。

★ **关键认知**：乐观更新能提升手感，但必须准备**回滚**：先改 UI，请求失败再改回去。⚠ 对金额、库存、权限这类敏感状态不要乐观更新。

## 7.2 缓存与去重

- 读多写少的数据加内存缓存与过期时间；相同请求用 in-flight Promise 去重。
- 写操作成功后主动失效相关缓存；列表返回详情时先用旧数据渲染，再后台刷新。

## 7.3 错误文案原则

| 场景 | 文案方向 |
|------|----------|
| 网络断开 | 说明原因 + 检查连接建议 |
| 超时 | 说明可以重试 |
| 401 | 引导重新登录 |
| 403 | 说明无权限，不要诱导重试 |
| 5xx | 简短致歉 + 重试入口 |
| 业务规则 | 直接展示服务端可读消息 |

# 8. 练习

1. 给下面四种情况分类并写出处理方式：断网、接口返回 500、接口返回 HTML 错误页、响应 `{code: 40001}`。
2. 实现带指数退避和抖动的重试函数，要求 500/429 重试、400 不重试、最多 3 次。
3. 封装一个 `http` 函数，统一处理超时、错误对象和 JSON 解析，并用它完成一次登录请求。

# 9. 小结

| 主题 | 要点 |
|------|------|
| 错误分层 | 网络、HTTP、解析、业务、取消分开处理 |
| 错误对象 | 用 `kind`/`status` 携带上下文，别只留字符串 |
| 重试 | 只重试可恢复错误，指数退避 + 抖动 |
| 幂等 | POST 自动重试必须配幂等键 |
| 取消 | 取消静默，超时提示，错误名要区分 |
| 封装 | 全局/模块/单次三层配置，拦截器放横切逻辑 |
| 安全 | 防 XSS、防 CSRF，前端不存密钥，CORS 不是鉴权 |

到这里，AJAX 模块从概念、XHR、fetch、跨域、数据格式、进度取消到错误处理已经闭环。下一步可以进入同级的 `Axios` 目录，看工程化请求库如何把拦截器、取消、自动 JSON 和错误处理打包成开箱即用的方案。
