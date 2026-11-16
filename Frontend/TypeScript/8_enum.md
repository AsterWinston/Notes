枚举是 TS 少有的**会产生运行时代码**的特性（其他类型特性都被擦除）。本篇讲清数字枚举的反向映射、const enum 的内联机制与陷阱，并给出"该不该用枚举"的现代答案——`as const` + 联合字面量在多数场景是更优解。

# 1. 数字枚举

## 1.1 声明与自增

```typescript
// 不给值的成员从 0 开始自增
enum Direction {
  Up,        // 0
  Down,      // 1
  Left,      // 2
  Right,     // 3
}

// 显式值：从该成员起重新自增
enum HttpStatus {
  OK = 200,
  Created,       // 201（从上一个显式值继续 +1）
  Moved = 301,
  Found,         // 302
}

const d: Direction = Direction.Up;    // 类型注解可用枚举名
console.log(d);                       // 0：运行时就是数字
```

★ **关键认知**：数字枚举的成员在运行时就是**普通数字**，赋值时做的是同义词检查（`const d: Direction = 0` 在非枚举字面量场景是允许的——`0` 能赋给 `Direction` 类型，这是数字枚举的**松散类型**特性）。⚠ 这意味着 `const d: Direction = 999` 编译通过（任何 number 都能赋给数字枚举类型），类型安全性远低于字符串枚举与联合字面量，这是数字枚举被逐渐弃用的核心原因。

## 1.2 反向映射（数字枚举独有）

```typescript
enum Color {
  Red = 1,
  Green = 2,
}

// 编译产出的 Color 对象是双向的：
// { 1: "Red", 2: "Green", Red: 1, Green: 2 }
console.log(Color[1]);       // "Red"：数字 → 名称（反向映射）
console.log(Color.Red);      // 1：名称 → 数字（正向）
```

★ **关键认知**：数字枚举的编译产物是一个**双向映射对象**（IIFE 构造），正向 `Color.Red → 1` 之外还有反向 `Color[1] → "Red"`。⚠ 反向映射带来三个坑：① `for (const k in Color)` 会遍历出两种键（数字键和字符串键），必须过滤 `isNaN(Number(k))`；② 对象的体积翻倍；③ **字符串枚举没有反向映射**——`Color["Red"]` 只有一个方向，混用两种枚举写遍历代码时极易踩雷。

```typescript
// ===== 数字枚举的编译产物（target: ES2015 简化示意） =====
var Color;
(function (Color) {
  Color[Color["Red"] = 1] = "Red";     // 一行同时建立正向与反向映射
  Color[Color["Green"] = 2] = "Green";
})(Color || (Color = {}));
```

★ **关键认知**：`Color[Color["Red"] = 1] = "Red"` 这个双赋值技巧是数字枚举反向映射的实现根源：先算 `Color["Red"] = 1`（返回 1），再以 1 为键写入 `"Red"`。`(Color || (Color = {}))` 的 IIFE 写法让多个文件的同名枚举可以**增量合并**（枚举声明合并），这也是 TS 枚举支持跨文件"打开"扩展的机制。

# 2. 字符串枚举与异构枚举

## 2.1 字符串枚举

```typescript
enum Theme {
  Light = "LIGHT",       // 每个成员必须显式赋字符串值
  Dark = "DARK",         // 字符串枚举不自增，没有"隐式值"概念
}

const t: Theme = Theme.Dark;   // 运行时值就是 "DARK"
// const t2: Theme = "DARK";  // ✗ TS2322: 字符串字面量不能直接赋给枚举类型
```

★ **关键认知**：字符串枚举是**封闭的名义集合**——即使运行时值与某个成员的字符串相同，也不能直接把裸字符串赋给枚举类型（这是它与联合字面量的最大区别，双向都断开）。它没有反向映射，编译产物就是普通的 `{ Light: "LIGHT", Dark: "DARK" }` 对象。**类型擦除**之后枚举对象依然存在（因为它是值），但"Theme"这个类型名消失。

## 2.2 异构枚举（不推荐）

```typescript
// 数字与字符串混在一个枚举里
enum Mixed {
  No = 0,            // 数字成员（有反向映射）
  Yes = "YES",       // 字符串成员（无反向映射）
}
```

★ **关键认知**：异构枚举在实现上是"数字枚举对象 + 字符串枚举对象"的拼接，反向映射只存在于数字成员一侧，遍历逻辑必须分别处理两种成员，心智负担高。⚠ 官方文档与社区共识都建议**避免异构枚举**：它带来的表达力收益（一个枚举里既有编号又有标签）远小于维护成本，需要"两套值"时通常意味着该拆成两个类型。

# 3. const enum 与内联

## 3.1 const enum 的机制

```typescript
// const enum：编译期完全内联，不产生运行时对象
const enum Level {
  Debug = 0,
  Info = 1,
}

function log(level: Level): void {
  if (level <= Level.Info) { ... }   // Level.Info 在编译后变成字面量 1
}

log(Level.Debug);   // 编译后 ≡ log(0)：引用被替换为值
```

★ **关键认知**：普通枚举产生一个运行时对象（每次访问 `Level.Info` 是一次属性查找），`const enum` 则在编译期把引用**直接内联为字面量**，产物里没有 `Level` 对象——零运行时开销、零包体积。这在性能敏感的热路径或体积敏感的库代码里有实际价值。

## 3.2 ⚠ const enum 与 isolatedModules 的冲突

```typescript
// tsconfig: { "isolatedModules": true } 时——
const enum Level { Debug, Info }
// ✗ TS2748: 无法在 isolatedModules 模式下访问 const enum，
//   除非使用 "const enum" 的 preserveConstEnums 或内联语义受限
```

★ **关键认知**：`isolatedModules`（esbuild/swc/Babel 等**单文件转译工具**的前提）要求每个文件能独立转译，而 `const enum` 的内联需要**跨文件的类型信息**（要知道成员的值才能内联），这与单文件转译根本冲突。现代工程（Vite、swc、esbuild）默认开启 `isolatedModules`，因此 **`const enum` 在现代 TS 项目中基本被判了死刑**。替代方案：普通枚举、`as const` 对象，或 TS 5.0+ 的 `erasableSyntaxOnly` 思路（只用可擦除语法）。

# 4. 枚举的类型语义

## 4.1 枚举成员类型：每个成员是独立子类型

```typescript
enum Shape {
  Circle = "CIRCLE",
  Square = "SQUARE",
}

// 每个成员有自己的"字面量类型"，是枚举类型的子类型
let s: Shape.Circle = Shape.Circle;   // Shape.Circle 是独立类型
let anyShape: Shape = s;              // 子类型 → 枚举类型，OK
// let s2: Shape.Circle = Shape.Square; // ✗ 不同成员类型互不兼容
```

★ **关键认知**：TS 为每个枚举成员推断出**成员类型**（如 `Shape.Circle`），它是枚举类型 `Shape` 的子类型。这与字符串字面量类型的子类型化同理，但枚举成员类型是**名义**的——`"CIRCLE"` 裸字符串不等于 `Shape.Circle`。成员类型在可辨识联合（第 5 篇的模式）之外提供了一种"枚举风格"的判别类型写法。

## 4.2 联合枚举类型与穷尽检查

```typescript
enum Event {
  Click = "CLICK",
  Hover = "HOVER",
}

// 枚举类型本质是所有成员类型的联合：Event ≡ Event.Click | Event.Hover
function handle(e: Event): string {
  switch (e) {
    case Event.Click: return "clicked";
    case Event.Hover: return "hovered";
    // 若删掉一个 case：
    // const _exhaustive: never = e;  // ✗ 报错：Event.Hover 不能赋给 never
  }
  return "";
}
```

★ **关键认知**：联合枚举类型天然支持**穷尽检查**——配合 `never` 断言（第 5 篇的 `assertNever` 模式），遗漏分支在编译期报错。⚠ 但枚举的判别靠"值比较"而非**可辨识联合**的判别属性，switch 中的 `e` 不会自动把类型收窄到成员类型（数字枚举尤其模糊），在大型状态机里**可辨识联合 + 字面量类型**的收窄体验更好。

## 4.3 switch 收窄与穷尽检查的落地

```typescript
enum State {
  Idle = "IDLE",
  Run = "RUN",
}

function tick(s: State): string {
  switch (s) {
    case State.Idle:
      return s;                     // 此处 s 被收窄为 State.Idle
    case State.Run:
      return s;                     // 此处 s 被收窄为 State.Run
    default:
      const unreachable: never = s; // 穷尽检查：所有成员都处理完，s 应为 never
      return unreachable;
  }
}
```

★ **关键认知**：枚举成员参与 switch 收窄时，`s` 会按 case 匹配到的**成员类型**逐分支收窄——机制与可辨识联合一致，只是判别依据从"tag 属性"换成了"值属于哪个成员"。`default: never` 断言是**穷尽检查**的标准落地：将来给 `State` 新增成员，default 分支的 `never` 赋值立刻报错，逼迫你处理新分支。⚠ 数字枚举的收窄体验较差（任意 number 都可赋给枚举类型），switch 中残留的类型可能比预期宽，这也是优先选字符串枚举/联合字面量的理由之一。

# 5. 现代替代：as const + 联合字面量

## 5.1 基本写法

```typescript
// as const 对象：值与类型都在编译期确定，无枚举运行时对象
const Direction = {
  Up: "UP",
  Down: "DOWN",
} as const;

// typeof Direction["Up"] 提取字面量类型，组成联合
type Direction = (typeof Direction)[keyof typeof Direction];   // "UP" | "DOWN"

const d: Direction = "UP";       // 直接用字符串字面量，零转换
console.log(Direction.Up);       // 运行时仍有常量对象可查
```

★ **关键认知**：`as const` 把对象的每个属性收窄为**只读的字面量类型**，再用 `typeof` + 索引访问类型提取出联合。这套组合拳同时提供了"值的运行时载体"（对象）与"类型的封闭联合"（`"UP" | "DOWN"`），且完全由**可擦除语法**构成——类型擦除后剩下一个冻结的普通对象，无任何编译器私有语义。

## 5.2 完整对比表

| 维度 | `enum` | `as const` + 联合字面量 |
|------|--------|------------------------|
| 运行时产物 | 枚举对象（IIFE，数字枚举含反向映射） | 普通对象（可 `Object.freeze`） |
| 类型安全性 | 数字枚举可被任意 number 赋值 ⚠ | 字面量联合，严格封闭 |
| 裸字符串赋值 | 字符串枚举拒绝，需转换 | 直接可赋（就是字符串） |
| tree-shaking | ⚠ IIFE 难以摇树，常整体保留 | 纯对象引用，可被摇掉未用属性 |
| `isolatedModules` | ⚠ const enum 冲突 | 完全兼容 |
| 跨语言互操作 | TS 专属语法，其他工具链需插件 | 就是普通 JS，任何语言可消费 |
| 反向查找 | 数字枚举内置 `Color[1]` | 需要自己写 `invert()` |
| 声明合并 | 支持跨文件扩展 | 不支持（普通对象） |
| 阅读体验 | `Theme.Dark` 自带命名空间 | `Direction.Up` 同样有命名空间 |

```typescript
// 需要反向查找时，as const 方案补一个小工具即可
function invert<T extends Record<string, string>>(obj: T) {
  // Object.entries 反转键值，值变键、键变值
  return Object.fromEntries(
    Object.entries(obj).map(([k, v]) => [v, k])
  ) as { [K in keyof T as T[K]]: K };   // 映射类型 + as 重映射（第 11 篇）
}

const Rev = invert(Direction);   // { UP: "Up", DOWN: "Down" }
```

★ **关键认知**：实践建议可以收敛成一句话——**新代码默认用 `as const` + 联合字面量，只在两种情况下用枚举**：① 需要枚举的声明合并/跨文件打开扩展；② 明确需要数字编码且接受其松散类型（如协议里的线格式就是数字）。与 Java 的 `enum`（带实例方法、单例语义、EnumSet 等运行时能力）相比，TS 的枚举只是"带命名空间的常量对象"，把 Java 的使用直觉直接搬过来会高估它的能力。

```plaintext
枚举在编译流水线中的位置（对照第 1 篇的流水线图）：

  enum Theme { Light = "LIGHT" }
        │
        ▼
  Checker：创建名义类型 Theme 与成员类型 Theme.Light
        │
        ▼
  Transformer：生成运行时对象 var Theme; (function(Theme){...})(Theme)
        │
        ▼
  产物：类型 Theme 消失，对象 Theme 保留（类型擦除 + 值保留）

  as const 的对照：
  const Theme = { Light: "LIGHT" } as const
        │
        ▼
  Checker：推断 { readonly Light: "LIGHT" }，类型全擦除
        │
        ▼
  产物：const Theme = { Light: "LIGHT" }（纯 JS，无编译器魔法）
```

★ **关键认知**：这张图揭示了选择的本质——`enum` 是"**值与类型都由编译器托管**"的方案，产物里携带编译器生成的 IIFE；`as const` 是"**值是普通 JS、类型是编译期推断**"的方案。在类型擦除哲学（TS 类型零运行时开销）的语境下，`as const` 更符合 TS 的设计本意，这也是它在现代 TS 社区（以及 deno、svelte 等生态）中逐步取代枚举的根本原因。⚠ 迁移存量代码时不必强行替换：枚举不是"错误"，只是"不再是最优解"。

## 5.3 选型与迁移速查

| 你的需求 | 推荐方案 | 理由 |
|---------|---------|------|
| 一组相关常量 + 命名空间访问 | `as const` 对象 | 有运行时载体，可 tree-shaking |
| 常量值就是字符串，对外传输 | `as const` + 字面量联合 | 裸字符串直接可用，无转换 |
| 协议/数据库规定数字编码 | 数字枚举 | 接受其松散类型，用注释警示 |
| 需要 `Color[1]` 反向查找 | 数字枚举或自写 `invert` | 权衡便利与体积 |
| 需要跨文件扩展同一组常量 | 枚举的声明合并 | `as const` 做不到 |
| 追求最小运行时开销 | `const enum`（旧工具链）/ `as const`（现代） | 视 isolatedModules 而定 |

★ **关键认知**：这张表的决策轴其实只有两根——**"要不要运行时载体"**与**"要不要名义封闭"**。`as const` + 联合字面量回答"要载体、不封闭"（字符串即值），字符串枚举回答"要载体、要封闭"（裸字符串不可赋入），数字枚举额外提供编码与反查但付出松散类型与体积的代价。⚠ 唯一建议立即动手替换的组合是 `const enum` + 现代构建工具链（isolatedModules 冲突会让构建直接失败），其余场景按表选型即可。
