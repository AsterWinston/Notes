Node.js 提供了海量能力,但仍有边界——调用 C/C++ 库、直接操作硬件、用系统 DLL。**原生模块(native addons)** 与 **ffi** 填上这缺口。本篇讲清 N-API/node-gyp 的原理、Electron 里的 ABI 重编译问题、预编译分发、原生模块的选型权衡,以及常见崩溃定位。

# 1. Node 能力的边界

## 1.1 纯 JS 能做什么、不能做什么

```text
能:文件、网络、进程、绝大多数系统 API(靠 Node 内置模块 + 大量 npm 包)
不能:直接调 C/C++ 库、唯一硬件接口、极致的 CPU 性能
```

★ **关键认知**：Node 的 JS 层能力已经**覆盖绝大多数桌面应用需求**(文件、网络、进程、系统)。⚠ 边界在三类——**要调用某个 C/C++ 库**、**要直接驱动硬件**、**CPU 极密集**(JS 慢)——这些才需要原生模块/ffi。别一上来就上原生,先看 npm 有没有现成的 JS 封装。

# 2. native addons(原生扩展)

## 2.1 什么是 native addon

```text
native addon = 用 C/C++ 写的 Node 模块(编译成 .node 二进制)
在 JS 里 require 它,像普通模块一样用
典型:better-sqlite3、sharp(图像处理)、native 加密库
```

★ **关键认知**：**native addon** 是**用 C/C++ 写的 Node 模块**,编译成 `.node` 动态库,`require` 进来像普通模块用。⚠ 你平时可能已在用(`better-sqlite3`、`sharp` 都是)——它们给 JS 补上"原生性能"与"调 C 库"的能力,但带着"重编译"的成本(见下)。

## 2.2 Electron 的重编译问题(ABI 与 context)

```text
Node 的 ABI(二进制接口)与 Electron 不同
→ 为普通 Node 编译的 addon,不一定能在 Electron 里用
→ 通常需要 electron-rebuild 针对 Electron 重新编译
```

★ **关键认知**：Electron 内嵌的 Node 与系统 Node 的 **ABI 不同**,为普通 Node 编译的 addon **可能在 Electron 里加载失败**。⚠ 解法:`electron-rebuild`(重新针对 Electron 编译 addon)或 `electron-builder install-app-deps`;这是 Electron 使用 native addon 的头号坑。典型报错是 `NODE_MODULE_VERSION mismatch`(expected 125, got 115 之类)——数字对不上就是"按别人的 ABI 编的";每个 Electron 版本有自己的 modules 号,用 `process.versions.modules` 查,N-API addon 不报这个错(正是它的价值)。

## 2.3 node-gyp 与工具链

```powershell
# native addon 编译工具链
npm install -g node-gyp
# 需要平台编译器:Windows 用 VS Build Tools,macOS 用 Xcode
npx electron-rebuild -f -w better-sqlite3    # -f 强制 -w 指定包
```

★ **关键认知**：编译 native addon 需要**本地编译工具链**(Windows 的 VS Build Tools、macOS 的 Xcode Command Line Tools)。⚠ 这是 native addon 的隐性成本——在 CI/多机器上要装齐工具链;纯 JS 项目则完全无此烦恼,这也是"能纯 JS 就纯 JS"的理由。node-gyp 还要下载对应 Electron 的头文件(网络受限时设 `ELECTRON_MIRROR`);Windows 上"npm install 某包后 python/MSBuild 报错"几乎都是它。

## 2.4 两条 API 线:V8 直绑与 N-API

```text
┌──────────────┐   V8 API(C++,随 V8 版本变)  ┌─────────────┐
│  C/C++ 源码  ├─────────────────────────────┼│ node_modules│
│  (addon)     │   N-API(C 接口,ABI 稳定)   ││    .node    │
└──────────────┼─────────────────────────────┼└─────────────┘
      node-gyp 编译: binding.gyp 描述目标/头文件/链接库
```

★ **关键认知**：addon 与 JS 之间隔着两种接口——老的 **V8 C++ API** 跟着 V8 版本变,**每个 Node/Electron 大版本都要重编**;**N-API** 是 Node 官方提供的**稳定 C ABI**,一次编译跨版本通用(配套的 JS 侧封装叫 **node-addon-api**)。⚠ 选型时先看依赖包的 issue:老包重编译报 V8 错误,多半是它还在用 V8 API;新写 addon 一律 N-API 起步,能省掉后面全部 ABI 麻烦。

## 2.5 预编译分发(prebuildify)

```text
成熟包的免编译姿势(用户装包时不碰编译链):
  作者 CI(矩阵) → 各平台/各 ABI 预编 .node
    prebuildify → 产物直接打进 npm 包(prebuilds/ 目录,按平台命名)
      → 用户 npm install 只做"选对文件"的事,零编译
  备选:prebuild + node-gyp-build(发布为 GitHub Release 附件,装时下载)
```

★ **关键认知**：`better-sqlite3`、`sharp` 这类包"装完就能用",是因为作者用 **prebuildify/node-gyp-build 预编好了各平台产物**,npm install 只是挑文件;找不到预制品时才回退本地编译(于是工具链缺失的用户报错)。⚠ Electron 侧要确认包的 prebuilds 覆盖 **electron runtime**(文件名里带 `electron`/`napi` 字样);只带 `node-v115` 的预编译在 Electron 里仍会失败——这正是 N-API 制品通吃的场景。

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

★ **关键认知**：ffi 是"**快速但脆弱**"的桥——声明错一个类型参数,轻则崩溃、重则内存损坏。⚠ 取舍:**调用原生函数的数量少且稳定 → ffi 够用;要正式、稳定、长期维护 → 写正规 native addon**(有类型安全与封装)。ffi 更像"脚本化地试一下原生能力"。另一个现实问题:ffi-napi 系包长期依赖旧 NAPI 版本,新 Node/Electron 上常要等社区分支(如 koffi 这类新一代 FFI 库),选型时先看它的 Electron 兼容性 issue。

# 4. 选型与决策

| 需求 | 方案 |
|------|------|
| 有 npm 现成 JS 包 | 直接用(native 由包自己处理) |
| 调 C/C++ 库、要性能 | native addon(重编译) |
| 只调库里的几个函数 | ffi-napi |
| CPU 极密集计算 | native addon 或 worker |

★ **关键认知**：决策顺序——**先找 npm 现成包 → 再考虑 native addon → 最后才 ffi**。⚠ 每向"原生"走一步,复杂度(编译链、跨平台、类型)都上一个台阶;非必要不原生,是 Electron 开发的务实态度。

## 4.1 Rust/napi-rs 这条新线

```bash
npm create napi@latest my-addon   # napi-rs 脚手架(Rust 写 addon)
npx napi build --platform         # 一条命令产出各平台 .node
# CI 用 napi-rs 官方 action 直接交叉出 win/mac/arm/x64 全家桶
```

★ **关键认知**：现代原生模块的新常态是 **Rust(napi-rs)/ Zig 写内核 + N-API 出口**——napi-rs 自动生成 TS 类型、一条命令交叉编译全平台制品,比 C++/node-gyp 的老链路省心一个量级(sharp 的后继者、大量"高性能替代包"都走这条路)。⚠ 但对"用模块"的 Electron 开发者,收益主要落在**包作者**侧;应用开发者该关心的是:**你依赖的原生包是否提供 napi 预编译制品**——有,基本告别 electron-rebuild;没有,回到第 2.3 节的重编译循环。

# 5. 跨平台注意

★ **关键认知**：native addon 与 ffi 都**与平台强绑定**(.dll/.so/.dylib、不同架构)——一套原生代码要**每个平台各编译一遍**。⚠ 这是原生方案的隐藏成本:你的 JS 跨平台,但原生部分要分担"三平台编译 + 测试"的开销;只有确有需要时才值得。

## 5.1 崩溃定位

```text
"打包后闪退/dev 正常"的原生模块排查顺序:
1. 报错文案: NODE_MODULE_VERSION → ABI 不匹配 → electron-rebuild
2. .node 是否真进了包: 装完看 resources/app.asar.unpacked/
   (asar 里的 .node 无法 dlopen,必须 asarUnpack —— 头号"打包后崩")
3. 依赖的 DLL 是否在场: Windows 上用 Dependencies 工具看缺失
   (VC++ 运行库没装 → "找不到模块"其实缺的是 msvcp140.dll)
4. 架构: arm64 设备装了 x64 addon(Node 的 process.arch 核对)
5. 终极手段: windbg/lldb 挂上,崩在哪个 .so 的哪个符号一眼明了
```

★ **关键认知**：Electron + 原生模块最常见的两个崩溃根因是 **ABI 不匹配**(electron-rebuild 解)和 **`.node` 被 asar 归档**(dlopen 需要真实路径,electron-builder 用 `asarUnpack: ["**/*.node"]` 解,第 13 篇)。⚠ 报错文案会骗人:Windows 上"找不到指定模块"经常不是 `.node` 不在,而是它**依赖的 VC++ 运行库/第三方 DLL 不在**——用户机器没有开发者环境,安装包带上 redistributable 或静态链接才稳。

# 6. 本篇小结与下一步

★ **关键认知**：Node 能力边界在三类(调 C 库/硬件/极致性能);addon 走 N-API 才有稳定 ABI,node-gyp 需本机工具链;Electron 的 ABI 差异靠 electron-rebuild/install-app-deps,成熟包靠 prebuildify 预编译制品免编译;ffi 直接调动态库(快但脆弱);新趋势是 napi-rs/Rust 交叉编译全家桶;崩溃定位先查 NODE_MODULE_VERSION 与 asarUnpack,再查 DLL 依赖与架构。选型"先 JS 包、再 addon、最后 ffi",原生方案跨平台成本高。

下一步第 19 篇:测试与调试——单元测试、e2e(Playwright)、DevTools 与日志。
