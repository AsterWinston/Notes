联合类型让一个变量"身兼数职"，类型收窄则让编译器沿控制流逐步锁定具体类型——两者结合是 TS 日常开发的核心循环。本篇讲清联合的建模能力（可辨识联合）、全部收窄手段，以及类型守卫、断言函数、never 穷尽检查这套组合拳。

# 1. 联合类型基础

## 1.1 语法与可赋值性

```typescript
// 联合：值是成员类型之一，用 | 分隔
let id: string | number = 42;
id = "abc";            // ✓ 是成员之一
id = true;             // ✗ 编译错误：boolean 不在联合里

// 可赋值性是"单向"的：成员可流向联合，联合不能流向成员
let n: number = id;    // ✗ 编译错误：id 可能是 string
let u: string | number = n;   // ✓ number 是联合的成员
```

★ **关键认知**：联合类型的可赋值性是**成员方向**的——值 → 联合向上兼容（是任一成员即可），联合 → 成员向下不允许（它可能是别的成员）。对联合只能调用**所有成员共有的成员**：`id.toUpperCase()` 会报错，因为 number 上没有这个方法，必须先收窄。联合是类型的"**或**"，与交叉 `&`（与）相对（见第 5 篇）。

## 1.2 联合与字面量的组合

```typescript
// 字面量联合：编译期的"取值集合"，比 enum 轻量（对比见第 8 篇）
type Direction = "up" | "down" | "left" | "right";
type Dice = 1 | 2 | 3 | 4 | 5 | 6;

// 联合自动拍平：嵌套的 | 会合并成一层
type T = (string | number) | boolean;   // 等价于 string | number | boolean

function move(d: Direction) {
  // switch 或 if 按字面量分支，见第 3 节收窄
}
```

★ **关键认知**：字面量联合是**有限取值集合**的编译期表达，零运行时开销——运行时它就是普通的 string / number。配合第 2 篇的 `as const` 可以从常量表直接导出字面量联合（`typeof arr[number]`）。⚠ 字面量联合与 Java 的 enum 不同：它没有运行时载体，拿不到"所有成员列表"，需要枚举值清单时必须自己保留常量数组。

# 2. 可辨识联合（discriminated union）

## 2.1 tag 字段模式

```typescript
// 每个成员都携带同名的字面量属性 ok（判别字段 / tag）
type Result =
  | { ok: true; data: string }
  | { ok: false; error: Error };

function handle(r: Result) {
  if (r.ok) {
    return r.data;      // ✓ 已收窄到成功分支
  }
  return r.error;       // ✓ else 分支自动收窄到失败分支
}
```

★ **关键认知**：可辨识联合的三要素：**同名 tag 属性、tag 取字面量类型、各成员携带不同载荷**。编译器把 tag 的字面量比较翻译成对整个联合的成员筛选，比逐个 `in` 判断优雅得多。它与 Java 用继承建模"多态结果"相比轻得多——零类开销、还能**穷尽检查**（见第 6 节）。

## 2.2 建模状态机与请求结果

```typescript
// 请求生命周期：idle → loading → success | failure
type FetchState =
  | { status: "idle" }
  | { status: "loading"; startedAt: number }
  | { status: "success"; data: string[] }
  | { status: "failure"; error: string };

function render(s: FetchState): string {
  switch (s.status) {
    case "idle":    return "等待中";
    case "loading": return `加载中（${Date.now() - s.startedAt}ms）`;
    case "success": return `共 ${s.data.length} 条`;
    case "failure": return `出错：${s.error}`;
  }
}
```

```
             ┌──────────┐  发起请求  ┌───────────┐
             │   idle   │ ──────────▶ │  loading  │
             └──────────┘            └───────────┘
                                          │     │
                              成功        │     │  失败
                              ▼           ▼     ▼
                        ┌──────────┐   ┌───────────┐
                        │ success  │   │  failure  │
                        └──────────┘   └───────────┘

  每个状态 = 联合的一个成员；status 字段 = 判别依据；其余字段 = 该状态专属载荷
```

★ **关键认知**：可辨识联合把**状态机直接编码进类型**——非法状态（如 loading 却携带 data）根本构造不出来，这是 Java 接口继承体系难以低成本做到的。⚠ tag 属性名必须各成员一致，且必须是**字面量类型**（写成 `status: string` 就无法辨识了）；载荷字段也应尽量做成"该状态才存在"，用可选属性混搭会削弱建模精度。

# 3. 收窄手段全集

**收窄（narrowing）**指编译器沿控制流把宽类型逐步锁定为窄类型。各手段一览：

| 手段 | 判定依据 | 典型场景 |
|------|---------|---------|
| `typeof x === "string"` | 运行时类型标签 | 原始类型联合 |
| 真值收窄 `if (x)` | truthy / falsy | 去掉 null / undefined |
| `"prop" in x` | 属性存在性 | 无 tag 的结构化联合 |
| `x instanceof Date` | 原型链 | 类的实例联合 |
| `x.kind === "a"` | 字面量比较 | 可辨识联合 |
| 赋值收窄 | 赋值语句本身 | 变量被重新赋值后 |

## 3.1 typeof 收窄

```typescript
function pad(v: string | number) {
  if (typeof v === "string") {
    return v.padStart(8);   // v: string
  }
  return v.toFixed(2);      // v: number（else 分支自动收窄）
}
```

★ **关键认知**：`typeof` 收窄基于 JS 运行时的类型标签，编译器把 `typeof v === "string"` 翻译成"true 分支取交集、false 分支取差集"。⚠ `typeof null === "object"` 是 JS 的历史遗留（语言之坑），null 的收窄要靠真值判断或 `=== null`，typeof 帮不上忙。

## 3.2 真值收窄及其陷阱

```typescript
function len(v: string | null) {
  if (v) {
    return v.length;        // v: string（null 被真值收窄去掉）
  }
  return 0;
}

// ⚠ 真值收窄把 "" 与 0 一起"收掉"了
type T = "" | "hello" | 0 | 1;
function f(x: T) {
  if (x) {
    x;   // 这里 x: "hello" | 1 —— "" 和 0 被当成 falsy 排除
  }
}
```

★ **关键认知**：真值收窄复用 JS 的 truthy / falsy 语义（falsy：`""`、`0`、`-0`、`0n`、`NaN`、`null`、`undefined`、`false`），它去掉的是"可能是 falsy"的那部分类型。⚠ 当联合里含有 `0` 或 `""` 这类**合法业务值**时，`if (x)` 会错误地把它们排除——想只去 null/undefined 请写 `if (x !== null)` 或用 `??` / `?.`。

## 3.3 in 操作符收窄

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(a: Fish | Bird) {
  if ("swim" in a) {
    a.swim();       // a: Fish
  } else {
    a.fly();        // a: Bird
  }
}
```

★ **关键认知**：`in` 检查的是**属性存在性**，对没有 tag 的结构化联合（尤其是外部数据的形状）特别好用，还能收窄可选属性（`if ("name" in u)`）。与 Go 的类型断言 `v, ok := x.(T)` 相比，`in` 不产生新值，而是对原变量**原地收窄**；⚠ 可选属性收窄后类型带 undefined（`"name" in u` 只保证键存在，不保证值非空）。

## 3.4 instanceof 收窄

```typescript
function log(e: Date | string) {
  if (e instanceof Date) {
    console.log(e.getTime());       // e: Date
  } else {
    console.log(e.toUpperCase());   // e: string
  }
}
```

★ **关键认知**：`instanceof` 检查原型链上的 `prototype`，收窄"类的实例联合"。这与 Java 的 `instanceof` 直接对应，但 TS 把结果**编译进类型层**（Java 21 的模式匹配 instanceof 才追平这个体验）。⚠ 跨 realm（iframe、vm 上下文）的实例会因原型不同而判假，这是 JS 语义问题而非 TS 的锅。

## 3.5 字面量比较与赋值收窄

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; w: number; h: number };

function area(s: Shape) {
  if (s.kind === "circle") {
    return Math.PI * s.radius ** 2;   // circle 分支
  }
  return s.w * s.h;                   // else 自动剩余为 rect
}

// 赋值收窄：重新赋值即触发重新收窄
let v: string | number = "hi";        // 此刻 v: string
v.toUpperCase();                      // ✓
v = 42;                               // 重新赋值
v.toFixed(2);                         // ✓ 此刻 v: number
```

★ **关键认知**：`===` 比较字面量时编译器做"**相等取交集、不等取差集**"的收窄；赋值收窄则是控制流分析（CFA）的最简形式——TS 对**每个程序点**维护变量的当前类型。⚠ 收窄不会跨函数边界存活：把变量捕获进回调后，编译器不再信任外部的收窄结果（回调执行时变量可能已被改写），需要在回调内重新收窄，这个细节在第 13、21 篇展开。

# 4. 自定义类型守卫

## 4.1 谓词 x is T

```typescript
// 返回类型写 "x is T"：承诺 true ⇒ x 是 T
function isString(x: unknown): x is string {
  return typeof x === "string";
}

function f(x: string | number) {
  if (isString(x)) {
    x.toUpperCase();   // ✓ 调用处被收窄为 string
  }
}
```

★ **关键认知**：类型守卫把"**运行时判断 + 类型承诺**"打包成一个函数，收窄能力从函数体内**延伸到调用处**——这是普通布尔函数做不到的。⚠ 守卫是**人肉担保**：谓词实现与声明不一致（比如实现判断 `typeof x === "number"` 却声明 `x is string`）编译器完全不查，运行时照样炸。与 Java 的模式匹配相比，TS 把"判定逻辑"下放给用户代码，灵活但危险。

## 4.2 手写守卫实战

```typescript
// 非空守卫：从数组里过滤掉 null / undefined
function isNonNullable<T>(x: T): x is NonNullable<T> {
  return x !== null && x !== undefined;
}

// 数组守卫：注意内置 Array.isArray 收窄出的是 any[]，泛型守卫更精确
function isArray<T>(x: unknown): x is T[] {
  return Array.isArray(x);
}

const vals: (string | null)[] = ["a", null, "b"];
const clean = vals.filter(isNonNullable);   // string[]
```

★ **关键认知**：`isNonNullable` 配合 `filter` 是最高频的守卫用法——⚠ 这里埋着一个著名问题：`filter` 只有在传入**谓词函数**时才会改写元素类型，传普通布尔函数（包括 `Boolean`）不会，详细机理与解法见第 6 篇 4.2。

# 5. 断言函数

```typescript
// asserts x is T：调用后把参数 x 收窄为 T；函数本身无返回值
function assertIsString(x: unknown): asserts x is string {
  if (typeof x !== "string") {
    throw new Error(`expected string, got ${typeof x}`);
  }
}

function g(x: string | number) {
  assertIsString(x);
  x.toUpperCase();   // ✓ 断言函数执行后收窄生效
}

// asserts x：只断言"非空/为真"，不指定具体类型
function assertTruthy(x: unknown): asserts x {
  if (!x) throw new Error("expected truthy");
}
```

★ **关键认知**：断言函数是"**抛错式收窄**"——适合放在函数入口做参数校验：一处断言，后续全函数体都带着收窄后的类型继续编译。与谓词守卫的区别是**控制流语义**：守卫用 `if` 分支消费结果，断言函数用异常兜底、直通后续代码。⚠ 断言函数与守卫一样是人肉担保；另外它改变了函数的控制流（可能抛异常），调用处隐含"校验不通过就不该继续"的前提，声明处最好写清会抛什么。

# 6. never 与穷尽检查

## 6.1 switch + never 实现穷尽检查

```typescript
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "rect":   return s.w * s.h;
    default: {
      // 所有成员都处理完后，s 被收窄为 never
      const exhaustive: never = s;
      return exhaustive;
    }
  }
}

// 日后给 Shape 增加 { kind: "tri"; ... } 而忘记补分支：
// default 里 s 是 tri 成员，赋给 never 立刻编译报错 → 提醒你补 case
```

## 6.2 assertNever 工具函数

```typescript
// 通用兜底：永不返回，任何遗漏都在编译期暴露
function assertNever(x: never): never {
  throw new Error(`unexpected: ${JSON.stringify(x)}`);
}

function area2(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "rect":   return s.w * s.h;
    default: return assertNever(s);   // 漏分支时这里编译报错
  }
}
```

★ **关键认知**：穷尽检查的原理是**收窄到空**——default 分支处 `s` 的类型是"排除所有已处理成员后的剩余"，全部处理完即 `never`，never 可赋给 never；一旦有成员遗漏，剩余类型非 never，赋值立刻报错。这让"新增联合成员"从**运行时漏处理**升级为**编译期清单**：改类型定义的那一刻，所有需要补分支的 switch 集体点亮。与 Go 的 switch 对比：Go 编译器不校验 case 是否覆盖所有可能（只能靠 lint 启发式），Java 的 switch 同样不查枚举完备性（IDE 提示而已）——TS 靠 never 把完备性变成硬约束。⚠ default 分支必须**真正把 s 赋给 never 变量或传给 assertNever**，只写 `default: break;` 不产生任何检查。
