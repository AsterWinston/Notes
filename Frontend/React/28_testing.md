"功能改了一个地方,别处悄悄崩了"——这种事靠**自动化测试**兜底。React 测试的主流组合是 **Vitest/Jest(测试框架)+ React Testing Library(组件测试)**。本篇讲清测试金字塔、RTL 的核心 API、"以用户视角测试"的哲学,以及 Hook 与 e2e 测试。

# 1. 测试金字塔

```
       [E2E 端到端](少,慢,全链路)
      /            \
 [集成测试](中) [组件测试](多,核心)
    /                \
 [单元测试](多,快,纯函数)
```

★ **关键认知**：测试金字塔——**底层单元测试多快,顶层 e2e 少慢**;React 项目的重点是**中间层:组件测试**(用起来最多、价值最高)。⚠ 别只写"纯函数单测"(覆盖不到交互),也别只写 e2e(慢且脆);**组件测试是 React 测试的主战场**。

# 2. React Testing Library

## 2.1 以用户视角测试

```jsx
import { render, screen, fireEvent } from '@testing-library/react'
import Counter from './Counter'

test('点击按钮,计数 +1', () => {
  render(<Counter />)                          // 渲染组件
  const button = screen.getByRole('button')    // 按"角色"找元素(像用户:找按钮)
  fireEvent.click(button)                      // 模拟用户点击
  expect(button).toHaveTextContent('1')        // 断言:按钮文本变成 1
})
```

★ **关键认知**：RTL 的核心哲学是"**测用户能看到/做到的事**"——用 `getByRole('button')`(用户视角:找按钮)而非 `querySelector('.btn')`(实现细节:找类名)。⚠ 这让测试**贴近行为、不绑定实现**,重构内部实现测试仍绿,是 RTL 与老式"测 DOM 结构"的根本区别。

## 2.2 查询与断言

```jsx
screen.getByText('提交')            // 按文本
screen.getByLabelText('姓名')       // 按 label
screen.getByRole('button', { name: '提交' })  // 按角色 + 可访问名
screen.queryByText('加载中')        // querBy:不存在返回 null(用于断言"没有")
// fireEvent / userEvent(更真实,模拟键入、点击)
```

★ **关键认知**：RTL 的查询优先**可访问性**(role/text/label),而非类名/结构;`getBy` 找不到会抛错,`queryBy` 返回 null。⚠ 用 `userEvent`(更贴真人的键入/交互)替代 `fireEvent`(底层但粗糙),是实践推荐。

# 3. 测试什么

## 3.1 测"行为"而非"实现"

```jsx
// ✓ 好:断言"点击后显示数字 1"(用户可见的行为)
// ✗ 差:断言"内部 state.count 等于 1"(窥探实现,重构就断)
```

★ **关键认知**：好测试断言**可观察的行为**(渲染出什么),坏测试断言**内部实现**(state 值、内部函数调用)。⚠ "测行为不测实现"与"用 role 不用类名"是一体两面,都让测试**经得起重构**。

## 3.2 异步与 Mock

```jsx
// 测异步组件:用 findBy(等待异步出现) + mock 请求
test('加载并显示用户', async () => {
  vi.mock('../api', () => ({ fetchUser: vi.fn().mockResolvedValue({ name: 'Alice' }) }))
  render(<UserProfile />)
  expect(await screen.findByText('Alice')).toBeInTheDocument()   // findBy 等待
})
```

★ **关键认知**：异步测试用 **`findByXxx`**(等待元素异步出现);外部依赖(请求)用 **mock** 掉。⚠ `findBy` 会轮询等待,是"数据到了才断言"的关键;暴 mock 请求则让测试**稳定、快、脱离网络**。

# 4. Hook 测试与 e2e

## 4.1 Hook 测试 renderHook

```jsx
import { renderHook, act } from '@testing-library/react'

const { result } = renderHook(() => useCounter())   // 渲染一个 Hook
act(() => { result.current.increment() })           // 触发更新
expect(result.current.count).toBe(1)
```

★ **关键认知**：自定义 Hook 可用 **`renderHook`** 独立测试(不需要套组件);`act` 包裹"会触发状态更新"的操作。⚠ 抽好 Hook(第 13 篇)后,逻辑测试比组件测试更轻更快,是"逻辑复用"的额外红利。

## 4.2 e2e(Playwright/Cypress)

★ **关键认知**：**e2e(端到端)** 用真实浏览器(Playwright/Cypress)测**整个应用流程**(登录→下单→支付)。⚠ 它最慢最脆,但覆盖"各环节真的串起来"这件事;放在**少数关键路径**(核心用户旅程)而非大量铺开。类型:组件测试(RTL)为主,e2e 补关键链路。

# 5. 工具链

```powershell
# 可选 Vitest(快,配 Vite 项目)或 Jest
npm install -D vitest @testing-library/react @testing-library/user-event jsdom
```

★ **关键认知**：Vitest(配 Vite)/Jest 是测试框架,`@testing-library/react` 是 RTL,`jsdom` 提供浏览器-like 环境。⚠ 现代 Vite 项目普遍用 **Vitest + RTL**;配置一个"写好就能跑"的测试环境,是让测试"持续被跑"的前提。

# 6. 本篇小结与下一步

★ **关键认知**：测试以"组件测试(RTL)"为主,"测行为不测实现";查询靠 role/text;异步用 findBy/mock;Hook 用 renderHook;e2e 补关键链路。工具:Vitest/Jest + RTL。

下一步第 29 篇:服务端与流式渲染进阶——流式渲染、Suspense for SSR、缓存,把"首屏快"做到极致。