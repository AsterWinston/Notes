泛型让"同一段逻辑适用于多种类型"且不丢失类型信息，是 TS 从"标注工具"升级为"类型编程语言"的第一级台阶。本篇讲清类型参数的推断、`extends` 约束机制，并对照 Java 的类型擦除与 Go 的类型集合约束，说明 TS 泛型"编译期展开"的本质。

# 1. 泛型函数

## 1.1 基本语法与推断

```typescript
// T 是类型参数：调用时由实参推断，或显式指定
function identity<T>(x: T): T {
  return x;                     // 返回值类型与入参一致，由 T 统一约束
}

const a = identity(42);         // 推断 T = number，返回 number
const b = identity("hi");       // 推断 T = string
const c = identity<number>(42); // 显式传类型参数，多用于约束调用范围
// identity<number>("hi");      // ✗ 显式指定后，实参必须是 number
```

★ **关键认知**：TS 的泛型**只存在于编译期**（类型擦除），编译后的 `identity` 就是普通函数，运行时没有任何"泛型实例化"开销。调用处的 `T` 由**实参反向推断**得出，推断依据是"最小公共超类型/最佳匹配"算法（第 21 篇详述推断规则）。⚠ 与 Java 不同，TS 的泛型没有运行时的 `T.class`，无法 `new T()`、`T[]`（直接写不行）、`instanceof T`——一切运行时类型操作都与 T 无关。

## 1.2 多类型参数与推断顺序

```typescript
// 经典 pair：两个独立类型参数
function pair<A, B>(a: A, b: B): [A, B] {
  return [a, b];                // 元组类型标注保证顺序（第 6 篇）
}

const p = pair(1, "x");         // [number, string]，逐个位置推断

// 转换函数：T 是输入，U 是输出
function map<T, U>(x: T, f: (v: T) => U): U {
  return f(x);                  // U 由回调返回值推断
}

const len = map("hello", s => s.length);   // U = number
```

★ **关键认知**：类型参数按**从左到右**的顺序推断，后续类型参数可引用先推断出的（如 `f: (v: T) => U` 中 U 依赖 T）。⚠ 当推断失败时 TS 回退为 `unknown`（strict 下）或隐式 `any`（非 strict 下报 TS7006），所以"泛型写复杂了反而推断不出来"是常见现象——推断不出时宁可让调用方显式传参，也不要给类型参数乱加默认 `any`。

# 2. 泛型约束

## 2.1 extends 约束接口

```typescript
// T 必须"至少"包含 name 属性：约束是结构化类型的上界
function greet<T extends { name: string }>(x: T): string {
  return `Hello, ${x.name}`;    // 约束保证 x.name 存在，可安全访问
}

greet({ name: "Alice", age: 3 });   // 多余属性 OK：约束是"下限"不是"全等"
// greet({ age: 3 });               // ✗ 缺少 name 属性
```

★ **关键认知**：`extends` 约束定义的是**类型参数的上界**（upper bound）——T 可以是约束类型的子类型（更宽的结构）。这是结构化类型的自然延伸：约束 `{ name: string }` 匹配任何"含 name: string 的对象"，与 Java 泛型 `<T extends Comparable<T>>` 的名义约束形成对照。⚠ 约束只影响类型检查，运行时**不做任何校验**：传入 `{}` 并 `as any` 依然会在运行时炸。

## 2.2 keyof 约束模式

```typescript
// K 被约束为 T 的键的联合：杜绝拼错属性名
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];              // T[K] 是索引访问类型（第 5 篇）
}

const user = { name: "Alice", age: 30 };
const n = getProp(user, "name");   // string（K 推断为 "name"）
// getProp(user, "email");         // ✗ "email" 不能赋给 "name" | "age"
```

★ **关键认知**：`K extends keyof T` + `T[K]` 是 TS 最重要的**类型安全模式**之一，它把"属性名"与"返回值类型"联动起来——传 `"age"` 返回 `number`，传 `"name"` 返回 `string`。这在 JS 里只能写 `obj[key]` 返回 `any`。这套模式也是内置工具类型 `Pick`/`Record` 的底层骨架（第 11、12 篇）。

## 2.3 约束默认值

```typescript
// 给类型参数默认值：调用方不传时使用，类似函数默认参数
function createStore<T = string>(initial: T): { get(): T } {
  let value = initial;
  return { get: () => value };   // 闭包保存，返回类型仍由 T 决定
}

const s1 = createStore("a");     // T 推断为 string
const s2 = createStore<number>(1);  // 显式覆盖
const s3 = createStore();        // T 用默认值 string
```

★ **关键认知**：类型参数默认值的求值顺序是"**推断优先，推断不出用默认，没有默认报错**"。默认值可以引用前面的类型参数（`<T, U = T[]>`），但不能引用后面的。⚠ 默认值让泛型"可省略"，对库作者是提升易用性的利器；但对内部工具函数，显式传参更利于重构（默认值会让调用处的类型"看起来无参"，改默认值时影响面隐蔽）。

# 3. 泛型接口与泛型类

## 3.1 泛型接口

```typescript
// 泛型接口：类型参数在使用接口时确定
interface Repository<T> {
  findById(id: string): T | undefined;
  save(entity: T): void;
}

// 实现时固定 T = User
class UserRepo implements Repository<{ id: string; name: string }> {
  findById(id: string) { return undefined; }   // 返回值可推断
  save(entity: { id: string; name: string }) { }
}
```

★ **关键认知**：泛型接口的类型参数在**使用处**（implements/类型注解）被实例化，之后就是一个普通接口。这与 Java 的泛型接口（擦除后是原始类型 + 桥接方法）不同——TS 的 `Repository<User>` 与 `Repository<Order>` 在编译期是两个完整展开的结构，运行时没有任何 Repository 相关代码。

## 3.2 泛型类与方法

```typescript
// 泛型类：类型参数绑定在类的实例上
class Stack<T> {
  private items: T[] = [];              // 泛型字段

  push(item: T): void {                 // 方法复用类的 T
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  // 泛型方法：可声明自己的类型参数 U，与类的 T 并存
  map<U>(f: (v: T) => U): Stack<U> {
    const out = new Stack<U>();         // 实例化时显式传 U
    this.items.forEach(v => out.push(f(v)));
    return out;
  }
}

const s = new Stack<number>();          // 类型参数在 new 处确定
s.push(1);
// s.push("x");                         // ✗ T 已固定为 number
```

★ **关键认知**：⚠ 泛型字段**不能有初始化器引用 T 的实例化**以外的复杂操作，且注意 `new Stack<U>()` 在泛型方法里是合法的——因为 `Stack` 的构造函数不依赖 T（T 是编译期的）。但如果想写 `new T()`，TS 直接报错：**运行时没有 T**。这是 TS 泛型与 Java/C# 泛型最硬的边界，需要运行时工厂时应传入构造函数参数（`constructor(private factory: () => T)`）。

# 4. 常见模式实战

## 4.1 工厂函数与容器类型

```typescript
// 工厂函数：返回值类型用泛型锁定
function create<T>(factory: () => T): T {
  return factory();                     // 延迟到调用处确定 T
}

// Result 容器：用可辨识联合表达"成功/失败"（第 5 篇模式）
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function safeParse<T>(json: string): Result<T> {
  try {
    return { ok: true, value: JSON.parse(json) as T };   // 断言见第 2 篇
  } catch (e) {
    return { ok: false, error: e as Error };
  }
}

const r = safeParse<{ n: number }>('{"n":1}');
if (r.ok) {
  r.value.n;                   // 收窄后拿到 T
} else {
  r.error.message;
}
```

★ **关键认知**：`Result<T, E>` 是"泛型 + 可辨识联合"的组合拳：泛型承载成功值的类型，判别属性 `ok` 驱动**类型收窄**。Java 需要用 `Optional`/异常体系，Go 用 `(T, error)` 多返回值——TS 用联合类型做到了编译期强制处理错误分支（不收窄就拿不到 `value`）。

## 4.2 类型安全的工具函数

```typescript
// map：输入数组元素类型 T，输出 U
function mapArray<T, U>(arr: T[], f: (v: T, i: number) => U): U[] {
  return arr.map(f);                    // 复用原生 map，类型由签名保证
}

// first：返回 T | undefined，逼调用方处理空数组
function first<T>(arr: T[]): T | undefined {
  return arr[0];                        // 不谎报"一定有值"
}

const nums = [1, 2, 3];
const strs = mapArray(nums, n => n.toFixed(2));   // string[]
const f = first(nums);                            // number | undefined
// f.toFixed(2);                                 // ✗ 可能 undefined（strictNullChecks）
```

★ **关键认知**：工具函数的泛型设计原则是"**让类型从入参流向出参**"，签名比实现更重要。`first` 返回 `T | undefined` 而不是 `T`，是把 JS 的运行时不确定性诚实地反映到类型里（对比 Java `Optional<T>`、Go `(T, bool)`）。⚠ 切勿用 `as T` 把 `arr[0]` 断言成 `T` 来"消灭" undefined——那是把编译期问题推迟到运行时。

## 4.3 Box 容器与链式变换

```typescript
// Box：最简单的泛型容器，演示"类型随变换流动"
class Box<T> {
  constructor(private value: T) {}

  // 变换后返回新类型的 Box：U 由回调返回值推断
  map<U>(f: (v: T) => U): Box<U> {
    return new Box(f(this.value));
  }

  // 取出值：类型完整交还调用方
  unbox(): T {
    return this.value;
  }
}

const len = new Box("hello")   // Box<string>
  .map(s => s.length)          // Box<number>
  .map(n => n > 3)             // Box<boolean>
  .unbox();                    // boolean
```

★ **关键认知**：`map<U>` 是泛型方法的标准范式——**输入类型 T 出现在参数位置、输出类型 U 出现在返回值位置**，链式调用中每一步的类型自动传递，全程无需注解。这与 Java `Stream<R>.map(Function<T,R>)` 的类型流动完全同构，但 TS 的产物里 Box 就是普通类、泛型全部消失（零运行时成本）；对比 Go 没有方法级泛型，同样的 API 只能写成包级函数 `Map[T, U](b Box[T], f func(T) U) Box[U]`，链式体验大打折扣。

# 5. 泛型与联合类型的分布直觉

## 5.1 类型参数实例化为联合

```typescript
function wrap<T>(x: T): { value: T } {
  return { value: x };
}

// T 实例化为联合类型时，T 整体代入（这里是包装，不拆开）
const w = wrap<string | number>(1);   // { value: string | number }

// 对比条件类型里的"分布"：T extends U ? X : Y 会把联合拆开逐个判断
type ToArray<T> = T extends unknown ? T[] : never;
type A = ToArray<string | number>;    // string[] | number[]（分布了！）
```

★ **关键认知**：**泛型实例化是"整体代入"，条件类型是"分布展开"**——这是初学类型编程最容易混淆的直觉差异。`wrap<string | number>` 的 T 就是整个联合，而条件类型遇到裸类型参数会自动分发（distributive）。分布的机制、阻断技巧（`[T] extends [U]`）是第 10 篇的核心内容。

# 6. 三种语言的泛型对比

| 维度 | TypeScript | Java | Go |
|------|-----------|------|-----|
| 实现方式 | 编译期类型展开（全擦除） | 类型擦除 + 桥接方法 | GCShape 单态化 + 字典 |
| 运行时开销 | 零（类型消失） | 装箱/桥接的隐性成本 | 极低（可内联） |
| `new T()` | ✗ 不可能 | ✗ 也不行（擦除） | ✗ 需约束构造 |
| 约束形态 | `extends` 结构上界 | `extends` 名义上界 | 类型集合 `int \| float64` |
| 约束默认值 | ✅ `<T = string>` | ✗ | ✗ |
| 方法级泛型 | ✅ | ✅ | ✗（只函数/类型级别） |
| 变型标注 | 协变/逆变自动推断（第 13 篇） | `? extends` / `? super` 显式 | 不支持变型标注 |
| 联合类型作参数 | ✅（且有条件类型分布） | ✗ | 类型集合可并集 |

```plaintext
泛型在编译流水线中的位置（对照第 1 篇）：

  function identity<T>(x: T): T { return x }
        │
        ▼
  Checker：为每次调用展开 T → identity(42) 检查 number → number
        │
        ▼
  Transformer：T 无运行时意义，直接丢弃
        │
        ▼
  产物：function identity(x) { return x }   ← 与手写 JS 完全一致

  对照 Java：  擦除 T 为 Object + 生成 checkcast 桥接
  对照 Go：    为 int/float64 各生成一份机器码（或共享 GCShape）
```

★ **关键认知**：TS 泛型的本质是"**编译期的类型函数**"——类型参数在 Checker 阶段被实参展开代入，产物里没有任何泛型痕迹。这带来零开销，也带来边界：**泛型无法影响运行时行为**。凡是需要运行时类型分支（序列化、工厂、反射）的场景，都必须把"类型"物化为值（构造函数、校验函数、tag），这是从 Java/C# 转过来的开发者最需要重建的直觉。⚠ 另一个隐蔽差异：Java 泛型不支持原始类型之外的特化，Go 的泛型是"值类型友好"的；而 TS 泛型的战场在类型层——它的强项是**类型变换**（条件类型、映射类型），这在另外两门语言里根本不存在对应物。
