Node.js 提供了海量能力,但仍有边界——调用 C/C++ 库、直接操作硬件、用系统 DLL。**原生模块(native addons)** 与 **ffi** 填上这缺口。本篇讲清 Electron 里的 Node 能力边界、native addons 的重编译问题、以及 ffi 直接调原生库。

# 1. Node 能力的边界

## 1.1 纯 JS 能做什么、不能做什么

```
能:文件、网络、进程、绝大多数系统 API(靠 Node 内置模块 + 大量 npm 包)
不能:直接调 C/C++ 库、唯一硬件接口、极致的 CPU 性能
```

★ **关键认知**：Node 的 JS 层能力已经**覆盖绝大多数桌面应用需求**(文件、网络、进程、系统)。⚠ 边界在三类——**要调用某个 C/C++ 库**(如调用公司的加密库)、**要直接驱动硬件**、**CPU 极密集**(JS 慢)——这些才需要原生模块/ffi。别一上来就上原生,先看 npm 有没有现成的 JS 封装。

# 2. native addons(原生扩展)

## 2.1 什么是 native addon

```
native addon = 用 C/C++ 写的 Node 模块(编译成 .node 二进制)
在 JS 里 require 它,像普通模块一样用
典型:better-sqlite3、sharp(图像处理)、native 加密库
```

★ **关键认知**：**native addon** 是**用 C/C++ 写的 Node 模块**,编译成 `.node` 动态库,`require` 进来像普通模块用。⚠ 你平时可能已在用(`better-sqlite3`、`sharp` 都是)——它们给 JS 补上"原生性能"与"调 C 库"的能力,但带着"重编译"的成本(见下)。

## 2.2 Electron 的重编译问题

```
Node 的 ABI(二进制接口)与 Electron 不同
→ 为普通 Node 编译的 addon,不一定能在 Electron 里用
→ 通常需要 electron-rebuild 针对 Electron 重新编译
```

★ **关键认知**：Electron 内嵌的 Node 与系统 Node 的 **ABI 不同**,为普通 Node 编译的 addon **可能在 Electron 里加载失败**。⚠ 解法:`electron-rebuild`(重新针对 Electron 编译 addon)或 `electron-builder install-app-deps`;这是 Electron 使用 native addon 的头号坑。

## 2.3 node-gyp 与工具链

```powershell
# native addon 编译工具链
npm install -g node-gyp
# 需要平台编译器:Windows 用 VS Build Tools,macOS 用 Xcode
```

★ **关键认知**：编译 native addon 需要**本地编译工具链**(Windows 的 VS Build Tools、macOS 的 Xcode Command Line Tools)。⚠ 这是 native addon 的隐性成本——在 CI/多机器上要装齐工具链;纯 JS 项目则完全无此烦恼,这也是"能纯 JS 就纯 JS"的理由。

# 3. ffi(直接调原生库)

## 3.1 ffi-napi 调 DLL

```javascript
const ffi = require('ffi-napi')

// 直接调用系统/第三方动态库里的函数
const user32 = ffi.Library('user32', {
  'MessageBoxW': ['int32', ['int32', 'string', 'string', 'int32']]
  //  函数名       返回类型  参数类型...
})
user32.MessageBoxW(0, '你好', '标题', 0)   // 弹出系统对话框
```

★ **关键认知**：**ffi(Foreign Function Interface)** 让 JS **直接调用动态库(.dll/.so/.dylib)里的函数**,不必写 C++ 包装。⚠ 它适合**"只想调库里的几个函数、不值得写完整 addon"**的场景;代价是**类型要手写声明**(上述 `['int32', [...]]`),易错、无编译期保护。

## 3.2 ffi 的风险与取舍

★ **关键认知**：ffi 是"**快速但脆弱**"的桥——声明错一个类型参数,轻则崩溃、重则内存损坏。⚠ 取舍:**调用原生函数的数量少且稳定 → ffi 够用;要正式、稳定、长期维护 → 写正规 native addon**(有类型安全与封装)。ffi 更像"脚本化地试一下原生能力"。

# 4. 选型与决策

| 需求 | 方案 |
|------|------|
| 有 npm 现成 JS 包 | 直接用(native 由包自己处理) |
| 调 C/C++ 库、要性能 | native addon(重编译) |
| 只调库里的几个函数 | ffi-napi |
| CPU 极密集计算 | native addon 或 worker |

★ **关键认知**：决策顺序——**先找 npm 现成包 → 再考虑 native addon → 最后才 ffi**。⚠ 每向"原生"走一步,复杂度(编译链、跨平台、类型)都上一个台阶;非必要不原生,是 Electron 开发的务实态度。

# 5. 跨平台注意

★ **关键认知**：native addon 与 ffi 都**与平台强绑定**(.dll/.so/.dylib、不同架构)——一套原生代码要**每个平台各编译一遍**。⚠ 这是原生方案的隐藏成本:你的 JS 跨平台,但原生部分要分担"三平台编译 + 测试"的开销;只有确有需要时才值得。

# 6. 本篇小结与下一步

★ **关键认知**：Node 能力边界在三类(调 C 库/硬件/极致性能);native addon 补缺口但需 electron-rebuild 重编译 + 编译工具链;ffi 直接调动态库(快但脆弱);选型"先 JS 包、再 addon、最后 ffi",原生方案跨平台成本高。

下一步第 19 篇:测试与调试——单元测试、e2e(Playwright)、DevTools 与日志。