函数是 JS 的一等公民，TS 为函数准备了完整的类型表达：签名、重载、this 类型、丰富的参数形态。本篇讲清函数类型的两种书写体系、可选/默认/剩余参数的规则细节，以及重载与 this 参数的底层机制。

# 1. 函数类型的两种写法

## 1.1 箭头式签名

```typescript
// 变量注解：(参数) => 返回类型
let greet: (name: string) => string;

// 实现处"上下文推断"：n 的类型从签名流入，无需注解
greet = (n) => `Hi, ${n}`;

// type 别名给函数类型起名，写法与箭头式一致
type Mapper = (x: number) => string;
const toStr: Mapper = (x) => x.toFixed(2);
```

★ **关键认知**：函数类型是**结构化**的——参数名毫无意义（只是文档），匹配只看位置和类型；参数名不一致照样兼容。这与 Java 的函数式接口（`Function<Integer, String>` 这类**名义**接口）不同，TS 的签名可以随处内联，不必先建接口。

## 1.2 接口式签名（调用签名）

```typescript
// 接口里写"调用签名"：看起来像没有函数名的函数声明
interface Greeter {
  (name: string): string;
}

const g: Greeter = (n) => `Hi, ${n}`;   // 可调用对象
g("Alice");
```

★ **关键认知**：调用签名让"**对象 + 可调用**"成为可能——接口里可以混入属性和其他调用签名，这是箭头式做不到的。两种写法的取舍：

| 维度 | 箭头式 `(x: T) => U` | 接口式 `interface { (x: T): U }` |
|------|---------------------|--------------------------------|
| 书写场景 | 局部变量、参数、类型别名 | 需附加属性或多重载的可调用对象 |
| 函数重载 | 不支持 | 支持（多条调用签名即重载） |
| 附加属性 | 不支持 | 支持（属性 + 调用混合） |
| 声明合并 | 不适用 | ✓ 同名接口自动合并 |

⚠ 类型别名也能写对象式的调用签名（`type F = { (x: number): string }`），但别名不能声明合并——需要扩展能力时用 interface。

# 2. 参数形态

## 2.1 可选参数

```typescript
function build(host: string, port?: number) {
  return `${host}:${port ?? 8080}`;   // port 可能是 undefined，用 ?? 兜底
}

build("a.com");           // ✓ 可省略
build("a.com", 443);
// build();              // ✗ 必填参数不能省，与 JS 的 undefined 行为不同
```

★ **关键认知**：可选参数的本质是**参数类型带 undefined**：`port?: number` 等价于 `port: number | undefined`，读取前必须处理 undefined（strictNullChecks 开启时）。⚠ 可选参数必须排在必填参数之后；`build("a.com", undefined)` 也是合法调用——undefined 与"没传"在类型层同义。

## 2.2 默认参数

```typescript
// 默认参数的类型从默认值推断，也可显式标注
function log(msg: string, level: "info" | "warn" = "info") {
  console.log(`[${level}] ${msg}`);
}

// 默认参数之后不能有必填参数；想跳过中间的默认参数只能显式传 undefined
function f(a = 1, b: string) {
  return `${a}-${b}`;
}
f(undefined, "x");        // a 用默认值 1
```

★ **关键认知**：默认参数在类型上等价于**可选参数**（`a?: number`），只是多了运行时赋默认值这层行为；它出现在签名里使参数变成"可省"。⚠ 顺序规则与可选参数相同：默认参数之后不能跟必填参数（TS 报错 TS1016）——需要"中间参数可选"的语义时，改用显式传 `undefined` 或把可选项收拢成一个对象参数。

## 2.3 剩余参数

```typescript
// 普通剩余参数：任意个 number
function sum(...nums: number[]): number {
  return nums.reduce((a, b) => a + b, 0);
}

// 元组剩余参数：精确描述"前缀固定 + 尾部同类型"
function tuple(...args: [name: string, ...scores: number[]]) {
  const [name, ...scores] = args;   // name: string, scores: number[]
}
tuple("Alice", 90, 85, 77);         // ✓
// tuple(90, 85);                  // ✗ 编译错误：第一个参数必须是 string
```

★ **关键认知**：`...args: T[]` 是 JS 剩余参数的直接类型化；**元组剩余参数**（variadic tuple）则让变长函数获得**逐位置精确签名**——第一个参数固定为 string，其余全是 number，调用处逐个检查。元组的全部玩法见第 6 篇。⚠ 剩余参数必须是数组/元组类型，且只能有一个、必须在参数列表最后。

## 2.4 解构参数的类型注解

```typescript
// 解构参数：类型注解写在解构模式之后
function draw({ x, y }: { x: number; y: number }, scale = 1) {
  return [x * scale, y * scale];   // x、y 从解构模式推断为 number
}

// 带可选属性的解构参数：配合默认值处理 undefined
function log({ name, level = "info" }: { name: string; level?: string }) {
  console.log(`[${level}] ${name}`);
}
```

★ **关键认知**：解构参数的注解是"**整个解构模式的类型**"，内部变量名逐个从模式中取出——这与 Go 的多返回值解构完全不同（Go 无需注解，TS 是同一参数位置的结构拆解）。⚠ 解构参数写了类型注解后就失去"可省略"能力：`draw()` 会报错（缺少整个对象），想让参数可省只能标注 `opts?: { ... }` 并在函数体内判空。

# 3. 函数重载

## 3.1 重载签名 vs 实现签名

```typescript
// 重载签名：对外可见的多条签名（调用方只看得到这些）
function parse(input: string): unknown;
function parse(input: string, reviver: (k: string, v: unknown) => unknown): unknown;

// 实现签名：对外不可见，只负责兜底实现
function parse(input: string, reviver?: (k: string, v: unknown) => unknown): unknown {
  return JSON.parse(input, reviver);
}

parse("[1]");                  // ✓ 命中第 1 条重载
// parse(42);                  // ✗ 无匹配重载 —— 实现签名对调用方不存在
```

★ **关键认知**：重载是**编译期多对一映射**：多条可见签名（重载签名）+ 一条不可见的实现签名。实现签名不参与调用检查，但必须**兼容所有重载签名的参数**（否则实现体内部报错）。⚠ 重载按**声明顺序**匹配，命中第一条就停——精确签名要写在前、宽泛签名写在后，顺序反了会让精确签名永远不可达。

## 3.2 重载 vs 联合参数 vs 泛型

```typescript
// 联合方案：一个签名搞定，但返回类型也是联合，调用方要再收窄
function firstOr(x: string | number[]): string | number {
  return Array.isArray(x) ? x[0] : x;
}

// 泛型方案：返回类型跟着输入走，保持精确
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}
first([1, 2]);   // number | undefined
```

| 方案 | 适用场景 | 缺点 |
|------|---------|------|
| 重载 | 参数类型组合决定**返回类型**不同 | 签名重复，实现体要兜底收窄 |
| 联合参数 | 各分支逻辑可统一实现 | 返回类型变联合，调用方需再收窄 |
| 泛型 | 输入输出类型**联动** | 约束复杂时签名难读 |

★ **关键认知**：能用联合或泛型就别用重载——重载的维护成本最高（改逻辑要同步多条签名）。只有当"不同参数类型 → 不同返回类型，且无法用条件类型表达"时才值得。横向对比：Java 的重载在编译期按**名义类型 + 最佳匹配**挑选，Go 干脆**不支持重载**（一个函数名一个签名），TS 介于两者之间且多一条"按声明顺序匹配"的规则。

# 4. this 参数

## 4.1 显式声明 this 类型

```typescript
interface Deck {
  cards: string[];
  draw(this: Deck, n: number): string[];
}

// "裸函数"将来绑定到 Deck 上使用时，this 参数钉死了期望的 this 类型
const draw = function (this: Deck, n: number) {
  return this.cards.splice(0, n);
};

const deck: Deck = {
  cards: ["A", "K", "Q"],
  draw,                 // ✓ this 类型匹配
};
```

★ **关键认知**：`this` 参数是**编译期的幻影参数**——它只能是参数列表的第一个，⚠ **不占运行时参数个数**，编译后被完全擦除。JS 的 this 是调用时动态绑定的（谁调用指向谁），TS 靠 `this` 参数把"期望的 this 形状"写进签名，在编译期拦住 `bind`/借用调用的错位。这与 Java 显式 `this` 引用、Go 显式 receiver 的静态风格殊途同归，但 TS 的 this 是**类型层概念**，运行时依然是 JS 的动态 this。

## 4.2 this: void 与 noImplicitThis

```typescript
// this: void —— 函数设计上不依赖 this（回调签名常用）
type Handler = (this: void, e: string) => void;

const h: Handler = function (e) {
  // this 在这里没有类型，this.xxx 会报错
  console.log(e);
};
```

★ **关键认知**：`this: void` 是一种**契约式禁用**——声明"这个函数不该被作为方法使用"，回调传参时防止 this 被意外捕获。`noImplicitThis`（strict 家族成员）开启后，未声明 this 的函数若无法从上下文推断出 this 类型就直接报错，⚠ 关掉它等于把 this 重新变回 any。

# 5. 返回值 void 与 undefined

```typescript
// void：调用方不应使用返回值
function log(msg: string): void {
  console.log(msg);
}

// undefined：返回值就是 undefined，是真实的值类型
function nothing(): undefined {
  return undefined;
}

// 回调场景：返回 void 的位置可以放"返回任意值"的函数
const nums = [1, 2, 3];
nums.forEach((n) => n * 2);   // ✓ 箭头返回 number 也不报错
```

★ **关键认知**：`() => void` 有一条特殊规则：**返回非 void 的函数可以赋给返回 void 的类型**——"别用返回值"的承诺不会因为实现多返回了值而被破坏，这让回调签名极其宽容（`forEach` 想要 `(n) => void`，传 `(n) => n * 2` 完全合法）。⚠ 反方向不行：`() => void` 不能赋给 `() => number`（人家要 number 你给不出）。参数方向的兼容恰好相反（**逆变**），细讲在第 13 篇。

# 6. 高阶函数与回调

## 6.1 回调类型

```typescript
// 高阶函数：参数或返回值是函数
function repeat(n: number, f: (i: number) => void): void {
  for (let i = 0; i < n; i++) f(i);
}

// 回调签名尽量"宽容"：参数少、返回 void，接受度最高
type Comparator<T> = (a: T, b: T) => number;
const byAge: Comparator<{ age: number }> = (a, b) => a.age - b.age;
```

★ **关键认知**：设计回调签名时要站在**使用者**角度：参数类型取宽、返回类型取 `void`，才能让各种实现都能传入（涉及函数兼容性的逆变/协变规则，第 13 篇展开）。这与 Go 的 `func(a, b T) int` 接口约束思路一致——签名即契约，但 TS 的契约是结构化匹配，无需显式声明实现。

## 6.2 箭头函数的上下文推断

```typescript
// 已知期望类型的位置：参数类型从签名"反向流入"
const handlers: Record<string, (e: number) => void> = {
  add: (e) => console.log(e),       // e: number，无需注解
  sub: (e) => console.log(-e),
};

// 数组方法的泛型签名同样是上下文类型来源
[1, 2, 3].map((x) => x * 2);        // x: number 由 map 的签名推断
```

★ **关键认知**：**上下文类型（contextual typing）**是 TS 最省心的推断之一——函数出现在已知期望类型的"目标位置"时，参数类型从期望签名反向流入，省去逐个注解的样板。⚠ 一旦给参数手写了注解、或把函数先存进无类型的变量再传递，上下文推断就会减弱甚至丢失，报错信息也变得更绕；排查"参数莫名变 any"时先检查函数是否脱离了目标位置。

## 6.3 参数个数的兼容规则

```typescript
type Cb = (a: number, b: number) => void;

// 实现的参数可以比签名少：调用方多给的实参被忽略（JS 运行时本就允许）
const short: Cb = (a) => console.log(a);    // ✓

// 但不能比签名多：调用方只承诺传 2 个，第三个参数永远是 undefined
// const long: Cb = (a, b, c) => {};       // ✗ 编译错误
```

★ **关键认知**："**实现参数 ≤ 签名参数**"是函数兼容的铁律——JS 调用时多传的参数被静默忽略，所以少参实现绝对安全；多参实现则意味着它依赖一个调用方从未承诺提供的值，编译期直接拦下。这与 Go 的固定参数列表（多一个少一个都编译不过）截然不同，是 TS 入乡随俗地贴合 JS 动态调用习惯的产物。⚠ 这条规则配合第 5 节的 `void` 返回宽容规则，共同构成"回调签名越宽、生态越顺"的设计哲学。
