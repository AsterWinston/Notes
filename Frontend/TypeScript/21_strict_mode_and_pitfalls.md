strict 一揽子开关是 TypeScript 质量的分水岭：开对了，编译器替你拦下大半运行时崩溃；不开，类型系统只剩一层自欺欺人的装饰。本篇逐开关拆解它们各拦什么错误，整理高频报错的修复套路，并给出"静态类型 ≠ 运行时校验"的边界防御写法。

# 1. strict 家族逐开关详解

## 1.1 总闸与分工

`"strict": true` 是**总闸**，一次性开启下面全部子开关；也可以逐个关掉某个子项（不推荐）。各开关的职责分工：

| 开关 | 拦截什么错误 |
|------|-------------|
| `strictNullChecks` | 可能为 null/undefined 的值被当确定值使用 |
| `noImplicitAny` | 参数/变量隐式推断成 any |
| `strictFunctionTypes` | 函数参数位置的不安全**逆变**赋值 |
| `strictBindCallApply` | call/apply/bind 的参数类型失守 |
| `strictPropertyInitialization` | 类字段声明后未初始化 |
| `noImplicitOverride` | 重写父类成员忘写 `override` |
| `noFallthroughCasesInSwitch` | switch 分支忘写 break 穿透 |
| `useUnknownInCatchVariables` | catch 变量默认是 any |
| `noUncheckedIndexedAccess` | 索引访问返回值可能是 undefined |

## 1.2 strictNullChecks：拦住 JS 头号崩溃

```typescript
// ===== 反例（关闭 strictNullChecks 时编译通过，运行时炸）=====
function getLen(user: { name?: string }) {
  return user.name.length;             // name 可能是 undefined！
}

// ===== 修复：显式收窄后再用 =====
function getLenSafe(user: { name?: string }): number {
  if (user.name === undefined) return 0;   // 判空收窄（类型收窄手段之一）
  return user.name.length;                 // 此处 name 已收窄为 string
}

// 可选属性 name?: string 的真实类型是 string | undefined
```

★ **关键认知**：JS 里 `Cannot read properties of undefined` 是运行时崩溃的头号来源，`strictNullChecks` 正是对着它去的。开启后 `null`/`undefined` 成为需要显式处理的独立类型，`string` 与 `string | undefined` 不再混同——这是 TS 从"类型注解工具"升级为"真类型系统"的标志性一步。⚠ 项目后期再补开它，存量代码可能一次冒出成百上千报错，所以第 1 篇才强调"新项目第一天就开 strict"。

## 1.3 noImplicitAny 与 strictFunctionTypes

```typescript
// ===== noImplicitAny 反例：参数推断不出类型，悄悄变 any =====
function add(a, b) {       // ✗ TS7006: 'a' implicitly has an 'any' type
  return a + b;
}
// 修复：补注解，或让调用点能推断（泛型）
function add2(a: number, b: number) { return a + b; }   // 显式注解

// ===== strictFunctionTypes 反例：参数逆变位置的危险赋值 =====
type Handler = (e: { id: number }) => void;
const risky: Handler = (e: { id: number; name: string }) => {
  console.log(e.name);        // 假设了更多参数字段，调用方并不保证
};
// ✗ 开启后报 TS2322：参数类型 { id: number } 不逆变兼容 { id; name }

// 修复：收窄回调的参数需求，或改用方法声明（方法参数是双变的）
const fine: Handler = (e) => console.log(e.id);   // 只依赖声明的字段
```

★ **关键认知**：`noImplicitAny` 堵住"忘记写类型 → 变 any → 类型检查失效"的暗道；`strictFunctionTypes` 则落实**函数参数逆变**——回调被传入的参数只会"更少"不会"更多"，允许上面的 `risky` 等于允许运行时 `e.name` 不存在（见第 13 篇变异性）。⚠ 注意例外：**方法简写声明**（`interface { m(e: T): void }`）的参数仍是双变（历史兼容），只有函数属性写法 `m: (e: T) => void` 才受逆变约束。

## 1.4 strictPropertyInitialization 与 strictBindCallApply

```typescript
// ===== strictPropertyInitialization：字段声明必须初始化 =====
class User {
  name: string;              // ✗ TS2564: Property 'name' has no initializer
  age = 0;                   // ✅ 有初始值
  email!: string;            // ⚠ 非空断言"骗过"编译器：延迟初始化时才用
  constructor(name: string) {
    this.name = name;        // 修复一：构造器里赋值
  }
}

// ===== strictBindCallApply：call/apply/bind 参数检查 =====
function pick(a: number, b: string): string {
  return `${a}-${b}`;
}
pick.call(null, 1, "x");        // ✅ 参数类型对得上
// pick.call(null, "1", 2);     // ✗ TS2345: call 的实参类型不匹配
```

★ **关键认知**：`strictBindCallApply` 把 `Function.prototype.call/apply/bind` 从 `any` 黑洞里救出来——它按原函数签名推导出"还差几个参数、各是什么类型"，让 `fn.call` 与 `fn` 受同一份契约约束。`strictPropertyInitialization` 则强制"对象创建完毕时字段必有值"，把"忘了初始化"从运行时 undefined 提前到编译期。⚠ `email!: string` 这种非空断言是合法出口（如框架延迟注入），但滥用等于局部关闭检查，见 3.2 节。

## 1.5 其余开关速览

```typescript
// ===== noImplicitOverride：重写父类成员必须写 override =====
class Base {
  greet(): void { console.log("hi"); }
}
class Child extends Base {
  override greet(): void { console.log("hello"); }   // ✅ 显式声明意图
  // greet(): void {}    // ✗ TS4114: 缺少 override 修饰符
}

// ===== noFallthroughCasesInSwitch：禁止 case 穿透 =====
function f(x: number): string {
  switch (x) {
    case 1:
      return "one";                  // ✅ 有 return 不算穿透
    case 2:
      console.log("two");            // ✗ TS7029: fallthrough（下个 case 无 break）
    case 3:
      return "three";
    default:
      return "other";
  }
}

// ===== useUnknownInCatchVariables / noUncheckedIndexedAccess =====
try {
  JSON.parse("{");
} catch (e) {                        // e 的类型是 unknown（不是 any）
  if (e instanceof Error) console.log(e.message);   // 先收窄再用
}

const xs = [1, 2, 3];
const first: number | undefined = xs[0];  // 开启后索引访问可能 undefined
if (first !== undefined) console.log(first.toFixed(0));  // 判空后收窄
```

★ **关键认知**：这三个开关管的是"**意图显性化**"——`override` 让重写意图可搜索、可被父类改名波及检出；`noFallthroughCasesInSwitch` 消灭 JS switch 最阴的隐式穿透；`useUnknownInCatchVariables` 与 `noUncheckedIndexedAccess` 则把两处"运行时天然可能缺失"的位置（异常对象、数组下标）老实地标成 `unknown`/`undefined`。⚠ `noUncheckedIndexedAccess` 对字典式代码影响很大（每处取值都要判 undefined），建议新项目开、老项目单独评估。

# 2. 高频报错清单

| 报错号 | 典型触发场景 | 修复套路 |
|--------|-------------|---------|
| TS2322 | 类型不兼容赋值（`let a: string = 1`） | 修正右值类型，或改声明类型 |
| TS2339 | 访问对象上不存在的属性（拼错/真没有） | 核对属性名；可辨识联合先收窄分支 |
| TS2345 | 函数实参类型/数量不匹配 | 对齐签名；检查联合类型是否漏收窄 |
| TS2531/TS2532 | 对象可能 null/undefined | 判空收窄、可选链 `?.`、非空断言（慎用） |
| TS7006 | 参数隐式 any | 补类型注解 |
| TS18046/TS2571 | 对 unknown/any 变量直接取属性 | `typeof`/`instanceof` 收窄后再用 |
| TS2551 | 拼写相近的属性（想写 name 写了 nmae） | 按提示改成正确拼写 |
| TS2352 | `as` 断言的两个类型重叠太少 | 先断言到中间层（`as unknown as T`）或改正类型 |
| TS2304 | 找不到名字（少 import / 少声明） | 补 import 或声明文件 |
| TS2564 | 属性未初始化 | 构造器赋值 / 初始值 / `!`（慎用） |
| TS2589 | 类型实例化过深（类型体操爆栈） | 见第 19 篇递归深度限制 |
| TS1272 | 严格模式下 override 检查报错 | 补 `override` 修饰符 |

★ **关键认知**：报错号的本质是"**错误分类索引**"——记住十来个高频号，配合编辑器红色波浪线能秒级定位问题类别，比逐字读完整英文报错快得多。⚠ TS2551 值得单独说：TS 会比对"你写的名"和"存在的名"的编辑距离，这类报错永远别用 `as` 绕过——绕过等于把拼写错误带进运行时。

# 3. 常见陷阱专题

## 3.1 any 治理：污染传播与替换路线

```typescript
// any 的"污染"：一处 any，处处 any，类型检查形同虚设
function parse(input: any) {        // 入口是 any
  return input.value.toFixed(2);    // 这里不报错……
}
const r = parse("not a number");    // 运行时才炸：s.value 是 undefined
console.log(r.length);              // r 也被推断成 any，继续放行

// 替换路线：unknown 入口 + 运行时收窄
function parse2(input: unknown): string {
  if (typeof input === "object" && input !== null && "value" in input) {
    const v = (input as { value: unknown }).value;   // 断言出字典再读
    if (typeof v === "number") return v.toFixed(2);  // 真正的运行时检查
  }
  throw new Error("bad input");                      // 不合法数据当场拒绝
}
```

★ **关键认知**：`any` 的毒性在于**双向逃逸**——它既能赋给任何类型（逃出去），也接受任何赋值（放进来），任一处 any 都会顺着调用链把整条链的类型检查变哑。治理策略是"**从边界往里挤**"：外部数据入口先改 `unknown`，配合收窄逐步恢复类型，最后才清掉内部的 any。与 Java 对比：Java 没有 any 这种"关闭检查"的逃生门，泛型警告也只是警告——TS 的 any 是更彻底的自我缴械。

## 3.2 非空断言 `!` 的滥用

```typescript
// ⚠ 用 ! 等于对这一行局部关闭 strictNullChecks：
const el = document.querySelector("#app")!;   // 断言"一定不为 null"
el.innerHTML = "hi";                          // 若元素不存在：运行时炸

// ✅ 正确姿势：收窄分支给出真实保证
const el2 = document.querySelector("#app");
if (el2) el2.innerHTML = "hi";                // 判空后类型自动收窄
```

★ **关键认知**：`!` 是**承诺**而非**检查**——它只让编译器闭嘴，运行时什么都没做。合法使用场景极少：框架延迟初始化字段（`email!: string`）、测试里确定存在的 fixture。代码评审时见到 `!` 都应多看两眼；与 Go 的 `_ = err`（显式忽略错误）相比，`!` 更危险，因为它是把"未知"直接标成"已知"。

## 3.3 可选链与空值合并的正确姿势

```typescript
const config: { db?: { host?: string } } = {};

// ✅ 可选链逐层防 undefined，空值合并给默认值
const host = config.db?.host ?? "localhost";   // host: string

// ⚠ 混淆坑：?? 与 || 的区别
const port1 = 0 ?? 8080;    // 0     —— ?? 只拦 null/undefined
const port2 = 0 || 8080;    // 8080  —— || 把 0 当假值放行了默认值
```

★ **关键认知**：`?.` 的返回类型会"传染"——链上任何一环可能缺，整个表达式就是 `T | undefined`，`??` 正好接住。⚠ 两个经典坑：`a?.b.c` 里 `a` 为 undefined 时短路，但 `a` 存在而 `b` 为 undefined 时照样炸；`??` 只认 null/undefined，`||` 认所有假值（`0`/`""`/`false`），配置合并场景用 `??` 才对。

## 3.4 接口不校验运行时数据、类型断言欺骗编译器

```typescript
// ⚠ interface 只是编译期承诺，对运行时数据零约束
interface User { id: number; name: string }
async function load(): Promise<User> {
  const res = await fetch("/api/user");
  return (await res.json()) as User;    // 断言！服务端少字段不会有任何报错
}

// ⚠ as 能把任意类型"说成"目标类型，编译器信了，运行时没变
const n = "123" as unknown as number;   // 双重断言：零转换，纯谎言
// n.toFixed(0)                        // 运行时：n.toFixed is not a function
```

★ **关键认知**：`as` **类型断言**只改写编译器的认知，不改写内存里的值——它是"说服编译器"的工具而不是"转换数据"的手段。真正需要转换时写运行时转换代码（`Number(x)`、校验函数），断言只应用于"编译器无法证明但你确有把握"的少数位置。Java 里的 `(Type) obj` 强转至少保留运行时检查（失败抛 ClassCastException），TS 的 `as` 连这层保险都没有。

## 3.5 `{}` 与 `object` 的坑

```typescript
let a: {};        // 除 null/undefined 外什么都收（包括原始类型！）
a = 42;           // ✅ 编译通过
a = "hi";         // ✅ 编译通过

let b: object;    // 只收对象（含数组/函数），但拿不到任何属性
b = { x: 1 };     // ✅
// b.x           // ✗ TS2339: 属性 x 不存在于 object 上

// ✅ 正确姿势：需要什么形状就写什么形状
let c: Record<string, unknown>;   // 明确表达"字符串键的字典"
c = { x: 1 };
console.log(Object.keys(c));      // 用途清晰，可安全遍历
```

★ **关键认知**：`{}` 的语义是"**非空**"而不是"空对象"，`object` 的语义是"**非原始**"但不可操作——两者都不描述对象长什么样，属于"挡了一半的门"。工程上一律用具体对象类型或 `Record<string, unknown>` 表达意图，别用这两个模糊类型。

## 3.6 枚举的坑（回顾第 8 篇）

```typescript
enum Direction { Up, Down }        // 数字枚举：双向映射 + 类型不安全

function move(d: Direction) { ... }
move(Direction.Up);                // ✅
move(0);                           // ⚠ 数字枚举居然也通过（任意 number 可赋）
move(999);                         // ⚠ 同样通过！运行时是无意义的值

// ✅ 替代：as const 对象 + 字面量联合（第 8 篇结论）
const Dir = { Up: "UP", Down: "DOWN" } as const;   // 冻结字面量
type Dir = (typeof Dir)[keyof typeof Dir];         // "UP" | "DOWN"
```

★ **关键认知**：数字枚举的"任意 number 可赋值"与反向映射是两大暗坑，`const enum` 又与 `isolatedModules` 冲突；**字面量联合 + `as const`** 几乎在所有维度上更优。⚠ 唯一保留枚举的理由是与既有枚举代码兼容，新代码没有理由再写数字枚举。

# 4. 静态类型 ≠ 运行时校验

## 4.1 信任边界在哪里

```
  外部数据（不可信）                     类型安全区（可信）
 ┌─────────────────────┐              ┌─────────────────────┐
 │ fetch 的 JSON 响应   │              │ 程序内部对象          │
 │ 表单/URL 输入        │   运行时校验  │ 函数间传递的值        │
 │ localStorage 读取    │  ──────────► │ （编译期已保证形状）  │
 │ 第三方 postMessage   │  "信任边界"   │                     │
 └─────────────────────┘              └─────────────────────┘
   类型标注帮不上忙                      interface 在这里生效
```

★ **关键认知**：**interface/type 只在编译期生效**（第 1 篇的**类型擦除**），跨过网络、磁盘、用户输入的数据不带任何类型信息。"给 JSON 断言成 User 就安全了"是 TS 最大的认知陷阱——静态类型是编译期契约，运行时校验是边界门卫，两者接力才能闭环。

## 4.2 手写极简校验函数

```typescript
interface User { id: number; name: string }   // 静态类型：编译期承诺

// 运行时校验器：返回类型谓词 is User，把 unknown 收窄成 User
function isUser(data: unknown): data is User {
  if (typeof data !== "object" || data === null) return false;  // 先排除非对象
  const d = data as Record<string, unknown>;                    // 断言出字典读字段
  return typeof d.id === "number"                               // 逐字段检查
      && typeof d.name === "string";
}

// 解析入口：只有校验通过才敢当 User 用
function parseUser(input: string): User | null {
  const data: unknown = JSON.parse(input);      // 外来数据一律 unknown
  return isUser(data) ? data : null;            // 守卫收窄后自动变 User
}

const u = parseUser('{"id":1,"name":"Alice"}');
if (u) console.log(u.name.toUpperCase());      // ✅ 名副其实的 User
```

★ **关键认知**：**类型守卫（`data is User`）是静态类型系统与运行时数据之间的桥**——校验函数把"我看过了，确实是 User"这个运行时事实翻译给编译器听，之后的代码才能合法使用 User 的字段。这正是第 3 篇类型收窄在数据边界上的实战形态。生态里 zod/valibot 等库把"声明 schema → 运行时校验 → 推导静态类型"做成了工业方案，底层思想与此完全相同。

★ **关键认知（全篇收束）**：strict 开关的本质是把"运行时才会暴露的不确定性"提前到编译期显式表达；而类型断言、非空断言、any 这三样"逃生门"，每一个都在用未来的运行时风险换眼前的编译通过。写 TS 的成熟度，就体现在使用逃生门的克制上。
