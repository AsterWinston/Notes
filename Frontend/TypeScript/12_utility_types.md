工具类型是 TS 标准库内置的一整套"类型函数"，它们把泛型、条件类型、映射类型的组合封装成日常可用的积木。本篇先给全景清单，再逐个手写实现核心成员——看懂实现比背签名有用十倍，最后讨论工具类型滥用的边界。

# 1. 工具类型清单总表

| 工具类型 | 签名 | 一句话作用 |
|---------|------|-----------|
| `Partial<T>` | `{ [K in keyof T]?: T[K] }` | 所有属性变可选 |
| `Required<T>` | `{ [K in keyof T]-?: T[K] }` | 所有属性变必填 |
| `Readonly<T>` | `{ readonly [K in keyof T]: T[K] }` | 所有属性变只读 |
| `Record<K,T>` | `{ [P in K]: T }` | 键集合 K 到值类型 T 的映射 |
| `Pick<T,K>` | 从 T 中挑出键 K | 局部取用 |
| `Omit<T,K>` | 从 T 中剔除键 K | Pick 的反面 |
| `Exclude<T,U>` | `T extends U ? never : T` | 联合差集 |
| `Extract<T,U>` | `T extends U ? T : never` | 联合交集 |
| `NonNullable<T>` | `Exclude<T, null \| undefined>` | 剔除空值成员 |
| `ReturnType<F>` | 提取函数返回值类型 | 反推签名输出 |
| `Parameters<F>` | 提取函数参数元组 | 反推签名输入 |
| `ConstructorParameters<C>` | 提取构造参数元组 | 反推构造签名 |
| `InstanceType<C>` | 提取构造的实例类型 | 类 ↔ 实例类型（第 7 篇） |
| `Awaited<T>` | 递归解包 Promise | `await` 的类型对应物 |
| `ThisParameterType<F>` | 提取显式 this 参数 | 方法借用场景 |
| `OmitThisParameter<F>` | 剔除 this 参数 | 把方法当普通函数传 |
| `Uppercase/Lowercase/...` | 字符串字面量变换 | 见第 11 篇 |

```typescript
// 用一个接口贯穿全篇示例
interface User {
  id: number;
  name: string;
  email?: string;          // 可选
  readonly createdAt: Date; // 只读
}

type U1 = Partial<User>;    // 全部可选
type U2 = Pick<User, "id" | "name">;   // { id: number; name: string }
type U3 = Omit<User, "email">;         // 去掉 email
type U4 = NonNullable<User["email"]>;  // string（undefined 被剔除）
```

★ **关键认知**：工具类型分两大族——**对象族**（Partial/Pick/Omit/Record，映射类型实现，作用于对象的键与修饰符）与**联合/签名族**（Exclude/Extract/ReturnType/Parameters，条件类型 + infer 实现，作用于类型集合与函数签名）。认清族谱后，记忆负担骤减：对象族想"键的循环"，签名族想"模式匹配"。

# 2. 手写实现核心工具类型

## 2.1 MyPick / MyRecord

```typescript
// Pick：从 T 中选出 K 集合的键（K 必须是 T 的键）
type MyPick<T, K extends keyof T> = {
  [P in K]: T[P];                // 遍历 K（不是 keyof T），逐键取原值类型
};

// Record：键来自调用方的 K，与 T 无关
type MyRecord<K extends keyof any, V> = {
  [P in K]: V;
};

// Omit：Pick 的反面 —— 先用 Exclude 剔键，再 Pick 剩下的
type MyOmit<T, K extends keyof any> = MyPick<T, MyExclude<keyof T, K>>;

const picked: MyPick<User, "id" | "name"> = { id: 1, name: "a" };
const table: MyRecord<"a" | "b", number> = { a: 1, b: 2 };
const withoutEmail: MyOmit<User, "email"> = { id: 1, name: "a", createdAt: new Date() };
// withoutEmail.email   ✗ 属性已被剔除，编译期即报错
```

★ **关键认知**：`MyPick` 的关键在 `P in K`——映射的遍历对象是**调用方指定的键子集**，`extends keyof T` 约束保证不选出不存在的键。对比 `MyRecord`：它遍历的是任意键集合，值类型是统一的 V。两者是同一种映射语法的两个参数化方向（**键来源不同**）。`MyOmit` 则是**两个工具类型的函数组合**：`MyExclude<keyof T, K>` 算出"剩下的键"，再交给 `MyPick` 映射——这展示了工具类型的可组合性（⚠ 它没有用映射 + as 过滤的写法，是因为组合式实现更直白、报错信息也更好读）。⚠ `MyPick`/`MyOmit` 都是同态映射（保留可选性），所以 `Pick<User, "email">` 的 email 仍是可选的。

## 2.2 MyExclude / MyExtract / MyNonNullable

```typescript
// 分布式条件类型驱动的集合运算（第 10 篇的分发机制）
type MyExclude<T, U> = T extends U ? never : T;   // T 有、U 没有 → 差集
type MyExtract<T, U> = T extends U ? T : never;   // T、U 都有 → 交集
type MyNonNullable<T> = MyExclude<T, null | undefined>;

type A = MyExclude<"a" | "b" | "c", "a" | "b">;   // "c"
type B = MyExtract<"a" | "b" | "c", "a" | "b">;   // "a" | "b"
type C = MyNonNullable<string | null | undefined>; // string
```

★ **关键认知**：三者是**同一分发引擎**的三个参数化用法：对联合的每个成员问"它 extends U 吗"，保留或消灭。`never` 在联合中是单位元（`X | never` ≡ `X`），所以"不要的置 never 再合并"就是集合运算的类型版。⚠ 这三个只作用于**联合类型**；作用于对象时它会退化成"整体判断"（需要 `[T] extends [U]` 阻断或改用映射），这是新手最常踩的语义错位。

## 2.3 MyReturnType / MyParameters

```typescript
// 模式匹配函数签名，infer 抓住返回值 / 参数元组
type MyReturnType<F extends (...args: any) => any> =
  F extends (...args: any[]) => infer R ? R : never;

type MyParameters<F extends (...args: any) => any> =
  F extends (...args: infer P) => any ? P : never;

declare function createUser(name: string, age: number): User;
type R1 = MyReturnType<typeof createUser>;      // User
type R2 = MyParameters<typeof createUser>;      // [name: string, age: number]
```

★ **关键认知**：约束 `F extends (...args: any) => any` 是给 infer 铺路——先把 T 限制在"函数形状"内，匹配才总是可行。`typeof createUser` 是"取值的类型"（第 7 篇的类/值双关同理），这两个工具的高频用法就是**从现成函数反推类型**，避免手写重复签名后漂移。⚠ 对重载函数，`ReturnType` 只取**最后一个重载**的返回值（顺序特性），复杂重载需手工处理。

## 2.4 MyPartial / MyRequired / MyReadonly

```typescript
// 同态映射 + 修饰符代数（第 11 篇的机制）
type MyPartial<T> = { [P in keyof T]?: T[P] };       // 加可选
type MyRequired<T> = { [P in keyof T]-?: T[P] };     // 去可选
type MyReadonly<T> = { readonly [P in keyof T]: T[P] }; // 加只读

// 三者可组合出"可写必填副本"：
type WritableRequired<T> = {
  -readonly [P in keyof T]-?: T[P];   // 同时去只读、去可选
};
```

★ **关键认知**：这三个是**修饰符维度**的变换，键与值类型都不动——纯"同态映射 + `+/-` 运算"。把它们与 MyPick（键维度）、MyExclude（集合维度）、MyReturnType（签名维度）并列，就凑齐了类型变换的四个维度：**键、值、修饰符、签名**。⚠ `MyRequired` 的 `-?` 会顺带剔除值类型里的 `undefined`（`email?: string` → `email: string`），与"只是把 ? 拿掉"的直觉略有出入，处理 `T | undefined` 时要额外小心。

## 2.5 MyAwaited（递归版）

```typescript
// 递归解包 Promise 嵌套，直到露出非 Promise
type MyAwaited<T> =
  T extends null | undefined ? T :
  T extends Promise<infer U> ? MyAwaited<U> :     // 递归下钻
  T extends { then(onfulfilled: (v: infer V) => any): any } ? MyAwaited<V> : T;
  // 第三行兜住"类 Promise 的 thenable 对象"

type D1 = MyAwaited<Promise<Promise<string>>>;    // string
type D2 = MyAwaited<Promise<number | Promise<Date>>>; // number | Date
```

★ **关键认知**：`MyAwaited` 展示了工具类型实现的三个进阶要素：**递归**（多层嵌套）、**结构匹配**（thenable 鸭子类型而非只认 Promise）、**终止条件**（非 Promise 原样返回）。它就是 `await` 表达式在类型层的镜像——JS 的 `await` 递归解包值，TS 的 `Awaited` 递归解包类型，二者一一对应。⚠ 递归无深度保护，恶意的无穷 thenable 类型可触发 TS2589（第 10 篇的深度限制）。

## 2.6 手写实现的模式对照

| 工具类型 | 用到的机制 | 一句话记忆 |
|---------|-----------|-----------|
| MyPick / MyOmit | 映射 + Exclude 组合 | 键的挑选与剔除 |
| MyRecord | 映射（非同态） | 凭空造键值映射 |
| MyExclude / MyExtract / MyNonNullable | 分布式条件类型 | 联合的 filter |
| MyReturnType / MyParameters | infer 模式匹配 | 拆函数签名 |
| MyPartial / MyRequired / MyReadonly | 同态映射 + 修饰符代数 | 增删 ? / readonly |
| MyAwaited | 递归条件类型 | 类型层的 await |

★ **关键认知**：背签名不如背**机制列**——这张表的第二列才是真正的"渔"。遇到陌生工具类型先问三个问题：键从哪来（`keyof T` 还是调用方给）？要不要分发？要不要递归？三问定位后，实现几乎可以默写出来。⚠ 内置版本在细节上可能比手写版更精确（如 `Omit` 用 `Exclude<keyof T, K>` 的同时保留了 `string | number` 索引签名的处理），手写版用于理解，工程里优先用内置。

# 3. 组合实战

## 3.1 类型安全的 extend & override

```typescript
// Omit & 重写：先剔除旧字段，再合并新定义（override 的类型版）
type Override<T, U> = Omit<T, keyof U> & U;

interface Base {
  id: number;
  name: string;
  role: "guest";
}

type Admin = Override<Base, { role: "admin"; permissions: string[] }>;
// { id: number; name: string } & { role: "admin"; permissions: string[] }
// 用 & 是为了保留未覆盖字段；展开后语义 = id/name/role: "admin"/permissions

const a: Admin = { id: 1, name: "root", role: "admin", permissions: ["*"] };
```

★ **关键认知**：`Omit<T, keyof U> & U` 是"**类型安全的属性覆盖**"标准式——直接 `T & U` 会让冲突字段变成交叉（`"guest" & "admin"` = never，⚠ 这是交叉类型最常见的翻车），先 Omit 再交叉才得到"以 U 为准"的结果。与 Java 方法重写的对照：Java 重写由名义继承 + 运行时虚方法表保证；TS 的 override 只是**类型层面的字段替换**，运行时就是普通的对象合并。

## 3.2 DeepPartial 与类型安全事件发射器

```typescript
// 递归可选：深层配置合并场景
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// 事件发射器：事件名 → 回调签名的映射，全部类型安全
type EventMap = {
  click: (x: number, y: number) => void;
  change: (value: string) => void;
};

interface Emitter<E> {
  on<K extends keyof E>(event: K, cb: E[K]): void;      // 事件名约束为键
  emit<K extends keyof E>(event: K, ...args: Parameters<E[K]>): void;
  //                                    ^^^ 复用 Parameters：参数类型联动
}

declare const em: Emitter<EventMap>;
em.on("click", (x, y) => console.log(x + y));   // 参数自动推断
em.emit("change", "new");                        // 参数类型校验
// em.emit("click", "wrong");                    // ✗ 参数不匹配
```

★ **关键认知**：这个组合把本篇所有积木串起来：`keyof E` 约束（第 9 篇）、`Parameters<E[K]>`（签名提取）、事件名联动（映射思路）。事件发射器在 JS 里历来是 `any` 重灾区，此写法让"**事件名 → 回调参数**"的契约在编译期闭环。⚠ 泛型约束只能保证"键存在"，若事件回调本身类型写错（如把 `(x: number)` 写成 `(x: string)`），错误会延后到 `on` 调用处才暴露——类型系统校验的是契约一致性，不是业务正确性。

## 3.3 类型安全的配置合并签名

```typescript
// override 手法的函数化：完整默认值 + Partial 覆盖 = 完整配置
function withDefaults<T extends object>(defaults: T, overrides: Partial<T>): T {
  return { ...defaults, ...overrides };   // 运行时合并，类型上仍返回 T
}

interface ServerConfig {
  host: string;
  port: number;
  debug: boolean;
}

const cfg = withDefaults(
  { host: "localhost", port: 8080, debug: false },   // defaults 钉死 T 的结构
  { port: 3000 }                                      // overrides 只许部分字段
);
cfg.host;              // string：合并结果是完整配置
// cfg.port.toFixed(); ✗ 若覆盖对象写了不存在的字段，新鲜字面量检查直接拦下
```

★ **关键认知**：`defaults: T + overrides: Partial<T> → T` 是配置 API 的**标准签名三件套**——`T` 的结构由 defaults 一次钉死，`Partial<T>` 只允许"同名字段的部分替换"，返回值还原为完整 `T`，调用方拿到的永远是无缺失的配置。与 Java 的 Builder 模式对照：Builder 用一串方法调用逐步积累不变量，TS 直接用"完整默认值 + 部分覆盖"表达同样的约束，代码量小一个量级；⚠ 代价是运行时合并逻辑（`{...a, ...b}`）不受类型保护，深嵌套对象会被浅覆盖，需要深合并时配合 3.2 的 `DeepPartial` 并自己写递归 merge。

# 4. ⚠ 工具类型滥用的边界

```typescript
// 反面教材：炫技三连，可读性归零
type X<T> = {
  [K in keyof T as T[K] extends Function ? never : `get${Capitalize<K & string>}`]:
    () => T[K] extends object ? DeepPartial<T[K]> : T[K] | null;
};

// 正面替代：拆开写、起名字、注释意图
type NonFunctionKeys<T> = {                      // 第 1 步：筛掉方法
  [K in keyof T as T[K] extends Function ? never : K]: K
}[keyof T];
type Getters<T> = {                              // 第 2 步：生成 Getter
  [K in NonFunctionKeys<T> as `get${Capitalize<K & string>}`]: () => T[K]
};
```

★ **关键认知**：类型编程的可读性准则与函数编程一致——**给中间结果起名字**，让每一步只做一件事。判断该不该用工具类型的标准：① 团队成员能否在 30 秒内读懂？② 它是否复用 ≥ 3 处？③ 报错信息是否还可读（复杂类型的报错会显示展开后的巨型类型）？三条有一条不过，就该定义**具名接口**（`interface UserDTO { ... }`）而不是堆砌工具类型。与 Java 对比：Java 社区早就放弃了"用类型系统表达一切"（注解 + 运行时校验反而更务实），TS 的类型体操同样需要克制——**类型是给协作者读的文档，不是炫技的舞台**。

```plaintext
工具类型四维度决策树：

  想做什么？                      用什么维度                典型工具
  ├─ 改键（取/删/改名/筛）   →  映射 + as           →  Pick / Omit / Getters
  ├─ 改属性修饰符           →  同态映射 + +/-      →  Partial / Required / Readonly
  ├─ 集合运算（联合的交并差）→  分布式条件类型       →  Exclude / Extract / NonNullable
  └─ 拆签名（函数/构造/Promise）→ infer 模式匹配    →  ReturnType / Parameters / Awaited
```

★ **关键认知**：把这棵树记住，遇到任何类型变换需求先问"是哪个维度"，再查对应积木——大多数"看起来要写复杂类型体操"的需求，其实只是某个维度的标准模式加一次组合。⚠ 最后一个提醒：工具类型的报错信息往往指向**展开后的形态**而非你的别名（报 `{ [x: string]: ... }` 而不是 `MyPartial<User>`），排查时用 IDE 悬浮查看别名展开，或临时用 `type Debug = YourType` 断点调试类型。
