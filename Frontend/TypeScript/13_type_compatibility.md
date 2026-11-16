类型兼容性回答"什么类型可以赋给什么类型"，是理解 TS 一切报错的底层依据；类型层级则把全部类型排成一座可赋值方向分明的塔。本篇讲清宽度兼容、函数的逆变/协变、数组协变的历史包袱，并用品牌类型给结构化类型补上"名义"的锁。

# 1. 结构化类型与宽度兼容

## 1.1 目标类型的属性源都有即可

```typescript
interface Point {
  x: number;
  y: number;
}

// 源类型属性更多 → 可赋给属性更少的目标（宽度兼容）
const p = { x: 1, y: 2, z: 3 };
const point: Point = p;          // OK：多余属性 z 不影响兼容

// 直接写字面量则被拦：
// const point2: Point = { x: 1, y: 2, z: 3 };   // ✗ 对象字面量多属性检查
```

★ **关键认知**：结构化类型的可赋值方向是"**源 ≥ 目标**"——目标要求的属性源必须全有（且类型兼容），源多出来的属性被忽略。这与 Java 的名义兼容完全不同：Java 里 `class Point3D extends Point` 才行，TS 里"长得像"就行。这决定了 TS 的可赋值性是**非对称**的（Point3D 结构能赋给 Point 结构，反过来不行）。

## 1.2 ⚠ 新鲜对象字面量检查（excess property check）

```typescript
function f(p: Point) { }

f({ x: 1, y: 2, z: 3 });          // ✗ TS2353: 对象字面量只能指定已知属性
f({ x: 1, y: 2 });                // OK

// 绕过方式 1：先赋给变量（变量不是"新鲜"字面量）
const tmp = { x: 1, y: 2, z: 3 };
f(tmp);                           // OK

// 绕过方式 2：类型断言（第 2 篇）
f({ x: 1, y: 2, z: 3 } as Point); // OK，但 z 成了幽灵属性
```

★ **关键认知**：**多余属性检查只对"新鲜对象字面量"（fresh object literal）生效**——直接内联的 `{...}` 在赋值瞬间做一次"属性白名单"校验，捕获拼写错误（`nmae` 拼错会立刻报错）；而经过变量中转后对象"失去新鲜度"，回归宽度兼容。⚠ 这条规则是双刃剑：它几乎总能逮住拼写错误，但也让"故意传扩展配置"必须写断言或中间变量。与 Go 对照：Go 的接口满足看方法集、结构体赋值看逐字段，同样没有"名义"概念，但也没有新鲜字面量检查这层额外保护。

# 2. 函数兼容性

## 2.1 参数逆变、返回值协变

```typescript
// 目标签名：收一个 { x: number; y: number }，返回 number
type Handler = (p: { x: number; y: number }) => number;

// 源函数参数更宽（Point3D → 只用 x/y）→ 逆变方向，OK
const h1: Handler = (p: { x: number; y: number; z: number }) => 0;

// 源函数参数更窄 → 不安全，strictFunctionTypes 开启时报错
const narrower = (p: { x: number }) => 0;
// const h2: Handler = narrower;   // ✗ 参数逆变违规

// 返回值协变：源返回值更具体（string 是 number 的子类型…此处示意）
type Make = () => string;
const m: Make = () => "hi";        // 返回值子类型 OK
```

★ **关键认知**：函数参数是**逆变（contravariant）**的——源函数的参数类型必须是目标参数类型的**超类型**（能处理更宽的输入才安全），返回值是**协变（covariant）**的（返回更具体的安全）。记住方向口诀："**参数逆变（反着来）、返回协变（顺着来）**"。⚠ `strictFunctionTypes` 开启后参数才严格逆变；关闭时参数是**双变**（bivariant），错误的赋值也能通过——这是 strict 家族里被低估的一个开关。

## 2.2 ⚠ 方法语法 vs 函数属性语法的双变差异

```typescript
interface Api {
  // 方法语法：即使 strictFunctionTypes 开启，参数仍是双变
  process(x: string | number): void;
}

// 函数属性语法：严格逆变
interface ApiStrict {
  process: (x: string | number) => void;
}

// 方法语法允许这种危险赋值（参数收窄为 string，仍通过）：
const bad: Api = { process: (x: string) => {} };   // ⚠ 方法位置双变
// 换成 ApiStrict 则报错：(x: string) => void 不能赋给 (x: string | number) => void
```

★ **关键认知**：TS 出于兼容存量 JS（大量方法风格 API 依赖参数双变，如数组的 `sort(comparefn)`）的考虑，让**方法语法声明**（`process(x): void`）在 `strictFunctionTypes` 下依然**双变**，而**函数属性声明**（`process: (x) => void`）严格执行逆变。这是规范里明文的例外，也是类型安全的一条暗缝。⚠ 写接口时优先用**函数属性语法**获得严格检查；故意需要双变时才用方法语法。

## 2.3 可选参数与剩余参数

```typescript
// 目标：可选参数；源：必需参数 —— 不兼容（少给了可选性）
type WithOpt = (a: number, b?: string) => void;
const req = (a: number, b: string) => {};
// const w: WithOpt = req;    // ✗ 目标可能只传一个参数，源需要两个

// 剩余参数可以匹配任意数量的固定参数
type Rest = (...args: number[]) => void;
const fixed = (a: number, b: number) => {};
const r: Rest = fixed;        // OK：固定参数可赋给 rest 签名
```

★ **关键认知**：参数数量的可赋值方向同样逆变——源函数**不能要求更多**（调用方按目标签名可能给更少）。剩余参数 `...args: T[]` 是"任意长度的参数列表"，可接受固定参数的函数（重载兼容），反向则不行。⚠ JS 的 `Function.length` 在运行时不受影响，类型层面的参数数量检查是纯编译期的契约。

## 2.4 void 返回值的特殊规则

```typescript
type VoidFn = () => void;

// 返回值任意的函数可以赋给 () => void："不关心返回值"语义
const f1: VoidFn = () => 42;        // OK：返回 number 也被接受

// 反向不行：() => void 不能赋给需要返回值的签名
const g: () => number = f1;         // ✗ void 不能赋给 number

// 这正是 forEach 回调宽松的原因
[1, 2, 3].forEach(n => n.toFixed());   // 回调返回 string，forEach 不在乎
```

★ **关键认知**：`() => void` 的语义是"**调用方不使用返回值**"，而不是"返回值是 undefined"——因此任何返回类型的函数都可赋给它（返回值被忽略是安全的）。反方向的拒绝保证了另一头的安全：不会把"没有返回值"误当作"有值返回"。⚠ 这条规则看似与返回值协变矛盾（string 不是 void 的子类型），实则是规范明文的**兼容性豁免**：目标位置是 void 时放宽检查，特判优先于一般规则——排查兼容性报错时先确认目标返回类型是不是 void，能省不少弯路。

# 3. 数组与元组的协变

## 3.1 ⚠ 数组协变的历史问题

```typescript
// 数组是协变的：子类型数组可赋给超类型数组
const nums: number[] = [1, 2, 3];
const mixed: (number | string)[] = nums;   // 编译通过（协变）

// 危险：通过宽类型引用写入后，原数组的类型被破坏
mixed.push("oops");              // 运行时 nums 变成 [1, 2, 3, "oops"]
nums[3].toFixed(0);              // 运行时炸：Number.prototype.toFixed.call("oops")
```

★ **关键认知**：TS 的数组**协变**（与 Java 数组的协变如出一辙）是刻意的历史兼容决策——JS 代码里"把 `number[]` 传给收 `(number|string)[]` 的函数"太常见，严格不变（invariant）会引发海量报错。代价就是上例的**运行时污染**：编译器认为安全的写操作可能破坏原始数组的类型假设。⚠ 缓解手段：把参数声明为**只读** `readonly number[]`（只读数组在写位置天然安全，且仍可协变读取），这是 API 设计的推荐默认。

## 3.2 元组的协变与逆变

```typescript
// 元组按元素逐一兼容，整体协变
type Pair = [number, string];
const p: Pair = [1, "a"];

// 元组也参与函数参数逆变：源参数元组元素可以更宽
type Fn = (t: [number, string]) => void;
const wide = (t: [number | string, string | number]) => {};   // 元素更宽
const f: Fn = wide;     // OK：宽元组参数可处理窄元组实参
```

★ **关键认知**：元组的兼容是**逐元素、带位置语义**的（第 6 篇的命名元组也参与），并且在函数参数位置按**逆变**处理元素。对比 Go：Go 的 slice 不变（`[]int` 与 `[]interface{}` 互不赋值），需要显式复制转换；TS 选择"宽松协变 + 只读防护"，是动态语言生态妥协的产物。

# 4. 对象变异性与 readonly

```typescript
// readonly 属性可以接受可变来源（读更安全），反过来不行
interface RO { readonly x: number }
interface RW { x: number }

const rw: RW = { x: 1 };
const ro: RO = rw;        // OK：只读位置可接收可变对象

const ro2: RO = { x: 1 };
// const rw2: RW = ro2;   // ✗ 可变位置不能接收只读（否则可通过 rw2 修改）

// 对象属性本身是"协变读、逆变写"的合成
type Getter<T> = { get(): T };      // 只出现在返回值 → 协变
type Setter<T> = { set(v: T): void }; // 只出现在参数 → 逆变
type GetterSetter<T> = { get(): T; set(v: T): void };  // 读写皆有 → 不变
```

★ **关键认知**：对象类型的变异性由**成员中类型出现的位置**合成：只读成员协变、只写成员逆变、读写皆有则不变（invariant）。`readonly` 的兼容规则是它只读性的直接推论。⚠ TS 的变异性检查是**启发式**的（对明显不安全的场景报错，深层嵌套有时放行），不能当作完全的运行时安全证明——与 Rust 的所有权系统相比，TS 只是"尽力而为的静态提醒"。

# 5. 类型层级全景

```plaintext
                    never（底类型，无人能赋值给它）
                      ▲  可赋值方向：下 → 上
        ┌─────────────┼─────────────┐
   字面量类型 "a"  1  true   null  undefined
        ▲   ▲
        │   └──── 联合/交叉收窄出的子类型
   "a" | "b"   1 | 2        string | number
        ▲
      string   number   boolean（原始类型）
        ▲
   String（包装对象类型，与 string 不同！）
        ▲
      object   { ... }   类的实例类型（结构化兼容横跨整层）
        ▲
     {} （非空类型：除 null/undefined 外的一切）
        ▲
     unknown（顶类型：任何类型都能赋给它）
        ▲
     any（双向任意：既能赋给一切，也能接受一切）
```

```typescript
// 每层各验证一条可赋值方向
const n: never = undefined as never;   // never 只能由断言或收窄得到
const lit: "a" = "a";                  // 字面量是 string 的子类型
const s: string = lit;                 // 子 → 父，OK
const o: object = { a: 1 };            // 对象 → object
const u: unknown = o;                  // 一切 → unknown
const a: any = u;                      // unknown → any OK
const back: string = a;                // any → 一切（双向任意）

// unknown 是"安全的 any"：接收一切，使用前必须收窄
function f(x: unknown) {
  // x.toFixed(2);                     // ✗ unknown 不能直接调用
  if (typeof x === "number") x.toFixed(2);   // 收窄后可用（第 5 篇）
}
```

★ **关键认知**：可赋值方向沿层级**自下而上**：`never` 是底（空集）、`unknown` 是顶（全集）、`any` 是**双向黑洞**（既在顶也在底——它可以赋给任何类型、任何类型也都能赋给它，类型检查对它完全放弃）。⚠ `any` 与 `unknown` 的一字之差是 strict 项目的分水岭：对外部输入用 `unknown` + **类型收窄**，把 `any` 留给真正无法描述的边界。历史层的 `String`/`Number` 包装对象类型与原始类型**不兼容**（`String` 不能赋给 `string`），几乎总是 bug，见到就删。

# 6. 模拟名义类型：品牌类型

## 6.1 品牌的构造

```typescript
// 结构相同的两个"单位类型"会被视为同一类型 —— 危险
type Meters = { value: number };
type Seconds = { value: number };
const m: Meters = { value: 10 };
// const s: Seconds = m;    // ⚠ 编译通过！米被当成了秒

// 品牌类型：加一个不可见的"品牌字段"破坏结构兼容
type Brand<T, B extends string> = T & { readonly __brand: B };
type MetersB = Brand<number, "meters">;
type SecondsB = Brand<number, "seconds">;

const dist = 10 as MetersB;              // 只能通过断言"铸造"
const time = 5 as SecondsB;
// const oops: SecondsB = dist;          // ✗ 品牌不同，结构不再兼容

// 带品牌的安全构造函数：唯一入口
const toMeters = (n: number) => n as MetersB;
const toSeconds = (n: number) => n as SecondsB;
function speed(d: MetersB, t: SecondsB): number {
  return (d as number) / (t as number);   // 内部使用时剥掉品牌
}
speed(dist, time);        // OK
// speed(time, dist);     // ✗ 参数顺序错也编译不过
```

★ **关键认知**：品牌类型（brand / opaque type）的原理是**给结构加一个名义锚点**——`__brand` 字段永远不存在于运行时（类型擦除），但足以让结构化类型系统"误以为"两者不同，从而恢复名义类型的隔离性。这是结构化系统里模拟 Java `newtype`/Scala 专有类型的经典技巧。⚠ 代价是所有创建点都要过 `as` 断言（"铸造"），应把铸造函数**集中收口**（如 `toMeters`）并在命名上警示，否则人人都随手 `as`，品牌就名存实亡。

## 6.2 何时值得上品牌

| 场景 | 是否用品牌 | 替代方案 |
|------|-----------|---------|
| 单位/货币/ID 混用风险高 | ✅ 用 | 运行时包装类 |
| 表单字段（email/username） | 可选 | 校验函数返回收窄类型 |
| 临时 DTO 形状区分 | ❌ 过重 | 具名接口即可 |
| 跨模块公共 API 边界 | ✅ 用 | — |
| 项目内部局部变量 | ❌ 过重 | 直接用原始类型 |

★ **关键认知**：品牌类型是**零运行时开销的防呆锁**，它的经济学与单元测试相反——越是"编译器本该拦住却拦不住"的低级混用（米/秒、元/分、用户 ID/订单 ID），收益越大。与 Go 对照：Go 的 `type Meters float64` 天然就是名义类型（自定义类型不与底层类型互赋），不需要品牌；TS 反其道而行之选择了结构化，再用品牌补课——两种设计各有代价：结构化换来了鸭子类型的灵活（mock、适配器零成本），名义换来了安全。理解了这座天平，才算真正理解 TS 的类型哲学。
