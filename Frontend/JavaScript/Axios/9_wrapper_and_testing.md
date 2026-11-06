前面八篇讲的是零件：实例、拦截器、取消、错误、上传下载、类型、安全。真正进入项目时，这些零件需要组装成一个**统一请求层**，否则每个页面仍会各写各的。本篇给出一套可落地的分层封装，并补上 Mock 与单元测试，让请求层能被验证，而不是靠手点页面。

# 1. 为什么要封装

## 1.1 不封装的代价

每个页面都手写 `Authorization`、baseURL 和错误判断，结果是改一处要全项目搜；错误处理不一致，有的弹窗、有的静默；类型全是 `any`；无法统一取消、去重、重试、脱敏；测试只能靠手点。

## 1.2 分层目标

```text
页面 → API 模块（modules/user.ts）→ request.ts → http.ts → Axios
       定义接口与类型            业务码/归一化   实例/拦截器
```

★ **关键认知**：封装的目标不是"把 Axios 包一层"，而是**把变化点集中**。baseURL、鉴权、错误提示、业务码这些容易变的东西只出现一次；页面只依赖稳定的业务函数。⚠ 如果封装后页面仍然要写 `response.data.code`，说明封装没有到位。

# 2. 目录结构

```text
src/api/
├─ http.ts          # 实例、拦截器、鉴权
├─ request.ts       # 泛型请求、业务码、错误归一化
├─ error.ts         # AppError、normalizeError
├─ types.ts         # ApiResponse、PageResult
└─ modules/
   ├─ user.ts
   └─ order.ts
```

| 文件 | 职责 | 不应该做什么 |
|------|------|-------------|
| `http.ts` | 连接 Axios、挂拦截器 | 写具体业务接口 |
| `request.ts` | 统一入参/出参、业务码 | 关心某个页面的 UI |
| `modules/*` | 描述业务接口 | 处理 Toast、路由跳转 |
| 页面 | 调用业务函数、处理交互 | 拼 baseURL、读 Token |

# 3. 核心封装

## 3.1 通用类型

```typescript
// src/api/types.ts
export interface ApiResponse<T> {
  code: number
  message: string
  data: T
}

export interface PageResult<T> {
  list: T[]
  total: number
}
```

## 3.2 错误归一化

```typescript
// src/api/error.ts
export type AppError =
  | { type: 'http'; status: number; message: string; raw: unknown }
  | { type: 'business'; code: number; message: string; raw: unknown }
  | { type: 'network' | 'timeout' | 'canceled' | 'unknown'; message: string; raw: unknown }

export class BusinessError extends Error {
  constructor(public code: number, message: string) {
    super(message)
    this.name = 'BusinessError'
  }
}
```

`normalizeError` 的实现与第 5 篇一致：依次判断取消、业务错误、HTTP、超时/网络、未知，最后统一成 `AppError`。这里不再重复贴代码。

## 3.3 HTTP 实例

```typescript
// src/api/http.ts
import axios, { type AxiosInstance } from 'axios'

export const http: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10000
})

http.interceptors.request.use((config) => {
  const token = sessionStorage.getItem('access_token')
  if (token && !config.skipAuth) {
    config.headers.set('Authorization', `Bearer ${token}`)
  }
  return config
})
// 401 刷新逻辑见第 3 篇
```

★ **关键认知**：实例层只负责"请求发出前"和"响应返回后"的通用动作，**不负责弹 Toast、不负责跳路由**。这些属于 UI 决策，应由页面或全局错误处理器决定。⚠ 如果拦截器里直接 `window.location.href = '/login'`，单元测试和 SSR 都会很难受；`skipAuth` 字段由第 7 篇的模块扩展声明。

## 3.4 泛型请求函数

```typescript
// src/api/request.ts
import type { AxiosRequestConfig, AxiosResponse } from 'axios'
import { http } from './http'
import { BusinessError, normalizeError } from './error'
import type { ApiResponse } from './types'

export async function request<T, D = unknown>(
  config: AxiosRequestConfig<D>
): Promise<T> {
  try {
    const response = await http.request<ApiResponse<T>, AxiosResponse<ApiResponse<T>>, D>(config)
    const body = response.data
    if (body.code !== 0) throw new BusinessError(body.code, body.message)
    return body.data
  } catch (error) {
    throw normalizeError(error)
  }
}
```

## 3.5 业务 API 模块

```typescript
// src/api/modules/user.ts
import { request } from '../request'
import type { PageResult } from '../types'

export interface User {
  id: number
  name: string
}

export function getUsers(params: { page: number; pageSize: number }) {
  return request<PageResult<User>>({ url: '/users', method: 'get', params })
}

export function createUser(data: { name: string }) {
  return request<User, { name: string }>({ url: '/users', method: 'post', data })
}
```

页面调用：

```typescript
import { getUsers } from '@/api/modules/user'

const page = await getUsers({ page: 1, pageSize: 20 })
// page 已经是 PageResult<User>，不需要 response.data
```

# 4. 错误码约定

| 层级 | 负责判断 | 输出 |
|------|---------|------|
| Axios | HTTP 状态码 | `AxiosError` |
| 响应拦截器 | 鉴权、刷新、通用日志 | 原始错误或刷新后重放 |
| `request` | 业务码 `code` | 业务数据或 `AppError` |
| 页面 | 展示方式 | Toast、表单错误、空状态 |

可以用 `BIZ_CODE` 常量集中维护 `0`、`40100`、`40300`、`42200`、`50000` 等业务码。⚠ 业务码是后端契约，前端不要猜；未知码展示兜底文案，而不是静默吞掉。

# 5. 取消与去重接入

取消应通过 `signal` 从页面传进来，业务函数增加第二个 `config` 参数并展开即可。去重适合放在拦截器里，按 `method + url + params` 生成 key，新请求取消旧请求，具体实现见第 4 篇。⚠ 去重只适合"结果只关心最后一次"的查询接口，不适合下单、支付这类写操作。

# 6. Mock

## 6.1 axios-mock-adapter

直接拦截实例，适合单元测试：

```javascript
import MockAdapter from 'axios-mock-adapter'
import { http } from './http'

const mock = new MockAdapter(http, { delayResponse: 300 })
mock.onGet('/users').reply(200, {
  code: 0, message: 'ok', data: { list: [], total: 0 }
})
```

它能覆盖拦截器、取消、错误路径，适合测试封装层。MSW 则在**网络层**拦截，适合组件测试与本地开发；测封装用 mock adapter，测页面数据流用 MSW。

# 7. 单元测试

## 7.1 测什么

| 用例 | 关注点 |
|------|--------|
| 成功 | URL、method、params 是否正确，返回值是否解包 |
| 业务错误 | `code !== 0` 是否转成 `BusinessError` |
| HTTP 错误 | 500 是否归一化为 `http` |
| 取消 | `signal` 是否生效，是否标记为 `canceled` |
| 鉴权 | 请求头是否带上 Token |
| 401 刷新 | 是否只刷新一次，队列是否重放 |

★ **关键认知**：请求层的测试重点不是 Axios 本身，而是**你们约定的行为**：解包规则、错误结构、鉴权头、取消语义。⚠ 只测 happy path 等于没测，失败路径才是封装的价值所在。

## 7.2 Vitest + axios-mock-adapter

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest'
import MockAdapter from 'axios-mock-adapter'
import { http } from './http'
import { request } from './request'

const mock = new MockAdapter(http)

beforeEach(() => mock.reset())
afterEach(() => mock.restore())

describe('request', () => {
  it('成功时解包 data', async () => {
    mock.onGet('/users').reply(200, {
      code: 0, message: 'ok', data: [{ id: 1, name: 'Ada' }]
    })
    await expect(request({ url: '/users' })).resolves.toEqual([{ id: 1, name: 'Ada' }])
  })

  it('业务错误转成 BusinessError', async () => {
    mock.onGet('/users').reply(200, {
      code: 1001, message: '无权限', data: null
    })
    await expect(request({ url: '/users' })).rejects.toMatchObject({
      type: 'business', code: 1001
    })
  })

  it('HTTP 500 归一化为 http', async () => {
    mock.onGet('/users').reply(500, { message: 'boom' })
    await expect(request({ url: '/users' })).rejects.toMatchObject({
      type: 'http', status: 500
    })
  })
})
```

## 7.3 测试取消

```typescript
it('取消后标记为 canceled', async () => {
  const controller = new AbortController()
  mock.onGet('/slow').reply(() => new Promise((resolve) =>
    setTimeout(() => resolve([200, { code: 0, data: null }]), 1000)))

  const promise = request({ url: '/slow', signal: controller.signal })
  controller.abort()
  await expect(promise).rejects.toMatchObject({ type: 'canceled' })
})
```

## 7.4 401 刷新要测什么

至少覆盖：两个并发请求同时 401 只发一次 refresh；刷新成功后两个请求都用新 Token 重放；刷新失败队列全部 reject；刷新接口自身 401 不再次触发刷新。可用 `replyOnce` 与计数器断言。

# 8. 最佳实践与反模式

| 反模式 | 后果 | 改法 |
|--------|------|------|
| 每个页面 `axios.create` | 配置分散、拦截器重复 | 统一实例 |
| 拦截器里弹所有错误 | 重复提示、取消也弹 | 分层处理，取消静默 |
| 拦截器解包 + 页面再解包 | 运行时 `undefined` | 二选一 |
| 泛型全用 `any` | 类型形同虚设 | 强制声明返回类型 |
| 重试无上限 | 重试风暴 | 限次数 + 退避 |
| 上传手写 Content-Type | 后端收不到文件 | 交给浏览器 |
| 日志打印完整 config | Token 泄露 | 统一脱敏 |
| 一个实例管所有后端 | 超时、鉴权互相干扰 | 按后端拆分 |
| 只测成功路径 | 失败路径无保障 | 覆盖错误与取消 |

# 9. 综合练习

## 9.1 练习一：组装请求层

实现 `http.ts`、`request.ts`、`error.ts` 与一个 `modules/user.ts`。要求业务成功返回 `data`，业务失败抛 `BusinessError`，HTTP/网络/超时/取消统一转成 `AppError`，页面调用处不出现 `response.data`。

## 9.2 练习二：补齐测试

为请求层写至少 5 个测试：成功、业务错误、500、网络错误、取消，并用 `toMatchObject` 断言错误结构。

## 9.3 练习三：设计 401 刷新测试

用 mock adapter 模拟两个并发请求同时返回 401，断言 refresh 只调用一次、两个请求最终都成功、旧 Token 被替换。

# 10. 小结

## 10.1 模块回顾

| 篇目 | 核心 |
|------|------|
| 1 | Axios 定位与 fetch/XHR 对比 |
| 2 | 实例与配置、环境变量 |
| 3 | 拦截器顺序与 401 刷新 |
| 4 | 取消、超时、去重、并发 |
| 5 | 错误分类、归一化、重试 |
| 6 | FormData、进度、Blob、大文件 |
| 7 | TypeScript 泛型与模块扩展 |
| 8 | Token、XSS/CSRF、CORS、脱敏 |
| 9 | 统一封装、Mock、测试 |

★ **关键认知**：好的请求层应该让业务代码**只表达意图**——"取用户列表""创建订单"，而不是重复表达"怎么发 HTTP"。把鉴权、错误、取消、类型、安全这些横切关注点收进封装，再用测试锁住行为，Axios 才算真正用明白了。⚠ 封装不是越厚越好：每加一层抽象，都要能回答"它替业务消除了哪种重复"，否则只是在制造新的理解成本。
