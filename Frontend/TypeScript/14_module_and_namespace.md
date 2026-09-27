TypeScript 的模块系统同时承接 ES 模块与 CommonJS 两套体系，还额外提供**声明合并**与 `declare global` 这些 JS 里不存在的组织机制。本篇讲清 TS 中两类模块写法的差异、模块解析策略与路径别名的运行时陷阱，并把声明合并的完整规则一次性讲透。

# 1. ES 模块与 CommonJS：TS 中的两种写法

## 1.1 导入导出对照

```typescript
// ===== ES 模块（ESM）：主流写法 =====
import { formatDate } from "./utils";        // 具名导入一个值
import type { User } from "./types";         // 只导入类型（编译后整行消失）
import config from "./config";               // 默认导入
export { formatDate };                       // 具名导出
export type { User };                        // 只导出类型
export default formatDate;                   // 默认导出

// ===== CommonJS（CJS）：TS 专用的对应语法 =====
import fs = require("fs");                   // 等价 JS 里的 require("fs")
export = formatDate;                         // 等价 module.exports = formatDate
```

★ **关键认知**：`import x = require()` 与 `export =` 是 TS 专门为 CJS 设计的语法，存在原因只有一个：`require` 在 JS 里是**运行时函数调用**，普通 `import` 语法表达不了"整个模块导出一个值（`module.exports = fn`）"的 CJS 语义。⚠ 写了 `export =` 的模块不能被 ESM 具名导入，消费端只能 `import x = require(...)`，或者开启 `esModuleInterop` 后用 `import x from` 接收。

## 1.2 import type / export type 与 isolatedModules

```typescript
// types.ts
export interface User { id: number; name: string }   // 接口：纯类型
export type Role = "admin" | "user";                 // 类型别名：纯类型

// ===== 错误示范：把类型混在值导出里 =====
// export { User, Role };    // ⚠ isolatedModules 开启时报错：类型无法单独转译

// ===== 正确写法：类型导出显式标注 =====
export type { User, Role };                 // ✓ 类型导出，编译后不产生代码

// main.ts
import { fetchUser } from "./api";          // 值导入：保留到运行时
import type { User } from "./api";          // 类型导入：编译后整行删除
import { type Role, loadRole } from "./api"; // 内联 type 修饰：Role 擦除、loadRole 保留
```

★ **关键认知**：`isolatedModules` 要求**每个文件能被单独转译**——esbuild、swc 这类工具只看单文件，无法判断 `User` 是类型还是值，`export { User }` 会让它们无从下手。tsc 做全程序分析所以平时不报错，打开 `isolatedModules` 后才强制你写 `export type`。⚠ 无论开关如何，`import type` 导入的名字**只能当类型用**，当值用直接报 TS1361。类型导入被完全**类型擦除**，运行时零成本。

## 1.3 动态 import() 的类型

```typescript
// math.ts
export function add(a: number, b: number): number { return a + b; }   // 具名导出
export const VERSION = "1.0.0";                                      // 具名导出

// main.ts
async function main() {
  const math = await import("./math");        // 类型：Promise<typeof import("./math")>
  math.add(1, 2);                             // ✓ 命名空间对象，类型完整
  const v: string = math.VERSION;             // ✓ 导出值的类型精确

  const { add } = await import("./math");     // 解构写法，推断同样精确
}

// 类型层面引用模块：typeof import("...")
type MathModule = typeof import("./math");    // 模块命名空间类型
type AddFn = MathModule["add"];               // (a: number, b: number) => number
```

★ **关键认知**：`import()` 的返回类型是 `Promise<typeof import("...")>`——TS 把模块也当作一个**结构化类型**（导出集合的形状）参与推断，所以动态导入后依然有完整补全。⚠ 但 `module: "commonjs"` 时 tsc 会把 `import()` 降级成 `Promise.resolve().then(() => require(...))`，缓存、循环依赖、异步边界的行为与原生 ESM 不同；要保留原生动态导入，用 `module: "node16"` / `"esnext"` 或交给不降级的工具链。

## 1.4 ESM 与 CJS 互操作的类型开关

```typescript
// 被导入方是老 CJS 库（module.exports = fn）
import fn from "cjs-lib";                     // ✓ esModuleInterop 开启时合法
// import { fn } from "cjs-lib";              // ✗ CJS 没有具名导出，TS 与 Node 运行时都拿不到
```

| 选项 | 作用 | 取舍建议 |
|------|------|---------|
| `esModuleInterop` | 生成 `__importDefault` / `__importStar` 辅助代码，允许默认导入 CJS | 一律开启（默认就是 true） |
| `allowSyntheticDefaultImports` | 仅**类型层**允许默认导入 CJS，不生成辅助代码 | 借 `esModuleInterop` 自动生效，一般不单独开 |
| `forceConsistentCasingInFileNames` | 强制 import 与磁盘文件大小写一致 | 开启，防 Windows/Linux 大小写不一致翻车 |

★ **关键认知**：互操作的本质是**两套模块语义的翻译**——ESM 的默认导出与 CJS 的 `module.exports` 并不天然等价，`esModuleInterop` 在产出里插入包装函数来弥合差异。这与 Java/Go 的"单一模块语义"完全不同：Go 的 import 永远指向包路径，不存在"同一个包两种加载格式"的互操作问题。

# 2. 模块解析策略

## 2.1 解析流水线

```
   import { x } from "@app/utils"
            │
            ▼
   ┌────────────────────────────┐
   │ 1. paths / baseUrl 别名命中？│──是──▶ 映射为项目内路径
   └────────────────────────────┘
            │ 否
            ▼
   ┌────────────────────────────┐
   │ 2. 相对路径 ./ ../ / 绝对？  │──是──▶ 试 ./utils.ts / .d.ts / .tsx / index.*
   └────────────────────────────┘
            │ 否
            ▼
   ┌────────────────────────────┐
   │ 3. 视为包名，逐级向上找      │
   │    node_modules/@app/utils  │──▶ package.json "types"/"exports" → index.d.ts
   └────────────────────────────┘
            │ 全部失败
            ▼
      报错 TS2307: Cannot find module
```

★ **关键认知**：模块解析就是 tsc 把 `from "..."` 这个**字符串**对应到磁盘文件的过程，规则由 `moduleResolution` 决定。⚠ 解析失败报 TS2307，是配置类报错里最高频的一条；排查顺序固定为：拼写 → 扩展名要求 → `paths` 映射 → `node_modules` 是否真装了包 → 解析策略是否匹配运行环境。

## 2.2 node / node16 / bundler 行为差异

| 维度 | `node`（node10） | `node16` / `nodenext` | `bundler` |
|------|------------------|----------------------|-----------|
| 可省略扩展名 `./a` | ✓（依次试 `.ts/.tsx/.d.ts` 与 `index`） | ✗（ESM 文件必须写 `./a.js`） | ✓ |
| 目录 index 解析 `./dir` | ✓（找 `dir/index.ts`） | ✗（须写 `./dir/index.js`） | ✓ |
| `package.json` `exports` 字段 | ❌ 完全忽略 | ✓ 严格遵守 | ✓ 严格遵守 |
| `type: "module"` 影响 | 忽略 | 决定 `.js` 的模块格式 | 忽略 |
| 默认配套 `module` | `commonjs` | `node16` / `nodenext` | `esnext` / `preserve` |
| 典型场景 | 老 CJS 项目 | Node 原生 ESM 运行 | 打包器构建的项目 |

★ **关键认知**：node16 要求写 `./a.js` 而不是 `./a.ts`，不是刁难——ESM 规范要求 import 说明符是可寻址的，TS 按**运行时的真实文件名**解析，这样产出后无需改写路径。`bundler` 是 node16 的放宽版：认 `exports` 字段但允许省略扩展名，是现代前端项目的默认选择。⚠ `moduleResolution` 与 `module` 有合法组合约束（如 `bundler` 需搭配 `module: "esnext"` 或 `"preserve"`），配错直接报 TS5095 一类的配置错误。

# 3. 路径别名：baseUrl 与 paths

## 3.1 配置与使用

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",                       // 别名解析的基准目录
    "paths": {
      "@utils/*": ["src/utils/*"],        // @utils/date → src/utils/date.ts
      "@/*": ["src/*"]                    // 通配符 * 必须在映射两端同时出现
    }
  }
}
```

```typescript
// 引用处：告别五层 ../
import { formatDate } from "@utils/date";        // 等价 import from "../../utils/date"
import type { Config } from "@/types/config";    // 类型导入同样走别名

// 编译产出 dist/index.js 后，这一行依然是 require("@utils/date")！
```

★ **关键认知**：⚠ **`paths` 只影响类型解析，tsc 绝不改写产出 JS 里的模块路径**。上面的例子编译后 Node 会直接抛 `Cannot find module '@utils/date'`。运行时必须配套方案：打包器的 alias 配置，或 Node 原生的 `imports` 字段（`#utils/*`，node16 解析原生支持），或构建后处理改写路径。这是"编译全绿、一跑就崩"最常见的成因。

## 3.2 各语言的模块映射机制对比

| 语言 | 路径映射机制 | 运行时由谁解析 |
|------|-------------|---------------|
| TypeScript + 打包器 | `paths` 别名 + 打包器 alias | 打包器在构建期替换 |
| Java | package 声明 + classpath | JVM 类加载器 |
| Go | 导入路径即包的完整路径 | go toolchain 编译期解析 |
| Python | `sys.path` + 包相对导入 | 解释器运行时解析 |

★ **关键认知**：TS 的别名是**编译期幻觉**，Go 的导入路径是**文件系统事实**——前者需要运行时工具配合，后者天然一致。理解这一点就能理解为什么 `paths` 的坑反复踩：类型世界与运行时世界在这里是脱节的。

# 4. 声明合并

## 4.1 interface 互并

```typescript
// 同名 interface 自动合并：成员取并集
interface Box {
  height: number;                // 非函数成员
  width: number;
}
interface Box {
  scale: number;                 // 合并后 Box 共 3 个属性
  clone(): Box;                  // 函数成员
}
interface Box {
  clone(depth: number): Box;     // 同名函数成员 → 合并为重载
}

const b: Box = { height: 1, width: 2, scale: 3, clone: () => b };
b.clone();                       // 匹配无参重载
b.clone(2);                      // 匹配有参重载

// interface Box { height: string; }   // ✗ TS2717：同名属性必须类型完全一致
// type Box = { height: number };      // ✗ TS2300：type 别名不参与合并
```

★ **关键认知**：**声明合并（declaration merging）**指同名声明被合成一个定义。interface 合并规则：非函数成员必须唯一，同名则必须**类型完全相同**；同名函数成员合并成重载链，**后声明的接口重载组排在前面、优先匹配**（例外：参数是单个字符串字面量的"特化签名"会被提升到重载顶部）。⚠ `type` 别名同名直接报重复标识符——这正是 `type` 与 `interface` 的核心取舍点：interface 可被跨文件扩展（利用合并），type 不能。

## 4.2 namespace 与 class / function / enum 的合并

```
   class Album ──────────┐   （值空间：构造器 + 实例类型）
                         │  声明合并
   namespace Album ──────┤   （值空间 + 类型空间）
                         ▼
                ┌────────────────────────┐
                │ Album（合并后的符号）    │
                ├────────────────────────┤
                │ new Album()            │ ← class 贡献
                │ Album.AlbumLabel（类）  │ ← namespace 贡献（静态值）
                │ Album.Status（类型）    │ ← namespace 贡献（类型）
                └────────────────────────┘
```

```typescript
// class + namespace：给类补充"静态侧"成员与内嵌类型
class Album {
  label: Album.AlbumLabel;        // 合并后可引用命名空间内的类型
}
namespace Album {
  export class AlbumLabel {}      // 必须 export，否则合并后不可见
  export type Status = "draft" | "published";
}
const label = new Album.AlbumLabel();     // 命名空间成员成为 Album 的静态属性
const s: Album.Status = "draft";          // 类型挂在 Album 命名空间下

// function + namespace：给函数挂静态属性（JS 里 fn.xxx = ... 的类型化）
function buildLabel(name: string): string {
  return buildLabel.prefix + name + buildLabel.suffix;
}
namespace buildLabel {
  export let suffix = "";                 // 函数的"静态字段"
  export let prefix = "Hello, ";
}
buildLabel("TS");                         // 普通调用
buildLabel.prefix = "Hi, ";               // 访问静态属性

// enum + namespace：给枚举扩展静态成员
enum Color { Red = 1, Green = 2, Blue = 4 }
namespace Color {
  export const Yellow = Color.Red | Color.Green;   // 组合常量
}
```

★ **关键认知**：合并发生在**符号表**层面——class/enum 既是值又是类型，namespace 也同时占据值空间与类型空间，同名声明在两个空间分别拼接。⚠ 硬性顺序规则：与 class/function/enum 合并时 **namespace 必须写在被合并声明之后**；namespace 内成员必须 `export`，非导出成员只在"自己那一段"可见，其他合并片段看不到它。

## 4.3 合并规则总表

| 组合 | 效果 | 限制 |
|------|------|------|
| interface + interface | 成员并集 / 函数变重载 | 同名属性须同类型 |
| namespace + namespace | 成员并集 | 非 `export` 成员互相不可见 |
| class + namespace | namespace 成员变静态成员 | namespace 在 class 之后；成员须 export |
| function + namespace | 函数获得静态属性 | namespace 在 function 之后 |
| enum + namespace | 枚举获得静态成员 | namespace 在 enum 之后 |
| class + class | ✗ 不允许（TS2300） | 用 mixin 模式模拟 |
| class + interface | 不合并（interface 约束实例类型） | — |

★ **关键认知**：⚠ 合并没有任何"运行时魔法"——namespace 合并编译后就是对象属性拼接（IIFE 或对象字面量），class 的静态成员就是构造器属性。TS 允许拆分声明，是为了兼容 JS 社区"给函数/类挂属性"的历史习惯；Java 那种"一个类必须完整定义在一个编译单元"的名义类型世界观里没有对应物。

# 5. namespace 的现代用法与衰落

## 5.1 为什么 ESM 时代不再用 namespace 组织代码

| 维度 | `namespace` | ES 模块 |
|------|------------|---------|
| 组织单位 | 同名符号跨文件合并 | 一个文件一个模块 |
| 依赖表达 | 隐式（依赖加载顺序） | 显式 `import` |
| 静态分析 | 差（合并依赖全局符号表） | 好（import 依赖图清晰） |
| tree shaking | 难（对象整体难拆） | 天然支持 |
| 产出形态 | IIFE / 对象字面量（非标准） | 标准 ESM / CJS |

★ **关键认知**：namespace 想解决的"命名空间管理、声明拆分"问题，ESM 用"**文件即模块**"解决得更干净：依赖显式可查、可静态分析、可摇树。TS 官方立场是 ESM 项目一律用模块，namespace 只保留给类型层场景。

## 5.2 今天仍然需要 namespace 的两个场景

```typescript
// 场景1：声明文件里给"全局库"分组（老式全局脚本库的类型化）
declare namespace MyLib {
  function format(s: string): string;         // MyLib.format
  namespace utils {
    function trim(s: string): string;         // MyLib.utils.trim
  }
}

// 场景2：描述"带静态属性的函数"的类型（JS 常见习惯 fn.xxx）
declare function route(path: string): void;   // 可调用
declare namespace route {
  export const version: string;               // route.version 的类型
}
```

★ **关键认知**：这两个场景的共同点是**给"值"描述嵌套属性**，而不是组织源码——namespace 在纯类型层依然不可替代。⚠ 新业务代码里拿 namespace 组织模块属于历史包袱，一律改成 `import` / `export`；`declare namespace` 只应出现在**声明文件**（`.d.ts`）里。

# 6. declare global：全局类型增强实战

## 6.1 给全局作用域扩展类型

```typescript
// global.d.ts —— 关键：文件含 import/export 才是"模块"，declare global 才合法
export {};                                 // 把本文件变成模块（少了它直接报错）

declare global {
  // 1) 扩展全局接口：所有数组获得 last() 的类型（运行时必须自己实现！）
  interface Array<T> {
    last(): T | undefined;
  }

  // 2) 声明宿主注入的全局变量（运行时由启动脚本赋值）
  var appConfig: {
    version: string;
    debug: boolean;
  };
}

// 任意模块文件里直接使用，无需 import
const v: string = appConfig.version;        // ✓ 全局可见
const tail = [1, 2, 3].last();              // ✓ 类型合法，tail: number | undefined
```

★ **关键认知**：⚠ `declare global` 是**纯类型增强、零运行时效果**——`Array.prototype.last` 必须自己写实现并挂上去，否则调用即抛 `not a function`。这再次印证**类型擦除**的结论：声明在运行时不留痕迹。⚠ 另一个反直觉点：非模块文件（没有 import/export）本身就处于全局作用域，直接写声明即可，此时再写 `declare global` 反而报错。

## 6.2 三种"增强"的分工

| 目标 | 语法 | 典型用途 |
|------|------|---------|
| 全局作用域 | `declare global { ... }` | 扩展内建接口（Array/String 等） |
| 已有模块的导出 | `declare module "x" { ... }`（模块增强） | 给第三方库补方法 |
| 全新的无类型模块 | `declare module "x" { ... }`（ambient 声明） | 给纯 JS 库补类型（见第 15 篇） |

★ **关键认知**：模块增强有两条硬限制——⚠ 增强里**只能给已有声明打补丁**（不能新增顶层导出），⚠ **默认导出无法增强**（`default` 是保留字），只能增强具名导出。增强文件里写的模块名解析规则与 `import` 完全一致，这保证了"增强"与"使用"指向同一实体。
