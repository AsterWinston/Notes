搜索联想、页面切换、重复点击提交，这些场景都会产生"已经没用了的请求"。放任不管，轻则浪费带宽、旧响应覆盖新结果，重则重复下单。Axios 提供取消与超时两套机制，本篇从 AbortController 讲到并发池，把"什么时候停、怎么停、停完怎么收尾"讲清楚。

# 1. 为什么需要取消

## 1.1 三个典型场景

搜索联想会被旧响应覆盖，页面切换后回调可能修改已卸载组件，重复提交会创建多笔订单，大文件下载会持续占用带宽。这些问题的共同解法都是"主动取消"。

## 1.2 取消的本质

取消是**客户端主动终止这次请求**：浏览器 abort 对应的 XHR，Promise 以特殊错误 reject。

```text
abort() → signal 触发 → adapter 取消 → Promise reject(ERR_CANCELED)
```

★ **关键认知**：取消**不等于服务端停止处理**。请求可能已到达服务器，服务器仍可能完成写库、扣款。客户端取消只能阻止"继续等待响应"和"后续回调"，不能当作事务回滚。⚠ 支付、下单这类接口必须由服务端保证幂等。

# 2. AbortController 现代方案

## 2.1 基本用法

```javascript
const controller = new AbortController()

const promise = http.get('/api/search', {
  params: { q: 'axios' },
  signal: controller.signal // 把信号交给 Axios
})

controller.abort() // 需要取消时

try {
  await promise
} catch (error) {
  if (axios.isCancel(error)) {
    console.log('请求已取消')
  } else {
    throw error
  }
}
```

## 2.2 识别取消

Axios 1.x 取消时抛出 `CanceledError`：

```javascript
try {
  await http.get('/api/slow', { signal: controller.signal })
} catch (error) {
  console.log(axios.isCancel(error))                 // true
  console.log(error.code)                            // ERR_CANCELED
  console.log(error instanceof axios.CanceledError)  // true
}
```

`axios.isCancel` 同时兼容旧的 `CancelToken`，是跨版本最稳的判断。⚠ 不要把取消当网络错误弹提示；统一错误处理应最先排除 `ERR_CANCELED`。

## 2.3 用信号做超时

现代浏览器支持 `AbortSignal.timeout()`：

```javascript
await http.get('/api/report', {
  signal: AbortSignal.timeout(5000) // 5 秒后自动 abort
})
```

`timeout` 由 Axios/adapter 计时，`AbortSignal.timeout` 是浏览器标准信号；两者同时设置时先到期先触发。手工 `setTimeout + abort` 适合兼容旧环境。

## 2.4 组合多个信号

同一个请求可能既要"页面离开取消"，又要"最多等 5 秒"：

```javascript
const pageController = new AbortController()

const signal = AbortSignal.any([
  pageController.signal,
  AbortSignal.timeout(5000)
])

await http.get('/api/data', { signal })
```

⚠ `AbortSignal.any` 与 `AbortSignal.timeout` 都是较新的标准，旧浏览器需降级为手工 `setTimeout` + 单个 controller。

# 3. CancelToken 旧 API

## 3.1 曾经的写法

```javascript
// 已废弃，仅用于维护老项目
const source = axios.CancelToken.source()

http.get('/api/users', { cancelToken: source.token })
source.cancel('用户离开了页面')
```

## 3.2 为什么被废弃

| 维度 | CancelToken | AbortController |
|------|-------------|-----------------|
| 标准 | Axios 私有 | Web 标准 |
| 跨库复用 | 只能给 Axios | fetch、XHR、流共用 |
| 状态 | 已废弃 | 推荐 |

Axios 在 0.22.0 引入 `signal`，1.x 仍保留 CancelToken 但不再推荐。迁移只需把 `cancelToken: source.token` 换成 `signal: controller.signal`，把 `source.cancel()` 换成 `controller.abort()`；自定义取消消息可给 `abort()` 传 reason。

# 4. 超时

## 4.1 timeout 配置

```javascript
const http = axios.create({
  baseURL: '/api',
  timeout: 10000 // 10 秒；0 表示不限制
})

await http.get('/api/export', { timeout: 120000 })
```

超时错误通常长这样：

```javascript
{
  code: 'ECONNABORTED', // 部分适配器或版本可能是 ETIMEDOUT
  message: 'timeout of 10000ms exceeded',
  config: { /* ... */ }
}
```

判断时不要只盯一个 code：

```javascript
const isTimeout = (error) =>
  axios.isAxiosError(error)
  && (error.code === 'ECONNABORTED' || error.code === 'ETIMEDOUT')
```

## 4.2 timeout 不等于 signal 超时

- `timeout` 是 Axios 的请求配置，作用于整个请求生命周期。
- `AbortSignal.timeout` 是通用取消信号，Axios 只是监听它。
- 两者同时设置时，先到期的先触发。

★ **关键认知**：超时不是"请求失败"的同义词，而是"等待超过阈值"。服务端可能仍在处理。⚠ 只有幂等请求（GET、带幂等键的 PUT）才适合在超时后重试；POST 自动重试可能重复创建。无论成功、失败还是超时，都要在 `finally` 里复位 loading、清理定时器，并避免更新已卸载组件的状态。

# 5. 重复请求取消

## 5.1 思路

为同一个请求生成唯一 key，发出新请求前取消旧的：

```text
请求 A 发出 → pending[key] = A
请求 B 发出 → 发现 A，abort A → pending[key] = B
A 的 catch 收到 ERR_CANCELED → 静默忽略
B 正常返回
```

## 5.2 拦截器实现

```javascript
const pending = new Map()

function keyOf(config) {
  return [
    config.method,
    config.url,
    JSON.stringify(config.params || {})
  ].join('&')
}

http.interceptors.request.use((config) => {
  const key = keyOf(config)
  pending.get(key)?.abort() // 取消上一次同 key 请求

  const controller = new AbortController()
  config.signal = controller.signal
  pending.set(key, controller)
  return config
})

function cleanup(config) {
  if (config) pending.delete(keyOf(config))
}

http.interceptors.response.use(
  (response) => {
    cleanup(response.config)
    return response
  },
  (error) => {
    cleanup(error.config)
    return Promise.reject(error)
  }
)
```

★ **关键认知**：清理必须放在**成功和失败两条路径**上。只在成功回调里 `delete`，请求一失败 key 就永远挂在 Map 里，后续同 key 请求会误取消一个已结束的请求。⚠ 更稳妥的是在 `finally` 里清理，或给 pending 加超时兜底。

## 5.3 两个注意点

- 上面的写法直接覆盖 `config.signal`，调用方自己的 signal 会被丢掉。需要合并时用 `AbortSignal.any([config.signal, controller.signal])`。
- `JSON.stringify(FormData)` 得到 `{}`，不同文件上传会被误判为同一个 key。上传请求应按 `url + 文件名 + size` 去重，或干脆不去重。

# 6. 并发控制

## 6.1 为什么要限制

浏览器对同域名并发连接有限制，但业务层一次 `Promise.all` 发几百个请求，仍会造成请求排队、服务端瞬时压力过大、失败重试放大流量。

## 6.2 一个简单的并发池

```javascript
async function runPool(tasks, limit) {
  const results = new Array(tasks.length)
  let cursor = 0

  async function worker() {
    while (cursor < tasks.length) {
      const index = cursor++
      try {
        results[index] = { status: 'fulfilled', value: await tasks[index]() }
      } catch (error) {
        results[index] = { status: 'rejected', reason: error }
      }
    }
  }

  await Promise.all(
    Array.from({ length: Math.min(limit, tasks.length) }, () => worker())
  )
  return results
}

const tasks = urls.map((url) => () => http.get(url))
const results = await runPool(tasks, 3) // 最多同时 3 个
```

每个 worker 取一个任务，完成后再取下一个，天然形成"最多 limit 个在跑"的效果；结果按下标回填，顺序不会乱。

⚠ 并发池只控制"同时发出多少"，不负责去重、重试和取消。需要这些能力时，可把 `p-limit` 与拦截器组合，而不是自己造一个全能调度器。

# 7. 常见坑

## 7.1 一个 controller 给多个请求共用

`abort()` 会取消**所有**监听同一个 signal 的请求。搜索框每次输入都应新建 controller，只 abort 上一次那个。

## 7.2 请求结束后才 abort

响应已经返回，再调用 `abort()` 不会产生任何效果。不要指望用它"撤销已完成的业务"。

## 7.3 取消后仍然更新 UI

```javascript
try {
  const { data } = await http.get('/api/data', { signal })
  setData(data)
} catch (error) {
  if (axios.isCancel(error)) return // 组件已卸载或请求被替换
  throw error
}
```

## 7.4 用超时自动重试写操作

超时只说明"没等到响应"，不说明"服务端没执行"。POST 创建订单超时后自动重试，可能产生两笔订单。写操作要么不重试，要么带服务端幂等键。

# 8. 练习

## 8.1 练习一：可取消的搜索

每次输入新建 `AbortController`，取消上一次请求；捕获 `ERR_CANCELED` 时静默返回。连续快速输入三次，最终只渲染最后一次结果。

## 8.2 练习二：组合超时

用 `AbortSignal.any` 组合"页面卸载信号"与"5 秒超时信号"，请求一个慢接口。分别验证两种触发方式都能 reject，并打印 `axios.isCancel(error)`。

## 8.3 练习三：并发池

有 10 个 URL，最多同时请求 3 个，返回结果按原顺序排列，并统计成功与失败数量。

# 9. 小结

| 概念 | 要点 |
|------|------|
| AbortController | 标准取消方式，`signal` 交给 Axios |
| 识别取消 | `axios.isCancel` / `code === 'ERR_CANCELED'` |
| CancelToken | 已废弃，迁移到 signal |
| 超时 | `timeout` 是 Axios 配置，超时不等于服务端停止 |
| 重复请求 | key 去重 + 成功失败都清理 |
| 并发控制 | worker 池限制同时进行的请求数 |

下一篇进入错误处理：HTTP 错误、网络错误、业务错误如何区分，`AxiosError` 有哪些字段，如何设计统一错误结构，以及重试与全局兜底策略。
