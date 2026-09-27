映射类型让"批量变换对象类型"成为类型层面的循环，模板字面量类型则把字符串拼接引入类型系统。二者结合，可以从一个接口派生出 Getter 名、事件名、配置键等整套类型——内置的 `Partial`/`Required`/`Record` 全是映射类型的特例。本篇拆解它们在类型层面到底"做了什么"。

# 1. 映射类型基础

## 1.1 遍历键的语法

```typescript
// 映射类型：对 T 的每个键 K，生成属性 K: F<T[K]>
type MyReadonly<T> = {
  readonly [K in keyof T]: T[K];      // 逐键复制，只加 readonly
};

interface Point {
  x: number;
  y: number;
}

type FrozenPoint = MyReadonly<Point>;   // { readonly x: number; readonly y: number }
```

★ **关键认知**：`{ [K in keyof T]: X }` 是类型层面的 **for-in 循环**：`keyof T` 提供键的联合，`K in` 逐个绑定，`T[K]`（索引访问类型）取值类型。生成的新类型与 T **结构上一一对应**，但每个属性的类型/修饰符可被改写。⚠ 映射类型产出的是**全新类型**（不是引用），修改它不会影响原接口——这是"类型变换不可变"的体现。

## 1.2 同态映射与修饰符保留

```typescript
// 同态（homomorphic）：形如 { [K in keyof T]: ... } 的映射
// 自动保留 T 的可选性与 readonly
type MyPartial<T> = { [K in keyof T]?: T[K] };   // ? 保留原可选性再加可选

interface User {
  name: string;
  age?: number;              // 本来就可选
}
type P = MyPartial<User>;
// { name?: string; age?: number }：age 的可选性被自然保留
```

★ **关键认知**：同态映射（键表达式恰好是 `keyof T`）会**继承原类型的修饰符语义**：可选的还是可选、readonly 还在，然后你追加的修饰符叠加其上。非同态映射（如 `Record<string, T>`，键是 `string` 而非 `keyof T`）则从零生成，没有"原属性"可保留。这是理解 `Partial` 与 `Record` 行为差异的钥匙。

## 1.3 与索引签名的互动

```typescript
// 带索引签名的类型：键不是具体字面量，而是 string/number 本身
interface Dict {
  [key: string]: number;        // 索引签名：任意字符串键都是 number
  known: number;
}

type DictKeys = keyof Dict;     // string | number（JS 对象键会转字符串）

// 映射类型遍历索引签名时，产出仍是索引签名形态
type Doubled = { [K in keyof Dict]: number };   // ≡ { [key: string]: number; ... }

// 想要"封闭的具体键"，声明处就得用显式键源
type Closed = Record<"a" | "b", number>;        // { a: number; b: number }
```

★ **关键认知**：`keyof` 对**索引签名**返回 `string | number`（JS 中数字键被转成字符串，两者都进键集合），对普通接口才返回字面量键的联合。映射类型遍历索引签名时无法"展开"成具体键——生成的还是 `[x: string]: ...`。⚠ 这就是"索引签名一旦写上，类型层面的键约束就塌了"的原因：想让键集合参与类型运算（as 重映射、按键过滤），声明时就必须用 `Record<K, V>` 或普通属性，索引签名只留给"键真的不可枚举"的场景。

# 2. 修饰符改写：+/- 号

## 2.1 readonly 与可选性的增删

```typescript
// + 表示"添加"（默认，可省略），- 表示"移除"
type Mutable<T> = {
  -readonly [K in keyof T]-?: T[K];   // 移除 readonly，移除可选
};

interface Config {
  readonly host: string;
  port?: number;
}
type C = Mutable<Config>;
// { host: string; port: number }：只读与可选都被剥掉

// Required 的实现原理就是 -?
type MyRequired<T> = { [K in keyof T]-?: T[K] };
```

★ **关键认知**：`+readonly/-readonly`、`+?/-?` 是**属性修饰符的代数运算**——映射类型不只是"复制属性"，还能对修饰符做增删。`Required` 的本质是"对每个键移除可选标记"，注意 `-?` 同时会把类型里的 `undefined` 剔除（`string | undefined` 在 `-?` 后变 `string`），这是它的隐蔽副作用。⚠ `-readonly` 只影响类型检查，运行时对象依然是原来那个（没有真正的解冻）。

# 3. as 子句：键的重映射

## 3.1 改名与 Getter 生成

```typescript
// as 子句（TS 4.1+）：对键本身做变换
type Getters<T> = {
  [K in keyof T as `get${Capitalize<K & string>}`]: () => T[K];
};

interface User {
  name: string;
  age: number;
}
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }
```

★ **关键认知**：`as` 让映射类型从"值变换"升级为"**键变换**"——`K in keyof T` 遍历旧键，`as 新表达式` 产出新键。`Capitalize<K & string>` 里 `K & string` 是必要的收窄（keyof 可能含 symbol/number，模板运算只吃 string），这类小技巧是类型编程的日常。⚠ 键改名后原键消失（新旧键并存需要手动写联合）。

## 3.2 键过滤（as + never）

```typescript
// as 后跟 never 会把该键"筛掉"：这就是过滤
type OnlyStringKeys<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

interface Mixed {
  name: string;
  age: number;
  email: string;
}
type S = OnlyStringKeys<Mixed>;   // { name: string; email: string }
```

★ **关键认知**：`as` 表达式求值为 `never` 时该属性被**删除**——`never` 是空联合，映射到空即不存在。这把条件类型的判断力引入了映射类型：**按值类型筛键**（只留字符串属性）、**按键名筛**（剔除 `onXxx`）都靠它。`Exclude<keyof T, K>` 配合 as 也能做同样的事（第 12 篇的 MyOmit 正是这个思路）。

# 4. 模板字面量类型

## 4.1 类型层面的字符串拼接

```typescript
// 反引号在类型位置 = 模板字面量类型
type EventName = `on${string}`;              // 任何以 on 开头的字符串
type Greeting = `hello ${"world" | "ts"}`;   // "hello world" | "hello ts"

// 与类型参数结合：自动生成事件名集合
type PropEventName<K extends string> = `${K}Changed`;
type E = PropEventName<"width" | "height">;  // "widthChanged" | "heightChanged"
```

★ **关键认知**：模板字面量类型是**类型层面的字符串插值**，且具有**分配律**：模板里出现联合类型，结果是所有组合的联合（`A|B` 参与拼接会展开为多个字面量类型）。⚠ 组合爆炸很快：两个各含 10 个成员的联合拼接会产出 100 个字面量，编译器对字面量联合的规模也有上限（超限退化为 `string`）。

## 4.2 事件名与配置键的应用

```typescript
// 类型安全的事件接口：事件名与回调参数联动
interface User {
  name: string;
  age: number;
}

type UserEvents = {
  [K in keyof User as `on${Capitalize<K>}Change`]: (value: User[K]) => void;
};
// { onNameChange: (v: string) => void; onAgeChange: (v: number) => void }

// 配置键校验：只允许已知键的路径字符串
type ConfigPath<T> = {
  [K in keyof T & string]: T[K] extends object
    ? `${K}.${ConfigPath<T[K]>}`     // 嵌套对象：递归拼出 "db.host" 形式
    : K;                             // 叶子：键名本身
}[keyof T & string];                 // 索引取值，把映射收成联合

type Path = ConfigPath<{ db: { host: string }; name: string }>;
// "db.host" | "name"
```

★ **关键认知**：模板字面量 + 映射 + 递归三件套可以把"运行时的字符串约定"（事件名、路径、CSS 键）提升为**编译期可校验的类型**。对比 Java：字符串常量做不到这一点（注解处理器才可能）；对比 Go：没有类型级字符串计算。这是 TS 类型系统独有的应用面——把"约定"变成"契约"。

## 4.3 模板字面量 + infer：类型层的正则

```typescript
// 按字面量切割字符串：infer 就是"捕获组"
type SplitOnce<S extends string> =
  S extends `${infer Head}:${infer Tail}` ? { key: Head; rest: Tail } : never;

type R = SplitOnce<"id:name:value">;   // { key: "id"; rest: "name:value" }

// 占位类别：${string} / ${number} / ${bigint} 各自约束嵌入值
type Route = `/users/${string}`;        // 用户路由前缀
const r: Route = "/users/42";           // OK
// const bad: Route = "/posts/42";      // ✗ 前缀不匹配

type Price = `$${number}`;              // 价格字符串
const p: Price = "$19.9";               // OK
// const p2: Price = "$free";           // ✗ 不是数字
```

★ **关键认知**：模板字面量里夹 `infer` 就是**类型层的正则表达式**：`${infer A}x${infer B}` 按字面量 `x` 切割，且第一个 infer **贪婪度最低**（`A` 只吃到第一个匹配处，剩余归 `B`）。`${string}`/`${number}`/`${bigint}` 是三个占位类别，参与模板时自动约束嵌入位置的值。⚠ 模板匹配只有精确的前缀/后缀/切分语义，**没有量词与回溯**——`a:b:c` 的多次切割要靠递归 + infer 组合（第 10 篇的递归模式），复杂解析别硬写一层模板。

# 5. 字符串操作类型

```typescript
// 四个内置字符串变换（类型层面，运行时零开销）
type A = Uppercase<"hello">;       // "HELLO"
type B = Lowercase<"HELLO">;       // "hello"
type C = Capitalize<"hello">;      // "Hello"（只改首字母）
type D = Uncapitalize<"Hello">;    // "hello"

// 组合出大小写不敏感的键匹配
type CaseInsensitive<T extends string> = `${Lowercase<T>}` | `${Uppercase<T>}`;
type Mode = CaseInsensitive<"read">;   // "read" | "READ"
```

★ **关键认知**：这四个类型由**编译器内建**（Checker 内部实现，不是 lib 声明的普通类型），对字面量类型逐字符做 ASCII 变换。⚠ 它们只处理 ASCII 字母（`Uppercase<"中">` 原样输出），且对联合类型自动分发（属于条件类型之外的另一处"自动展开"）。与 JS 的 `String.prototype.toUpperCase()` 对照记忆：一个是编译期类型变换，一个是运行时值变换，名字相同、世界不同。

## 5.1 组合应用：从常量表生成键空间

```typescript
const Metrics = ["cpu", "mem", "disk"] as const;   // 常量表（运行时唯一载体）

type Metric = (typeof Metrics)[number];             // "cpu" | "mem" | "disk"
type MetricKey = `metric.${Metric}` | `metric.${Uppercase<Metric>}`;
// "metric.cpu" | "metric.mem" | "metric.disk"
// | "metric.CPU" | "metric.MEM" | "metric.DISK"

const key: MetricKey = "metric.CPU";                // 键的合法性由类型保证
// const bad: MetricKey = "metric.gpu";             // ✗ 表里没有 gpu
```

★ **关键认知**：字符串操作类型常与 `as const` 常量表搭配——`typeof arr[number]` 提取字面量联合（第 8 篇的替代枚举套路），再用模板与大小写变换生成整个"键空间"。运行时只保留 `Metrics` 数组本身，键的合法性全部在编译期闭环。⚠ `Uppercase<Metric>` 让联合翻倍，常量表很长时组合爆炸的警告（4.1 节）同样适用——键空间的规模要控制在编译器预算内。

# 6. 内置工具类型的映射原理拆解

```typescript
// Record：非同态映射，凭空生成"键集合 → 值类型"的映射
type MyRecord<K extends keyof any, T> = {
  [P in K]: T;                   // K 直接给出键（联合或 string/number/symbol）
};
type R = MyRecord<"a" | "b", number>;   // { a: number; b: number }

// Partial：同态映射，逐键加可选
type MyPartial<T> = { [P in keyof T]?: T[P] };

// Required：同态映射，逐键移除可选（顺带剔除 undefined）
type MyRequired<T> = { [P in keyof T]-?: T[P] };

// Readonly：同态映射，逐键加只读
type MyReadonly<T> = { readonly [P in keyof T]: T[P] };
```

| 工具类型 | 映射形态 | 键来源 | 修饰符运算 | 保留原修饰符 |
|---------|---------|--------|-----------|-------------|
| `Record<K,T>` | 非同态 | 调用方指定 | 无 | 否（全新键） |
| `Partial<T>` | 同态 | `keyof T` | `+?` | 是（叠加可选） |
| `Required<T>` | 同态 | `keyof T` | `-?` | 是（仅删可选） |
| `Readonly<T>` | 同态 | `keyof T` | `+readonly` | 是（叠加只读） |

```plaintext
映射类型的求值模型（把类型想成"编译期的循环"）：

  type F<T> = { [K in keyof T as New<K>]: G<T[K]> }

  对 T = { a: string; b?: number } 求值：
     │
     ├─ 轮次 1：K = "a"   → as 变换键 → New<"a">   → 值类型 G<string>
     ├─ 轮次 2：K = "b"   → as 变换键 → New<"b">   → 值类型 G<number | undefined>
     │                      （可选属性的 T[K] 带 undefined）
     ▼
  合并各轮结果 → 新对象类型（同态时继承修饰符，as→never 的轮次丢弃）
```

★ **关键认知**：把映射类型理解为"编译期对键集合的 for 循环 + 可选的键改名/丢弃 + 值类型变换 + 修饰符代数"这四个可组合的动作后，所有花式写法都只是这四个动作的排列组合。`Record` 与 `Partial` 一族的本质差异是**键的来源**：前者由调用方供给（非同态），后者从原类型继承（同态）。与 Java 的注解处理器、Go 的 code generation 对照：TS 把"批量生成相似类型"的活直接内化进了类型语言，无需任何构建期工具——代价是这些计算全部压在 Checker 上，⚠ 大型映射（对含几百个键的类型做递归变换）会实打实地拖慢编译。
