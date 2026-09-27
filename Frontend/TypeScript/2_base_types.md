基础类型的注解语法一学就会，真正决定日常开发体验的是**类型推断**的边界与几个"特殊类型"的微妙语义。本篇讲清原始类型与复合类型的注解写法、推断与拓宽（widening）机制，并彻底厘清 any / unknown / never / void / object / {} 这六个"看着都像占位符"的类型有何本质区别。

# 1. 原始类型注解

## 1.1 八种原始类型

```typescript
// TS 的原始类型直接对应 JS 的原始值，注解写在冒号之后
let name: string = "Alice";         // 字符串
let age: number = 30;               // number 同时表示整数与浮点（对应 JS 的 Number）
let ok: boolean = true;             // 布尔
let big: bigint = 100n;             // 大整数，注意字面量后缀 n
let uid: symbol = Symbol("uid");    // 全局唯一标识符
let empty: null = null;             // null 是独立类型
let missing: undefined = undefined; // undefined 也是独立类型
```

★ **关键认知**：TS 把 JS 的数字**整体建模为单一的 `number` 类型**，不像 Java 拆成 byte/short/int/long/float/double——整数与浮点在类型层面毫无区别。需要任意精度大整数时用 `bigint`，⚠ `number` 与 `bigint` 之间**不能隐式混合运算**，`1 + 1n` 直接编译报错，必须先 `Number(big)` 或 `BigInt(num)` 显式转换。

与 Java / Go 的原始类型横向对比：

| TypeScript | Java | Go | 说明 |
|-----------|------|-----|------|
| `number` | `int` / `long` / `double`… | `int` / `float64`… | TS 不区分整数与浮点 |
| `bigint` | `BigInteger`（类） | `big.Int`（结构体） | TS 是原始类型，带 `n` 后缀字面量 |
| `string` | `String` | `string` | JS 字符串是 UTF-16 码元序列 |
| `boolean` | `boolean` | `bool` | — |
| `symbol` | 无对应 | 无对应 | 独一无二，常作对象的私有键 |
| `null` / `undefined` | `null` | 零值 `nil` | TS 两者是**互相独立的类型** |

## 1.2 数组与对象的注解写法

```typescript
// 数组的两种等价写法：T[] 语法糖与泛型 Array<T>
let nums: number[] = [1, 2, 3];
let names: Array<string> = ["a", "b"];      // 与 string[] 完全等价

// 元素类型复杂时 Array<T> 更清晰：函数数组的 T[] 写法需要括号
let cbs: Array<() => void> = [];
// 等价但更难读：let cbs: (() => void)[] = [];

// 对象类型字面量：直接描述"形状"，不需要先定义类
let user: { name: string; age: number } = { name: "Alice", age: 30 };
```

★ **关键认知**：TS 的对象类型是**结构化描述**而非类的附属品——`{ name: string }` 本身就是合法的完整类型。这延续了第 1 篇的**结构化类型系统**：任何满足该形状的对象都能赋值。⚠ 但**对象字面量**直接赋值时会触发**多余属性检查（freshness check）**：多打一个字段会被编译器拦下，而先存进变量再赋值则不拦——这是字面量特有的"新鲜度"行为，详见第 5 篇。

# 2. 类型推断

## 2.1 变量初始化推断

```typescript
// 声明时未写注解，TS 从初始化表达式反推类型
let count = 10;             // 推断为 number
let title = "hello";        // 推断为 string
let items = [1, 2];         // 推断为 number[]

count = "ten";              // ✗ 编译错误：不能把 string 赋给 number
```

★ **关键认知**：类型推断不是动态反射，而是**编译期的单向数据流**——初始化表达式一旦定型，变量就有了静态类型，后续所有赋值按此检查。这与 Go 的短变量声明 `:=` 神似（都从右侧推断），区别是 Go 推出的永远是具体静态类型，TS 在字面量场景还会做"**拓宽**"（见 2.4）。

## 2.2 最佳通用类型

```typescript
class Animal {}
class Dog extends Animal {}
class Cat extends Animal {}

// 情况1：候选里没有"大家共同的候选" → 推断为联合
const mixed = [new Dog(), new Cat()];             // (Dog | Cat)[]

// 情况2：某个候选是其余所有候选的父类型 → 选它（最佳通用类型）
const zoo = [new Dog(), new Cat(), new Animal()]; // Animal[]
```

★ **关键认知**：TS 从多个候选（数组元素、多分支表达式）推断类型时执行**最佳通用类型（best common type）**算法：优先选"其余候选都能赋值给它"的那个候选；选不出来就退化为**联合类型**。⚠ 空数组 `let arr = [];` 在 `noImplicitAny` 下依赖"隐式演化数组"机制——先 push 什么就推成什么，多处 push 不同类型会并成联合，容易埋雷，建议声明时显式标注 `number[]` 之类。

## 2.3 函数返回值推断

```typescript
// 返回值类型可完全省略，由所有 return 语句反推
function add(a: number, b: number) {
  return a + b;               // 推断返回类型为 number
}

// 没有任何 return 表达式 → 推断为 void
function log(msg: string) {
  console.log(msg);           // 推断返回类型为 void
}

// ⚠ 递归函数无法从"调用自身"推断，必须显式标注返回类型
function fact(n: number): number {
  return n <= 1 ? 1 : n * fact(n - 1);
}
```

★ **关键认知**：返回值推断是**从内向外的单遍推导**：编译器收集全部 `return` 表达式求最佳通用类型。递归函数因推导环路必须人工注解。实践上**导出的公共 API 建议显式写返回类型**——它既是文档，也能防止实现细节的改动悄悄改变对外契约：返回类型一变，全项目调用处立刻集体报错，这正是重构安全感的来源。

## 2.4 类型拓宽与字面量类型

```typescript
// const 保留字面量类型，let 发生"拓宽（widening）"
const direction = "up";     // 类型是 "up"（字面量类型）
let dir = "up";             // 类型是 string（拓宽了）

// 字面量类型流向"可变位置"时拓宽，流向"常量位置"时保持
let d1 = direction;         // string —— 拓宽
const d2 = direction;       // "up"  —— 保持

// 显式标注字面量类型可阻止拓宽，形成"取值约束"
let status: "on" | "off" = "on";
status = "maybe";           // ✗ 编译错误：只能赋 "on" 或 "off"
```

★ **关键认知**：**字面量类型（literal type）**是 `string` / `number` / `boolean` 的子类型——`"up"` 既是值也是类型。`let` 推断时子类型会**拓宽**回基类型（`"up"` → `string`），因为可变变量的用途通常不是永远等于某个字面量；`const` 与显式注解则保留字面量类型。⚠ 这就是 `const x = "up"` 的类型能当约束用、而 `let x = "up"` 不能的原因——`as const` 的意义正是**手动阻止拓宽**（见 4.2）。

```
  字面量类型 "up" ──拓宽（let / 可变位置）──▶ 基类型 string
        ▲                                       │
        └────收窄 narrowing（见第 3 篇）◀────────┘
```

# 3. 六个易混类型：any / unknown / never / void / object / {}

## 3.1 语义对比表

| 类型 | 本质 | 能赋给谁 | 谁能赋给它 | 使用前需收窄 |
|------|------|---------|-----------|-------------|
| `any` | 类型检查逃生舱 | 任意类型 | 任意类型 | 否（检查停摆） |
| `unknown` | 顶层类型，"未知值" | `unknown` / `any` | 任意类型 | **是** |
| `never` | 底类型，"永不出现值" | 任意类型 | 只有 `never` | — |
| `void` | "无返回值，别用结果" | `void` / `undefined` | `undefined` | — |
| `object` | 非原始类型 | `object` | 对象 / 数组 / 函数 | — |
| `{}` | 非 null/undefined 的任意值 | `{}` | 几乎一切（除 null/undefined） | — |

## 3.2 any 的污染传播 vs unknown 的收窄要求

```typescript
// ===== any：类型"污染"一路向下扩散 =====
let a: any = JSON.parse("{}");
const b = a.foo.bar();      // b 是 any —— 检查器完全停摆，属性名写错也不报
const c = b.baz;            // c 还是 any —— 从 a 开始全链路失守

// ===== unknown：必须先收窄才能使用 =====
let u: unknown = JSON.parse("{}");
// u.foo;                   // ✗ 编译错误：unknown 上不存在任何属性
if (typeof u === "string") {
  u.toUpperCase();          // 收窄为 string 后才允许使用
}
```

★ **关键认知**：`any` 的可怕之处不在"它什么都行"，而在**它会污染下游**——一旦沾上 any，从它推导出的一切表达式都变成 any，整个子图脱离类型检查，报错从这里开始"静音"。`unknown` 则是安全的"值垃圾桶"：什么都能塞进来，但**用之前必须收窄**。接收外部数据（解析结果、跨模块传值）应一律首选 `unknown`。⚠ `any` 还会悄悄改变重载解析与泛型推断结果，排查"推断莫名变成 any"时，先沿数据流找 any 的源头。

## 3.3 never 的底类型地位

```typescript
// 抛异常或死循环的函数返回 never
function fail(msg: string): never {
  throw new Error(msg);       // 永不正常返回
}

// never 能赋给任意类型（底类型可向上流向一切）
let s: string = fail("boom");

// 但任何值都不能赋给 never
// let n: never = 42;        // ✗ 编译错误
```

★ **关键认知**：`never` 是类型层级的**底（bottom）类型**——表示"这个位置永远不会产生值"。因此它可赋给任何类型（向上的兼容是无限的），而除了 never 自己没有任何类型能赋给它。无解的交叉 `string & number` 化简后也是 never。⚠ 正常会返回的函数声明 `: never` 会报错；`never` 的真正威力在**穷尽检查**，见第 3 篇。

## 3.4 void、undefined、object、{} 的微妙差别

```typescript
// void：函数"没有有意义的返回值"，调用方不应使用结果
function warn(msg: string): void {
  console.log(msg);
}
const r = warn("hi");       // r 的类型是 void —— 使用 r 本身就被禁止

// undefined 是真实的值类型；void 是"别用返回值"的语义标记
let u1: undefined = undefined;
let u2: void = undefined;   // 只有 undefined 能赋给 void

// object：只接受非原始值
let o1: object = { x: 1 };  // ✓
let o2: object = [1, 2];    // ✓ 数组也是对象
// let o3: object = "str";  // ✗ 编译错误：string 是原始类型

// {}：接受一切非 null/undefined，但几乎什么都用不了
let e1: {} = "str";         // ✓ 意外地合法！
// e1.toUpperCase();        // ✗ 编译错误：{} 上没有 toUpperCase
```

★ **关键认知**：`void` 与 `undefined` 的差别是**意图**而非能力——`undefined` 说"值就是 undefined"，`void` 说"请不要关心返回值"（正因如此，`() => void` 的回调位置可以放返回任意值的函数，规则见第 4 篇）。`object` 与 `{}` 是两类"对象"：`object` 排除原始类型，`{}` 排除 null/undefined 却**放行原始类型**。⚠ `{}` 是新手重灾区：它不是"空对象结构"，而是"没声明任何属性的近乎顶层类型"——赋什么都过、用什么都报错。Java 程序员可类比：`unknown` ≈ `Object`（用前必须强转），`any` ≈ 彻底放弃检查的 `Object`；但 TS 的收窄是编译期完成的，零运行时开销。

# 4. 类型断言

## 4.1 as 断言

```typescript
// 场景：TS 无法静态得知具体类型（如取联合的某个成员）
let value: string | number = "hi";
const len = (value as string).length;   // string 是联合成员，允许断言

// 断言只在"方向相关"的类型之间允许
// let bad = value as boolean;          // ✗ 编译错误：毫无关系的类型
```

★ **关键认知**：`as` 是**对编译器的单方面承诺**，不是运行时转换——`(value as string)` 编译后完全消失，运行时 `value` 还是原来的值。断言方向必须"说得通"（两类型互有子类型关系时才允许），以拦住 `number` 断成 `string` 这种低级错误。⚠ 断言错误没有任何编译或运行时提示，它只是把炸弹的引爆点从"看得见的报错"推迟到"更深的诡异行为"。

## 4.2 常量断言 as const

```typescript
// 没有 as const：元素被拓宽为 string
const routes = ["GET", "POST"];            // string[]
routes.push("DELETE");                     // ✓ 可变

// 有 as const：推断为只读元组，元素是字面量类型
const methods = ["GET", "POST"] as const;  // readonly ["GET", "POST"]
type Method = (typeof methods)[number];    // "GET" | "POST"
// methods.push("DELETE");                // ✗ 编译错误：只读
```

★ **关键认知**：`as const` 是对整个表达式的**深度常量断言**：属性变 `readonly`、数组变**只读元组**、所有值保留字面量类型不拓宽（对照 2.4 的拓宽机制理解）。它的经典用途是"**用数据定义类型**"——先声明常量表，再用 `typeof` + 索引类型导出字面量联合（`(typeof methods)[number]` 取出所有元素类型的联合）。⚠ 断言后不可修改，需要可变副本时用展开 `[...methods]` 重新推断。

## 4.3 非空断言 !

```typescript
interface User {
  name?: string;
}

// 场景：你确定此刻不是 null/undefined，但编译器无法证明
function show(u: User) {
  console.log(u.name!.toUpperCase());   // ! 告诉编译器"这里一定有值"
}
```

★ **关键认知**：`!` 是"**我保证非空**"的编译期承诺，⚠ **运行时不做任何检查**——如果 `u.name` 真是 undefined，它和裸写 `u.name.toUpperCase()` 一样抛 `TypeError`。优先用真值收窄（`if (u.name) { ... }`，见第 3 篇）替代 `!`；`!` 只用于类型确实无法表达的不变量，并在旁边注释写清"保证"的来源。

## 4.4 双重断言

```typescript
// 常规断言失败：两个类型毫无关系
declare const x: string;
// const y = x as number;             // ✗ 编译错误

// 双重断言：先断到逃生舱，再断到目标类型
const y = x as unknown as number;     // ✓ 编译通过，语义完全靠人肉担保
```

★ **关键认知**：双重断言是**逃生舱中的逃生舱**——它绕过 4.1 的"方向相关性"检查，本质是把类型系统关掉两秒。只应在修补类型声明不完善的第三方声明时偶尔使用，⚠ 每一处 `as unknown as` 都应记为一笔欠账：它的出现频率可以直接衡量代码库对类型系统的背离程度。

# 5. 类型查询与类型别名

## 5.1 typeof 类型查询

```typescript
const config = {
  host: "localhost",
  port: 8080,
};

// 类型位置上的 typeof：把"值的名字"变成"它的类型"
type Config = typeof config;            // { host: string; port: number }

// 经典组合：typeof + keyof 取出所有键的联合
type ConfigKey = keyof typeof config;   // "host" | "port"
```

★ **关键认知**：类型位置上的 `typeof` 是**编译期类型查询**，与运行时 `typeof` 运算符（返回 `"string"` 这类字符串）是**两个完全不同的机制**，只是恰好同名。⚠ 只能作用于"名字"（变量、属性访问），不能作用于任意表达式；查询到的是变量声明时的推断类型，之后重新赋值不会改变已提取出的类型别名。

## 5.2 type 类型别名

```typescript
// 给任意类型起名字：原始、联合、元组、函数……都可以
type ID = string | number;
type Pair = [string, number];
type Callback = (ok: boolean) => void;
type Predicate<T> = (value: T) => boolean;   // 别名也能带泛型参数
```

★ **关键认知**：`type` 别名是**纯粹的编译期命名**，与第 1 篇的类型擦除一脉相承——运行时根本不存在 `ID` 这个东西。它能命名 `interface` 命名不了的东西（联合、元组、原始类型、条件/映射类型的计算结果）。与 `interface` 的取舍（声明合并、继承方式、显示名、递归能力）将在第 5 篇用对比表展开，简单记忆：**描述对象形状用 interface，命名"类型表达式"用 type**。
