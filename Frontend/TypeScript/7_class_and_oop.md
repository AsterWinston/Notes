TypeScript 的 class 在 JS class 之上补全了类型层：字段类型、修饰符、抽象类与接口实现。本篇讲清类的类型标注规则，并重点剖析 TS 类与 Java 类在本质上的差异——TS 的类兼容是**结构化类型**（鸭子类型），而非名义继承。

# 1. 类的声明与成员类型

## 1.1 字段声明与构造器

```typescript
// TS 要求字段先声明后使用（JS 的 class 字段是运行时特性，TS 补的是类型）
class Point {
  x: number;             // 声明字段并标注类型
  y: number;             // 若开了 strictPropertyInitialization，必须赋值

  constructor(x: number, y: number) {
    this.x = x;          // 构造器内完成初始化
    this.y = y;
  }

  // 方法：参数与返回值都可标注类型
  distanceTo(other: Point): number {
    return Math.hypot(this.x - other.x, this.y - other.y);
  }
}

const p = new Point(3, 4);   // 构造调用同样做类型检查
```

★ **关键认知**：`strictPropertyInitialization`（strict 总闸包含它）要求**每个声明的字段在构造完成前被赋值**，否则报 TS2564。这把 JS 里最常见的 `undefined` 来源之一——"忘了在构造器里初始化字段"——在编译期就拦掉了。⚠ 若字段确实可能没有值，应显式写 `x?: number` 或 `x!: number`（**确定赋值断言**，告诉编译器"别处会赋值"），后者是类型断言的一种，用多了会重新引入 `undefined` 风险。

## 1.2 参数属性简写

```typescript
// 参数属性：在构造器参数前加修饰符，一步完成"声明字段 + 赋值"
class User {
  constructor(
    private name: string,        // 声明私有字段 this.name 并赋值
    public readonly age: number, // 只读公共字段
    protected email: string,     // 受保护字段
  ) {}

  describe(): string {
    return `${this.name} (${this.age})`;  // 内部可访问所有参数属性
  }
}
```

★ **关键认知**：`constructor(private name: string)` 是纯**编译期语法糖**——编译后等价于"声明字段 + 在构造器第一行赋值"，运行时没有任何特殊机制（类型擦除之外，这个语法本身也会被展开）。它只是消样板的简写。⚠ 参数属性让构造器签名的可读性下降（看不出"这个参数还会被存下来"），团队规范里常约定只在简单 DTO 类中使用。

# 2. 修饰符与封装

## 2.1 三级访问修饰符

```typescript
class Animal {
  public kind: string = "animal";      // 默认就是 public，可省略
  private age = 0;                     // 仅类内部可访问
  protected name = "anon";             // 类内部 + 子类可访问

  protected grow(): void {
    this.age += 1;                     // 类内部访问 private，OK
  }
}

class Dog extends Animal {
  rename(n: string): void {
    this.name = n;                     // 子类访问 protected，OK
    // this.age = 1;                   // ✗ TS2341: 属性 "age" 是私有的
  }
}

const d = new Dog();
d.kind;                                // public，外部可访问
// d.name;                            // ✗ TS2445: 属性 "name" 受保护
```

★ **关键认知**：`public/private/protected` 全部是**编译期检查**，编译后产物里没有任何访问控制——运行时 `d.name` 照样能读到值。这一点与 Java 的 `private`（字节码层面可被反射绕过，但常规代码受 JVM 检查）完全不同，也与 C++ 的 `private`（链接期检查）不同。⚠ 这意味着"封装"在 TS 里是**约定而非强制**：恶意或粗糙的 JS 调用方可以 `(d as any).name` 直接摸到私有字段。

## 2.2 `#私有字段` vs `private`：真私有与假私有

```typescript
class Secret {
  private token = "ts-private";   // 编译期私有：运行时就是普通属性
  #key = "js-private";            // 真私有：基于 WeakMap 的运行时私有

  leak(): string {
    return this.token + this.#key;
  }
}

const s = new Secret();
(s as any).token;                 // 能读到 "ts-private"：运行时可绕过！
// s.#key;                       // ✗ 语法错误：#字段只能在类体内访问
// (s as any).#key;               // ✗ 任何路径都无法从外部访问 #字段
```

★ **关键认知**：`#field` 是 **JS 语言级的私有字段**（ES2022），它的私密性由**运行时引擎**保证——底层通常实现为每个实例关联的内部槽或 WeakMap，字段名根本不出现在对象的可枚举属性里。而 `private` 只是类型系统的记号，编译后被完全擦除（类型擦除）。两者的选择标准：需要**真正的封装边界**（防绕过、防序列化泄漏）用 `#`，只做 API 表面收窄用 `private`。

| 维度 | `private x` | `#x` |
|------|------------|------|
| 检查时机 | 仅编译期 | 编译期 + 运行时 |
| 编译产物 | 普通属性（擦除修饰符） | 引擎级私有槽 |
| `as any` 能否绕过 | 能 | 不能 |
| 子类能否访问 | 能（同名声明会冲突） | 不能（连继承都拿不到） |
| JSON.stringify | 会被序列化 | 不会被序列化 |
| 与 Java 对比 | ≈ 无运行时强制 | ≈ Java private + 不可反射 |

⚠ `#` 字段与 `private` 字段**不能相互覆盖**：子类里重新声明父类的 `private` 字段名会报错（TS 认为它们是不同属性），而 `#` 字段在子类中直接不可见。混用两种风格的类会让维护者困惑，一个类内应统一。

## 2.3 readonly 字段

```typescript
class Config {
  readonly host: string;          // 只读：只能在声明处或构造器内赋值
  readonly port = 8080;           // 声明处初始化

  constructor(host: string) {
    this.host = host;             // 构造器内赋值，OK
  }

  change(h: string) {
    // this.host = h;             // ✗ TS2540: 无法分配到 "host"，因为它是只读属性
  }
}
```

★ **关键认知**：`readonly` 同样只在编译期生效，运行时对象依然可写（`(c as any).host = "x"` 成功）。它约束的是**通过类的类型化引用的赋值路径**，与 `Object.freeze`（运行时冻结）是正交的两回事。⚠ readonly 保护的是"属性引用不被重新赋值"，不保护对象内部：`readonly items: string[]` 仍允许 `this.items.push("x")`——这与函数式语言的不可变数据结构有本质差距。

# 3. 存取器、静态成员与抽象类

## 3.1 get/set 的类型规则

```typescript
class Celsius {
  #deg: number;

  constructor(d: number) {
    this.#deg = d;
  }

  // getter 的返回值类型就是属性的类型
  get fahrenheit(): number {
    return this.#deg * 9 / 5 + 32;
  }

  // setter 只能标注参数类型，不能标注返回类型（只能是 void）
  set fahrenheit(f: number) {
    this.#deg = (f - 32) * 5 / 9;
  }
}

const c = new Celsius(100);
c.fahrenheit;          // 212：读取走 getter
c.fahrenheit = 32;     // 写入走 setter
```

★ **关键认知**：TS 对存取器有一条关键推断规则——**只声明了 getter 的属性自动推断为只读**，对它赋值会报 TS2540。另外 getter 与 setter 的类型必须兼容（setter 参数应能接受 getter 返回的类型），这是编译期的一致性检查。⚠ 与 Java 的 getX()/setX() 方法不同，TS 的 get/set 在调用侧伪装成普通属性，这会影响重构（把 getter 改成方法是破坏性变更）。

## 3.2 静态成员与静态块

```typescript
class Counter {
  static count = 0;                     // 静态字段：挂在类上，不挂在实例上
  static #secret = Math.random();       // 静态真私有字段

  static {
    // 静态初始化块（ES2022）：可写任意初始化逻辑
    console.log("类加载时执行一次");
  }

  static reset(): void {
    Counter.count = 0;                  // 静态方法内只能通过类名访问静态成员
  }

  constructor() {
    Counter.count += 1;                 // 实例方法可读写静态成员
  }
}
```

★ **关键认知**：静态成员属于**类构造函数对象本身**，不在原型链上，子类通过 `extends` 继承静态成员（`class Sub extends Counter` 后 `Sub.count` 可见）。⚠ 静态成员的类型是"类的构造函数类型"的一部分：两个类的实例结构兼容，不代表它们的静态成员兼容——这在把 `typeof Counter` 当作值传递时会暴露出来。

## 3.3 抽象类与接口实现

```typescript
// 抽象类：不能实例化，专门做基类
abstract class Shape {
  abstract area(): number;                    // 抽象方法：只有签名，子类必须实现

  describe(): string {                        // 具体方法：提供默认实现
    return `面积 = ${this.area().toFixed(2)}`;
  }
}

// 接口约束类的实例结构
interface Printable {
  print(): string;
}

class Circle extends Shape implements Printable {
  constructor(private r: number) { super(); }  // 有派生就必须调 super()
  area(): number { return Math.PI * this.r ** 2; }
  print(): string { return `Circle(r=${this.r})`; }
}

// const s = new Shape();   // ✗ TS2511: 无法创建抽象类的实例
```

★ **关键认知**：`abstract` 是编译期的"**不可实例化 + 强制实现**"约束，编译后抽象类就是普通类（JS 没有抽象类概念）。`implements` 更纯粹——它**只做结构检查**，不产生任何继承关系或运行时代码：类只要"长得像"接口就通过。⚠ 与 Java 的 `implements`（名义契约，运行时 `instanceof` 可检测）不同，TS 的 `implements` 连 `instanceof` 都不会多出任何痕迹。

# 4. 类作为类型

## 4.1 实例类型

```typescript
class Box {
  constructor(public value: string) {}
}

// 类名用在类型位置时，表示"该类的实例类型"
function takeBox(b: Box): string {
  return b.value;
}

// 等价的显式写法（InstanceType 的预告）：先取构造函数类型，再取实例类型
type BoxInstance = InstanceType<typeof Box>;   // ≡ Box
```

★ **关键认知**：**类名在类型位置指实例，在值位置指构造函数**，`typeof Box` 才是"构造函数类型"。这条双关规则是 TS 最常见的困惑源之一。`InstanceType<T>` 的实现原理（`new (...args: any[]) => infer R` 里提取 `R`）见第 10 篇的 infer 一节。

## 4.2 构造器签名

```typescript
// 描述"能被 new 的东西"的类型
type Constructor = new (x: number) => Box;

function makeBox(Ctor: Constructor): Box {
  return new Ctor(42);              // 按构造签名实例化
}

makeBox(Box);                       // 传类本身（构造函数值）OK
// makeBox({});                     // ✗ 对象字面量没有构造签名
```

★ **关键认知**：类同时携带两种类型——**实例类型**（`Box`）与**构造函数类型**（`typeof Box`，含 `new` 签名与静态成员）。写框架级代码（依赖注入、工厂）时用的是后者。⚠ 与 Go 的"接口隐式实现 + 传构造函数"的惯例相比，TS 把构造函数做成了**一等类型**，可以显式书写 `new (...) => T`，表达力更强但也更容易写出难读的签名。

# 5. 结构化类型下的类兼容

## 5.1 类兼容是"鸭子类型"

```typescript
class Duck {
  constructor(public name: string) {}
  quack(): void { console.log("quack"); }
}

class Person {
  constructor(public name: string) {}
  quack(): void { console.log("I quack too"); }
}

// Person 从未继承 Duck，但结构相同，直接互赋值
const d: Duck = new Person("Alice");   // 编译通过！
d.quack();                             // 输出 "I quack too"
```

★ **关键认知**：这是 TS 与 Java **最本质的分野**。Java 是**名义类型**：`class Person extends Duck` 之外的任何相似类都与 Duck 无关。TS 是**结构化类型**：只要成员结构匹配（属性名、方法签名兼容），类与类、类与接口就互相兼容——`implements` 只是把这个结构检查显式化，写不写都不影响兼容性。这与 Go 的接口隐式实现同源（"像鸭子就是鸭子"），但 TS 把它推广到了**类与对象**，而不只是接口。

## 5.2 私有/保护成员：结构兼容的破坏者

```typescript
class Money {
  private cents = 0;                 // 带私有成员
  constructor(public amount: number) {}
}

class Amount {
  constructor(public amount: number) {}   // 结构上只有一个 amount 字段
}

// const m: Money = new Amount(5);  // ✗ TS2322: 类型 "Amount" 与 "Money" 不兼容
```

★ **关键认知**：TS 有一条特殊规则来挽救封装性——**当源类型或目标类型声明了 private/protected 成员时，兼容性退化为"名义式"**：两个类必须**继承自同一个声明了该私有成员的类**才互相兼容。这条规则让 `private` 重新获得了类似 Java 的名义隔离效果，是结构化类型系统里少见的"名义补丁"。⚠ `#` 字段不参与这条规则也不参与结构比较（它们连反射都看不到），实际效果同样是阻断兼容。

## 5.3 与 Java/OOP 传统的对比总结

| 维度 | TypeScript 类 | Java 类 |
|------|--------------|---------|
| 类型系统 | 结构化（鸭子类型） | 名义（继承关系决定） |
| `implements` 的作用 | 显式化结构检查，无运行时痕迹 | 建立名义契约，可反射/instanceof |
| `private` | 仅编译期，可 `as any` 绕过 | JVM 级访问控制 |
| `#field` | 运行时真私有 | —（Java 无对应物） |
| 多重继承 | 单继承 + 多 `implements` | 单继承 + 多接口 |
| 类型擦除 | 全擦除（类名也只是变量） | 仅泛型参数擦除 |
| 抽象类 | 编译期约束 | 字节码级约束（不可实例化） |

```plaintext
Java 的类关系：                TS 的类关系：
  Animal                        Animal（只是一组结构签名）
    ▲ nominal                     ▲
    │ extends（血缘）              │ extends（共享结构与实现）
    │                             │
  Dog  implements Pet             Dog  implements Pet（结构检查）
                                    │
  Pet（名义契约，运行时存在）        Pet（结构契约，编译后消失）
                                    │
                                  Anything that "looks like Pet" 都算 Pet
```

★ **关键认知**：理解这一节的模型后，TS 的许多"怪现象"都顺理成章：不需要 `implements` 就能传参给接口类型（第 13 篇的兼容性规则）、用对象字面量就能替代类做 DTO、mock 测试时随手写个同结构对象即可。代价是**语义丢失**：编译器无法区分"米"和"秒"这样结构相同语义不同的类型，需要用**品牌类型**模拟名义类型（第 13 篇给出完整实现）。⚠ 从 Java 转 TS 的人最大的习惯性错误是把 `implements` 当作"运行时类型标签"来用 `instanceof` 断言——TS 里 `instanceof` 只能判断**类**（构造函数），不能判断接口。
