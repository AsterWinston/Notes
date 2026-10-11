Electron 应用横跨主进程与渲染进程,测试与调试也因此复杂——渲染进程能照搬前端那套(单测/e2e),主进程要单独测。本篇讲清 Electron 的测试分层、单元测试、e2e(Playwright)、以及主/渲染进程各自的调试手段。

# 1. 测试分层

## 1.1 Electron 的测试结构

```
1. 单元测试:主进程的纯函数、preload 逻辑、工具函数(快、多)
2. 组件测试:渲染进程的 Vue/React 组件(用框架那套)
3. e2e:整个应用启动,模拟用户操作(Playwright,慢、少、关键链路)
```

★ **关键认知**：Electron 测试分三层——**主进程逻辑用单元测试、渲染组件用前端测试、全流程用 e2e**。⚠ 别试图"什么都靠 e2e"(慢且脆);核心业务逻辑(主进程的 handler、工具函数)抽成纯函数做单测,是性价比最高的部分。

# 2. 单元测试

## 2.1 主进程纯函数测试

```javascript
// 把主进程逻辑抽成可测的纯函数(第 11 篇的"集中 handler"此时受益)
// utils.js
function validatePath(p) { /* 路径校验逻辑 */ }
module.exports = { validatePath }

// utils.test.js(Vitest/Jest)
test('非法路径被拦截', () => {
  expect(validatePath('../etc')).toBe(false)
})
```

★ **关键认知**：**把主进程的业务逻辑抽成纯函数**,用 Vitest/Jest 直接测——不需要真的启动 Electron。⚠ 这需要在**写代码时就"逻辑与 Electron API 解耦"**(能纯函数化的都纯函数化),否则主进程几千行塞满 Electron 调用,只能靠 e2e 了。

## 2.2 隔离 Electron API

```javascript
// 难测的根源:逻辑里混着 ipcMain/dialog 等 Electron API
// 解法:依赖注入 / mock,把 Electron API 与纯逻辑分离
// 主进程 handler 只做"编排",核心计算放纯函数里测
```

★ **关键认知**：主进程可测性的关键是"**把 Electron API 与纯逻辑分离**"——handler 只做编排(收 IPC、调函数、回结果),真正的计算在纯函数里。⚠ 这个分层(第 11 篇)让单元测试成为可能,否则 Electron API 一掺和就难测。

# 3. e2e 测试

## 3.1 Playwright 测 Electron

```javascript
import { _electron as electron } from 'playwright'

test('启动应用并操作', async () => {
  const app = await electron.launch({ args: ['main.js'] })   // 启动 Electron
  const window = await app.firstWindow()                       // 拿窗口

  await window.click('text=新建')                              // 模拟点击
  expect(await window.textContent('body')).toContain('新建成功')

  await app.close()
})
```

★ **关键认知**：**Playwright 的 `_electron`** 能启动真实 Electron 应用并**模拟用户操作**(点击、输入)——真正的端到端。⚠ e2e 的收益是"真的跑通了完整链路";代价是慢、需稳定环境(能起窗口、能操作),所以只覆盖**关键用户旅程**(启动 → 核心功能)。

## 3.2 e2e 的边界

★ **关键认知**：e2e 里能测"**用户视角的行为**"(点了按钮发生什么),但**不适合测内部实现细节**。⚠ 把 e2e 留给"登录 → 保存 → 退出"这类关键链路,细节逻辑交给单测;e2e 太多会让构建又慢又脆。

# 4. 调试

## 4.1 渲染进程:DevTools

★ **关键认知**：渲染进程调试就是**熟悉的 Chrome DevTools**——断点、Network、Console 都有(它本质是 Chromium)。⚠ 开发时用 DevTools(第 2 篇);定位界面问题的第一手段就是它,前端经验无缝迁移。

## 4.2 主进程:断点与日志

```javascript
// 主进程调试:
// 1. VSCode 配 launch.json 里 "debugger" 断点主进程
// 2. 或 console.log 到主进程 stdout(用 electron 启动时可见)
console.log('主进程日志:', data)
```

★ **关键认知**：主进程调试用 **VSCode 断点调试**(配 launch.json)或**主进程日志**。⚠ 主进程没有 DevTools,靠断点/日志;生产环境的问题排查,依赖**完善的日志**(统一的 logger,而非散落的 console.log)。

## 4.3 统一日志

★ **关键认知**：为**主进程建立统一日志**(如用 log 级别、写文件)——生产环境的崩溃、异常,几乎都靠日志还原现场。⚠ 别用散乱的 `console.log`;主进程的关键路径(IPC 出入、错误、更新)记日志,是"发布后能定位问题"的保障。

# 5. 测试清单

| 层 | 工具 | 覆盖 |
|----|------|------|
| 主进程逻辑 | Vitest/Jest | 纯函数、handler 编排 |
| 渲染组件 | RTL/Vitest | 组件行为 |
| e2e | Playwright `_electron` | 关键用户旅程 |
| 调试 | DevTools / 断点 / 日志 | 开发与生产排查 |

# 6. 本篇小结与下一步

★ **关键认知**：Electron 测试分三层——主进程逻辑抽纯函数做单测(需 API 解耦)、渲染组件用框架测试、关键链路用 Playwright e2e;调试:渲染用 DevTools、主进程用断点+日志(生产靠统一日志)。

下一步第 20 篇:跨平台疑难——平台差异、路径/通知/快捷键坑点与排查。