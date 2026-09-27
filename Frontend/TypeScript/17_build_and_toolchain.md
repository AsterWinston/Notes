把 TypeScript 从"能编译"推进到"能发布"，要面对的是一个分工明确的工具链：tsc 负责类型检查与声明产出，esbuild/swc 负责高速转译，package.json 的 `exports` 字段决定消费端拿到什么。本篇只讲 TS 原生工具链的构建、发布与性能优化，给出一份可照抄的命令清单。

# 1. tsc 的定位：官方编译器

## 1.1 编译器 = 类型检查 + 转译 + 声明产出

```typescript
// 同一个源文件，tsc 能同时产出三种资产（见第 1 篇流水线）
// src/index.ts
export function add(a: number, b: number): number { return a + b; }   // 源码

// 产出 1：dist/index.js —— 类型擦除 + 语法降级后的可执行代码
// 产出 2：dist/index.d.ts —— 给使用者看的类型契约（--declaration）
// 产出 3：dist/index.js.map —— 报错栈映射回源码（--sourceMap）

// 命令行等价开关：
// npx tsc --declaration --sourceMap     一次编译三种产出
// npx tsc --emitDeclarationOnly         只产出 .d.ts（转译交给别的工具时用）
```

★ **关键认知**：tsc 是目前**唯一**同时做"类型检查 + 转译 + 声明产出"的工具——esbuild/swc 只做第二件事。声明产出依赖全程序类型分析，这就是为什么 `.d.ts` 无法由转译器生成、必须留给 tsc 的根本原因。

## 1.2 npx tsc --noEmit 在 CI 中的角色

```powershell
# CI 流水线里的类型关卡：只检查、不写任何文件
npx tsc --noEmit                          # 返回非零退出码 = 类型检查失败，流水线中断
```

```
  git push ──▶ CI ──▶ ┌────────────────────┐ ──▶ 测试 ──▶ 发布
                      │ npx tsc --noEmit   │
                      │ （类型关卡，秒级~） │
                      └────────────────────┘
```

★ **关键认知**：`--noEmit` 把 tsc 降级为**纯检查器**，这是"类型检查与转译分离"模式的半壁江山（另半壁是 esbuild 们）。⚠ 没有这道关卡，esbuild/sucrase 会"友好地"擦掉类型照跑不误——类型错误静默潜入产物，类型系统等于白开。CI 里它应当排在测试之前，因为它是成本最低的失败。

# 2. 转译器对比与分离模式

## 2.1 主流转译器对比

| 工具 | 实现语言 | 类型检查 | 转译速度量级 | 定位 |
|------|---------|---------|-------------|------|
| tsc | TypeScript/JS | ✓ 全量 | 慢（万行/秒级） | 官方编译器：检查 + 转译 + 声明 |
| esbuild | Go | ✗ | 极快（比 tsc 快 10~100 倍） | 打包器内置转译 |
| swc | Rust | ✗ | 极快 | 可编程转译器（插件生态） |
| sucrase | JS | ✗ | 快 | 开发期即时转译（小型工具） |

```typescript
// 各转译器对同一段 TS 的处理都是"删类型"，但能力边界不同
const x: number = 1;                       // 普通注解：人人会删
enum Color { Red }                         // ⚠ 枚举：需要产出运行时对象，sucrase 等走降级路径
const e = Color.Red;                       // enum 是"有运行时代码的类型特性"，删不干净
type T = { a: number };                    // 纯类型：人人可安全删除
```

★ **关键认知**：转译器之间的真实差异不在速度而在**哪些语法需要"生成代码"**——enum、namespace、参数属性（`constructor(public x)`）都属于"类型语法产出运行时逻辑"，是 transpile-only 工具的兼容性雷区。⚠ 这正是 `isolatedModules`（见第 16 篇）存在的原因：提前把这些写法拦在编译期。

## 2.2 类型检查与转译分离的工程模式

```
                 ┌──────────────────────┐
   src/*.ts ────▶│  esbuild / swc 转译  │────▶ dist/*.js      （速度线：秒级）
                 │  （擦类型 + 降级）    │
                 └──────────────────────┘
                 ┌──────────────────────┐
   src/*.ts ────▶│  tsc --noEmit        │────▶ 报错 / 通过    （质量线：分钟级）
                 │  （纯类型检查）        │
                 └──────────────────────┘
                 ┌──────────────────────┐
   src/*.ts ────▶│  tsc --emitDeclarationOnly │──▶ dist/*.d.ts （类型契约线）
                 └──────────────────────┘
```

★ **关键认知**：分离模式的本质是把 tsc 的三个能力**拆开各用其长**——转译交给快 10 倍以上的工具（开发体验），检查保留 tsc（准确性无可替代），声明产出按需执行（只有库需要）。⚠ 三条线的配置必须共享同一份 tsconfig 语义（target/lib 一致），否则"检查通过的代码"与"实际产出"出现偏差，报错会以诡异形式出现在运行时。

# 3. 声明产出

## 3.1 declaration 与 emitDeclarationOnly

```jsonc
// tsconfig.build.json —— 库项目专用的构建配置
{
  "extends": "./tsconfig.json",         // 继承检查配置
  "compilerOptions": {
    "declaration": true,                // 产出 .d.ts
    "declarationMap": true,             // 产出 .d.ts.map（跳转回源码）
    "emitDeclarationOnly": true,        // 只出声明，不出 js（js 归 esbuild）
    "outDir": "dist/types",             // 声明统一放 types 目录
    "rootDir": "src"
  },
  "include": ["src"]
}
```

★ **关键认知**：`emitDeclarationOnly` 是分离模式的第三条线专用开关——它让 tsc 只做"类型检查 + 声明产出"，把 `.js` 让给转译器。⚠ 声明产出会**继承** `declarationMap`/`rootDir`/`outDir` 的结构约束：`rootDir` 写错会导致 `dist/types` 里出现意料之外的嵌套层级，消费者的 `types` 路径随即失效。

## 3.2 声明聚合：api-extractor 与 rollup-plugin-dts

```typescript
// 问题：tsc 按源码结构产出声明，几十个 .d.ts 散落一地
// dist/types/index.d.ts      → export * from "./utils/math"   （内部结构全暴露）
// dist/types/utils/math.d.ts → export declare function add(...)   （深链可被 import）

// 方案A：api-extractor —— 聚合成单文件 + API 报告（检测破坏性变更）
// npx api-extractor run --local
// 产出 dist/my-lib.d.ts（单文件） + api-report.md（签名快照，可 diff）

// 方案B：rollup-plugin-dts / dts-bundle-generator —— 打包式聚合声明
// 适用：只想把 N 个 .d.ts 合成 1 个、不需要 API 报告的轻量场景
```

| 方案 | 产出 | 适用场景 | 额外能力 |
|------|------|---------|---------|
| tsc 原生 | N 个 `.d.ts` | 内部项目、不在乎入口暴露面 | 无 |
| api-extractor | 单文件 + API 报告 | 对外发布的库 | 破坏性变更检测、beta 标记 |
| rollup-plugin-dts | 单文件 | 轻量聚合 | 打包器插件形态 |

★ **关键认知**：聚合的价值不只是"文件少"，更是**封装**——不聚合时消费者可以 `import "my-lib/internal/secret"` 深链到内部模块，聚合后只有 `index.d.ts` 的导出面可见。⚠ 聚合产物必须与 `package.json` 的 `types`/`exports.types` 指向一致，否则消费端解析到的是聚合前的散装声明，封装白做。

# 4. 发布 npm 包

## 4.1 双格式 ESM + CJS 的目录结构

```
   my-lib/
   ├── src/                     TS 源码（唯一事实源）
   ├── dist/
   │   ├── esm/
   │   │   ├── index.js         ESM 产物（export ...）
   │   │   └── index.d.ts       ESM 侧声明
   │   └── cjs/
   │       ├── index.js         CJS 产物（exports.x = ...）
   │       ├── package.json     ← {"type":"commonjs"}（钉死目录格式！）
   │       └── index.d.ts       CJS 侧声明
   ├── package.json
   └── tsconfig.json
```

★ **关键认知**：双格式的核心矛盾是**扩展名与 `type` 字段的归属**——`.js` 的模块格式由最近的 `package.json` 的 `type` 决定，所以 `dist/cjs/` 里塞一个 `{"type":"commonjs"}` 的包内 `package.json` 是行业标准做法。⚠ 只在根声明 `"type": "module"` 而不动 cjs 目录，CJS 产物会被 Node 当 ESM 加载，直接 `exports is not defined`。

## 4.2 package.json 的入口字段详解

```jsonc
{
  "name": "my-lib",
  "version": "1.0.0",
  "type": "module",                          // 默认模块格式：ESM
  "main": "./dist/cjs/index.js",             // 旧解析器（CJS 入口）
  "module": "./dist/esm/index.js",           // 打包器识别的 ESM 入口（约定俗成、非标准）
  "types": "./dist/esm/index.d.ts",          // 旧 TS（<4.7）的类型入口
  "exports": {
    ".": {                                   // 主入口的条件导出
      "types": "./dist/esm/index.d.ts",      // ⚠ types 条件必须排第一
      "import": "./dist/esm/index.js",       // import 时用 ESM 产物
      "require": "./dist/cjs/index.js"       // require 时用 CJS 产物
    },
    "./package.json": "./package.json"       // 显式放行这个子路径
  },
  "files": ["dist"],                         // 发布白名单：只带 dist
  "sideEffects": false,                      // 声明无副作用，允许摇树
  "scripts": {
    "build": "tsc -p tsconfig.build.json && tsc -p tsconfig.cjs.json",
    "typecheck": "tsc --noEmit",
    "prepublishOnly": "npm run typecheck && npm run build"   // 发布前自动把关
  }
}
```

★ **关键认知**：`exports` 是**条件导出**——按消费方式（`types`/`import`/`require`）分流到不同产物，条件按**声明顺序**匹配，所以 `types` 必须放最前（TS 要求类型条件优先命中）。`main`/`module`/`types` 三个字段是 `exports` 之前的旧世界入口，保留它们是为了兼容老工具链；⚠ 新项目写了 `exports` 就以它为准，旧字段只是兜底。

## 4.3 exports 的封装性：最容易翻车的一坑

```typescript
// 消费端（my-lib 有 exports 字段时）
import { add } from "my-lib";               // ✓ 命中 exports["."]
import { secret } from "my-lib/dist/esm/utils.js";   // ✗ 报错！子路径被封装

// 只有显式列出的子路径才可导入：
// "exports": { ".": ..., "./utils": "./dist/esm/utils.js" }   // 放行 my-lib/utils
```

| 现象 | 成因 | 解法 |
|------|------|------|
| 子路径导入报 ERR_PACKAGE_PATH_NOT_EXPORTED | `exports` 封装未列出的路径 | 显式补 `"./子路径"` 映射 |
| 旧 TS 报"找不到模块的类型" | 旧 TS 不认 `exports` 的 `types` 条件 | 保留顶层 `types` 字段兜底 |
| 项目里同时存在两份库实例 | ESM/CJS 双份产物被分别加载（dual package hazard） | 尽量纯 ESM，或保证实例只经一种格式加载 |
| 发布包里多了一堆源码 | `files` 白名单没配 | 加 `"files": ["dist"]` 后 `npm pack --dry-run` 核对 |

★ **关键认知**：⚠ **`exports` 一旦出现就全面接管模块解析**——它同时是"放行清单"和"封装墙"，这是 Node 的刻意设计（让库作者锁死公共 API 面）。它与第 15 篇的声明封装是同一思想在两个层面的落地：类型层用聚合声明收敛 `types` 入口，运行时层用 `exports` 收敛 JS 入口。

# 5. 性能优化

## 5.1 四个关键开关

```jsonc
{
  "compilerOptions": {
    "incremental": true,          // 增量编译：只重查变化文件及其依赖
    "tsBuildInfoFile": ".cache/tsbuildinfo",  // 增量缓存文件位置（可提交也可忽略）
    "skipLibCheck": true,         // 跳过所有 .d.ts 的检查（第三方声明你修不了）
    "isolatedModules": true       // 单文件可转译：配合 esbuild，也减少跨文件分析
  }
}
```

| 开关 | 提速机制 | 代价 |
|------|---------|------|
| `incremental` | `.tsbuildinfo` 记录文件指纹，跳过未变文件 | 一个缓存文件；改 tsconfig 会全量重建 |
| `skipLibCheck` | 不检查 `.d.ts` 内部一致性 | 第三方类型的内部矛盾不会被发现 |
| `isolatedModules` | 逼代码符合"单文件转译"约束 | 部分写法报错（enum 跨文件、type 混合导出等） |
| `references` | 按包粒度跳过未变下游（见第 16 篇） | 多包结构与配置成本 |

★ **关键认知**：大项目里 `skipLibCheck` 往往是**单项提速之王**——node_modules 里的 `.d.ts` 数量远超自有代码，跳过它们直接砍掉大半检查时间。⚠ `incremental` 的缓存与 `outDir` 无关（默认写在 tsconfig 同级的 `.tsbuildinfo`），CI 缓存该文件能显著缩短构建时间；但改了 `compilerOptions` 必然全量重建，缓存别指望跨配置复用。

# 6. 可照抄执行的命令清单

```powershell
# ===== 1. npm 切国内镜像（安装提速，发布前记得切回） =====
npm config set registry https://registry.npmmirror.com
npm config get registry                # 确认输出 https://registry.npmmirror.com/

# ===== 2. 初始化库项目并安装 TS 工具链 =====
npm init -y
npm install -D typescript               # 编译器 + 类型检查器
npm install -D tsx                      # 本地直接跑 .ts（开发调试用）
npm install -D @microsoft/api-extractor # 声明聚合（可选，对外发布库推荐）

# ===== 3. 生成并精简 tsconfig（strict/open composite 等见第 16 篇） =====
npx tsc --init

# ===== 4. 类型检查（CI 关卡，不产出文件） =====
npx tsc --noEmit

# ===== 5. 构建产物（js + .d.ts + sourcemap，按 tsconfig 配置输出） =====
npx tsc

# ===== 6. 只构建声明（js 归 esbuild 等转译器时） =====
npx tsc --emitDeclarationOnly

# ===== 7. 声明聚合（可选） =====
npx api-extractor run --local

# ===== 8. 发布前检查：核对将发布的文件清单 =====
npm pack --dry-run                     # 确认 dist/ 与 .d.ts 都在包里

# ===== 9. 发布（⚠ 目标是官方 registry；镜像只加速安装） =====
npm config get registry                # 若指向镜像，先 npm config set registry https://registry.npmjs.org/
npm publish --access public            # scoped 包必须带 --access public
```

★ **关键认知**：这条流水线的每一环都对应本篇的一个机制——`--noEmit` 是质量线（第 1 章）、`--emitDeclarationOnly` 是类型契约线（第 3 章）、`npm pack --dry-run` 是封装面检查（第 4 章）。⚠ 与 Java 的 Maven 生命周期（validate → package → deploy 一条命令贯通）相比，TS 工具链是**拼装式**的：每一步独立可换，收益是灵活、代价是每一步的配置一致性要靠人守。
