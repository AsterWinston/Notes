**声明文件（.d.ts）**是 TypeScript 与无类型 JavaScript 世界之间的桥：它只描述类型、不产出任何运行时代码，却让整个 npm 生态能被类型系统覆盖。本篇讲清 `declare` 语法家族、手写类型声明的技巧，以及 `@types` 生态与 `allowJs` 渐进式加类型的路线。

# 1. 声明文件与 declare 语法家族

## 1.1 声明文件是什么

```typescript
// math.d.ts —— 声明文件：只有签名，没有实现体
export function add(a: number, b: number): number;   // 函数签名（结尾是分号，不是花括号）
export const VERSION: string;                        // 变量声明
export interface Point { x: number; y: number }      // 纯类型照常写
// export function add(a: number, b: number): number { return a + b; }  // ✗ 声明文件里禁止实现
```

★ **关键认知**：`.d.ts` 是**只含类型的编译单元**——它不被转译成 `.js`，只在类型检查时被消费。这与 Java 的 `.class` 元数据不同：Java 的类型信息是运行时可反射的产物，TS 的声明文件是编译期的纯静态契约，运行时同样被**类型擦除**得干干净净。⚠ 在 `.d.ts` 里写实现体（函数有花括号、变量有初始值）直接报 TS1183。

## 1.2 declare 语法全家福

```typescript
// vendor.d.ts —— ambient（环境）声明：描述"别处提供的存在"
declare const VERSION: string;                     // 全局常量（运行时由宿主注入）
declare function format(d: Date): string;          // 全局函数
declare class LegacyQueue {                        // 全局类：只有签名
  constructor(size: number);                       // 构造签名
  push(item: string): void;                        // 方法签名
  readonly size: number;                           // 属性签名
}
declare enum Level { Low, High }                   // 全局枚举（注意：会产出运行时对象的类型）
declare namespace LegacyUtils {                    // 全局命名空间（分组）
  function pad(s: string): string;                 // LegacyUtils.pad
}
declare module "legacy-lib" { ... }                // 见 2.1：为无类型模块补声明
declare global { ... }                             // 见第 14 篇：全局增强
```

★ **关键认知**：`declare` 的语义是"**我承诺这个名字存在于此**"，它把变量声明的"值创建"降级为"纯类型声明"。⚠ `declare` 出来的名字在运行时**不会被创建**——写了 `declare const VERSION` 却没人真的提供 `VERSION`，运行时访问就是 `ReferenceError`。这与 Go 的接口"鸭子契约"形似：都只约定形状，不提供实现。

## 1.3 声明文件的三种身份

| 身份 | 文件特征 | 生效方式 | 例子 |
|------|---------|---------|------|
| 模块声明文件 | 有 `export`/`import` | 被 `import` 时提供类型 | 自己库的 `index.d.ts` |
| 全局声明文件 | 无 `export`/`import` | 自动纳入全局作用域 | `global.d.ts` |
| 脚本式 ambient | 顶层 `declare` | 同上（老式全局库） | `jquery.d.ts` |

★ **关键认知**：⚠ 一个文件"是不是模块"决定了所有名字的可见性——无 `import`/`export` 的文件里一切声明都是全局的，容易与其他声明撞名报 TS2300。现代项目建议让 `.d.ts` 显式成为模块（写 `export`），全局性只留给真正需要的场景。

# 2. 为纯 JS 库手写类型声明

## 2.1 ambient 模块声明：declare module "lib-name"

```typescript
// types/legacy-lib.d.ts —— 为没有类型的 npm 包补声明
declare module "legacy-lib" {
  // 用法与普通模块声明完全一致：describe 而不是 implement
  export function parse(input: string): Record<string, unknown>;
  export function serialize(data: object): string;
  export default function main(): void;             // 默认导出的声明
  export interface ParseOptions { strict?: boolean } // 类型照常导出
}

// 业务代码里照常导入，从此拥有完整类型
import { parse, type ParseOptions } from "legacy-lib";   // ✓ 解析到上面的声明
```

★ **关键认知**：`declare module "包名"` 是 **ambient 模块声明**——它告诉编译器"这个字符串对应的模块长这样"，从而接管该包的全部类型来源。⚠ 它同时**屏蔽**了包内可能存在的 `index.d.ts`：一旦你为某包手写了 ambient 声明，官方类型不再被读取，维护时要同步跟进上游 API 变化。

## 2.2 通配模块声明：declare module "*.ext"

```typescript
// types/assets.d.ts —— 为"非 JS 资源"的导入语句补类型
declare module "*.txt" {
  const content: string;                            // import x from "./a.txt" 的类型是 string
  export default content;
}
declare module "*.raw" {
  export function read(): string;                   // 通配声明也能描述复杂导出
}
```

★ **关键认知**：通配声明按**后缀模式**匹配 import 说明符，解决的是"打包器把 `.txt`/`.css` 这类资源也当作模块处理"的类型缺口（⚠ 实际把资源打进产物的是打包器，TS 只补类型）。这与 Java 的 SPI、Go 的 `embed` 有本质不同：那两种是运行时机制，TS 的通配声明纯粹是编译期的语法兜底。

# 3. 泛型库的声明技巧

## 3.1 泛型函数与重载声明

```typescript
// types/generic-lib.d.ts
declare module "generic-lib" {
  // 泛型函数：类型参数照常写
  export function first<T>(items: readonly T[]): T | undefined;

  // 重载声明：多条签名 + 不写实现（声明文件里没有实现签名）
  export function pick(obj: object, key: string): unknown;
  export function pick<T, K extends keyof T>(obj: T, key: K): T[K];

  // 泛型接口 + 类
  export interface Repo<T> {
    get(id: string): Promise<T>;
    save(item: T): Promise<void>;
  }
  export class SqlRepo<T> implements Repo<T> {
    constructor(table: string);
    get(id: string): Promise<T>;
    save(item: T): Promise<void>;
  }
}
```

★ **关键认知**：声明文件里的重载**只有重载签名、没有实现签名**——实现体在 JS 那边，声明只负责"每个可调用形态"。⚠ 重载顺序依然遵循"从具体到宽泛"的匹配规则，顺序写反会让宽签名吃掉所有调用、窄签名永远匹配不到，类型精确度直接塌方。

## 3.2 接口 + 构造签名合并的库模式

```typescript
// 库常见模式：同一个名字既是"可调用函数"又是"可 new 的构造器"，还带静态属性
declare module "event-lib" {
  export interface Emitter {
    on(event: string, cb: (...args: unknown[]) => void): this;
    off(event: string, cb: (...args: unknown[]) => void): this;
    emit(event: string, ...args: unknown[]): boolean;
  }

  export const Emitter: {
    new (): Emitter;                    // 构造签名：new Emitter() 得到 Emitter 实例
    readonly version: string;           // 静态属性
    create(): Emitter;                  // 静态工厂
  };
}
```

★ **关键认知**：`interface Emitter` 描述**实例类型**，`const Emitter` 描述**值的形状**（含构造签名）——同名的类型声明与值声明在两个空间共存，互不冲突。这是 JS 库"一个导出多重身份"的标准建模手法，等价于第 14 篇的 namespace 合并思路在模块内的应用。⚠ 构造签名写 `new (): Emitter` 而不是 `constructor(): Emitter`，后者是 class 内部语法。

# 4. 扩展第三方模块与全局

## 4.1 模块增强（module augmentation）

```typescript
// augment.d.ts —— 给已有模块打补丁（注意：必须 import，文件才是模块）
import "geometry-lib";                              // 让本文件与目标模块建立关联

declare module "geometry-lib" {
  interface Point {
    distanceTo(other: Point): number;               // 给第三方接口追加方法
  }
  export function midpoint(a: Point, b: Point): Point;   // ⚠ 报错：不能新增顶层导出
}

// 使用处：原有 API 与增强 API 混用，类型完整
import { Point } from "geometry-lib";
declare const p: Point;
p.distanceTo(p);                                    // ✓ 增强生效
```

★ **关键认知**：模块增强把声明**合并进目标模块的原有声明**（效果等同第 14 篇的声明合并），但有两条硬限制：⚠ 只能给**已有声明**打补丁（新增顶层导出报错）；⚠ **默认导出无法增强**（`default` 是保留字），只能增强具名导出。运行时你仍需自己把方法挂到原型上——类型增强不产生实现。

## 4.2 全局增强（global augmentation）

```typescript
// augment-global.d.ts
export {};                                          // 先成为模块，declare global 才合法

declare global {
  interface String {
    truncate(max: number): string;                  // 所有字符串获得新方法的类型
  }
  namespace NodeJS {                                // 常见：给宿主全局对象补字段
    interface ProcessEnv {
      MY_APP_KEY: string;                           // 环境变量的类型收窄
    }
  }
}
```

★ **关键认知**：全局增强与模块增强同源——都是"把新声明合并进既有符号"，区别只在目标是全局作用域还是某个模块。⚠ 同样不产出运行时代码：`String.prototype.truncate` 必须自己实现。第 14 篇 6.1 的 Array 扩展例子与此完全同构。

# 5. @types 生态与发布类型包

## 5.1 DefinitelyTyped 与 types 字段

```jsonc
// 装别人写好的类型（绝大多数无类型库都有社区维护）
// npm install -D @types/lodash          // @types/<包名> 约定俗成

// 自己的包在 package.json 里声明类型入口
{
  "name": "my-lib",
  "version": "1.0.0",
  "types": "./dist/index.d.ts",         // 类型入口：解析器优先读这里
  "main": "./dist/index.js"             // 运行时入口
}
```

★ **关键认知**：`@types/*` 由 DefinitelyTyped 仓库集中维护，装包即获得类型——这是 TS 生态规模化的关键设计（与 Go 的"类型就在源码里"、Java 的"签名编进字节码"形成三种路线）。⚠ `types` 字段指向的文件不存在时 TS 不会报错于构建，而是悄悄回退到 `main` 附近的 `index.d.ts` 查找，最后才报 TS7016（隐式 any）。

## 5.2 typesVersions：按 TS 版本分流类型

```jsonc
// package.json —— 给不同 TS 版本的用户提供不同声明
{
  "types": "index.d.ts",                // 默认入口（最老版本兼容）
  "typesVersions": {
    ">=4.5": { "*": ["ts4.5/*"] },      // TS ≥ 4.5 的用户走 ts4.5/ 目录
    "*": ["ts4.1/*"]                    // 其余用户走 ts4.1/ 目录
  }
}
```

★ **关键认知**：`typesVersions` 解决"新 TS 语法写的类型声明老 TS 读不懂"的兼容矩阵问题，匹配按**声明顺序**取第一个命中的范围。⚠ 它只影响类型解析，运行时代码完全不受影响——又一处"类型世界与运行时世界平行"的例子。

## 5.3 发布自己的类型包：可照抄命令清单

```powershell
# ===== 1. npm 切国内镜像（安装提速） =====
npm config set registry https://registry.npmmirror.com
npm config get registry                  # 确认输出 https://registry.npmmirror.com/

# ===== 2. 初始化类型包（@types/xxx 约定命名） =====
npm init -y
npm install -D typescript

# ===== 3. 写 index.d.ts 后做一次类型自检 =====
npx tsc --noEmit                         # 声明文件自身也要过检查

# ===== 4. 只发布声明：package.json 的 types 指向 index.d.ts，files 白名单带上它 =====
npm pack --dry-run                       # 预览将发布的文件清单，确认 index.d.ts 在内

# ===== 5. 发布（⚠ 目标 registry 是官方源，镜像只加速安装） =====
npm config get registry                  # 发布前确认；若指向镜像需切回官方
npm publish --access public              # scoped 包必须加 --access public
```

★ **关键认知**：类型包与普通包的发布流程完全一致——npm 不区分二者，`types` 字段是唯一的类型入口约定。⚠ 别忘了 `files` 白名单与 `.npmignore` 会过滤产物，最常见的翻车是 `npm publish` 后消费者拿到的包里没有 `index.d.ts`，表现为全库隐式 any。`npm pack --dry-run` 是发布前最便宜的保险。

# 6. allowJs / checkJs 与 JSDoc：渐进式加类型

## 6.1 JSDoc 类型注释

```typescript
// math.js —— 注意这是 .js 文件，用 JSDoc 注释描述类型（注释语法即类型语法）
/**
 * 两数相加
 * @param {number} a   加数
 * @param {number} b   加数
 * @returns {number}   两数之和
 */
export function add(a, b) {
  return a + b;                    // checkJs 开启后，此处类型错误同样被揪出
}

/** @type {import("./types").User} */     // 用 JSDoc 引用 .d.ts 里的类型
const u = { id: 1, name: "Alice" };       // ✓ 获得类型检查与补全
```

```jsonc
// tsconfig.json —— 渐进式开关组合
{
  "compilerOptions": {
    "allowJs": true,        // 允许 .js 参与编译（否则直接报错）
    "checkJs": true,        // 对 .js 也做类型检查（不开则只转译不检查）
    "declaration": true,    // 可从带 JSDoc 的 .js 反推出 .d.ts
    "strict": true          // 检查强度与 .ts 一致
  }
}
```

★ **关键认知**：JSDoc 的类型注释会被 tsc 解析为**与 TS 注解等价的类型信息**——`@param {number}` 就是 `a: number`。这一机制是"零迁移成本给存量 JS 上保险"的关键：不改扩展名、不引入构建改动，先获得检查。⚠ JSDoc 与 TS 注解混用时以 `.ts` 文件的注解为准；`@ts-check` 注释可对单个 `.js` 文件临时开启检查（与 `checkJs` 是文件级/项目级关系）。

## 6.2 渐进式加类型 vs 迁移到 TS

| 维度 | allowJs + checkJs 渐进式 | 迁移为 .ts |
|------|-------------------------|-----------|
| 改动成本 | 低（只加 JSDoc 注释） | 高（改扩展名 + 修全部报错） |
| 类型表达力 | 受 JSDoc 语法限制 | 完整（泛型、条件类型等） |
| 构建链路 | 不变 | 需接入转译/声明产出 |
| 长期可维护性 | 注释与代码易脱节 | 类型是内建语法 |
| 适合场景 | 存量大 JS 库、只想上保险 | 新代码、核心模块 |

★ **关键认知**：两条路线的本质区别是**类型信息住在注释里还是语法里**——注释可被随意打破而不报错（`@param {number}` 写错 tsc 未必拦得住参数名），语法注解则是编译器强制校验。工程上的成熟做法是"新代码一律 `.ts`，存量 `.js` 用 `checkJs` 守住底线，逐模块迁移"。⚠ `checkJs` 对无注释的 JS 只能推断出最宽的类型，别指望它凭空变出精确类型。
