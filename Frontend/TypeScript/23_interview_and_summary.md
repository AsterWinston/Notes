本篇是全系列的收官：先把 TypeScript 面试最高频的 12 道题按"要点 + 示例"过一遍，再汇总手写工具类型的考点与实现，最后画出 23 篇的知识地图和进阶路线。无论备战面试还是系统复盘，这篇都值得收藏。

# 1. 高频面试题 12 道

## 1.1 any 与 unknown 的区别

**要点**：`any` 双向逃逸（既赋给一切也接受一切，等于关闭检查）；`unknown` 只进不出，使用前必须**类型收窄**。外部数据入口用 unknown 是纪律。

```typescript
let a: any = "s";
const n1: number = a;        // ✅ any 污染：字符串溜进了 number
let u: unknown = "s";
// const n2: number = u;    // ✗ TS2322: unknown 不可赋给 number
if (typeof u === "string") console.log(u.length);   // 收窄后才可用
```

★ **关键认知**：`any` 是"放弃治疗"，`unknown` 是"先扣押后审查"——安全边界上永远选 unknown（第 2、21 篇）。

## 1.2 type 与 interface 的区别

**要点**：interface 支持**声明合并**、`extends` 继承、报错信息友好；type 能表达**联合/元组/映射/模板字面量**等任意类型，不能合并。

```typescript
interface Box { value: number }
interface Box { unit: string }    // 声明合并：Box 现有 value + unit

type Id = string | number;        // 联合类型只能用 type 表达
// interface Id2 = string | number;  // ✗ interface 不能这么写
```

| 能力 | interface | type |
|------|-----------|------|
| 声明合并 | ✅ 同名自动合并 | ❌ 同名直接报错 |
| `extends` 继承 | ✅ | ✅（用交叉 `&`） |
| 联合/元组/映射/模板字面量 | ❌ | ✅ |
| 被 implements | ✅（类实现） | ✅（对象字面量类型） |
| 报错信息 | 通常更可读 | 复杂展开后较冗长 |

## 1.3 为什么函数参数是逆变

**要点**：**协变**=赋值方向与子类型方向一致（返回值），**逆变**=相反（参数）。参数逆变保证"被调用方收到的只会比预期少、不会多"。

```typescript
type Handler = (e: { id: number }) => void;
// 父参数类型的 handler 不能赋给子参数类型的期望（逆变）
// const h: Handler = (e: { id: number; name: string }) => {};  // ✗ TS2322

// 返回值是协变：子类型可安全替代父类型
type Factory = () => { id: number };
const f: Factory = () => ({ id: 1, name: "x" });   // ✅ 多返回字段无害
```

★ **关键认知**：若参数协变，调用方只传 `{id}` 而函数内部读 `e.name` 就会崩；逆变从编译期堵死这种"读到不存在的字段"（第 13 篇）。

## 1.4 extends 的两处含义

**要点**：`T extends U` 在**泛型约束**里是"T 必须可赋给 U"；在**条件类型**里是"若 T 可赋给 U 则取前者分支"，联合类型还会**分布式**展开。

```typescript
type ToArr<T extends string> = T[];        // 约束：T 必须是 string 子类型
type IsString<T> = T extends string ? "yes" : "no";   // 条件类型
type R = IsString<"a" | 1>;                // "yes" | "no"（分布式！分布式条件类型）
type R2 = [1 | 2] extends [string] ? 1 : 2; // 2（包进元组可阻止分布）
```

★ **关键认知**：同一关键字、两种语义——约束是"准入检查"，条件是"类型层三元表达式"；裸类型参数上的条件类型会按联合逐成员分发（第 10 篇）。

## 1.5 类型收窄的手段全集

**要点**：`typeof`、`instanceof`、`in`、真值/相等性判断、可辨识联合判别属性、自定义类型守卫 `is`、断言函数 `asserts`。

```typescript
function f(x: string | number | Date) {
  if (typeof x === "string") return x.length;      // typeof 收窄
  if (x instanceof Date) return x.getTime();       // instanceof 收窄
  return x.toFixed(0);                             // 剩余收窄为 number
}

type Shape = { kind: "circle"; r: number } | { kind: "square"; s: number };
function area(s: Shape): number {
  return s.kind === "circle"                       // 判别属性收窄（可辨识联合）
    ? Math.PI * s.r ** 2
    : s.s ** 2;
}
```

★ **关键认知**：收窄的本质是**控制流分析**——编译器沿分支追踪变量类型的"排除法"，`never` 出现即代表该分支已被排除干净（第 3 篇）。

## 1.6 as const 的作用

**要点**：冻结**字面量类型**、属性全部 `readonly`、数组变元组——对抗类型拓宽。

```typescript
const arr1 = ["a", "b"];        // string[]
const arr2 = ["a", "b"] as const;   // readonly ["a", "b"]（元组 + 字面量）
type Second = (typeof arr2)[1];     // "b"（精确到字面量）

const cfg = { port: 8080 } as const;   // { readonly port: 8080 }
// cfg.port = 1;                        // ✗ TS2540: readonly 不可赋值
```

★ **关键认知**：`as const` 是类型编程的"常量声明"——没有它，`"a"` 会被**类型拓宽**成 `string`，第 19 篇的 schema 映射就是靠它保字面量的。

## 1.7 interface 与 type 的实践取舍

**要点**：对象形状优先 `interface`（可合并、可扩展、报错友好）；联合、元组、工具类型、映射用 `type`；对外发布的 API 用 interface 留出**声明合并**的扩展口。

```typescript
// 对外 API 用 interface：使用方可用声明合并补充字段
interface UserProps { name: string }
// 使用方增强：declare module 后 interface UserProps { age?: number }

// 工具与组合用 type：语义更精确
type WithId<T> = T & { id: string };    // 泛型包装只能 type
```

⚠ 团队里最好二选一定为规范，别在同一代码库混用两套审美；发布库时注意 interface 的合并特性是把双刃剑（可增强，也可能被误增强）。

## 1.8 结构化类型与名义类型

**要点**：TS 是**结构化类型**（看结构不看名字），Java/Go 风格的名义类型靠"品牌"模拟。

```typescript
interface Point { x: number; y: number }
class Vec2 { constructor(public x: number, public y: number) {} }
const p: Point = new Vec2(1, 2);   // ✅ 结构相同即兼容，无需 implements

// 模拟名义类型：加品牌字段防止"米"当"秒"用
type Meters = { value: number; readonly brand: unique symbol };
type Seconds = { value: number; readonly brand: unique symbol };
```

★ **关键认知**：结构化类型带来灵活性，代价是语义混淆（第 1、13 篇）；`unique symbol` 品牌是零成本的名义化手段。

## 1.9 装饰器的两套标准

**要点**：legacy `experimentalDecorators`（TS 私有提案，签名随目标而变，配合 `emitDecoratorMetadata` 产出 `design:paramtypes`）vs TC39 标准装饰器（统一 `(value, context)`，纯运行时函数，无内建元数据）。

```typescript
// legacy：方法装饰器三个参数，可改写描述符
function log(t: object, k: string | symbol, d: PropertyDescriptor) {
  const orig = d.value;                          // 保存原方法
  d.value = (...a: unknown[]) => orig(...a);     // 替换为包装函数
}
// TC39：统一两参，返回替换函数
function log2(value: unknown, context: ClassMethodDecoratorContext) {
  return function (this: unknown, ...a: unknown[]) {
    return (value as Function).apply(this, a);   // context.name 可取方法名
  };
}
```

★ **关键认知**：分水岭在"要不要构造器元数据"——依赖注入需要 `design:paramtypes` 只能用 legacy，其余场景新项目优先 TC39（第 20 篇）。

## 1.10 静态类型与运行时校验的关系

**要点**：类型编译后被**类型擦除**，对外部数据零约束；信任边界（API 返回、表单、本地存储）必须运行时校验，用**类型守卫**把结果接回类型系统。

```typescript
interface User { id: number; name: string }
function isUser(d: unknown): d is User {            // 类型守卫桥接两边
  const o = d as Record<string, unknown>;           // 断言出字典读字段
  return typeof o?.id === "number" && typeof o?.name === "string";
}
const data: unknown = JSON.parse('{"id":1}');       // 外来数据一律 unknown
if (isUser(data)) console.log(data.name);           // 校验通过才收窄为 User
```

★ **关键认知**：静态类型是编译期契约，运行时校验是边界门卫，两者接力才闭环（第 21 篇）。

## 1.11 void 与 undefined 的区别

**要点**：`undefined` 是具体值类型；`void` 表示"**不关心返回值**"，是函数返回位置的约定。

```typescript
const u: undefined = undefined;      // 只能存 undefined
function log2(msg: string): void {   // 不打算用返回值
  console.log(msg);
}
const cb: () => void = () => 42;     // ✅ 返回值可被"忽略"（JS 回调惯例）
// const v: void = 42;               // ✗ TS2322: void 不能装值
```

⚠ `void` 与 `undefined` 在严格模式下互不通用；返回 `void` 的函数结果不能再参与运算。

## 1.12 never 的用途

**要点**：`never` 是"不会产生值"的底部类型——穷尽检查、永错函数、永不返回的收窄出口。

```typescript
type Shape = { kind: "circle" } | { kind: "square" };
function assertNever(x: never): never {   // 穷尽检查的收窄锚点
  throw new Error(`未处理分支: ${JSON.stringify(x)}`);
}
function f(s: Shape) {
  switch (s.kind) {
    case "circle": return 1;
    case "square": return 2;
    // 新增联合成员后，下面这行立即报错——穷尽检查生效
    default: return assertNever(s);       // s 在此被收窄为 never
  }
}
```

★ **关键认知**：`never` 是所有类型的子类型、不能赋给任何变量，恰好适合做"这条路不该到达"的标记；联合分支全部排除后剩下的就是 never（第 3 篇）。

# 2. 手写题清单与参考实现

## 2.1 题单总览

| 题目 | 一行考点 |
|------|---------|
| MyPick | 映射类型 + `K extends keyof T` 约束 |
| MyOmit | Pick + Exclude 组合 |
| MyExclude | 联合类型上的分布式条件类型 |
| MyReturnType | `infer` 提取函数返回类型 |
| MyPartial | 映射类型 + `+?` 修饰符 |
| MyParameters | `infer` 提取参数元组 |
| MyNonNullable | 分布式条件类型排除 null/undefined |
| DeepReadonly | 递归条件类型 |

## 2.2 参考实现

```typescript
// MyPick：按键子集选取属性（考点：映射类型遍历 K）
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type P1 = MyPick<{ a: 1; b: 2 }, "a">;           // { a: 1 }

// MyExclude：从联合 T 中剔除可赋给 U 的成员（考点：分布式条件类型）
type MyExclude<T, U> = T extends U ? never : T;   // 命中就变 never 被丢弃
type E1 = MyExclude<"a" | "b" | "c", "a">;        // "b" | "c"

// MyOmit：按键子集剔除属性（考点：Pick 与 Exclude 组合）
type MyOmit<T, K extends keyof T> = MyPick<T, MyExclude<keyof T, K>>;
type O1 = MyOmit<{ a: 1; b: 2 }, "a">;            // { b: 2 }
```

```typescript
// MyReturnType：提取函数返回类型（考点：infer 在返回位推断）
type MyReturnType<F> = F extends (...args: never[]) => infer R ? R : never;
type R1 = MyReturnType<() => string>;             // string

// MyParameters：提取参数元组（考点：infer 在参数位推断）
type MyParameters<F> = F extends (...args: infer P) => unknown ? P : never;
type Pa1 = MyParameters<(a: number, b: string) => void>;  // [number, string]

// MyPartial：全部属性变可选（考点：映射类型 + ? 修饰符）
type MyPartial<T> = { [K in keyof T]?: T[K] };
type F1 = MyPartial<{ a: 1 }>;                    // { a?: 1 }

// MyNonNullable：剔除 null/undefined（考点：条件类型过滤）
type MyNonNullable<T> = T extends null | undefined ? never : T;
type N1 = MyNonNullable<string | null>;           // string

// DeepReadonly：递归只读（考点：递归条件类型 + readonly 修饰）
type DeepReadonly<T> = T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K]> }  // 逐层递归加 readonly
  : T;
type D1 = DeepReadonly<{ a: { b: number } }>;     // { readonly a: { readonly b: number } }
```

★ **关键认知**：八道题覆盖了类型编程的全部基本件——映射类型（`in` 遍历键）、条件类型（三元分支）、分布式（裸类型参数分发）、`infer`（模式提取）、递归（深度展开）。面试写这些题时，能顺手说出"分布式条件类型""infer 推断位""映射类型修饰符"这三个词的机制，比只写出代码更能拿分。

⚠ 高频失分点：① `MyExclude` 的 `T` 必须是裸类型参数才分布式展开；② `MyReturnType` 的参数写 `(...args: never[])` 而非 `any[]`（更安全，且避免逆变问题）；③ `MyOmit` 忘了 `K extends keyof T` 约束。

# 3. 全系列知识地图

```
            TypeScript 学习路线（23 篇 · 四阶段）
 ┌────────────┐   ┌────────────┐   ┌────────────┐   ┌────────────┐
 │阶段一 基础  │ → │阶段二 类型  │ → │阶段三 工程  │ → │阶段四 高级  │
 │ 1 简介环境  │   │ 7 类与OOP  │   │ 14 模块     │   │ 19 类型体操 │
 │ 2 基础类型  │   │ 8 枚举     │   │ 15 声明文件 │   │ 20 装饰器   │
 │ 3 联合收窄  │   │ 9 泛型基础 │   │ 16 tsconfig │   │ 21 严格陷阱 │
 │ 4 函数类型  │   │ 10 条件infer│  │ 17 构建发布 │   │ 22 类型测试 │
 │ 5 对象接口  │   │ 11 映射模板│   │ 18 异步类型 │   │ 23 面试总结 │
 │ 6 数组元组  │   │ 12 工具类型│   │             │   │             │
 │            │   │ 13 兼容层级│   │             │   │             │
 └────────────┘   └────────────┘   └────────────┘   └────────────┘
   值与类型注解      类型系统内核      真实项目落地      表达力与质量
```

| 阶段 | 篇目 | 核心一句话 |
|------|------|-----------|
| 一 基础 | 1~6 | 注解、推断、收窄、函数/对象/数组的类型表达 |
| 二 类型系统 | 7~13 | 类、泛型、条件/映射/模板类型、兼容性与类型层级 |
| 三 工程化 | 14~18 | 模块、.d.ts、tsconfig、构建发布、异步类型 |
| 四 高级 | 19~23 | 类型体操、装饰器、strict 陷阱、类型测试、面试 |

★ **关键认知**：四阶段的暗线是"**表达力递增、离运行时渐远**"——阶段一学"怎么标注值"，阶段二学"类型本身怎么运算"，阶段三学"类型如何服务工程"，阶段四学"类型的极限与守护"。与 Java 的学习路径对照：TS 缺了"字节码/反射/JVM"这层运行时，却多了"类型即函数"的编译期世界，两门语言的深水区完全不同。

# 4. 学习路径与进阶方向

| 方向 | 做法 | 收益 |
|------|------|------|
| 类型体操练习场 | 刷 type-challenges 之类的分级类型题 | 条件类型/递归直觉 |
| 阅读优秀 .d.ts | 读 typescript 包内置 lib.d.ts 与流行库声明 | 学惯用法与边界处理 |
| 参与 DefinitelyTyped | 给无类型 JS 库补 @types 声明 | 理解声明文件的实战坑 |
| 跟踪版本发布说明 | 读每个 TS 版本的 release notes | 掌握新语法（如 const 类型参数） |
| 结构化类型深挖 | 复盘第 13 篇变异性 + 品牌类型 | 看懂复杂报错的根因 |

```typescript
// 进阶示例：TS 5.0+ 的 const 类型参数——函数级 as const
function routes<const T extends readonly string[]>(paths: T): T {
  return paths;                                   // 泛型参数原样保留字面量
}
const r = routes(["/a", "/b"]);                   // readonly ["/a", "/b"]
type First = (typeof r)[0];                       // "/a"（无需调用处写 as const）
```

★ **关键认知**：进阶的正道不是背更多 API，而是**读类型定义、写类型测试、追编译器演进**三件事循环——`.d.ts` 是别人写好的类型编程范本，类型测试是验证理解的试金石，release notes 则告诉你类型系统每年都在变强。

★ **关键认知（全系列收束）**：TypeScript 的本质是给 JavaScript 披上一层**编译期的、可编程的、零运行时开销**的类型外衣；它的全部威力在"编译期拦截 + 契约表达"，全部边界在"运行时零痕迹"。把 23 篇浓缩成一句话：**用类型表达不变量，用收窄处理不确定性，用校验守住信任边界，用类型测试锁住契约**——做到这四点，你已经越过大多数 TS 使用者的水平线。
