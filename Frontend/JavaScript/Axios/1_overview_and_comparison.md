Axios 是前端最常用的 HTTP 客户端之一。很多人对它的印象停留在"比 fetch 少写几行"，但它真正的价值在于**实例化配置、拦截器、统一错误模型和跨端适配**。本篇先把它放回 HTTP 客户端的位置，和 fetch、XMLHttpRequest 做对照，再用一个最小请求跑通浏览器与 Node 两条链路。

# 1. Axios 的定位

## 1.1 它解决什么问题

浏览器发请求的底层能力一直存在，麻烦的是**每次都要重复处理同一批杂事**：

- 拼接 baseURL、查询参数、请求头。
- 把对象序列化成 JSON，再把响应文本解析回对象。
- 判断 HTTP 状态码，决定抛错还是返回数据。
- 统一加 Token、统一弹错误提示、统一处理 401。
- 上传进度、超时、取消、重试。

Axios 把这些能力收进一个**基于 Promise 的客户端**，并允许你通过实例和拦截器集中配置。业务代码只需要关心"请求哪个接口、拿什么数据"。

```javascript
// 没有封装时，业务代码很容易变成这样
const res = await fetch(`${BASE_URL}/users?page=1`, {
  headers: { Authorization: `Bearer ${token}` }
})
if (!res.ok) throw new Error(`HTTP ${res.status}`)
const json = await res.json()

// 使用 Axios 并做好实例配置后
const { data } = await http.get('/users', { params: { page: 1 } })
```

★ **关键认知**：Axios 不是新的网络协议，它仍然是**对 XMLHttpRequest 或 Node http 模块的封装**。它提供的是"请求生命周期管理"，而不是"更快的网络"。⚠ 因此它不能绕过同源策略、CORS、证书校验这些浏览器或服务端规则。

## 1.2 核心能力一览

| 能力 | 说明 | 典型用途 |
|------|------|---------|
| Promise API | 所有请求返回 Promise | async/await 写法 |
| 自动 JSON | 请求体序列化、响应体反序列化 | REST 接口 |
| 实例配置 | `axios.create()` 固化 baseURL、timeout、headers | 多后端、多环境 |
| 拦截器 | 请求前、响应后插入逻辑 | 鉴权、刷新 Token、错误提示 |
| 取消 | AbortController、CancelToken | 搜索联想、页面切换 |
| 进度 | 上传、下载进度回调 | 文件上传、导出 |
| 跨端 | 浏览器与 Node 共用 API | SSR、脚本、CLI |
| 错误模型 | `AxiosError` 携带 config、response、code | 统一错误处理 |

# 2. 与 fetch / XHR 的对比

## 2.1 三者是什么关系

```text
┌──────────────────────────────────────────────┐
│  业务代码                                     │
│  axios.get() / fetch() / new XMLHttpRequest() │
└───────────────┬──────────────────────────────┘
                │
     ┌──────────┼───────────┐
     ▼          ▼           ▼
┌─────────┐ ┌────────┐ ┌──────────────┐
│ Axios   │ │ fetch  │ │ XMLHttpRequest│
│ 适配层   │ │ 浏览器  │ │ 浏览器原生    │
└────┬────┘ └────────┘ └──────────────┘
     │
     ├─ 浏览器: xhr adapter → XMLHttpRequest
     └─ Node:   http adapter → http/https
```

XMLHttpRequest 是最底层的浏览器 API，fetch 是它的现代化替代，Axios 则是**站在适配层之上的封装**。在 Axios 1.x 中，浏览器默认仍走 xhr adapter，Node 走 http adapter；也可以显式选择 fetch adapter。

## 2.2 关键差异对照

| 维度 | Axios | fetch | XMLHttpRequest |
|------|-------|-------|----------------|
| API 风格 | Promise，配置对象 | Promise，配置对象 | 事件回调 |
| 默认 JSON | 自动序列化/反序列化 | 需要手动 `JSON.stringify` / `.json()` | 手动 |
| HTTP 错误 | 默认 4xx/5xx 直接 reject | 只有网络错误才 reject | 需自己判断 status |
| 超时 | `timeout` 配置 | 需配合 AbortSignal | `xhr.timeout` |
| 取消 | AbortController / CancelToken | AbortController | `xhr.abort()` |
| 上传进度 | `onUploadProgress` | 不直接支持 | `xhr.upload.onprogress` |
| 下载进度 | `onDownloadProgress` | 需读 stream | `xhr.onprogress` |
| 拦截器 | 内置 | 无，需自行包装 | 无 |
| 跨端 | 浏览器 + Node 共用 | 现代浏览器 + Node 18+ | 仅浏览器 |
| 体积 | 约十几 KB | 浏览器内置，零依赖 | 浏览器内置 |

★ **关键认知**：fetch **不会因为 404、500 而 reject**。它只在网络层失败（断网、DNS 失败、被 CORS 拦截）时 reject，HTTP 错误仍然是一个"成功的响应"，必须自己检查 `response.ok`。Axios 默认把 2xx 之外的状态码都当作错误抛出，这是两者最容易踩混的地方。

```javascript
// fetch：业务层必须自己判断状态码
const res = await fetch('/api/not-exist')
console.log(res.ok)        // false
const body = await res.json() // 仍然可以读响应体

// axios：非 2xx 直接进入 catch
try {
  await axios.get('/api/not-exist')
} catch (err) {
  console.log(err.response.status) // 404
}
```

## 2.3 什么时候选谁

| 场景 | 建议 | 原因 |
|------|------|------|
| 中大型业务、接口多 | Axios | 实例、拦截器、统一错误 |
| 需要上传/下载进度 | Axios | 原生回调支持完善 |
| 只想发一两个简单请求 | fetch | 零依赖，浏览器内置 |
| 需要精细控制流式响应 | fetch | 可直接读 `response.body` |
| 兼容老浏览器 | Axios + polyfill | XHR 链路成熟 |
| Node 脚本 | Axios 或 fetch | Node 18+ 自带 fetch |

⚠ **不要为了"新"而盲目换 fetch**：如果项目已经有成熟的 Axios 封装，换 fetch 意味着重新实现拦截器、错误归一化、取消与重试，收益通常抵不过迁移成本。

# 3. 安装与第一个请求

## 3.1 安装

```bash
# npm
npm install axios

# pnpm
pnpm add axios

# yarn
yarn add axios
```

Axios 1.x 同时提供 CommonJS 与 ESM 入口，现代打包器会优先使用 ESM。浏览器直接通过 `<script>` 引入时，会暴露全局变量 `axios`。

```html
<script src="https://cdn.jsdelivr.net/npm/axios@1/dist/axios.min.js"></script>
```

## 3.2 最小请求

```javascript
import axios from 'axios'

async function loadUsers() {
  try {
    const response = await axios.get('https://jsonplaceholder.typicode.com/users', {
      params: { _limit: 3 } // 会被拼成 ?_limit=3
    })
    console.log(response.data) // 已经是解析好的数组
  } catch (error) {
    // 网络错误、超时、非 2xx 都会走到这里
    console.error('请求失败:', error.message)
  }
}

loadUsers()
```

`axios.get(url, config)` 的第二个参数是配置对象，不是请求体。POST 才是 `axios.post(url, data, config)`。

```javascript
// GET：第二个参数是 config
await axios.get('/api/users', { params: { page: 1 } })

// POST：第二个参数是 data，第三个才是 config
await axios.post('/api/users', { name: 'Ada' }, { timeout: 5000 })
```

## 3.3 响应结构

Axios 返回的响应对象不是裸数据，而是一个固定结构的对象：

```javascript
{
  data: {},          // 服务端返回的响应体，JSON 已被自动解析
  status: 200,       // HTTP 状态码
  statusText: 'OK',  // 状态文本
  headers: {},       // 响应头
  config: {},        // 本次请求的最终配置
  request: {}        // 底层请求对象（XHR 或 http.ClientRequest）
}
```

因此业务里常见的写法是 `const { data } = await axios.get(...)`。如果响应拦截器已经统一解包，那么返回值可能直接就是业务数据，两种风格不要混用。

★ **关键认知**：`response.data` 的类型取决于 `responseType` 和服务端 `Content-Type`。默认情况下，JSON 会被解析成对象；`text` 返回字符串；`blob` 返回 Blob；`arraybuffer` 返回 ArrayBuffer。⚠ 如果服务端返回 JSON 但 Content-Type 写成 `text/plain`，Axios 可能不会自动解析，需要检查响应头。

# 4. 浏览器与 Node 的差异

## 4.1 适配器机制

Axios 通过 adapter 把统一 API 映射到不同运行时：

```text
axios.request(config)
        │
        ├─ 浏览器默认 xhr adapter
        │    └─ new XMLHttpRequest()
        │
        ├─ Node 默认 http adapter
        │    └─ http.request() / https.request()
        │
        └─ 可选 fetch adapter
             └─ fetch()
```

这也是同一份请求代码可以同时跑在浏览器和 Node 的原因。但"API 一样"不等于"行为完全一样"。

## 4.2 常见差异

| 维度 | 浏览器 | Node |
|------|--------|------|
| 相对 URL | 可用，基于当前页面 | 不可用，需要绝对 URL 或 baseURL |
| Cookie | `withCredentials` 控制跨域携带 | 需手动设置 `Cookie` 头 |
| FormData | 原生 `FormData` | Node 18+ 原生，旧版本需 `form-data` |
| Blob/File | 原生支持 | Node 18+ 有 Blob，File 视版本 |
| 流式响应 | xhr adapter 不支持 `stream` | `responseType: 'stream'` 可用 |
| 代理 | 浏览器无 | 可用 `httpAgent` / `httpsAgent` |
| 超时默认值 | 0，表示不超时 | 0，表示不超时 |
| CORS | 受同源策略限制 | 不受浏览器 CORS 限制 |

```javascript
// Node 环境：URL 必须能解析成绝对地址
await axios.get('https://api.example.com/users')

// 只写相对路径在 Node 会报错，除非配置了 baseURL
await axios.get('/users', { baseURL: 'https://api.example.com' })
```

⚠ **Node 里没有浏览器的同源策略**，所以 CORS 问题不会出现；但这也意味着服务端请求可以访问内网地址，做 SSR 或代理时必须自行校验目标 URL，防止 SSRF。

# 5. 常见误区

## 5.1 把 Axios 当成 fetch 的替代品

Axios 的默认行为更"业务友好"，例如非 2xx 抛错、自动 JSON。迁移到 fetch 时要补齐这些行为，否则错误处理会静默失效。

## 5.2 忘记 await 或忘记解构

```javascript
// 错误：拿到的是 Promise，不是响应
const data = axios.get('/api/users')
console.log(data.data) // undefined

// 正确
const { data } = await axios.get('/api/users')
```

## 5.3 认为取消请求也会走普通错误分支

取消产生的错误 `code` 是 `ERR_CANCELED`，应该单独识别，不要当成网络失败弹提示。

```javascript
try {
  await axios.get('/api/search', { signal: controller.signal })
} catch (error) {
  if (axios.isCancel(error)) return // 主动取消，静默处理
  throw error
}
```

## 5.4 在浏览器里手动设置 FormData 的 Content-Type

浏览器需要自己生成 `multipart/form-data; boundary=...`。手动写死 `Content-Type: multipart/form-data` 会丢掉 boundary，服务端无法解析。这个坑在第 6 篇会详细展开。

# 6. 练习

## 6.1 练习一：对比错误行为

分别用 fetch 和 Axios 请求一个返回 404 的地址，打印各自的走向：fetch 是否会进入 catch，Axios 是否会进入 catch，以及两者的 `status` 从哪里取。

## 6.2 练习二：封装一个最小 GET

写一个 `getJSON(url, params)`，内部使用 Axios，返回 `response.data`，并在失败时抛出一个包含 `status` 与 `message` 的新错误对象。

## 6.3 练习三：判断运行环境

写一个函数 `buildUrl(path)`：浏览器环境返回相对路径，Node 环境拼接 `process.env.API_ORIGIN` 后返回绝对 URL。提示：可以用 `typeof window === 'undefined'` 判断。

# 7. 小结

| 概念 | 要点 |
|------|------|
| 定位 | 基于 Promise 的 HTTP 客户端，封装 XHR/http |
| 与 fetch | Axios 默认对非 2xx 抛错、自动 JSON、内置拦截器 |
| 与 XHR | XHR 是底层 API，Axios 在其上做生命周期管理 |
| 响应结构 | `data`、`status`、`headers`、`config`、`request` |
| 跨端 | 浏览器走 xhr adapter，Node 走 http adapter |
| 边界 | 不能绕过 CORS、同源策略、证书校验 |

下一篇进入实例与配置：用 `axios.create()` 把 baseURL、timeout、headers、params 固化下来，并讲清默认配置的合并顺序与环境变量的注入方式。
