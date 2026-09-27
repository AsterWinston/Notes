tsconfig.json 是 TypeScript 项目的总控台：几十个编译选项里真正值得吃透的不过二十来个，但每一个都直接决定类型检查的强度与产物的形态。本篇按"结构 → 选项精选 → strict 家族逐开关 → 工程化引用 → 报错排查"的顺序把 tsconfig 讲深讲透。

# 1. tsconfig 的结构

## 1.1 顶层字段一览

```jsonc
// tsconfig.json —— 完整骨架
{
  "extends": "./tsconfig.base.json",    // 继承基础配置（可组合）
  "compilerOptions": {                  // 编译选项（本文第 2 章逐项讲解）
    "strict": true
  },
  "files": ["src/types/global.d.ts"],   // 显式列出的文件（精确、不可被 include 覆盖）
  "include": ["src"],                   // glob 模式圈定参与编译的文件
  "exclude": ["src/**/*.test.ts"],      // 从 include 结果里剔除（默认排除 node_modules）
  "references": [                       // 工程引用（见第 4 章）
    { "path": "./packages/core" }
  ]
}
```

★ **关键认知**：参与编译的文件集合 = `files` ∪ `include` − `exclude`，而**类型检查的范围不受此限**——被 `import` 到的外部文件即使不在集合里也会被拉进类型图（只是不产出）。这解释了一个经典疑惑："我 exclude 了某目录，为什么它的错误还在报？"⚠ 只有 `files`/`include` 决定**产出**哪些 `.js`，`exclude` 挡不住 `import` 带来的类型检查。

## 1.2 extends 继承与组合

```jsonc
// tsconfig.base.json —— 团队统一的基础配置
{
  "compilerOptions": {
    "strict": true,                     // 团队底线：严格检查
    "target": "ES2022",                 // 统一产物语法下限
    "skipLibCheck": true,               // 统一跳过 .d.ts 检查
    "declaration": true,                // 统一产出声明
    "paths": { "@/*": ["src/*"] }       // 统一路径别名
  }
}

// tsconfig.json —— 项目级覆盖
{
  "extends": "./tsconfig.base.json",    // 相对路径相对于"本文件"解析
  "compilerOptions": {
    "declaration": false,               // 按 key 覆盖：本项目不产出声明
    "outDir": "dist"                    // 新增 key：base 里没有的照常添加
  },
  "include": ["src"]                    // files/include/exclude 是整体替换，不是合并
}
```

★ **关键认知**：`extends` 的合并语义是"**compilerOptions 按 key 覆盖**，`files`/`include`/`exclude` **整体替换**"——数组不拼接，这点和多数人直觉相反。⚠ `extends` 的路径相对于**发起继承的那个文件**解析（而不是项目根），把 base 放进上层目录时要写 `../tsconfig.base.json`。⚠ 别忘了 `paths` 里的相对路径同样相对 base 文件所在目录解析，这是多项目复用 base 时的高频翻车点。

## 1.3 与 Java/Go 构建配置的对比

| 维度 | tsconfig.json | Java（Maven pom.xml） | Go（go.mod + 命令行） |
|------|--------------|----------------------|----------------------|
| 定位 | 编译器全部行为的声明式配置 | 依赖 + 构建生命周期 | 仅模块依赖 |
| 继承机制 | `extends` 单继承 | parent + BOM | 无 |
| 选项数量 | 100+ | 插件体系间接决定 | 少量 env/flag |
| 类型检查开关 | strict 家族细粒度 | 编译器固定规则 | 固定规则、不可关 |

★ **关键认知**：TS 把"检查强度"做成了**可配置的连续光谱**，而 Go 选择"规则固定、不可协商"。灵活的代价是配置组合爆炸——所以本篇只精选真正常改的选项，其余交给 `npx tsc --init` 生成的注释版配置查表。

# 2. 编译选项精选

## 2.1 产物类选项：决定产出什么

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",           // 语法降级下限：决定类/async 等被翻译成什么形态
    "module": "ESNext",           // 产出的模块格式（ESM / CJS / AMD…）
    "moduleResolution": "bundler",// 模块解析策略（见第 14 篇对比表）
    "lib": ["ES2022", "DOM"],     // 类型检查时认得哪些全局 API（不注入任何运行时代码）
    "outDir": "dist",             // 产出目录（保持源码目录结构）
    "rootDir": "src",             // 源码根：决定 dist 下的相对层级
    "outFile": "bundle.js",       // 合并单文件（仅 module: amd/system 有效）
    "declaration": true,          // 产出 .d.ts（发布库必备）
    "declarationMap": true,       // .d.ts.map：跳转定义能回到 .ts 源码
    "sourceMap": true             // .js.map：运行时报错栈映射回 .ts
  }
}
```

| 选项 | 取舍建议（一句话） |
|------|------------------|
| `target` | 跟着运行时的最低支持走，浏览器项目写 `ES2017`+，Node 18+ 写 `ES2022` |
| `module` | 要产出 ESM 写 `ESNext`，纯 Node CJS 项目写 `CommonJS` |
| `lib` | 服务端项目别加 `DOM`，否则 `window`/`document` 能过编译、运行时不存在 |
| `outDir` | 永远指向独立目录（`dist`/`build`），别把产物混进源码树 |
| `rootDir` | 显式写死 `src`，防止测试文件被意外纳入后目录层级错乱 |
| `declaration` | 发布 npm 包必开；纯应用可关闭以加速编译 |
| `sourceMap` | 生产排查必备，成本极低（产物多几个 `.map` 文件） |

★ **关键认知**：⚠ `lib` 与 `target` 是**正交**的——`target` 决定语法怎么降级，`lib` 决定"哪些 API 名字合法"。`target: "ES5"` + `lib: ["ES2022"]` 会得到"语法降级了、`Promise`/`Array.flat` 也认了，但旧引擎上照崩"的组合——新 API 永远不会被自动注入，缺失只能靠 polyfill。

## 2.2 检查类选项：决定查多严

```jsonc
{
  "compilerOptions": {
    "strict": true,                  // 严格家族总闸（见第 3 章逐开关）
    "noUnusedLocals": true,          // 未使用的局部变量报错
    "noUnusedParameters": true,      // 未使用的参数报错（_ 前缀可豁免）
    "noImplicitReturns": true,       // 有分支没 return 时报错
    "noFallthroughCasesInSwitch": true, // switch 穿透报错（strict 含此开关）
    "exactOptionalPropertyTypes": true, // 可选属性不能显式赋 undefined（更苛刻）
    "noPropertyAccessFromIndexSignature": false // 点号访问索引签名属性是否报错
  }
}
```

★ **关键认知**：检查类选项分两档——`strict` 家族是**语义正确性**（不这么查会埋运行时炸弹），`noUnused*` 是**代码卫生**（只影响整洁度）。⚠ 存量项目一次性开 `exactOptionalPropertyTypes` 或 `noUnusedLocals` 会雪崩式报错，正确姿势是先开 `strict`、其余卫生开关分批补。

## 2.3 工程类选项：决定怎么组织

```jsonc
{
  "compilerOptions": {
    "incremental": true,        // 增量编译：只重查变化文件（缓存写入 .tsbuildinfo）
    "composite": true,          // 工程引用的入场券（强制 declaration + incremental）
    "skipLibCheck": true,       // 跳过 .d.ts 的检查，大项目提速显著
    "isolatedModules": true,    // 强制"每个文件可单独转译"（配合 esbuild/swc）
    "baseUrl": ".",             // 解析基准目录（paths 的前置）
    "paths": { "@/*": ["src/*"] }, // 路径别名（⚠ 仅编译期，见第 14 篇）
    "tsBuildInfoFile": ".cache/tsbuildinfo" // 增量缓存位置自定义
  }
}
```

| 选项 | 取舍建议（一句话） |
|------|------------------|
| `incremental` | 无脑开启，构建时间几乎必然缩短，代价只是一个缓存文件 |
| `composite` | 仅在多包 monorepo 里开（单项目没有收益还多约束） |
| `skipLibCheck` | 开——第三方 `.d.ts` 的报错你修不了，跳过是行业默认 |
| `isolatedModules` | 用 esbuild/swc/tsx 转译就开，把问题拦在编译期而不是打包器里 |
| `paths` | 少用、集中管理；用它就必须配运行时方案（打包器 alias 或 `imports`） |

★ **关键认知**：`isolatedModules` 是**工具链契约**——它把"全程序分析才能保证的正确性"降格为"单文件即可转译"，换来与现代转译器的兼容。⚠ 它会让 `export { SomeType }`、跨文件 `const enum` 等写法直接报错，但这些报错正是打包器里会静默出 bug 的地方，早报早好。

# 3. strict 家族逐开关对照表

| 开关 | 拦截什么错误 | 反例（一行） |
|------|-------------|-------------|
| `strictNullChecks` | null/undefined 的解引用 | `user.name.toUpperCase(); // name?: string` |
| `noImplicitAny` | 隐式 any（参数/属性无注解且推断不出） | `function f(x) { return x; } // x: any` |
| `strictFunctionTypes` | 函数参数逆变被错误放宽 | `const f: (x: string) => void = (x: number) => {};` |
| `strictBindCallApply` | `bind/call/apply` 丢参失检 | `fn.bind(null, 123); // 参数错也通过` |
| `strictPropertyInitialization` | 字段声明未初始化 | `class A { x: number; } // x 从未赋值` |
| `noImplicitOverride` | 子类覆盖父类成员未写 `override` | `class B extends A { foo() {} } // 缺 override` |
| `noFallthroughCasesInSwitch` | switch 分支穿透 | `case 1: doA(); case 2: doB(); // 穿透` |
| `useUnknownInCatchVariables` | catch 变量被当 any 滥用 | `catch (e) { console.log(e.message); }` |
| `noUncheckedIndexedAccess` | 索引访问忽略"可能越界" | `const v = arr[0].name; // arr[0] 可能 undefined` |
| `alwaysStrict` | 产出非严格模式代码 | `// 缺 "use strict" 的 CJS 产出` |

```typescript
// strictNullChecks 的靶子：可选属性的真实类型是 string | undefined
function getUserName(user: { name?: string }) {
  return user.name.toUpperCase();        // ✗ TS2532: Object is possibly 'undefined'
}

// strictPropertyInitialization 的靶子：字段没初始化
class User {
  name: string;                          // ✗ TS2564: Property has no initializer
  constructor() { /* this.name 未赋值 */ }
}

// noUncheckedIndexedAccess 的靶子：下标访问后多出 undefined
const xs = [1, 2, 3];
const first = xs[0];                     // 类型是 number | undefined（不是 number！）
```

★ **关键认知**：`strict: true` 等价于**同时打开表中全部开关**（外加 `noImplicitThis`），各开关也可单独关闭来"给存量代码留缓冲"。⚠ 唯一例外是 `noUncheckedIndexedAccess`——它**不在** `strict` 家族里（太苛刻会波及所有数组/对象索引访问），需要显式开启。这与 Go"全部固定开启"的哲学再次形成对照：TS 允许逐项协商强度，但代价是团队间配置漂移。

# 4. project references 与 monorepo

## 4.1 composite 与 references 配置

```jsonc
// packages/core/tsconfig.json —— 被依赖的基础包
{
  "compilerOptions": {
    "composite": true,          // 工程引用必备：强制 declaration + incremental
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}

// packages/app/tsconfig.json —— 依赖 core 的上层包
{
  "compilerOptions": {
    "composite": true,
    "outDir": "dist",
    "rootDir": "src",
    "paths": {}                 // 引用包靠 references 解析，不靠 paths 硬指
  },
  "references": [
    { "path": "../core" }       // 声明依赖：构建顺序 + 类型来源
  ],
  "include": ["src"]
}
```

```typescript
// packages/app/src/main.ts —— 引用方式与普通包一致
import { Core } from "core";   // 类型来自 core 的 .d.ts 产物（不是源码）
```

## 4.2 增量构建机制

```
        ┌──────────────────────────────────────────┐
        │  tsc --build（构建模式：按依赖图拓扑排序） │
        └──────────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │  core   │ ───▶ │  utils  │ ───▶ │   app   │   依赖方向：app → utils → core
   │ .tsbuild│      │ .tsbuild│      │ .tsbuild│
   │  info   │      │  info   │      │  info   │
   └─────────┘      └─────────┘      └─────────┘
        │                 │                 │
        └── 只重编"指纹变了"的包及其下游 ──────┘
```

★ **关键认知**：`composite` + `references` 的增量机制基于**声明产物指纹**——下游包只读上游的 `.d.ts`（而非 `.ts` 源码），上游没改出新声明时下游完全跳过编译。⚠ 由此推论：改了 core 的实现但没改导出签名，`tsc --build` 依然跳过 app（正确且高效）；但 ⚠ 手删 `dist` 里的 `.d.ts` 会让下游解析失败，删产物必须连 `.tsbuildinfo` 一起删（或直接 `tsc --build --clean`）。

## 4.3 与 Go monorepo / Java 多模块的对比

| 维度 | TS project references | Go 多包 | Java Maven 多模块 |
|------|----------------------|---------|------------------|
| 依赖声明 | `references` 显式列出 | import 路径隐式推导 | `<dependency>` 显式 |
| 依赖产物 | 上游 `.d.ts` + `.js` | 源码直接编译 | jar + 元数据 |
| 增量单位 | 指纹（`.tsbuildinfo`） | 包级重编 | 模块级 |
| 循环依赖 | 手动杜绝（无强制） | 编译报错拒绝 | 构建报错拒绝 |

★ **关键认知**：⚠ TS 的引用图**不检查循环引用**——A 引 B、B 引 A 能配置通过，但运行时会拿到 `undefined`（CJS 循环 require 的经典症状）。monorepo 里应靠 lint 工具或 code review 保证依赖单向，这是 TS 工程化至今的短板之一。

# 5. 常见报错与排查

| 报错 | 含义 | 排查方向 |
|------|------|---------|
| TS2307 Cannot find module | 模块解析失败 | 扩展名要求、`paths` 映射、包是否安装、`moduleResolution` 是否匹配 |
| TS2304 Cannot find name | 名字不存在 | 缺 `lib`、缺声明文件、拼写错误 |
| TS2503 Cannot find namespace | 命名空间/类型不存在 | 缺 `@types/*`、`types` 字段指错 |
| TS2300 Duplicate identifier | 声明撞名 | 全局声明冲突、同名 `type` 合并、重复装 `@types` |
| TS2717 Subsequent property declarations… | interface 合并属性类型不一致 | 两处同名属性类型不同（见第 14 篇） |
| TS1259 import can only be default-imported… | CJS 默认导入未开互操作 | 开 `esModuleInterop` |
| TS7006 Parameter implicitly has an 'any' | 隐式 any 参数 | 补注解或开 `noImplicitAny` 后逐个修 |
| TS6133 'x' is declared but never read | 未使用变量 | 删掉或开/关 `noUnusedLocals` |
| TS6305 Output not built… | 工程引用未先构建 | 先 `tsc --build` 上游包 |
| 运行时 undefined（无 TS 报错） | 循环引用 | 画出 import 图打断环、改用惰性引用 |

```typescript
// 排查 TS2307 的三板斧（按成本从低到高）
// 1) 看报错里的模块名：相对路径拼写、大小写是否与磁盘一致
import { x } from "./util";                 // 若磁盘是 Utils.ts，Linux 上必炸
// 2) 看 moduleResolution：bundler/node16 对扩展名要求不同（见第 14 篇）
// 3) 看包本身：npm ls <包名> 确认真装了，且 types 字段指向的文件存在
```

★ **关键认知**：⚠ 循环引用是表中唯一"**编译器沉默、运行时爆炸**"的项目——TS 的类型检查允许循环 import（类型层面环是良性的），但 CJS 的 `require` 环会让先加载方拿到未初始化的 `exports`。检测靠工具（依赖图分析）、根治靠架构：让依赖方向保持单向分层，是所有模块系统通用的纪律。
