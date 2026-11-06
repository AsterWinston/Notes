拦截器是 Axios 最有价值的能力，也是封装请求层时最容易写乱的地方。它让你在"请求发出前"和"响应返回后"插入统一逻辑，例如加 Token、解包数据、处理 401。本篇讲清执行顺序、鉴权注入、Token 刷新队列，以及那几个几乎人人都踩过的坑。

# 1. 拦截器是什么

## 1.1 生命周期位置

```text
http.get() → 请求拦截器 → adapter → 响应拦截器 → 业务代码
              加 Token              解包 / 错误
```

## 1.2 注册与注销

```javascript
const requestId = http.interceptors.request.use(
  (config) => config, // 必须返回 config
  (error) => Promise.reject(error)
)

const responseId = http.interceptors.response.use(
  (response) => response,
  (error) => Promise.reject(error)
)

http.interceptors.request.eject(requestId)
http.interceptors.response.eject(responseId)
```

★ **关键认知**：请求拦截器的 `onRejected` **不负责处理 HTTP 错误**。HTTP 错误发生在 adapter 之后，应由响应拦截器的错误回调处理；请求拦截器的错误回调只处理"前一个请求拦截器失败"。

# 2. 执行顺序

## 2.1 请求倒序，响应正序

Axios 内部把请求拦截器**从后往前**插入执行链，把响应拦截器**从前往后**追加：

```text
注册顺序：request A → request B
          response A → response B

实际执行：请求 B → 请求 A → adapter
          adapter → 响应 A → 响应 B
```

```javascript
http.interceptors.request.use((config) => {
  console.log('请求 A')
  return config
})
http.interceptors.request.use((config) => {
  console.log('请求 B')
  return config
})
http.interceptors.response.use((response) => {
  console.log('响应 A')
  return response
})
http.interceptors.response.use((response) => {
  console.log('响应 B')
  return response
})

await http.get('/ping')
// 输出：请求 B → 请求 A → 响应 A → 响应 B
```

★ **关键认知**：请求拦截器是**栈式**的，后注册先执行；响应拦截器是**队列式**的，先注册先执行。这对应"洋葱模型"：越晚包装的请求逻辑，越早接触原始请求，也越晚接触最终响应。⚠ 依赖顺序的拦截器（例如"先加 traceId 再加 Token"）必须倒着注册。

# 3. 请求拦截器实战

## 3.1 注入鉴权头

```javascript
export const http = axios.create({
  baseURL: '/api',
  timeout: 10000
})

http.interceptors.request.use((config) => {
  if (config.skipAuth) return config

  const token = localStorage.getItem('access_token')
  if (token) {
    // Axios 1.x 的 headers 是 AxiosHeaders，优先用 set
    config.headers.set('Authorization', `Bearer ${token}`)
  }
  return config
})

await http.post('/auth/login', data, { skipAuth: true })
```

`skipAuth` 不是 Axios 内置字段，TypeScript 下需要模块扩展，第 7 篇会讲。

## 3.2 改写请求配置

```javascript
http.interceptors.request.use((config) => {
  // 给所有 GET 请求加时间戳，绕过某些缓存
  if (config.method === 'get') {
    config.params = { ...config.params, _t: Date.now() }
  }
  return config
})
```

★ **关键认知**：拦截器里对 `config` 的修改会直接影响真实请求，因此**不要做有副作用且不可重复执行的操作**（例如自增计数器）。请求可能因为重试再次经过拦截器，逻辑必须幂等。

# 4. 响应拦截器实战

## 4.1 解包响应

```javascript
http.interceptors.response.use(
  (response) => response.data,
  (error) => Promise.reject(error)
)

const users = await http.get('/users') // 直接就是数据
```

⚠ 解包后 `http.get()` 的运行时返回值不再是 `AxiosResponse`，但类型默认仍是。要么让拦截器只做错误处理、业务层统一 `const { data } = await http.get()`，要么拦截器解包并配套修正 TypeScript 类型。**两种风格不要混用**。

## 4.2 业务错误

很多后端用 HTTP 200 返回业务错误：

```json
{ "code": 1001, "message": "余额不足", "data": null }
```

```javascript
http.interceptors.response.use(
  (response) => {
    const body = response.data
    if (body && typeof body.code === 'number' && body.code !== 0) {
      const error = new Error(body.message || '业务处理失败')
      error.name = 'BusinessError'
      error.code = body.code
      error.response = response
      return Promise.reject(error)
    }
    return body.data
  },
  (error) => Promise.reject(error)
)
```

★ **关键认知**：业务错误和 HTTP 错误是**两个层次**。HTTP 200 只代表"服务器成功处理了这次 HTTP 请求"，不代表业务成功。必须约定哪一层负责判断 `code`，否则调用方会拿到"成功但 data 为 null"的诡异结果。

# 5. 401 刷新 Token

## 5.1 单飞与队列

多个并发请求同时收到 401 时，如果每个都去刷新 Token，会发出多次刷新请求，甚至互相覆盖。正确做法是**单飞（single-flight）**：只允许一个刷新请求，其余排队等待。

```text
请求 A ─┐
请求 B ─┼─ 401 ─→ 已在刷新？
请求 C ─┘            │
                     ├─ 否：发起 refresh，其他入队
                     └─ 是：直接入队
                              │
                     refresh 成功 → 队列统一重放
```

## 5.2 完整示例

```javascript
const http = axios.create({ baseURL: '/api', timeout: 10000 })

// 刷新专用实例：不挂业务拦截器，避免死循环
const refreshClient = axios.create({ baseURL: '/api', timeout: 10000 })

let isRefreshing = false
let waitingQueue = [] // { resolve, reject, config }

function flushQueue(error, token = null) {
  waitingQueue.forEach(({ resolve, reject, config }) => {
    if (error) {
      reject(error)
    } else {
      config.headers.set('Authorization', `Bearer ${token}`)
      resolve(http(config)) // 重放原请求
    }
  })
  waitingQueue = []
}

http.interceptors.response.use(
  (response) => response,
  async (error) => {
    const { response, config } = error

    if (!response || response.status !== 401 || config._retry) {
      return Promise.reject(error)
    }
    if (config.url.includes('/auth/login') || config.url.includes('/auth/refresh')) {
      return Promise.reject(error)
    }

    config._retry = true

    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        waitingQueue.push({ resolve, reject, config })
      })
    }

    isRefreshing = true
    try {
      const refreshToken = localStorage.getItem('refresh_token')
      const { data } = await refreshClient.post('/auth/refresh', { refreshToken })
      localStorage.setItem('access_token', data.accessToken)
      flushQueue(null, data.accessToken)
      return http(config)
    } catch (refreshError) {
      flushQueue(refreshError)
      localStorage.removeItem('access_token')
      localStorage.removeItem('refresh_token')
      window.location.assign('/login')
      return Promise.reject(refreshError)
    } finally {
      isRefreshing = false
    }
  }
)
```

## 5.3 关键点

| 点 | 说明 |
|----|------|
| `_retry` | 防止同一个请求无限重试 |
| 刷新实例独立 | 避免刷新请求再次触发 401 逻辑 |
| 排除登录/刷新 | 这两个接口 401 不应走刷新 |
| 队列统一重放 | 刷新成功后所有等待请求继续 |
| `finally` 复位 | 保证 `isRefreshing` 回到 false |

⚠ 刷新成功后重放仍可能 401（例如会话被撤销）。`_retry` 只挡一次，第二次直接失败，这是符合预期的。

# 6. 常见坑

## 6.1 忘记 return config

```javascript
// 错误：后续拦截器和 adapter 收到 undefined
http.interceptors.request.use((config) => {
  config.headers.set('X-Token', token)
})

// 正确
http.interceptors.request.use((config) => {
  config.headers.set('X-Token', token)
  return config
})
```

## 6.2 在响应拦截器里吞掉错误

```javascript
// 危险：业务层的 catch 永远不执行
http.interceptors.response.use(
  (res) => res,
  (err) => {
    showToast(err.message) // 只提示，没有继续抛
  }
)
```

## 6.3 重复注册与重复提示

全局拦截器弹一次、页面 `catch` 又弹一次，是重复提示的常见来源。可以在错误对象上打标记，或约定"全局只处理网络/系统错误，业务错误交给页面"。

# 7. 练习

## 7.1 练习一：验证顺序

注册两个请求拦截器和两个响应拦截器，每个打印名字。先预测输出，再运行验证；然后对调请求拦截器的注册顺序，观察变化。

## 7.2 练习二：跳过鉴权

给请求配置增加 `skipAuth` 字段，使登录接口不携带 `Authorization`。分别用 JavaScript 和 TypeScript 实现，TypeScript 版本不能报类型错误。

## 7.3 练习三：改造刷新队列

把刷新失败时的"跳转登录页"改成"只清理 Token 并抛错"，由路由守卫决定是否跳转，并说明这样改的好处。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 请求拦截器 | 后注册先执行，必须 return config |
| 响应拦截器 | 先注册先执行，错误必须继续 reject |
| 解包风格 | 拦截器解包或业务层解包，二选一 |
| 401 刷新 | 单飞 + 队列 + `_retry` + 独立实例 |
| 常见坑 | 忘返回值、吞错误、重复注册、重复提示 |

下一篇进入取消与超时：AbortController 如何中断请求、CancelToken 为何被废弃、如何取消重复请求，以及如何用并发池控制同时进行的请求数量。
