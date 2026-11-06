直接使用全局 `axios` 发请求，项目一大就会出现两个问题：每个请求都在重复写 baseURL 和超时，以及某个页面改了全局配置导致其他页面跟着变。解决办法是**实例化**——把一组稳定的配置固化成一个独立的 `AxiosInstance`，再让业务代码只关心路径和数据。本篇讲清 `create()`、配置项、合并优先级，以及环境变量如何安全地注入。

# 1. 为什么需要实例

## 1.1 全局 axios 的问题

```javascript
// 每个请求都重复
axios.get('https://api.example.com/v1/users', { timeout: 10000 })
axios.post('https://api.example.com/v1/orders', data, { timeout: 10000 })

// 某处改了全局默认值，影响所有请求
axios.defaults.timeout = 3000
```

全局对象是**单例**，任何一次 `axios.defaults` 的修改都会影响整个应用；多个后端、多个超时策略时也无法区分。

## 1.2 实例的基本形态

```javascript
// src/api/http.js
import axios from 'axios'

const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL, // 例如 https://api.example.com/v1
  timeout: 10000,                              // 10 秒
  headers: {
    'Content-Type': 'application/json'
  }
})

export default http
```

★ **关键认知**：`axios.create(config)` 返回一个**全新的 AxiosInstance**，它拥有独立的 defaults、interceptors 和 adapter 配置。全局 `axios` 本身也是一个实例，`create()` 相当于复制一份默认配置再做覆盖。⚠ 实例之间不会共享拦截器，一个实例注册的拦截器不会影响另一个。

# 2. create 与常用配置

## 2.1 配置项全景

| 配置 | 作用 | 常见值 |
|------|------|--------|
| `baseURL` | 所有相对 URL 的前缀 | `https://api.example.com/v1` |
| `timeout` | 请求超时毫秒数 | `10000`，`0` 表示不超时 |
| `headers` | 默认请求头 | `Authorization`、`Content-Type` |
| `params` | URL 查询参数 | `{ page: 1 }` |
| `data` | 请求体 | 对象、FormData、字符串 |
| `responseType` | 响应类型 | `json`、`text`、`blob`、`arraybuffer`、`stream` |
| `withCredentials` | 跨域是否携带 Cookie | `true` / `false` |
| `auth` | HTTP Basic Auth | `{ username, password }` |
| `validateStatus` | 自定义成功状态码 | `status => status < 500` |
| `paramsSerializer` | 自定义查询参数序列化 | 数组、对象嵌套 |
| `maxContentLength` | 响应体最大字节数 | 防止超大响应 |
| `maxBodyLength` | 请求体最大字节数 | 主要用于 Node |
| `proxy` | Node 代理 | `{ host, port }` |
| `signal` | 取消信号 | `AbortController.signal` |

## 2.2 baseURL 的拼接规则

Axios 的 `combineURLs` 逻辑很直接：去掉 baseURL 末尾的 `/`，去掉相对 URL 开头的 `/`，再用一个 `/` 连接。

```javascript
const http = axios.create({ baseURL: 'https://api.example.com/v1' })

http.get('/users')  // → https://api.example.com/v1/users
http.get('users')   // → https://api.example.com/v1/users
```

★ **关键认知**：当请求 URL 本身是**绝对地址**时，`baseURL` 默认会被忽略。例如 `http.get('https://other.com/data')` 仍然请求 `other.com`，不会拼上 baseURL。Axios 1.8+ 提供 `allowAbsoluteUrls: false` 强制不允许绝对地址，适合需要锁死域名的场景。

⚠ 反向代理场景要特别注意：如果 baseURL 写成 `/api`，而 Nginx 把 `/api` 转发到后端时没有去掉前缀，最终后端收到的路径会是 `/api/users`，和预期可能不一致。

## 2.3 params 与 data 的区别

| 位置 | 配置项 | 出现在 | 适用方法 |
|------|--------|--------|---------|
| URL 查询串 | `params` | `?page=1&size=10` | GET、DELETE 等 |
| 请求体 | `data` | body | POST、PUT、PATCH |

```javascript
// GET 用 params
await http.get('/users', { params: { page: 1, size: 10 } })

// POST 用 data
await http.post('/users', { name: 'Ada', age: 36 })
```

数组的默认序列化是"方括号形式"：

```javascript
await http.get('/users', { params: { ids: [1, 2] } })
// 默认 → /users?ids[]=1&ids[]=2
```

需要逗号分隔或自定义格式时，用 `paramsSerializer.serialize` 改写序列化逻辑；Axios 1.x 也支持 `indexes: null` 等选项控制数组格式。

## 2.4 headers 的合并

Axios 1.x 使用 `AxiosHeaders` 管理请求头，合并顺序大致是：

```text
axios.defaults.headers.common
        ↓ 被实例覆盖
instance.defaults.headers.common
        ↓ 被方法级覆盖
instance.defaults.headers.get / post / put ...
        ↓ 被单次请求覆盖
config.headers
```

```javascript
const http = axios.create({
  baseURL: '/api',
  headers: {
    common: { 'X-App': 'notes' }, // 所有方法
    get: { 'Cache-Control': 'no-cache' },
    post: { 'Content-Type': 'application/json' }
  }
})

// 单次请求覆盖
await http.get('/users', {
  headers: { 'X-App': 'mobile' }
})
```

★ **关键认知**：请求头名称是**大小写不敏感**的，Axios 内部会用 `AxiosHeaders` 归一化。修改时优先用 `headers.set(name, value)` 或直接赋值，删除用 `headers.delete(name)`。⚠ 不要直接 `config.headers = {...}` 整体替换，容易丢掉 Axios 已经合并进去的默认头。

# 3. 默认配置与优先级

## 3.1 五层优先级

```text
全局默认 axios.defaults
        ↓
实例默认 instance.defaults
        ↓
请求配置 config
        ↓
请求拦截器对 config 的修改
        ↓
最终 adapter 收到的配置
```

后面的层级覆盖前面的层级。拦截器之所以在最后，是因为它在请求真正发出前执行。

```javascript
// 全局
axios.defaults.timeout = 3000

// 实例覆盖全局
const http = axios.create({ timeout: 10000 })

// 单次请求覆盖实例
await http.get('/slow-report', { timeout: 60000 })
```

## 3.2 修改默认配置的风险

```javascript
// 不推荐：全局副作用
axios.defaults.headers.common.Authorization = `Bearer ${token}`

// 推荐：只影响当前实例
http.defaults.headers.common.Authorization = `Bearer ${token}`
```

⚠ 在 SSR 场景下，全局默认值会被**多个请求共享**。如果登录用户的 Token 写进了全局 headers，可能泄露给下一个用户。这类状态必须放在"每次请求的上下文"里，而不是模块级单例上。

## 3.3 什么时候该建多个实例

| 场景 | 是否拆分 | 说明 |
|------|---------|------|
| 单一后端 | 一个实例 | 最简单 |
| 主站 + 第三方 API | 两个实例 | 超时、鉴权、错误处理不同 |
| 需要上传大文件 | 单独实例 | 超时更长、可能不重试 |
| 需要静默请求 | 单独实例 | 不弹全局错误提示 |

```javascript
// 业务接口
export const http = axios.create({ baseURL: '/api', timeout: 10000 })

// 文件服务：超时更长，不自动弹提示
export const uploadHttp = axios.create({
  baseURL: '/upload',
  timeout: 120000
})

// 第三方服务：完全独立的鉴权
export const thirdPartyHttp = axios.create({ baseURL: 'https://api.third.com' })
```

# 4. 环境变量与多环境

## 4.1 不同构建工具的变量前缀

| 工具 | 变量前缀 | 访问方式 |
|------|---------|---------|
| Vite | `VITE_` | `import.meta.env.VITE_API_BASE_URL` |
| Create React App | `REACT_APP_` | `process.env.REACT_APP_API_BASE_URL` |
| Next.js | `NEXT_PUBLIC_` | `process.env.NEXT_PUBLIC_API_BASE_URL` |
| Vue CLI | `VUE_APP_` | `process.env.VUE_APP_API_BASE_URL` |
| Node 脚本 | 自定义 | `process.env.API_BASE_URL` |

```bash
# .env.development
VITE_API_BASE_URL=/api
VITE_APP_TITLE=Notes Dev

# .env.production
VITE_API_BASE_URL=https://api.example.com/v1
```

```javascript
// src/api/http.js
const baseURL = import.meta.env.VITE_API_BASE_URL

if (!baseURL) {
  throw new Error('缺少 VITE_API_BASE_URL，请检查 .env 文件')
}

export const http = axios.create({ baseURL, timeout: 10000 })
```

★ **关键认知**：所有以构建工具前缀开头的变量都会**被打进前端产物**，任何访问页面的人都能看到。⚠ 因此 `.env` 只能放公开配置，例如 API 地址、埋点开关；**绝不能放数据库密码、第三方 Secret、私钥**。需要保密的值必须留在服务端。

## 4.2 环境判断与兜底

⚠ 构建时变量是**静态替换**的，不能靠 `.env` 实现"一次构建、多环境部署"；这种场景要用运行时配置文件或服务端注入。生产环境缺失 API 地址时应直接抛错，而不是回退到 `localhost`。

# 5. 经典坑

## 5.1 baseURL 末尾斜杠导致双斜杠

```javascript
// 不推荐
axios.create({ baseURL: 'https://api.example.com/' })
// 请求 '/users' 时通常没问题，但和手工拼接混用可能出现 //
```

统一约定：baseURL 不带末尾斜杠，请求路径以 `/` 开头，或者干脆全部用相对路径。

## 5.2 timeout 设成 0 以为会超时

`timeout: 0` 表示**不限制超时**，请求可能一直挂着。生产环境应显式设置一个合理值，并用 AbortController 处理页面切换。

## 5.3 在实例上重复注册拦截器

热更新或模块重复执行时，拦截器会被注册多次，导致同一个 Token 被加多次、错误提示弹多次。注册时保存返回的 id，在销毁时 `eject`，第 3 篇会给出完整写法。

## 5.4 把 Content-Type 写死在实例上

```javascript
// 不推荐：上传 FormData 时会被这个头覆盖
axios.create({
  headers: { 'Content-Type': 'application/json' }
})
```

上传文件时应让浏览器自动生成 `multipart/form-data; boundary=...`。实例级只设置 `Accept` 这类通用头，或者在具体请求里覆盖。

# 6. 练习

## 6.1 练习一：拆分两个实例

为一个项目创建 `http` 与 `uploadHttp` 两个实例：前者超时 10 秒、baseURL 为 `/api`；后者超时 120 秒、baseURL 为 `/upload`。要求两者都不在全局 `axios` 上留下配置。

## 6.2 练习二：验证优先级

依次设置全局 `timeout = 3000`、实例 `timeout = 8000`、单次请求 `timeout = 15000`，写一段代码打印最终生效的值。提示：可以在请求拦截器里打印 `config.timeout`。

## 6.3 练习三：安全的环境变量

写一个 `readApiBaseUrl()`，从 `import.meta.env` 读取地址；如果缺失则抛出明确错误，并说明为什么不能把 Secret 放进 `VITE_` 变量。

# 7. 小结

| 概念 | 要点 |
|------|------|
| create | 创建独立实例，拥有独立 defaults 与拦截器 |
| baseURL | 只拼接相对 URL；绝对 URL 默认绕过 |
| params/data | 查询串用 params，请求体用 data |
| headers | 多层合并，大小写不敏感，优先用 set |
| 优先级 | 全局 → 实例 → 请求 → 拦截器 |
| 环境变量 | 前缀变量会进入前端产物，禁止放密钥 |

下一篇进入拦截器：请求与响应拦截器的执行顺序、鉴权头的注入、401 刷新 Token 的队列方案，以及重复注册、无限重试这些高频坑。
