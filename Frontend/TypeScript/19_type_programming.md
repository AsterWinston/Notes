类型编程（俗称"类型体操"）是把 TypeScript 的类型系统当成一门小型函数式语言来用：输入类型，经过条件判断、映射与递归，产出新类型。本篇带你实现递归类型、字符串类型操作与类型算术，再用两个综合案例体会类型编程的实战威力，最后讨论它在工程中的边界。

# 1. 类型编程是什么

## 1.1 类型层面的"函数式编程"

```typescript
// 运行时函数：输入值 → 计算 → 输出值
function firstWord(s: string): string {   // 取字符串的首个单词
  return s.split(" ")[0];                // 按空格切分后取第一个
}

// 类型函数：输入类型 → 模式匹配 → 输出类型（泛型参数就是"入参"）
type FirstWord<S extends string> =
  S extends `${infer W} ${string}`        // 能按 "单词 + 空格 + 其余" 切开？
    ? W                                   // 能：W 是推断出的首词
    : S;                                  // 不能：整个字符串就是首词

type T1 = FirstWord<"hello type world">; // "hello"
type T2 = FirstWord<"solo">;             // "solo"
```

★ **关键认知**：类型编程的三个基本件是 **泛型**（参数）、**条件类型**（分支）、**递归**（循环），组合起来就是一门跑在编译期的迷你函数式语言。它没有赋值、没有副作用，只有"类型 → 类型"的纯变换；求值发生在编译器**实例化泛型**的那一刻，产物是展开后的类型，运行时零痕迹（**类型擦除**，见第 1 篇）。

## 1.2 与运行时编程的对应关系

| 运行时编程 | 类型编程 | 说明 |
|----------|---------|------|
| 值 value | 类型 type | 类型是值的"编译期影子" |
| 函数 function | 泛型 + 条件类型 | 泛型参数是入参，类型别名是返回值 |
| if / else | `T extends U ? X : Y` | 条件类型即类型层分支 |
| 循环 / 递归 | 递归条件类型 | 没有 for，只能递归 |
| 字符串处理 | **模板字面量类型** | `${infer A}${infer B}` 做模式匹配 |
| 数组 | 元组 tuple | 元组是"能数长度"的类型 |
| 函数调用 | 泛型实例化 `F<T>` | 编译期展开成具体类型 |
| 类型转换 | `as` **类型断言** | 只影响编译期，不做运行时转换 |

⚠ 类型编程里没有可变变量和循环，一切靠递归 + 元组累积器（accumulator）模拟。递归次数直接等于编译器的工作量，写类型体操时必须时刻惦记**编译性能**（见 2.3 节）。

三种语言泛型实现方式的差异，决定了谁的类型系统"可编程"：

| 语言 | 泛型实现 | 能否做类型级计算 |
|------|---------|----------------|
| TypeScript | 编译期展开类型 | 能：条件类型 + 递归 + 模板字面量 |
| Java | **类型擦除**（运行时只剩 Object） | 不能：拿不到 `T` 的具体信息 |
| Go | 单态化（每个实参生成一份代码） | 基本不能：无条件分支、无递归类型 |

# 2. 递归类型

## 2.1 DeepPartial 与 DeepReadonly

```typescript
// 内置 Partial 只"浅"一层：嵌套对象的内层不会跟着变可选
type Shallow<T> = { [K in keyof T]?: T[K] };    // 仅第一层加 ?

// 递归版：属性值若是对象就继续往下展开
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }      // 每个属性递归调用自身
  : T;                                           // 到达基本类型：原样返回

// 递归只读：每层属性都加 readonly
type DeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;

interface Config {
  server: { host: string; port: number };        // 嵌套一层
  debug: boolean;
}

const patch: DeepPartial<Config> = {
  server: { port: 8080 },                        // host 可省：内层也被递归成可选
};
```

★ **关键认知**：递归类型的写法套路固定——**条件类型判断"是否还能往下走"，能走就递归，走到底就返回原样**。`T extends object` 是"还有没有嵌套"的探针：基本类型（string/number 等）是递归出口。Java 的**类型擦除**决定了它写不出这种"逐层展开"的泛型，这是 TS 类型系统独有的表达力。

## 2.2 Flatten：元组递归

```typescript
// 把嵌套元组拍平成一层（元组是类型层的"数组"，可模式匹配）
type Flatten<T extends unknown[]> = T extends [infer First, ...infer Rest]
  ? First extends unknown[]                        // 首元素还是数组？
    ? [...Flatten<First>, ...Flatten<Rest>]        // 是：先展平它再拼剩余
    : [First, ...Flatten<Rest>]                    // 否：保留并递归剩余
  : [];                                            // 空元组：递归出口

type A = Flatten<[1, [2, [3, [4]]]]>;            // [1, 2, 3, 4]
type B = Flatten<[[1, 2], [3, [4, 5]]]>;         // [1, 2, 3, 4, 5]
```

★ **关键认知**：**元组 + `...infer Rest` 解构**是类型层的"数组遍历"：`[infer First, ...infer Rest]` 把"头元素"和"剩余部分"拆开，对应运行时递归里的 `head`/`tail`。展开运算符 `[...A, ...B]` 在类型层就是数组拼接，Flatten 本质是把递归结果不断拼进新元组。

## 2.3 递归深度限制与"excessively deep"

```typescript
// ⚠ 非尾递归的深展开会撞上限，报 TS2589:
//    Type instantiation is excessively deep and possibly infinite
type BadRepeat<T, N extends number, Acc extends unknown[] = []> =
  [...Flatten<[[T], [BadRepeat<T, N, [...Acc, unknown]>]]]];  // 故意写坏的深嵌套

// ✅ 尾递归写法：递归调用出现在条件类型的"最后一个位置"
type Repeat<T, N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N                    // 累加器够长就停
    ? Acc                                    // 返回累积结果
    : Repeat<T, N, [...Acc, T]>;             // 尾位置递归 + 累加器

type R3 = Repeat<"x", 3>;                   // ["x", "x", "x"]
```

★ **关键认知**：TS 4.1 起对**尾递归形式**（递归调用是条件分支的最终结果）做了消除优化，递归深度上限从大约 **50 层**提升到约 **999 层**。判断尾递归的直观标准：递归调用 `Repeat<...>` 是不是整个分支"光秃秃的返回值"，周围没有再套 `[...X]`、模板字面量之类的包装。⚠ 超过 1000 层、或非尾递归深嵌套，编译器会直接放弃并报 TS2589；此时应改成"分段递归 + 中间类型别名"，或重新审视这个类型是否真的需要这么复杂。

# 3. 字符串类型操作

## 3.1 Split 与 Join

```typescript
// Split：按分隔符把字符串切成元组（逐段递归）
type Split<S extends string, Sep extends string> =
  S extends `${infer Head}${Sep}${infer Tail}`  // 用 Sep 做锚点切一刀
    ? [Head, ...Split<Tail, Sep>]               // 头段进元组，递归剩余
    : [S];                                      // 切不开：最后一段

type Segments = Split<"a/b/c", "/">;           // ["a", "b", "c"]
type Words = Split<"hi ts types", " ">;        // ["hi", "ts", "types"]

// Join：把元组用分隔符拼回字符串
type Join<T extends string[], Sep extends string> =
  T extends [infer First extends string, ...infer Rest extends string[]]
    ? Rest["length"] extends 0                  // 只剩一个元素
      ? First                                  // 不加分隔符
      : `${First}${Sep}${Join<Rest, Sep>}`      // 拼上分隔符继续递归
    : "";                                       // 空元组：返回空串

type Path = Join<["a", "b", "c"], "-">;        // "a-b-c"
```

★ **关键认知**：**模板字面量类型**就是类型层的正则引擎：`${infer Head}${Sep}${infer Tail}` 把字符串"按字面量切开"，其中第一个 `infer` 贪婪度最低——`${infer A}${infer B}` 中 A 只吃一个字符，`${infer A}x${infer B}` 中 A 吃到第一个 `x` 之前。⚠ `Split<S, "">`（空分隔符）语义特殊，会退化成逐字符切分，实际工程中应显式禁止空分隔符入参。

## 3.2 Replace 与 Reverse

```typescript
// Replace：把第一次出现的 From 换成 To
type Replace<S extends string, From extends string, To extends string> =
  S extends `${infer Head}${From}${infer Tail}`  // 把 From 夹出来
    ? `${Head}${To}${Tail}`                      // 原位替换
    : S;                                         // 找不到：原样返回

type R = Replace<"hello world", "world", "TS">; // "hello TS"

// Reverse：逐字符倒序（递归 + 模板字面量拼接）
type Reverse<S extends string> =
  S extends `${infer First}${infer Rest}`       // First 吃一个字符
    ? `${Reverse<Rest>}${First}`                // 先递归剩余，首字符拼到末尾
    : "";                                        // 空串是递归出口

type Rev = Reverse<"abcd">;                    // "dcba"
```

★ **关键认知**：字符串递归与元组递归是同构的——**模板字面量当"数组"，`${infer A}${infer B}` 当解构，字符串拼接当 `[...A, ...B]`**。这类操作的实用价值在于"把命名规范做成类型"：如 `"user_name"` → `"userName"` 的 CamelCase 转换、路由模板解析（见 5.1 节）都是同一套路。

# 4. 类型算术

## 4.1 元组即计数器

```
 类型层面的"计数器"：用元组长度表示自然数
   0 → [ ]                  Length<[ ]>                → 0
   1 → [•]                  Length<[•]>                → 1
   2 → [•, •]               Length<[•, •]>             → 2
   3 → [•, •, •]            Length<[•, •, •]>          → 3
   ...
   N → 长度为 N 的元组 ──Length──► number 字面量类型

 加法 = 元组拼接后取长度：
   Add<2, 3> = Length<[•,•] ++ [•,•,•]> = Length<[•,•,•,•,•]> = 5
                  （++ 即类型层的展开拼接 [...A, ...B]）
```

## 4.2 Add 与 Length 的实现

```typescript
// Length：取元组长度（元组自带的"数数"能力）
type Length<T extends unknown[]> = T["length"];   // 长度是 number 字面量

// BuildTuple：把数字 N 变成长度为 N 的元组（类型层的 for 循环）
type BuildTuple<N extends number, Acc extends unknown[] = []> =
  Acc["length"] extends N                          // 数够了？
    ? Acc                                          // 返回累积器
    : BuildTuple<N, [...Acc, unknown]>;            // 否则加一个再数

// Add：两个数字先各自变成元组，拼起来再取长度
type Add<A extends number, B extends number> =
  [...BuildTuple<A>, ...BuildTuple<B>]["length"];  // 拼接 + 取长度

type Sum = Add<2, 3>;                             // 5
type Total = Length<[string, number, boolean]>;   // 3
```

★ **关键认知**：类型层没有 `+` 运算符，数字只能借"**元组长度**"这个原语来算：数字 → 元组（BuildTuple）→ 拼接 → 取长度。这与 JS 运行时的算术完全是两个世界——编译器只会做类型关系判定，不会真的算数，`Add<2, 3>` 的"5"来自编译器数了五个元组元素。Go/Java 的类型系统里连这种"数数"的钩子都没有，这是 TS 类型层图灵不完备的表达力里最有趣的一角。

⚠ 元组长度有硬上限（约 1000），且每加 1 都是一次类型实例化：`Add<900, 900>` 这类大数运算既慢又容易撞深度上限。类型算术只适合**小数字场景**。

## 4.3 实际用途：限制递归层数

```typescript
// 用数字参数给递归"上保险丝"：最多处理 N 个元素
type Take<N extends number, T extends unknown[], Acc extends unknown[] = []> =
  Acc["length"] extends N                    // 拿够 N 个就停
    ? Acc
    : T extends [infer F, ...infer R]        // 还有元素
      ? Take<N, R, [...Acc, F]>              // 取头元素进累积器（尾递归）
      : Acc;                                  // 元素不足：有几个给几个

type First3 = Take<3, ["a", "b", "c", "d"]>; // ["a", "b", "c"]
type Only2 = Take<2, [1, 2]>;                // [1, 2]
```

★ **关键认知**：给递归类型加"计数上限"是防御性类型编程的核心手段——它把**最坏情况的编译代价封顶**，也把报错信息从"excessively deep"变成可读的类型不匹配。类比运行时：这相当于给递归函数加 `maxDepth` 参数，只不过防的不是栈溢出，而是编译器爆栈。

# 5. 综合实战

## 5.1 类型安全的路由参数解析

```typescript
// 把 "user/:id/post/:postId" 解析成 { id: string; postId: string }
type Prettify<T> = { [K in keyof T]: T[K] } & {};  // 抹平交叉类型，报错更友好

// 按 "/" 逐段扫描路由串，递归收集 :param 段
type ParseParams<S extends string> = Prettify<ParseParamsInner<S>>;

type ParseParamsInner<S extends string> =
  S extends `${infer Segment}/${infer Rest}`       // 拆出首段与剩余
    ? ParseSegment<Segment> & ParseParamsInner<Rest> // 分别处理后交叉合并
    : ParseSegment<S>;                              // 最后一段

// 单段处理：":xxx" 是参数段，其余静态段忽略
type ParseSegment<S extends string> =
  S extends `:${infer Name}`                       // 形如 ":id"
    ? { [K in Name]: string }                      // 产出 { id: string }
    : Record<never, never>;                        // 空对象类型

type Route = ParseParams<"user/:id/post/:postId">;
// { id: string; postId: string }

// 用解析结果约束 handler 入参：路径变了，参数类型自动跟着变
declare function route<P extends string>(
  path: P,                                          // 字面量类型保留完整路径
  handler: (params: ParseParams<P>) => void         // 参数类型由路径推导
): void;

route("user/:id", (p) => console.log(p.id));        // ✅ p.id 自动补全
// route("user/:id", (p) => console.log(p.uid));    // ✗ TS2339: 属性 uid 不存在
```

★ **关键认知**：这个案例把前面所有技巧串了起来：**模板字面量切分**（按 `/` 解构）+ **递归**（逐段处理）+ **映射类型**（`{ [K in Name]: string }` 动态造键）+ **交叉合并**（把各段结果攒起来）。它的工程意义在于"**路径即类型**"：路由配置和参数处理函数之间形成编译期契约，改路径忘改取参立即报错。⚠ 交叉类型 `A & B` 直接展示很丑，`Prettify` 只是把显示"抹平"，不改变类型语义，但能显著改善报错信息。

## 5.2 简易 JSON Schema → TS 类型映射

```typescript
// 用对象字面量描述 schema，再在类型层"解释"成 TS 类型
type Schema = {
  type: "string" | "number" | "boolean" | "object" | "array"; // 节点类型
  properties?: Record<string, Schema>;   // object 节点的子字段
  items?: Schema;                        // array 节点的元素 schema
};

// 类型层"解释器"：按 type 字段分派到对应 TS 类型
type FromSchema<S extends Schema> =
  S["type"] extends "string" ? string :
  S["type"] extends "number" ? number :
  S["type"] extends "boolean" ? boolean :
  S["type"] extends "array"
    ? Array<FromSchema<NonNullable<S["items"]>>>   // 元素类型递归翻译
    : S["type"] extends "object"
      ? { [K in keyof NonNullable<S["properties"]>]:
            FromSchema<NonNullable<S["properties"]>[K]> } // 逐字段递归
      : never;

// 声明 schema：必须 as const，否则 type 会被拓宽成 string
const userSchema = {
  type: "object",
  properties: {
    name: { type: "string" },
    age: { type: "number" },
    tags: { type: "array", items: { type: "string" } },
  },
} as const;

type User = FromSchema<typeof userSchema>;
// { name: string; age: number; tags: string[] }
```

★ **关键认知**：`as const` 在这里不是锦上添花而是**生死线**——没有它，`type: "object"` 会被拓宽为 `string`，`FromSchema` 的条件分支全部落空、返回 `never`。这揭示了类型编程的一个底层机制：**字面量类型是类型编程的"常量"**，普通变量声明默认的**类型拓宽**（widening，见第 2 篇）会把这些常量抹掉。⚠ 此模式只解决"schema → 类型"这半边，运行时校验仍需另写（见第 21 篇），两者合起来才是完整的"schema 驱动"方案。

# 6. 类型体操的工程边界

## 6.1 何时该写、何时该停

| 信号 | 判断 |
|------|------|
| 类型在多个公共 API 间复用 | 值得写成工具类型 |
| 只为炫技、一次性场景 | 停手，写显式类型更清楚 |
| 编译时间肉眼可见变慢 | 停手，拆分或退化为简单类型 |
| 报错信息长到没人看得懂 | 停手，加 `Prettify` 或中间类型别名 |
| 团队一半人读不懂 | 停手，或配文档 + 类型测试（见第 22 篇） |

## 6.2 类型性能与报错恶化

```typescript
// ⚠ 反面教材：把递归类型嵌套进高频调用的公共签名
type Overkill<T> = T extends object
  ? { [K in keyof T]: Overkill<Flatten<Extract<T[K], unknown[]>>> }
  : T;                                   // 每次实例化都全量递归，编译器苦不堪言

// ✅ 工程做法：复杂度封顶 + 中间别名改善报错
type SafeOverkill<T> = T extends object
  ? { [K in keyof T]: T[K] }             // 两层封顶，够用就好
  : T;
```

★ **关键认知**：类型体操的成本有三笔账——**编译耗时**（递归展开是指数级工作量）、**报错可读性**（深层条件类型的错误信息像天书）、**团队认知负担**。工业界的共识是：公共库（工具类型、SDK 类型层）值得打磨体操，业务代码点到为止；每引入一个复杂类型，就该配一组**类型测试**（见第 22 篇）锁住它的行为。⚠ 与 Java/Go 团队协作时尤其注意：那两个语言的开发者没有"类型还能编程"的心智模型，过度体操会成为协作断层。

★ **关键认知（全篇收束）**：类型编程的本质是**把不变量交给编译器看守**——路由参数、schema 结构、递归数据形状，一旦用类型表达，破坏它的地方会在编译期集体现形。但类型系统是"编译期影子"，它再精巧也不产生任何运行时校验；真正的边界防御永远需要运行时代码兜底（第 21 篇）。
