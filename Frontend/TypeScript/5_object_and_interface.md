对象类型是 TS 使用频率最高的类型构造，而 interface 与 type 各有不可替代的领地。本篇讲清对象类型字面量、可选与只读属性、索引签名、接口声明合并这四大机制，并给出 type vs interface 的取舍框架与交叉类型的边界。

# 1. 对象类型字面量

## 1.1 形状即类型

```typescript
// 无需类载体：对象类型字面量直接描述"形状"
function dist(
  a: { x: number; y: number },
  b: { x: number; y: number },
) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

// 只要结构满足即可传入（结构化类型）
dist({ x: 0, y: 0 }, { x: 3, y: 4 });

// ⚠ 对象字面量直接传参会做"多余属性检查"，多一个字段就报错
// dist({ x: 0, y: 0, z: 0 }, { x: 3, y: 4 });   // ✗ z 不在形状里
```

★ **关键认知**：对象类型是**结构**的描述而非类的附属——这是第 1 篇结构化类型系统的落地。多余属性检查只对**新鲜的对象字面量**生效（编译器能确定它是"现写现用"的），存进变量再传则只做结构匹配，多出的属性不报错。⚠ 这个不对称经常造成困惑："变量赋值行、字面量赋值不行"不是 bug，是防拼写错误的新鲜度检查。

## 1.2 可选属性

```typescript
interface User {
  name: string;
  age?: number;       // 可有可无
}

const u: User = { name: "Alice" };
// u.age.toFixed();   // ✗ age 可能不存在

// 读取前必须处理 undefined：真值收窄（见第 3 篇）或 ?? 兜底
const age = u.age ?? 0;
```

★ **关键认知**：可选属性的真实类型是 `T | undefined`——`?` 只是"可能不出现"的语法糖，⚠ 在 `strictNullChecks` 下读出来就是 `number | undefined`，不做任何收窄直接用会被编译器拦住。对比 Java：Java 8 之前只能用 `null` + 注释表达"可选"，`Optional<T>` 还要运行时包装；TS 的 `?` 是零成本的编译期标注。

## 1.3 只读属性

```typescript
interface Config {
  readonly host: string;
  port: number;
}

const c: Config = { host: "localhost", port: 80 };
// c.host = "x";      // ✗ 编译错误：host 是只读
c.port = 443;          // ✓
```

★ **关键认知**：`readonly` 是**纯编译期约束**——⚠ 运行时对象照样可变（真要冻结得用 `Object.freeze`，而它也只冻到第一层）。`readonly` 还是**浅层**的：只读属性里的数组仍可 `push`，深只读要靠递归工具类型或 `as const`（第 6、11 篇）。语义对比：Java 的 `final` 字段是"引用不可换、内容随便改"，C++ 的 `const` 是深度传播的运行时约束，TS 的 `readonly` 是两者之外的第三种——只管编译期赋值。

## 1.4 readonly 的嵌套局限

```typescript
interface Nested {
  readonly tags: string[];
}

const n: Nested = { tags: ["a"] };
// n.tags = [];        // ✗ 编译错误：不能替换数组本身
n.tags.push("b");      // ✓ 数组内容照样可变——readonly 是浅的
```

★ **关键认知**：`readonly` 只冻结"**属性槽位**"，不深入属性值的内部结构——想让内容也只读，元素类型要自己写成 `readonly string[]`（第 6 篇）或用深只读工具类型（第 11 篇）。⚠ 这是最容易产生"假安全感"的地方：Java 的 `final List` 同样只锁引用不锁内容，C++ 的 `const` 才会深度传播——TS 与 Java 在这一点上同病相怜。

# 2. interface

## 2.1 基本用法

```typescript
interface User {
  name: string;
  greet(): string;        // 方法签名（也可写属性式 greet: () => string）
}

const u: User = {
  name: "Alice",
  greet() {
    return `Hi ${this.name}`;
  },
};
```

★ **关键认知**：interface 描述的是"**有哪些成员、各是什么类型**"，它不限定实现方式——对象字面量、类实例、函数返回值都能满足它（结构化匹配）。方法签名与属性式函数类型在**兼容性检查上有微妙差别**（方法参数是双向协变的宽松规则），第 13 篇展开。

## 2.2 接口继承

```typescript
interface Animal {
  name: string;
}
interface Pet {
  owner: string;
}

// 多继承：逗号分隔多个基接口，成员做结构合并
interface Dog extends Animal, Pet {
  bark(): void;
}

// 等价的交叉类型写法：
// type Dog = Animal & Pet & { bark(): void };
```

★ **关键认知**：`extends` 是**结构合并**，TS 接口可以多继承（Java 的接口同样支持多 `extends`，但类只能单继承 + 多实现）。⚠ 继承链上出现同名但类型不兼容的成员时直接报 TS2320——接口不像交叉类型那样"交成 never"（见第 6 节），而是早失败早纠正。

## 2.3 接口与类的协作：implements

```typescript
interface Serializable {
  toJSON(): string;
}

// 类用 implements 声明"结构上满足接口"
class Point implements Serializable {
  constructor(public x: number, public y: number) {}
  toJSON() {
    return JSON.stringify({ x: this.x, y: this.y });
  }
}

// 不写 implements 的类照样兼容（结构化类型的本色）
class P2 {
  constructor(public x: number, public y: number) {}
  toJSON() {
    return "p2";
  }
}
const s: Serializable = new P2(1, 2);   // ✓ 结构满足即可
```

★ **关键认知**：`implements` 是**结构检查**而非名义声明——它只是让编译器在类定义处提前自查"实例形状是否满足接口"，不写也不影响赋值兼容（这正是第 1 篇结构化类型的体现）。⚠ 与 Java 的 `implements` 有本质区别：Java 是名义契约（不写就不能当接口用），TS 是"写了多一层编译期自查"；且 `implements` 只校验实例成员，静态成员与私有字段不参与检查。

# 3. 索引签名

## 3.1 基本语法

```typescript
// 字符串索引签名：任意字符串键都返回 number（Go 的 map[string]number）
interface Scores {
  [name: string]: number;
}

const s: Scores = { alice: 90, bob: 85 };
s["carol"] = 77;            // ✓
// s["dave"] = "x";         // ✗ 编译错误：值类型不符
```

★ **关键认知**：索引签名是**开放字典**的类型表达——它声明"键空间是开放的，值统一为某类型"。与 Go 的 `map[string]int` 相比，TS 的索引签名纯编译期（运行时对象照旧），且可以和具体属性混用（见下）。⚠ 索引签名会让该类型的所有键访问都回落到兜底类型：`s.alice` 也是 `number`，编译器不记得你写过 `alice: 90`。

## 3.2 一致性约束与多种键类型

```typescript
interface Mixed {
  [key: string]: string | number;   // 索引签名是"兜底类型"
  id: number;                        // ✓ 具体属性必须是兜底类型的子类型
  name: string;                      // ✓
  // age: boolean;                   // ✗ TS2411：不在 string | number 范围内
}

interface NumKey {
  [index: number]: string;    // 数字索引：数组的抽象
  [key: string]: unknown;     // 字符串索引必须兼容数字索引的值类型
}
```

★ **关键认知**：两条硬规则——**具体属性必须可赋给索引签名类型**（TS2411），**数字索引的值类型必须是字符串索引值类型的子类型**（因为 JS 把数字键转成字符串存储，`obj[0]` 与 `obj["0"]` 是同一个键）。symbol 键需要独立的 `[sym: symbol]` 签名。⚠ 想要"已知键 + 动态键"混合建模时，先想清楚兜底类型够不够宽——兜底类型写窄了会连累具体属性，写宽了等于放弃检查。

# 4. 声明合并

## 4.1 同名 interface 自动合并

```typescript
interface Window {
  width: number;
}
interface Window {
  height: number;
}

// 合并结果：{ width: number; height: number }
const w: Window = { width: 800, height: 600 };
```

★ **关键认知**：**声明合并（declaration merging）**是 interface 独有的能力——同名接口（同一作用域、含跨文件）自动把成员并到一起。这是 TS 为"**开放式扩展**"预留的机制：第三方类型不够用时，无需改包源码，声明一个同名 interface 就能把字段补进去（第 15 篇扩展库类型全靠它）。⚠ type 别名同名直接报错——这是 interface 与 type 最本质的分野。

## 4.2 合并规则与坑

```typescript
interface A {
  x: string;
}
interface A {
  x: string;        // ✓ 重复声明同类型：允许（合并时视为一个）
}
// interface A { x: number; }   // ✗ TS2717：同名属性类型必须完全一致

interface B {
  f(x: string): void;
}
interface B {
  f(x: number): void;   // 函数成员特例：合并为重载（后声明的排前）
}
```

★ **关键认知**：合并的细则是"**非函数成员必须完全相同，函数成员合并成重载**"——这正是给库追加重载签名的惯用手法。⚠ 跨文件合并是隐式的：两个毫不相干的文件里的同名 interface 会悄悄并成一个，命名空间污染防不胜防，公共类型建议集中在类型文件里管理。Java 无此机制——同名类直接冲突，扩展只能走继承。

# 5. type 与 interface 的取舍

| 维度 | `interface` | `type` |
|------|------------|--------|
| 声明合并 | ✓ 同名自动合并 | ✗ 同名直接报错 |
| 继承方式 | `extends`（可多继承） | `&` 交叉 |
| 联合 / 元组 / 原始类型 | ✗ 不能 | ✓ 能 |
| 条件 / 映射 / 模板字面量类型 | ✗ | ✓ |
| 类 `implements` | ✓ | ✓（对象形状的别名也行） |
| 报错/IDE 显示名 | ✓ 直接显示名字 | 复杂类型常被内联展开 |
| 递归类型（树结构） | ✓ 可自引用 | ⚠ 递归交叉受限，需借助 interface |
| 编译性能 | 增量检查更快（结果可缓存） | 复杂交叉/递归更耗 |

★ **关键认知**：两者的核心差异是**表达力**与**可生长性**——interface 是"可生长、可合并的形状"，type 是"类型表达式的命名"。实践建议：**对象形状、对外公共 API、需要被他人扩展或声明合并的场合用 interface**；**联合、元组、函数类型、工具类型的计算结果用 type**。⚠ 团队内部风格统一远比逐个场景精挑细选重要——混用造成的认知成本比两者差异本身大得多。

# 6. 交叉类型

## 6.1 交叉与联合

```typescript
type A = { a: number };
type B = { b: string };

type AB = A & B;          // 同时具备 a 和 b（"与"）
const v: AB = { a: 1, b: "x" };

type U = A | B;           // 二选一（"或"）
```

| | `A \| B`（联合） | `A & B`（交叉） |
|------|------------------|-----------------|
| 语义 | 成员之一 | 成员兼具 |
| 可赋值条件 | 满足 A 或 B 其一 | 同时满足 A 与 B |
| 使用前 | 通常需收窄 | 直接可用全部成员 |
| 典型场景 | 状态机、请求结果 | 混入、配置叠加扩展 |

★ **关键认知**：交叉是**结构叠加**（所有成员拼在一起），联合是**成员择一**；两者是类型代数的"与"和"或"。它与 Java 的接口多实现（名义上"同时是两种类型"）形似而神不同——TS 交叉没有运行时身份，纯粹是编译期把属性并集到一起。⚠ 交叉也常用于给已有类型"打补丁"：`Config & { debug: boolean }` 就是加了一个字段的 Config。

## 6.2 交叉出 never 的场景

```typescript
// 同名属性也参与交叉：字面量相交无解 → never
type X = { kind: "a" } & { kind: "b" };   // kind: "a" & "b" ≡ never
// const bad: X = { kind: "a" };          // ✗ 这个类型永远无法构造

// 原始类型相交也是 never
type N = string & number;                  // never
```

★ **关键认知**：交叉把**同名属性也做交叉**——两个互斥的字面量类型相交化简为 `never`，整个类型变成不可构造的空类型。⚠ 真实事故通常是这样：交叉两份配置（`BaseConfig & UserConfig`），两边同名字段类型不一致，类型悄悄变成 never，报错却出现在十万八千里外的赋值处——遇到"某个类型永远构造不出来"的诡异报错，先回头检查交叉的每一份来源的同名字段。语义上这与集合论的交集一致：$\{a\} \cap \{b\} = \emptyset$。
