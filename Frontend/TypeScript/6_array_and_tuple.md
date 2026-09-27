数组与元组是 TS 最常见的容器类型，元组更是"定长定型"建模的利器。本篇讲清数组的两种写法与只读数组的变异性细节、元组的全部形态、`as const` 数组的推断差异，以及数组方法泛型签名的读法与 filter 的著名类型问题。

# 1. 数组类型

## 1.1 T[] 与 Array\<T\> 等价写法

```typescript
// 两种写法在编译期完全同义，选哪种是纯风格问题
const a: number[] = [1, 2, 3];           // 语法糖
const b: Array<number> = [1, 2, 3];      // 泛型形式

// 元素类型复杂时 Array<T> 更清晰：T[] 需要括号防歧义
const cbs: Array<() => void> = [];
// 等价但难读：const cbs: (() => void)[] = [];
```

★ **关键认知**：`T[]` 只是 `Array<T>` 的**语法糖**，两者连报错信息里的显示都逐步统一。嵌套数组推荐 `Array<Array<number>>` 或 `number[][]`，⚠ 别写 `Array<number>[]`——它表示"元素是 `Array<number>` 的数组"，意思一样但阅读顺序极反直觉。

## 1.2 只读数组与协变规则

```typescript
const ro: readonly number[] = [1, 2, 3];
// ro.push(4);                        // ✗ 编译错误：不能增删
// ro[0] = 9;                         // ✗ 编译错误：不能改元素

// ReadonlyArray<T> 是 readonly T[] 的等价泛型形式
const ro2: ReadonlyArray<number> = [1, 2];

// ⚠ 只读数组不能赋给可变数组（那等于绕过只读）
// const mut: number[] = ro;          // ✗ 编译错误
const copy: number[] = [...ro];       // ✓ 展开复制解围
```

★ **关键认知**：`readonly T[]` 是**只读视图**——不能改元素、不能增删，但元素本身仍是 `T`（**浅只读**，元素是对象时其内部照样可变）。变异性规则：可变数组 → 只读数组允许（放宽使用场景），只读 → 可变禁止（否则改坏了原数组）。⚠ 数组本身是**协变**的——`Dog[]` 能赋给 `Animal[]`，Java 数组同样协变（运行时抛 ArrayStoreException 兜底），Go 的 `[]T` 则完全不变；TS 编译期不查元素存储，深挖在第 13 篇。API 设计惯例：**入参收 `readonly T[]`（接受度最大），返回 `T[]`（留给调用方修改权）**。

## 1.3 数组字面量的推断细节

```typescript
// 空数组：进入"演化数组"状态，随 push 逐步定型
let buf = [];            // 初始推断为隐式 any[]（演化中）
buf.push(1);             // 演化为 number[]
// buf.push("x");        // ✗ 已定型为 number[]，再 push 字符串就报错

// 异构数组：元素之间无共同候选 → 元素类型联合
const mixed = [1, "a"];  // (string | number)[]

// 有共同父类时回到"最佳通用类型"（第 2 篇 2.2 的数组版）
class Animal {}
class Dog extends Animal {}
const zoo = [new Dog(), new Animal()];   // Animal[]
```

★ **关键认知**：数组字面量的推断走三条路——**空数组演化**（多次 push 累积类型，首次使用后定型）、**无共同候选取联合**、**有共同候选取最佳通用类型**。⚠ 演化机制只对未注解的空数组生效且极易失控（不同分支 push 不同类型会并成联合，报错点远离事故点），生产代码宁可写 `const buf: number[] = []` 也别依赖演化。

## 1.4 类数组不是数组

```typescript
function f() {
  // ⚠ arguments 是"类数组（array-like）"：有 length 与数字下标，但不是 Array
  // const args: number[] = arguments;   // ✗ 编译错误：IArguments 不是 number[]
  const args = Array.from(arguments);    // ✓ 转成真正的数组
}

// 类数组的类型刻画：有 length + 数字索引，但没有数组方法
type ArrayLike = { length: number; [i: number]: unknown };
```

★ **关键认知**：JS 里"像数组"与"是数组"是两回事——运行时判据是 `Array.isArray`，类型层对应的建模是内置工具类型 `ArrayLike<T>`。⚠ 类数组对象没有 `map` / `filter` 等方法，也不能赋给 `T[]`；`Array.from` 与展开 `[...xs]` 是两条常规的"转正"路径。Go 的切片天生就是"指针 + 长度"的正经数组、不存在此类混淆，Java 的 varargs 底层虽是数组但调用侧从不暴露类数组概念——这是 JS 动态传统留给 TS 的独特包袱。

# 2. 元组

## 2.1 基本形式与越界访问

```typescript
// 元组：定长 + 每个位置独立类型
const pair: [string, number] = ["age", 30];
pair[0].toUpperCase();     // ✓ string
pair[1].toFixed();         // ✓ number

// 动态下标：TS 不知道取的是哪个位置 → 退化为元素联合
const x = pair[Math.random() > 0.5 ? 0 : 1];   // string | number

// pair[2]                  // ✗ 编译错误：长度 2 的元组没有下标 2
```

★ **关键认知**：元组下标访问两条规则——**字面量下标**精确到该位置的类型，**动态下标**退化为所有元素类型的联合。⚠ JS 运行时的数组越界只返回 `undefined` 且毫无提示，元组在类型层直接拦死；开启 `noUncheckedIndexedAccess` 后连普通数组的 `arr[0]` 都会变成 `T | undefined`，但元组的字面量下标不受影响——这是元组优于数组的又一理由。

## 2.2 可选元素、命名元组、剩余元素

```typescript
// 可选元素：? 标记尾部位置，只能出现在尾部
type Row = [string, number?];
const r1: Row = ["a"];        // ✓ 第二个位置可省
const r2: Row = ["a", 1];

// 命名元组：给位置起名，让解构与补全有据可查
type Person = [name: string, age: number];
const p: Person = ["Alice", 30];

// 剩余元素：定长前缀 + 变长尾巴（元组剩余参数的底座，见第 4 篇 2.3）
type Args = [cmd: string, ...flags: string[]];
const args: Args = ["build", "-v", "--watch"];
```

★ **关键认知**：元组的三件扩展装备把它从"固定两元组"升级为**任意形状的参数列表建模**：可选元素对应"可能缺席的位置"，命名只影响可读性（运行时仍是普通数组），剩余元素把"前缀精确 + 尾部开放"写进类型。⚠ 可选元素必须在尾部（中间可选只能用联合位置技巧模拟）；剩余元素同样只能在最后，且至多一个。

## 2.3 多维元组与解构推断

```typescript
// 多维元组：矩阵、棋盘坐标、固定形状的嵌套数据
type Matrix = [number, number][];                  // 行的数组
type Grid = [[string, string], [string, string]];  // 2×2 网格

// 解构赋值按位置精确取类型
const [first, second] = pair;      // first: string, second: number
const [head, ...rest] = args;      // head: string, rest: string[]

// for...of 遍历拿到的是元素类型联合
for (const el of pair) {
  el;   // string | number
}
```

★ **关键认知**：`for...of` 的元素类型是**所有位置类型的联合**——TS 不跟踪迭代次数，第 1 次循环不会推成 string；想要逐位置的精确类型只能靠解构。⚠ 解构可选元组元素得到的变量带 `undefined`（`const [a, b] = r1` 中 `b: number | undefined`）；数组解构可跳位（`const [, second] = pair`）。

## 2.4 元组与数组的互转

```typescript
// 元组是数组的子类型：元组 → 数组直接赋值
const tup: [number, number] = [1, 2];
const arr: number[] = tup;               // ✓

// 数组 → 元组只能靠断言：⚠ 运行时并不检查长度与元素
const maybe = [1, 2, 3];
const t2 = maybe as [number, number];    // ✓ 编译通过，运行时 t2 仍是 3 个元素
```

★ **关键认知**：元组与数组的包含关系是单向的——**元组 ⊂ 数组**（元组多带了长度与逐位置类型信息），向下只能靠 `as` 断言且自负盈亏。⚠ 长度不符的断言没有任何运行时校验：解构 `const [a, b] = t2` 的类型写着 `[number, number]`，实际数据仍按原数组走——与 Java 泛型擦除的"强转即信任"如出一辙，运行时坑要靠自己的校验兜住。

# 3. as const 数组与普通数组推断差异

```typescript
const colors = ["red", "green"];              // string[]，元素被拓宽
const fixed = ["red", "green"] as const;      // readonly ["red", "green"]

type C1 = (typeof colors)[number];            // string
type C2 = (typeof fixed)[number];             // "red" | "green"
```

| 维度 | `const a = [...]` | `const a = [...] as const` |
|------|-------------------|----------------------------|
| 推断类型 | `T[]`（可变数组） | 只读元组 |
| 元素类型 | 基类型（`string`） | 字面量（`"red"`） |
| 可变性 | 可 push / 改元素 | 完全只读 |
| 作类型来源 | 导出的是基类型 | 导出字面量联合 |

★ **关键认知**：`as const` 让**数据即类型**——常量表（路由表、配置表、枚举替代物）声明一次，字面量联合自动导出（配合 `typeof arr[number]`，见第 2 篇 4.2）。它的机制正是第 2 篇讲的**阻止拓宽**在数组上的推广：元素保留字面量类型、整体变只读元组。⚠ 只读性是**深**的（数组与其中对象属性都 readonly），但运行时无冻结；需要可变副本用展开 `[...fixed]`，元素类型会重新推断。

# 4. 数组方法的泛型签名

## 4.1 map / filter / reduce 签名怎么读

```typescript
// map 的签名（简化）：map<U>(fn: (value: T, index: number, arr: T[]) => U): U[]
// U 从回调的返回值推断
const lens = ["a", "bc"].map((s) => s.length);    // number[]

// filter 有两条重载：谓词版返回 T[] 的子集类型，布尔版返回原 T[]
const nonEmpty = ["a", ""].filter((s) => s.length > 0);   // string[]

// reduce 的签名带第二个泛型 U：累加器可与元素类型不同
const sum = [1, 2, 3].reduce((acc, n) => acc + n, 0);            // number
const joined = [1, 2].reduce((acc: string, n) => acc + n, "");  // string
```

★ **关键认知**：数组方法签名是**泛型 + 重载 + 上下文推断**的三重奏——`map<U>` 的 `U` 由回调返回值反推，`reduce<U>` 的 `U` 由初始值或显式注解决定，不写初始值时 `reduce` 甚至有"首元素当初始值"的重载。⚠ 无初始值的 `reduce` 在空数组上**运行时抛 TypeError**，而类型仍是 `T`——类型层不知道数组空不空，安全写法是一律给初始值。

## 4.2 filter 不过滤类型的著名问题

```typescript
const raw: (string | null)[] = ["a", null, "b"];

// 著名问题：filter 不改写元素类型，null 仍在结果类型里！
const filtered = raw.filter(Boolean);   // 类型还是 (string | null)[]

// 解法1：谓词函数（第 3 篇 4.1 的类型守卫）——filter 的谓词重载生效
const cleaned = raw.filter((x): x is string => x !== null);   // string[]

// 解法2：复用泛型非空守卫，签名即文档
function isNonNullable<T>(x: T): x is NonNullable<T> {
  return x !== null && x !== undefined;
}
const cleaned2 = raw.filter(isNonNullable);   // string[]
```

★ **关键认知**：`filter` 的签名本质是**谓词重载**：传 `x is T` 谓词时返回 `T[]`，传普通布尔函数时返回原 `T[]`——"类型不过滤"是**签名设计使然**：普通谓词只返回 `boolean`，与元素类型毫无关联，编译器无从删减成员。⚠ `filter(Boolean)` 里 `Boolean` 只是个布尔转换函数、**不是类型守卫**，结果里 `null` 成员纹丝不动，后续 `filtered[0].length` 照样报错——这个坑在 JS→TS 迁移代码里出现频率极高。对比另外两门语言：Java 的 `Stream.filter()` 同样不改写泛型参数（类型擦除，`List<String>` 进 `List<String>` 出），Go 干脆没有泛型 filter 方法、只能手写循环逐个收窄；TS 的谓词重载是三者中唯一能做到"过滤即改类型"的，前提是把守卫写成 `x is T`。

### 4.3 其他常用方法的签名速读

```typescript
// find：找不到返回 undefined，类型层如实表达"可能落空"
const hit = [1, 2, 3].find((n) => n > 1);        // number | undefined

// every 传类型守卫可收窄整个数组（some 做不到）
const all: (string | number)[] = ["a", 1];
if (all.every((x): x is string => typeof x === "string")) {
  all;   // 此处 all: string[]
}

// flat 的深度是泛型参数，决定元素被展平几层
const nested = [[1], [2, 3]];
const flat1 = nested.flat();   // number[]
```

| 方法 | 签名要点 | 返回类型 |
|------|---------|---------|
| `find` | 谓词 / 守卫重载 | `T \| undefined` |
| `every` | 类型守卫可收窄数组整体 | `boolean` |
| `some` / `includes` | 不产生收窄 | `boolean` |
| `flat` / `flatMap` | 深度走泛型参数 | 展平后的 `T[]` |
| `slice` / `concat` | 不改元素类型 | `T[]` |
| `sort` / `reverse` | 原地变异、返回自身 | `T[]`（同一引用） |

★ **关键认知**：读泛型签名的口诀是"**看类型参数从哪来、到哪去**"——`map<U>` 的 U 来自回调返回值，`find` 的结果来自元素类型但可能落空，`every` 的守卫版把收窄作用于**整个数组引用**（复用第 3 篇的谓词收窄机制）。⚠ `sort` / `reverse` 是**原地变异**且返回自身引用，与 `map` / `filter` "返回新数组"的心智相反——链式混用时容易既改了原数组、又以为拿到了副本。

# 5. 与数组协作的实用模式

## 5.1 元组工具函数

```typescript
// 模式1：tuple 工具函数——展开传参保留精确形状
function tuple<T extends unknown[]>(...args: T): T {
  return args;
}
const t = tuple("a", 1);   // [string, number]，而非 (string | number)[]

// 模式2：交换元组元素顺序
function swap<A, B>(pair: [A, B]): [B, A] {
  return [pair[1], pair[0]];
}

// 模式3：对象转键值对数组（Object.entries 的产出就是元组数组）
const scores = { alice: 90, bob: 85 };
const pairs = Object.entries(scores);   // [string, number][]
```

★ **关键认知**：朴素写法 `(...args: unknown[])` 会把一切推成 `unknown[]`；用泛型 `T extends unknown[]` **捕获实参列表的变长元组形状**，返回类型原样带回精确结构——这是泛型（第 9 篇）与变长元组（第 4 篇 2.3）的经典联动。⚠ `Object.entries` 的值类型退化为所有值类型的联合（`number`），拿不到"哪个键对应哪种值"的精确配对，需要时得用映射类型自己造（第 11 篇）。

## 5.2 只读参数：入参宽、返回严

```typescript
// API 设计惯例：入参收 readonly T[]（可变/只读都能传），返回 T[]（修改权归调用方）
function unique(xs: readonly number[]): number[] {
  return [...new Set(xs)];
}

const src = [1, 1, 2] as const;   // readonly [1, 1, 2]
unique(src);                      // ✓ 只读数组照样传入
```

★ **关键认知**：把入参写成 `readonly T[]` 是 1.2 变异性规则的工程化应用——只读 → 可变被禁止、可变 → 只读被允许，于是"收只读"成为**接受面最大**的写法，还顺带向调用方承诺"我不会动你的数据"。⚠ 反例是返回 `readonly T[]` 但内部仍持可变引用并四处修改——只读是编译期纪律而非运行时保险，别用它冒充不可变性。
