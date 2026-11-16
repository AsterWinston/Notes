请求失败时，最容易出现的写法是 `catch (e) { alert('出错了') }`。但"出错"至少分五类：HTTP 状态码错误、网络层错误、超时、业务错误、主动取消。它们该不该提示、该不该重试、该不该上报，处理方式完全不同。本篇从 `AxiosError` 的结构出发，建立一套可落地的错误分类与重试策略。

# 1. 错误的五类来源

## 1.1 分类

| 类型 | 触发条件 | 典型特征 | 是否提示 |
|------|---------|---------|---------|
| HTTP 错误 | 状态码非 2xx | `error.response` 存在 | 视状态码而定 |
| 网络错误 | 断网、DNS 失败、CORS | 无 `response`，`ERR_NETWORK` | 提示检查网络 |
| 超时 | 超过 `timeout` | `ECONNABORTED` / `ETIMEDOUT` | 提示稍后重试 |
| 业务错误 | HTTP 200 但业务码失败 | 响应体里的 `code` | 展示后端消息 |
| 取消 | 主动 abort | `ERR_CANCELED` | 不提示 |

## 1.2 传播路径

```text
adapter 发出请求
      ├─ 网络失败 ─────────→ AxiosError(code=ERR_NETWORK)
      ├─ 超时 ─────────────→ AxiosError(code=ECONNABORTED)
      ├─ 主动取消 ─────────→ CanceledError(code=ERR_CANCELED)
      ├─ 状态码非 2xx ─────→ AxiosError(response 存在)
      └─ 2xx ──────────────→ then(response) → 业务 code 判断
```

★ **关键认知**：Axios 只负责把"HTTP 层"的结果翻译成 Promise 的成功或失败，**不负责判断业务是否成功**。HTTP 200 完全可能对应"余额不足""库存为零"。业务码判断必须由响应拦截器或业务封装负责。

# 2. AxiosError 的结构

## 2.1 关键字段

```javascript
{
  message: 'Request failed with status code 404',
  code: 'ERR_BAD_REQUEST',
  config: {},   // 本次请求配置
  request: {},  // 底层请求对象
  response: { data: {}, status: 404, headers: {}, config: {} }, // 无响应时为 undefined
  isAxiosError: true
}
```

| code | 含义 |
|------|------|
| `ERR_NETWORK` | 网络错误，没有拿到响应 |
| `ECONNABORTED` / `ETIMEDOUT` | 超时或被中止 |
| `ERR_CANCELED` | 主动取消 |
| `ERR_BAD_REQUEST` / `ERR_BAD_RESPONSE` | 4xx / 5xx 响应 |

## 2.2 类型守卫

```javascript
try {
  await http.get('/api/users/1')
} catch (error) {
  if (axios.isAxiosError(error)) {
    console.log(error.response?.status) // 可能 undefined
    console.log(error.code)
  } else if (error instanceof Error) {
    console.log(error.message)
  }
}
```

★ **关键认知**：不要用 `error.response.status` 直接取值。网络错误、超时、取消时 `response` 都是 `undefined`，直接点会抛 `TypeError`，把真正的错误覆盖掉。⚠ 一律用 `error.response?.status`，或先判断 `response` 是否存在。

# 3. HTTP 状态码处理

## 3.1 默认行为

Axios 默认 `validateStatus: (status) => status >= 200 && status < 300`，即 2xx 成功，其余全部 reject。想自己处理所有状态码，可以改成 `() => true`；但这样 404、500 都不会进 catch，业务层必须自己判断 `response.status`，不建议全局打开。

## 3.2 常见状态码策略

| 状态码 | 含义 | 建议 |
|--------|------|------|
| 400 | 参数错误 | 展示后端消息，通常不重试 |
| 401 | 未认证 | 刷新 Token，失败则登录 |
| 403 | 无权限 | 提示无权限，不重试 |
| 404 | 不存在 | 展示空状态或提示 |
| 408 / 429 | 超时 / 限流 | 退避后重试，遵守 `Retry-After` |
| 422 | 校验失败 | 展示字段级错误 |
| 500 / 502 / 503 / 504 | 服务端或网关异常 | 提示稍后重试，可有限重试 |

实现时可以在响应拦截器里按 `error.response?.status` 给 `error.userMessage` 赋可展示文案，再统一 `Promise.reject(error)`；不要在这里吞掉错误。

# 4. 网络错误与超时

## 4.1 如何区分

```javascript
function classify(error) {
  if (axios.isCancel(error)) return 'canceled'
  if (!axios.isAxiosError(error)) return 'unknown'
  if (error.response) return 'http'
  if (error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT') return 'timeout'
  return 'network'
}
```

有 `response` 说明服务端返回了响应，只是状态码不理想；无 `response` 且带超时码说明没在时限内收到响应；无 `response` 且是 `ERR_NETWORK` 则说明根本没连上，或被浏览器拦截。

## 4.2 处理建议

网络错误提示"网络异常，请检查连接"，不要无限重试；超时对 GET 可以退避重试 1 到 2 次，对 POST 谨慎。⚠ CORS 失败在 JS 里通常只表现为笼统的 `ERR_NETWORK`，必须看控制台与 Network 面板，不能只靠 `error.message` 排查。

# 5. 业务错误

## 5.1 200 也可能是失败

```json
{ "code": 1001, "message": "库存不足", "data": null }
```

HTTP 200 但业务失败。如果不判断 `code`，调用方会以为成功，拿着 `null` 继续往下走。

## 5.2 在响应拦截器里转换

```javascript
class BusinessError extends Error {
  constructor(code, message, data) {
    super(message)
    this.name = 'BusinessError'
    this.code = code
    this.data = data
  }
}

http.interceptors.response.use(
  (response) => {
    const body = response.data
    if (body && typeof body === 'object' && 'code' in body) {
      if (body.code !== 0) {
        throw new BusinessError(body.code, body.message || '业务失败', body.data)
      }
      return body.data
    }
    return body
  },
  (error) => Promise.reject(error)
)
```

★ **关键认知**：在响应拦截器的成功回调里 `throw`，会被 Promise 链自动转成 rejected，后续 `catch` 能接到。⚠ 但如果同时"解包 + 抛业务错误"，返回值类型就变成"业务 data 或抛错"，TypeScript 下要配套泛型声明。

## 5.3 业务错误不要吞

只 `showToast` 然后 `return`，调用方会拿到 `undefined`；正确做法是提示之后继续 `throw`，让 Promise 保持 rejected。

# 6. 统一错误结构

## 6.1 为什么需要归一化

上层组件不应该关心"这是 AxiosError 还是 BusinessError"，只想要一个稳定结构：

```typescript
interface AppError {
  type: 'http' | 'network' | 'timeout' | 'business' | 'canceled' | 'unknown'
  code?: string | number
  status?: number
  message: string
  raw: unknown
}
```

## 6.2 normalizeError

```javascript
function normalizeError(error) {
  if (axios.isCancel(error)) {
    return { type: 'canceled', message: '请求已取消', raw: error }
  }

  if (error?.name === 'BusinessError') {
    return { type: 'business', code: error.code, message: error.message, raw: error }
  }

  if (axios.isAxiosError(error)) {
    const status = error.response?.status
    const body = error.response?.data

    if (status) {
      return {
        type: 'http',
        status,
        code: body?.code,
        message: body?.message || error.message,
        raw: error
      }
    }

    const timeout = error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT'
    return {
      type: timeout ? 'timeout' : 'network',
      code: error.code,
      message: timeout ? '请求超时，请稍后重试' : '网络异常，请检查连接',
      raw: error
    }
  }

  return {
    type: 'unknown',
    message: error instanceof Error ? error.message : String(error),
    raw: error
  }
}
```

★ **关键认知**：归一化的价值在于**把"怎么判断"收敛到一处**。页面只需 `switch (appError.type)`，不用再写 `axios.isAxiosError`、`response?.status` 这些散落各处的判断。⚠ 归一化后的 `message` 应是**可展示给用户**的文案，原始技术信息留在 `raw` 里供日志使用。

## 6.3 全局兜底

浏览器监听 `unhandledrejection`，Node 监听 `process.on('unhandledRejection')`，在回调里先 `normalizeError`，跳过 `canceled`，再统一上报。上报前必须脱敏。

# 7. 重试

## 7.1 哪些请求可以重试

| 方法 | 可重试 | 说明 |
|------|--------|------|
| GET / HEAD | 是 | 幂等，最安全 |
| PUT / DELETE | 视情况 | 语义幂等，但服务端未必实现 |
| POST | 否（除非有幂等键） | 可能重复创建 |
| PATCH | 否 | 可能重复修改 |

状态码层面只对临时性失败重试：`408`、`429`、`502`、`503`、`504`。401 应走刷新 Token，不混进普通重试。

## 7.2 指数退避与抖动

```javascript
function sleep(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms))
}

function backoff(attempt) {
  const base = Math.min(1000 * 2 ** attempt, 8000)
  return base + Math.random() * 300 // 抖动，避免同时重试
}
```

## 7.3 用拦截器实现有限重试

```javascript
const RETRYABLE = new Set([408, 429, 502, 503, 504])

function shouldRetry(error) {
  if (!axios.isAxiosError(error) || axios.isCancel(error)) return false
  const method = (error.config?.method || 'get').toLowerCase()
  if (!['get', 'head', 'options'].includes(method)) return false
  if (error.response) return RETRYABLE.has(error.response.status)
  return true // 网络错误或超时
}

http.interceptors.response.use(undefined, async (error) => {
  const config = error.config
  if (!config || !shouldRetry(error)) return Promise.reject(error)

  config._retryCount = config._retryCount || 0
  if (config._retryCount >= 2) return Promise.reject(error)

  config._retryCount += 1
  await sleep(backoff(config._retryCount))
  return http(config)
})
```

⚠ 重试必须设上限，否则会无限循环。还要让"401 刷新"和"5xx 重试"两个拦截器职责分离，避免出现"401 → 刷新 → 重试 → 又 401 → 又刷新"。

# 8. 全局提示与上报

## 8.1 提示与去重

取消不提示；网络/超时用全局 toast；401/403 跳登录或无权限页；422 交给表单；业务错误优先展示后端 message。可用 `error.__notified` 标记避免全局与页面重复弹提示。上报前必须过滤 `Authorization`、Cookie、手机号、身份证号等敏感字段。

# 9. 练习

## 9.1 练习一：实现 classify

不看本篇代码，写一个 `classify(error)`，返回 `canceled`、`http`、`timeout`、`network`、`business`、`unknown` 之一，并用构造的错误对象测试。

## 9.2 练习二：401 与重试隔离

同时挂"401 刷新"与"5xx 重试"两个拦截器，构造 401 后重试仍失败的场景，确保不会触发 5xx 重试，也不会无限刷新。

## 9.3 练习三：统一错误结构

把 `normalizeError` 改写成 TypeScript 判别联合：`type === 'http'` 时 `status` 必填，`type === 'business'` 时 `code` 必填。

# 10. 小结

| 概念 | 要点 |
|------|------|
| AxiosError | `response` 可能不存在，取值用可选链 |
| 四类错误 | HTTP、网络、业务、取消，策略不同 |
| 业务错误 | HTTP 200 不等于业务成功 |
| 归一化 | 用统一 `AppError` 收敛判断 |
| 重试 | 只重试幂等请求与临时状态码，必须设上限 |
| 提示 | 全局与页面只选一个负责，取消不提示 |

下一篇进入上传与下载：FormData 的正确构造、上传进度、Blob 下载、Content-Disposition 文件名解析，以及大文件的分片与流式处理。
