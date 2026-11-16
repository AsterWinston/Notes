Axios 自带的类型定义相当完整，但很多人用着用着就退化成 `any`：`response.data` 是 `any`，`catch (e)` 里不敢碰 `e`，拦截器返回值类型对不上。问题通常不在于类型系统难，而在于没有把"配置类型、响应泛型、错误泛型、模块扩展"这几件事串起来。本篇按使用频率从高到低梳理。

# 1. 类型全景

## 1.1 常用类型

| 类型 | 作用 | 典型位置 |
|------|------|---------|
| `AxiosInstance` | 实例类型 | 封装模块的返回值 |
| `AxiosRequestConfig<D>` | 请求配置 | 封装函数的入参 |
| `InternalAxiosRequestConfig<D>` | 拦截器内的配置 | 请求拦截器 |
| `AxiosResponse<T, D>` | 响应对象 | 响应拦截器、底层调用 |
| `AxiosError<T, D>` | 错误对象 | `catch`、错误拦截器 |
| `AxiosInterceptorManager<V>` | 拦截器管理器 | 保存 eject id |
| `AxiosHeaders` | 请求头容器 | `headers.set()` |

## 1.2 三个泛型参数

以 `get` 为例：

```typescript
get<T = any, R = AxiosResponse<T>, D = any>(url: string, config?: AxiosRequestConfig<D>): Promise<R>
```

| 参数 | 含义 | 默认 |
|------|------|------|
| `T` | 响应体 `data` 的类型 | `any` |
| `R` | 整个返回值的类型 | `AxiosResponse<T>` |
| `D` | 请求体 `data` 的类型 | `any` |

★ **关键认知**：只写一个泛型时，改变的是 `response.data` 的类型，不是返回值的类型。`http.get<User[]>('/users')` 返回的仍然是 `AxiosResponse<User[]>`，需要 `.data` 才能拿到 `User[]`。⚠ 这也是"拦截器解包"之后类型对不上的根源：运行时返回了 `data`，类型却还写着 `AxiosResponse`。

# 2. AxiosInstance 与配置

## 2.1 给实例标注类型

```typescript
import axios, { type AxiosInstance, type AxiosRequestConfig, type InternalAxiosRequestConfig, type AxiosResponse, type AxiosError } from 'axios'

export const http: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10000
})

export function request<T, D = unknown>(
  config: AxiosRequestConfig<D>
): Promise<T> {
  return http.request<T, T, D>(config) // R = T，表示直接返回 T
}
```

`http.request<T, T, D>` 把第二个泛型 `R` 指定为 `T`，告诉 TypeScript"这个 Promise 最终 resolve 的是 T"。

## 2.2 配置对象的类型

`AxiosRequestConfig<D>` 的 `data` 由 `D` 约束，可以这样写：`const config: AxiosRequestConfig<{ name: string }> = { url: '/users', method: 'post', data: { name: 'Ada' } }`。

`AxiosRequestConfig` 的 `headers` 可选且宽松；拦截器里的 `InternalAxiosRequestConfig` 要求 `headers` 一定存在，并且是 `AxiosHeaders`：

```typescript
http.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  config.headers.set('X-Trace-Id', crypto.randomUUID())
  return config
})
```

★ **关键认知**：`AxiosRequestConfig` 是"调用方传入的配置"，`InternalAxiosRequestConfig` 是"Axios 合并默认值后交给拦截器的配置"。⚠ 写请求拦截器时用后者，否则 `config.headers` 可能被推断为 `undefined`。

# 3. 泛型响应

## 3.1 基础用法

```typescript
interface User {
  id: number
  name: string
  email: string
}

const response = await http.get<User[]>('/users')
const users: User[] = response.data // data 被推断为 User[]

interface CreateUserDto {
  name: string
  email: string
}

// POST 第三个泛型约束请求体；第二个泛型默认就是 AxiosResponse<User>
const created = await http.post<User, AxiosResponse<User>, CreateUserDto>(
  '/users',
  { name: 'Ada', email: 'ada@example.com' }
)
```

第二个泛型通常不用写；只有需要改变返回结构时才手工指定。

## 3.2 业务响应包

后端常见结构：

```typescript
interface ApiResponse<T> {
  code: number
  message: string
  data: T
}
```

封装函数负责拆包与业务码判断：

```typescript
export class BusinessError extends Error {
  constructor(public code: number, message: string) {
    super(message)
    this.name = 'BusinessError'
  }
}

export async function request<T, D = unknown>(
  config: AxiosRequestConfig<D>
): Promise<T> {
  const response = await http.request<
    ApiResponse<T>,
    AxiosResponse<ApiResponse<T>>,
    D
  >(config)

  const body = response.data
  if (body.code !== 0) {
    throw new BusinessError(body.code, body.message)
  }
  return body.data
}
```

调用侧就非常干净：

```typescript
const users = await request<User[]>({ url: '/users', method: 'get' })
```

## 3.3 不要同时"拦截器解包"和"泛型解包"

如果响应拦截器已经 `return response.data`，那么 `http.get<T>()` 的运行时返回值是 `T`，类型却仍是 `AxiosResponse<T>`。有两种修复方式：

```typescript
// 方式一：拦截器只做错误处理，业务层统一解包（推荐，类型最直观）
const response = await http.get<User[]>('/users')
const users = response.data

// 方式二：拦截器解包，封装层用 R = T 修正类型
const users2 = await http.request<User[], User[]>({ url: '/users' })
```

⚠ 两种方式混用是类型报错和运行时 `undefined` 的重灾区。团队内必须约定一种。

# 4. 拦截器类型

## 4.1 请求拦截器

```typescript
http.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    const token = getToken()
    if (token && !config.skipAuth) {
      config.headers.set('Authorization', `Bearer ${token}`)
    }
    return config
  },
  (error: unknown) => Promise.reject(error)
)
```

## 4.2 响应拦截器

```typescript
http.interceptors.response.use(
  (response: AxiosResponse) => response,
  (error: AxiosError<ApiResponse<unknown>>) => {
    const message = error.response?.data?.message
    if (message) error.message = message
    return Promise.reject(error)
  }
)
```

`AxiosError<T>` 的第一个泛型是**响应体类型**，不是错误类型。`error.response?.data` 会被推断为 `T`。

## 4.3 保存 eject id

`use()` 返回 `number`，保存后在清理时用 `eject(id)` 注销。在组件或热更新环境里重复执行 setup 而不 eject，会导致拦截器叠加。

★ **关键认知**：`AxiosInterceptorManager<V>` 的 `V` 是"拦截器处理的值的类型"，请求与响应分别使用 `InternalAxiosRequestConfig` 和 `AxiosResponse`。

# 5. 模块扩展

## 5.1 给配置加自定义字段

`skipAuth`、`_retry`、`silent` 这些字段 Axios 不认识，直接写会报类型错误。用模块扩展声明：

```typescript
// src/types/axios.d.ts
import 'axios'

declare module 'axios' {
  export interface AxiosRequestConfig<D = any> {
    skipAuth?: boolean   // 跳过鉴权头
    _retry?: boolean     // 401 刷新后是否已重试
    silent?: boolean     // 是否静默，不弹全局提示
  }
}
```

⚠ `AxiosRequestConfig` 在 Axios 1.x 中是泛型接口，扩展时**必须带上 `<D = any>`**，否则 TypeScript 会报"类型参数不一致"。如果你的版本里它不是泛型，去掉尖括号即可。

扩展后可以安全地写：

```typescript
await http.post('/auth/login', data, { skipAuth: true })
```

## 5.2 给错误加展示字段

```typescript
declare module 'axios' {
  export interface AxiosError<T = unknown, D = any> {
    userMessage?: string
  }
}
```

这样拦截器里可以安全地写 `error.userMessage = '...'`。

## 5.3 扩展文件要参与编译

`src/types/axios.d.ts` 需要在 `tsconfig.json` 的 `include` 范围内，或通过 `import './types/axios'` 显式引入。没有参与编译的声明文件不会生效，表现为"我明明扩展了，还是报错"。

# 6. 错误类型

## 6.1 catch 里的 unknown

开启 `useUnknownInCatchVariables`（`strict` 下默认开启）后，`catch (error)` 的 `error` 是 `unknown`：

```typescript
try {
  await request<User[]>('/users')
} catch (error) {
  if (axios.isAxiosError<ApiResponse<unknown>>(error)) {
    console.log(error.response?.status)
    console.log(error.response?.data.message)
  } else if (error instanceof BusinessError) {
    console.log(error.code)
  } else if (error instanceof Error) {
    console.log(error.message)
  }
}
```

★ **关键认知**：`axios.isAxiosError<T>(error)` 既是运行时守卫，也是类型守卫。传入 `T` 后，`error.response?.data` 会按 `T` 推断。⚠ 不要用 `(error as any).response` 绕过，否则等于把类型系统关掉。

## 6.2 错误响应的泛型

定义 `interface ApiErrorBody { code: number; message: string; field?: string }`，把 `AxiosError<ApiErrorBody>` 用在错误拦截器上，`error.response?.data.field` 就有类型。

# 7. 常见类型坑

| 坑 | 后果 | 改法 |
|----|------|------|
| 不传泛型 | `data` 是 `any`，一路传染 | 封装层强制声明返回类型 |
| 拦截器返回 `data` | 类型仍是 `AxiosResponse` | 只解包一次，或用 `R = T` 修正 |
| 混用两种配置类型 | `headers` 可能为 `undefined` | 拦截器用 `InternalAxiosRequestConfig` |
| 用 `as` 代替校验 | 运行时结构与类型不符 | 关键接口用 Zod 等校验 |
| 扩展没带泛型 | 模块扩展报类型参数不一致 | Axios 1.x 写 `<D = any>` |

# 8. 练习

## 8.1 练习一：类型化 API 模块

定义 `User`、`ApiResponse<T>` 和 `request<T>()`，实现 `getUser(id: number): Promise<User>` 与 `createUser(dto: CreateUserDto): Promise<User>`，要求调用时不出现 `any`。

## 8.2 练习二：扩展配置字段

通过模块扩展给 `AxiosRequestConfig` 增加 `skipAuth` 与 `_retry`，并在请求拦截器中使用 `skipAuth` 跳过鉴权。要求 `tsc --noEmit` 通过。

## 8.3 练习三：判别联合错误

定义 `type AppError = { type: 'http'; status: number } | { type: 'business'; code: number } | { type: 'network'; message: string }`，实现 `toAppError(error: unknown): AppError`，并让调用方在 `switch` 时获得正确的字段补全。

# 9. 小结

| 概念 | 要点 |
|------|------|
| 三个泛型 | `T` 响应体、`R` 整个返回值、`D` 请求体 |
| 实例类型 | `AxiosInstance` 标注封装模块 |
| 拦截器配置 | 用 `InternalAxiosRequestConfig`，headers 必填 |
| 错误 | `AxiosError<T>` 的 `T` 是响应体类型 |
| 模块扩展 | Axios 1.x 需带 `<D = any>` |
| 解包 | 拦截器解包与泛型解包二选一 |

下一篇进入安全与最佳实践：Token 该放哪里、XSS 与 CSRF 如何防、CORS 的边界、超时与重试的资源保护，以及日志脱敏。
