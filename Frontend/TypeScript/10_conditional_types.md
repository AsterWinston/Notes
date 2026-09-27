条件类型让类型系统具备了"**根据类型做分支**"的计算能力，`infer` 则让类型可以被"提取"。二者结合，TS 才真正成为一门可以编程的类型语言——内置的 `ReturnType`、`Awaited` 全部由此构成。本篇讲清分发机制、infer 提取与递归类型的深度限制。

# 1. 条件类型语法

## 1.1 三元分支

```typescript
// 条件类型：T extends U ? X : Y —— 类型层面的三元表达式
type IsString<T> = T extends string ? true : false;

type A = IsString<string>;     // true
type B = IsString<number>;     // false
type C = IsString<"hello">;    // true：字面量类型是 string 的子类型

// 实战：根据输入类型决定返回"数组"还是"错误提示"
type Wrap<T> = T extends string ? T[] : never;
type D = Wrap<string>;         // string[]
type E = Wrap<number>;         // never
```

★ **关键认知**：条件类型在**类型层面求值**，编译后完全不存在（类型擦除）。判断用的是 `extends` 的**可赋值性**（结构兼容 + 子类型关系，第 13 篇），与泛型约束的 `extends` 语法相同但语义位置不同（见第 5 节辨析）。⚠ 条件类型求值发生在类型实例化时，嵌套过深会触发编译器的计算预算（见第 4 节递归限制）。

## 1.2 与运行时三元的对照

| 维度 | 运行时三元 `a ? x : y` | 条件类型 `T extends U ? X : Y` |
|------|----------------------|-------------------------------|
| 求值时机 | 运行时 | 编译期（类型实例化时） |
| 判断依据 | 真值 | 可赋值性（结构 + 子类型） |
| 产物 | 值 | 类型 |
| 求值次数 | 每次执行都算 | 每次实例化算一次并随类型缓存 |
| 能否递归 | 能 | 能（⚠ 有深度预算） |

```typescript
// 嵌套条件类型：类型层的 else-if 链
type TypeName<T> =
  T extends string ? "string" :
  T extends number ? "number" :
  T extends boolean ? "boolean" :
  "other";

type T1 = TypeName<"a">;        // "string"
type T2 = TypeName<Date>;       // "other"
```

★ **关键认知**：条件类型就是**类型层面的三元表达式**，可嵌套出 else-if 链，产物是类型而非值。与运行时三元的关键差异在**求值时机**：类型在实例化时求值并随类型绑定复用（运行时三元每次执行都重算），代价是复杂条件类型会实打实地消耗编译时间。⚠ 嵌套链超过两三层就该拆成具名别名（像 `TypeName` 这样），一口气嵌五层的类型是可读性灾难的常见起点。

# 2. 分布式条件类型

## 2.1 联合自动分发

```typescript
// 裸类型参数 T 出现在 extends 左侧时，联合会自动"分发"逐个判断
type ToArray<T> = T extends unknown ? T[] : never;

type F = ToArray<string | number>;
// 计算过程：ToArray<string> | ToArray<number>
//        = string[] | number[]        ← 拆开处理再联合
// 而不是 (string | number)[]           ← 整体代入的结果
```

★ **关键认知**：**分布式条件类型（distributive conditional types）**是 TS 类型编程的分水岭规则——条件类型遇到**裸类型参数**（未被包装进元组/数组/对象的 `T`）在 `extends` 左侧时，自动对联合的每个成员求值再做联合。等价的伪代码是 `type ToArray<T> = T extends unknown ? T[] : never` 对联合 `A | B` 求值为 `ToArray<A> | ToArray<B>`。⚠ "裸"是关键：`T[]`、`[T]`、`{ x: T }` 里的 T 都不裸，不分发。

## 2.2 阻断与触发分发

```typescript
// 技巧：把 T 包进元组 [T]，阻断分发
type NoDistribute<T> = [T] extends [string] ? true : false;

type G = NoDistribute<string | number>;   // false（整体判断：联合不是 string 子类型）
// 对比裸写法：
type Distribute<T> = T extends string ? true : false;
type H = Distribute<string | number>;     // boolean ≡ true | false（分发后联合）

// 反过来：用分发实现集合运算（这是分发的正面价值）
type Exclude2<T, U> = T extends U ? never : T;   // 从 T 中剔除 U 的成员
type I = Exclude2<"a" | "b" | "c", "a">;         // "b" | "c"
```

★ **关键认知**：`[T] extends [U]` 是阻断分发的标准技巧——元组包装把"裸参数"变成"包装参数"，分发条件不再满足。分发本身则是**类型集合运算的引擎**：`Exclude`（差集）、`Extract`（交集）全靠它实现。⚠ 一个常见困惑：`string | number extends string ? ...` 直接写（无类型参数）是**整体判断**不分发——分发只发生在"实例化一个裸类型参数"时，不是语法层面对任何联合都生效。

## 2.3 never 的传播：分发式过滤

```typescript
// 过滤器模式：把"不要"的成员映射为 never，never 在联合中自动消失
type NonFunction<T> = T extends Function ? never : T;
type N = NonFunction<string | (() => void) | number>;   // string | number

// 结构化条件过滤：只保留含 id 属性的成员
type WithId<T> = T extends { id: unknown } ? T : never;
type M = WithId<{ id: string } | { tag: string } | { id: number }>;
// { id: string } | { id: number }
```

★ **关键认知**：分发 + `never` 的组合就是**类型层的 filter**——每个成员独立求值，被淘汰的成员变成 `never`，而 `never` 是联合的单位元（`X | never` ≡ `X`），自动从结果里消失。`WithId` 这种"结构条件过滤"是处理异构联合的利器（如第三方 API 返回的混合对象数组）。⚠ 过滤条件写在 `extends` 右侧的是**结构匹配**（`{ id: unknown }` 匹配任何带 id 的对象），粒度由条件的宽窄控制，写 `{ id: string }` 就会漏掉 `id: number` 的成员。

# 3. infer：类型提取

## 3.1 基本提取

```typescript
// infer 在条件类型的 extends 子句中声明"待推断的类型变量"
type ElementOf<T> = T extends (infer E)[] ? E : never;

type J = ElementOf<string[]>;              // string
type K = ElementOf<(number | boolean)[]>;  // number | boolean

// 提取 Promise 内层
type Unwrap<T> = T extends Promise<infer U> ? U : T;
type L = Unwrap<Promise<string>>;          // string
type M = Unwrap<number>;                   // number（不是 Promise，原样返回）
```

★ **关键认知**：`infer` 只能出现在**条件类型的 `extends` 子句**里，作用是"声明一个占位类型变量，由匹配过程绑定实际类型"。它的匹配是**结构模式匹配**：`T extends Promise<infer U>` 就是拿 T 去匹配 `Promise<X>` 的形状，匹配成功则 U 绑定为 X。⚠ infer 的位置影响提取结果（逆变位置取交、协变位置取联），复杂场景的行为可参考第 13 篇的变异性规则。

## 3.2 手写内置工具类型

```typescript
// ReturnType：提取函数返回值
type MyReturnType<F> = F extends (...args: any[]) => infer R ? R : never;
type R1 = MyReturnType<(x: number) => string>;     // string

// Parameters：提取参数元组（参数是逆变位置，infer 取元组）
type MyParameters<F> = F extends (...args: infer P) => any ? P : never;
type R2 = MyParameters<(a: string, b: number) => void>;   // [string, number]

// InstanceType：提取构造函数的实例类型（第 7 篇预告的实现）
type MyInstanceType<C> = C extends new (...args: any[]) => infer R ? R : never;
type R3 = MyInstanceType<typeof Date>;             // Date

// ConstructorParameters：提取构造参数
type MyCtorParams<C> = C extends new (...args: infer P) => any ? P : never;
type R4 = MyCtorParams<typeof Date>;               // [value: number | string]
```

★ **关键认知**：这四个内置工具类型是**同一模式的四连**——"把一个签名类型拆开，用 infer 抓住关心的部分"。函数返回值在协变位置、参数元组在逆变位置、构造实例在 new 签名的返回位置。理解这一组实现后，`ReturnType<typeof fn>` 这类高频写法背后的机制就完全透明了：**typeof fn 拿到函数类型 → 条件模式匹配 → infer 绑定返回类型**。

## 3.3 infer 与元组解构

```typescript
// 变量元组 + infer：提取首元素 / 末元素 / 剩余部分
type First<T extends unknown[]> = T extends [infer F, ...unknown[]] ? F : never;
type Last<T extends unknown[]> = T extends [...unknown[], infer L] ? L : never;
type Rest<T extends unknown[]> = T extends [unknown, ...infer R] ? R : never;

type F1 = First<[string, number, boolean]>;   // string
type F2 = Last<[string, number, boolean]>;    // boolean
type F3 = Rest<[string, number, boolean]>;    // [number, boolean]

// 多个 infer 同时出现：按位置分别绑定
type Swap<T> = T extends [infer A, infer B] ? [B, A] : never;
type S = Swap<[string, number]>;              // [number, string]
```

★ **关键认知**：元组模式 `[infer F, ...unknown[]]` 是类型层的"**头部解构**"——`...` 的位置决定 infer 抓头还是抓尾，剩余部分用 `...infer R` 接住，与 JS 的 `const [f, ...rest] = arr` 完全同构。多个 infer 按**出现位置**各自绑定，这让"交换、取前后缀、逐元素变换"这类元组操作都能在类型层完成。⚠ 递归处理变长元组时注意深度预算（见 4.2 节），超长元组照样触发 TS2589。

# 4. 递归条件类型

## 4.1 递归类型别名

```typescript
// DeepPartial：递归地把每层属性都变可选
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }   // 对象：映射 + 递归下钻
  : T;                                        // 基本类型：原样返回

interface Config {
  db: { host: string; port: number };
  name: string;
}
type DP = DeepPartial<Config>;
// { db?: { host?: string; port?: number }; name?: string }

// Flatten：递归展平嵌套数组
type Flatten<T> = T extends (infer E)[] ? Flatten<E> : T;
type F1 = Flatten<number[][][]>;    // number
```

★ **关键认知**：递归条件类型让类型计算具备了**遍历树结构**的能力——终止条件通常是"不再是对象/不再是数组"。映射类型（第 11 篇）负责"逐属性处理"，条件类型负责"判断要不要下钻"，二者是递归类型的左右手。⚠ 递归发生在**类型层面**，运行时零开销，但编译期成本全部转嫁给了 Checker。

## 4.2 ⚠ 递归深度限制

```typescript
// 过深的递归会触发编译器报错：
// TS2589: Type instantiation is excessively deep and possibly infinite.
type Bad<T> = T extends object ? Bad<{ [K in keyof T]: Bad<T[K]> }> : T;

// 实用的缓解：加深度计数器，到限停止
type DeepPartialN<T, D extends number = 8> = [D] extends [never]
  ? T                                  // 深度耗尽：原样返回，不再下钻
  : T extends object
    ? { [K in keyof T]?: DeepPartialN<T[K], Prev[D]> }
    : T;

// 用元组长度做数字递减表（类型层面的计数）
type Prev = [never, 0, 1, 2, 3, 4, 5, 6, 7, 8];
```

★ **关键认知**：TS Checker 对类型实例化有**递归深度预算**（约 50 层、99 层因版本而异，还有实例化次数上限），超限报 TS2589 或 TS2590。"元组长度当数字用"（`Prev` 查表递减）是类型编程里控制深度的标准手法——条件 `[D] extends [never]` 同时用到了第 2 节的阻断分发技巧。⚠ 缓解不是消除：对超大嵌套类型（如深度 JSON），更好的办法是**限制递归范围**（只递归需要的层）或干脆让 API 接受 `DeepPartialN<T, 3>` 这种显式深度。

# 5. extends 的两处含义辨析

```typescript
// 用法 1：泛型约束 —— 声明"类型参数的上界"
function f<T extends string>(x: T) { return x; }

// 用法 2：条件类型 —— 运行"类型层面的可赋值性判断"
type G2<T> = T extends string ? 1 : 0;

// 同一个关键字，两种角色：
type Check<T extends object> = T extends { id: string } ? T["id"] : never;
//      ↑ 约束 T 必须是对象        ↑ 判断 T 是否含 id
```

★ **关键认知**：两处 `extends` 的共同内核是**可赋值性**（assignability，第 13 篇），区别在角色：**约束是静态契约**（类型参数声明时的上界，违反则实例化报错），**条件是动态判断**（每次实例化时求值出 X 或 Y）。一句话记法：约束位置的 extends 是"必须是"，条件位置的 extends 是"如果是"。⚠ 语法陷阱：`<T extends U ? X : Y>` 是非法的——条件类型不能直接用在类型参数约束里，必须包一层（如 `type Alias<T> = T extends U ? X : Y`）。

| 位置 | 语义 | 求值时机 | 失败表现 |
|------|------|---------|---------|
| `<T extends U>` | 类型参数上界 | 实例化时检查 | 类型不满足约束报错 |
| `T extends U ? X : Y` | 可赋值性判断 | 实例化时求值 | 求值为 Y 分支 |
| `interface A extends B` | 接口继承（结构合并） | 声明时 | 成员冲突报错 |
| `class A extends B` | 类继承（运行时 + 结构） | 运行时原型链 | — |

```plaintext
条件类型的求值流水线：

  type F<T> = T extends Promise<infer U> ? U[] : T;
  F<string | Promise<number>>
        │
        ▼ 裸类型参数 → 分布式条件类型触发分发
  F<string> | F<Promise<number>>
        │
        ▼ 每个成员独立做模式匹配
  (string extends Promise<infer U> ? ... )    → string（不匹配，走 else）
  (Promise<number> extends Promise<infer U> ? ...) → U 绑定 number → number[]
        │
        ▼ 联合结果
  string | number[]
```

★ **关键认知**：把求值过程拆成"分发 → 匹配 → 联合"三步后，条件类型就不再是魔法。与 Go 泛型的"类型集合"对比：Go 的约束只回答"T 能不能用"（静态），TS 的条件类型能回答"T 是什么、进而算出新类型"（计算）——这是 TS 类型系统表达力远超其他主流语言的根源，也是"类型体操"文化诞生的土壤。⚠ 表达力的代价是可读性：团队代码里递归 + infer + 分布的组合应当克制使用（滥用边界的讨论见第 12 篇）。

## 5.2 约束与条件：门禁与分流

```typescript
// 同一意图的两种写法：约束"逼调用方"，条件"分情况"
function len1<T extends { length: number }>(x: T): number {
  return x.length;                          // 约束：不满足的输入直接报错
}

type Len2<T> = T extends { length: number } ? number : never;
type A2 = Len2<string>;                     // number：满足，走肯定分支
type B2 = Len2<number>;                     // never：不满足，走否定分支

// 组合骨架：先约束砍掉非法输入，再条件分流
type Describe<T extends object> = T extends { id: string }
  ? `entity:${string}`
  : "plain-object";
```

★ **关键认知**：**约束是静态门禁，条件是动态分流**——约束在类型参数实例化时检查并报错（没有"否则"的概念），条件在求值时产出两个分支之一（不报错，只是给出不同结果）。因此"输入必须满足 X"用约束，"根据输入是 X 还是 Y 算出不同结果"用条件。⚠ 二者常组合使用：`<T extends object>` 先把非法输入挡在门外、`T extends { id: string }` 再分流，这是复杂工具类型的常见骨架（第 12 篇的 `MyReturnType` 约束 `F extends (...args: any) => any` 正是"先门禁后分流"）。
