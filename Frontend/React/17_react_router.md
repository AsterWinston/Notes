SPA 的多页面体验靠路由把 URL 映射到组件。React Router 是 React 生态的官方路由。本篇讲清路由配置、动态/嵌套路由、导航与守卫、懒加载,以及数据路由(loader/action)的现代写法。

# 1. 基本配置

## 1.1 路由表与出口

```jsx
import { createBrowserRouter, RouterProvider, Outlet, Link } from 'react-router-dom'

const router = createBrowserRouter([
  { path: '/', element: <Home /> },          // path → 组件
  { path: '/about', element: <About /> }
])

function App() {
  return <RouterProvider router={router} />   // 挂到根
}

// 组件内:<Link to="/about">去关于</Link> 用于跳转,<Outlet /> 用于嵌套出口
```

★ **关键认知**：React Router 的现代用法是 **`createBrowserRouter` 建路由表 + `<RouterProvider>` 挂载**;`<Link>` 跳转、`<Outlet>` 渲染嵌套子路由。⚠ 与 Vue Router 对照:都是"path → 组件"映射 + `<RouterView>/<Outlet>` 出口,理念一致。

# 2. 动态与嵌套路由

## 2.1 动态参数

```jsx
const router = createBrowserRouter([
  { path: '/user/:id', element: <UserDetail /> }   // :id 动态段
])

function UserDetail() {
  const { id } = useParams()          // 读动态参数
  return <div>用户 {id}</div>
}
```

★ **关键认知**：`/user/:id` 定义动态段,`useParams()` 读取。⚠ 与 Vue 一致:同路由不同参数不会重建组件,参数变化要自己监听;React 里可用 `key={id}` 强制重建或监听 `id`。

## 2.2 嵌套路由与 Outlet

```jsx
const router = createBrowserRouter([
  {
    path: '/users',
    element: <UsersLayout />,          // 父布局组件(里面有 <Outlet/>)
    children: [
      { path: '', element: <UsersList /> },      // 默认子路由
      { path: ':id', element: <UserDetail /> }   // /users/123
    ]
  }
])

function UsersLayout() {
  return (
    <div>
      <h1>用户</h1>
      <Outlet />              {/* 子路由渲染在这 */}
    </div>
  )
}
```

★ **关键认知**：嵌套路由实现"**布局 + 子内容**"——子路由渲染在父组件里的 `<Outlet />`。⚠ 与 Vue 嵌套路由/RouterView 同构;父布局(导航/侧栏)与子内容(页面)分离,是路由组织的主流形态。

# 3. 导航与守卫

## 3.1 编程式导航

```jsx
import { useNavigate } from 'react-router-dom'

function Login() {
  const navigate = useNavigate()
  async function submit() {
    await doLogin()
    navigate('/dashboard')            // 跳转
  }
}
```

★ **关键认知**：`useNavigate()` 返回函数做**编程式跳转**(对应 Vue 的 `router.push`)。⚠ `navigate('/x')` 默认 push(留历史),`navigate('/x', { replace: true })` 替换(不留历史)。

## 3.2 路由守卫(无官方守卫,自己实现)

```jsx
// React Router 没有 Vue 那样的 beforeEach,通常做"受保护路由"组件
function Protected({ children }) {
  const isLoggedIn = useAuth()
  return isLoggedIn ? children : <Navigate to="/login" replace />
}

const router = createBrowserRouter([
  {
    path: '/dashboard',
    element: <Protected><Dashboard /></Protected>   // 包一层受保护
  }
])
```

★ **关键认知**：React Router **没有内置守卫钩子**,靠"**受保护组件包裹**"(检查登录,未登录 `<Navigate to="/login">`)实现鉴权。⚠ 这是 React 与 Vue 路由的显著差异——React 更"组合式",守卫就是普通组件;Vue 用 beforeEach 钩子集中处理。

# 4. 懒加载与数据路由

## 4.1 路由懒加载

```jsx
import { lazy, Suspense } from 'react'

const About = lazy(() => import('./About'))   // 懒加载组件

const router = createBrowserRouter([
  {
    path: '/about',
    element: <Suspense fallback={<div>加载中</div>}><About /></Suspense>
  }
])
```

★ **关键认知**：路由懒加载 = `lazy(() => import(...))` + `<Suspense>` 包裹,访问到才下载(配合第 23 篇代码分割)。⚠ 每个懒加载路由都要有 Suspense 兜住加载态;这是 SPA 优化首屏的标配。

## 4.2 loader / action(数据路由)

```jsx
const router = createBrowserRouter([
  {
    path: '/users/:id',
    element: <UserDetail />,
    loader: async ({ params }) => {          // 渲染前抓数据
      return fetchUser(params.id)
    }
  }
])

function UserDetail() {
  const user = useLoaderData()               // 读 loader 抓到的数据
  return <div>{user.name}</div>
}
```

★ **关键认知**：**数据路由** = 路由定义 `loader`(渲染前抓数据),组件用 `useLoaderData()` 读——数据获取与路由绑定,避免"挂载后再发请求"的空窗。⚠ 这是 React Router 新版的特色,把"进页面 + 拿数据"合一,与数据请求库(第 20 篇 React Query)可互补。

# 5. 本篇小结与下一步

★ **关键认知**：`createBrowserRouter` + RouterProvider + Link/Outlet;动态 `:id` 用 useParams;嵌套用 Outlet;守卫靠"受保护组件"(无官方钩子);懒加载 lazy+Suspense;数据路由用 loader/useLoaderData。

下一步第 18 篇:状态管理 Redux Toolkit——集中式的全局状态,dispatch/selector,与 Pinia 对照。