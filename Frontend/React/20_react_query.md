"服务端数据"不同于"客户端状态"——它要**请求、缓存、失效、重试、乐观更新**,这些用 useState/useEffect 手写既繁琐又易错。**React Query(TanStack Query)** 是专业选手。本篇讲清 useQuery/useMutation、缓存与失效、无限滚动与乐观更新。

# 1. 服务端状态 ≠ 客户端状态

## 1.1 为什么需要专门的库

```jsx
// 手写把"服务端数据"存进 useState:要自己处理加载/错误/缓存/重试/竞态…
// 而 React Query 一条 useQuery 搞定,还内置缓存与自动重试
```

★ **关键认知**：服务端数据的特点是**"远端、会过期、要缓存、会失败"**,与"本地的、当前会话的"客户端状态本质不同。⚠ 共识:**服务端数据交给 React Query,客户端状态交给 useState/状态库**——两者职责分离,是清晰状态模型的关键。

# 2. useQuery 查询

## 2.1 基本用法

```jsx
import { useQuery } from '@tanstack/react-query'

function UserList() {
  const { data, isLoading, isError, error } = useQuery({
    queryKey: ['users'],                    // 缓存键(唯一标识这次查询)
    queryFn: () => fetch('/api/users').then(r => r.json())  // 查询函数
  })

  if (isLoading) return <div>加载中</div>
  if (isError) return <div>出错:{error.message}</div>
  return <ul>{data.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

★ **关键认知**：`useQuery` 的 `queryKey`(缓存键)+ `queryFn`(取数函数)是核心;返回 `data/isLoading/isError` 等状态。⚠ **缓存**是 React Query 的灵魂——同样的 queryKey 命中缓存就直接返回(不重新请求),这是它相比"手写 useEffect 请求"最大的价值。

## 2.2 staleTime 与缓存

```jsx
useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  staleTime: 1000 * 60 * 5      // 5 分钟内数据是"新鲜的",不重新请求(直接用缓存)
})
```

★ **关键认知**：`staleTime` 定义"数据多久算新鲜"——新鲜期内直接读缓存**不发请求**;过期后"后台静默重取"(先显示旧缓存,拿到新的再更新)。⚠ 这是"**缓存优先 + 后台更新**"的经典模式,体验极佳;默认 `staleTime: 0`(每次挂载都重取),按需调大。

# 3. useMutation 变更

## 3.1 基本用法

```jsx
const mutation = useMutation({
  mutationFn: (newUser) => fetch('/api/users', {
    method: 'POST',
    body: JSON.stringify(newUser)
  })
})

// 触发
mutation.mutate({ name: 'Alice' })   // mutation.isPending / isSuccess / isError
```

★ **关键认知**：`useMutation` 处理**写操作**(POST/PUT/DELETE),`mutationFn` 是变更函数,`mutate()` 触发。⚠ 查询用 useQuery、变更用 useMutation——这是对称的两个入口,写操作还用 useQuery 是常见错误(写操作通常不缓存)。

## 3.2 变更后失效缓存

```jsx
const mutation = useMutation({
  mutationFn: addUser,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['users'] })   // 让 ['users'] 缓存失效,触发重新拉取
  }
})
```

★ **关键认知**：**写操作后要让相关缓存失效**——`invalidateQueries` 标记某 key 的缓存过期,界面自动刷新为最新。⚠ 这是"增删改后更新列表"的标准链路:变更成功 → 失效缓存 → 自动重取。

# 4. 进阶:无限滚动与乐观更新

## 4.1 无限滚动 useInfiniteQuery

```jsx
import { useInfiniteQuery } from '@tanstack/react-query'

const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ['posts'],
  queryFn: ({ pageParam }) => fetchPosts(pageParam),
  initialPageParam: 1,
  getNextPageParam: (lastPage) => lastPage.nextPage ?? undefined
})
// 滚动到底时 fetchNextPage() 加载下一页
```

★ **关键认知**：`useInfiniteQuery` 做**无限滚动/分页**——按 `pageParam` 一页页拉,`fetchNextPage` 加载下一页。⚠ 它内部拼合多页数据(`data.pages`),是"下拉加载更多"的标配。

## 4.2 乐观更新

```jsx
const mutation = useMutation({
  mutationFn: toggleTodo,
  onMutate: async (id) => {
    await queryClient.cancelQueries(['todos'])          // 先冻结旧查询
    const prev = queryClient.getQueryData(['todos'])    // 备份
    queryClient.setQueryData(['todos'], old => /* 立即改本地缓存 */)  // 乐观更新
    return { prev }
  },
  onError: (err, id, ctx) => {
    queryClient.setQueryData(['todos'], ctx.prev)       // 失败回滚
  }
})
```

★ **关键认知**：**乐观更新**——先假设请求会成功、**立即更新本地缓存**(界面即时响应),失败再回滚。⚠ 它是"点赞、待办勾选"这类高频操作的体验利器;核心是 onMutate 改缓存 + onError 回滚。

# 5. 本篇小结与下一步

★ **关键认知**：React Query 管"服务端状态"——useQuery(读+缓存,queryKey/staleTime)、useMutation(写)、invalidateQueries(失效)、useInfiniteQuery(无限滚动)、乐观更新(onMutate+回滚)。让它管请求,组件状态只管 UI。

下一步第 21 篇:表单 React Hook Form——用"非受控 + 最小重渲染"解决大表单的性能与啰嗦。