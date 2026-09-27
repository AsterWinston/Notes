TypeScript 是 JavaScript 的超集，它在 JS 之上加了一层静态类型系统，让编译器在代码运行之前就揪出类型错误。本篇讲清 TypeScript 的定位、tsc 的编译原理，并给出一套可照抄执行的环境搭建命令，为整个系列打好地基。

# 1. TypeScript 是什么

## 1.1 一句话定位

TypeScript = JavaScript + **静态类型系统**。它由微软开发并开源，任何合法的 JavaScript 代码都是合法的 TypeScript 代码——你可以把 `.js` 代码原样粘进 `.ts` 文件直接编译通过。

```typescript
// 这是合法的 TypeScript：和 JavaScript 一模一样（类型注解是可选的）
let name = "Alice";          // 未写注解，自动推断为 string

function add(a: number, b: number) {
  return a + b;              // 返回值未标注，自动推断为 number
}
```

★ **关键认知**：TypeScript 不是一门"新语言"，而是给 JavaScript 加了**编译期的类型层**。类型注解是可选的语法扩展，编译后被完全丢弃。这与 Java/Kotlin 这类"编译产物仍携带类型信息（字节码元数据）"的语言有本质区别。

## 1.2 类型只在编译期存在：类型擦除

```typescript
// ===== 编译前 greet.ts =====
function greet(name: string): string {
  return `Hello, ${name}`;
}

// ===== 编译后 greet.js =====
// 所有类型注解被"擦除"，产出就是普通 JS
function greet(name) {
  return `Hello, ${name}`;
}
```

★ **关键认知**：TS 的类型是**零运行时开销**的——它们只活在编译期，编译产物里没有任何类型痕迹。⚠ 由此有一个必须刻进脑子的推论：**运行时拿不到类型信息**。`interface`、`type` 在运行时根本不存在，你无法"反射"出一个参数是不是 string。凡是需要运行时类型判断（比如校验接口返回的 JSON），都必须手写校验代码（见第 21 篇）。

与 Java 的对比要分清两个概念：

| 语言 | 擦除范围 | 运行时还能拿到类型吗 |
|------|---------|-------------------|
| Java | 只擦**泛型参数**（类型擦除） | 能：类名、字段、方法签名可反射读取 |
| TypeScript | 擦除**全部类型注解** | 不能：interface/type 完全消失 |

## 1.3 结构化类型系统（鸭子类型）

```typescript
interface Point {
  x: number;
  y: number;
}

class Vector2 {
  constructor(public x: number, public y: number) {}
}

// Vector2 从没写过 "implements Point"，但结构上兼容，直接赋值通过
const p: Point = new Vector2(1, 2);
```

★ **关键认知**：TS 判断类型兼容看的是**结构**（有哪些属性、各是什么类型），而不是**名字**。这叫**结构化类型（structural typing）**，俗称鸭子类型；Java/C# 用的是**名义类型（nominal typing）**——两个字段完全相同的类，名字不同就不能互赋值。TS 里可以。

⚠ 结构化类型也有代价：两个语义完全不同但结构相同的类型（比如 `Meters` 和 `Seconds` 都是 `{ value: number }`）会被视为同一类型，编译器拦不住你把"米"当"秒"传。可以用**品牌类型（brand）**模拟名义类型，技巧见第 13 篇。

四种语言的类型系统横向对比：

| 维度 | JavaScript | TypeScript | Java | Go |
|------|-----------|-----------|------|-----|
| 类型检查时机 | 无（动态） | 编译期 | 编译期 | 编译期 |
| 类型在运行时存在 | — | 否（全擦除） | 是（反射） | 部分（reflect 包） |
| 类型兼容规则 | — | 结构化 | 名义 | 结构化（接口隐式实现） |
| 类型注解 | 无 | 可选 | 必写 | 必写 |
| 泛型实现方式 | 无 | 编译期类型展开 | 类型擦除 | 单态化 + 字典 |

# 2. tsc 编译器工作原理

## 2.1 编译流水线

```
 .ts 源码
    │
    ▼
 ┌─────────┐    ┌─────────┐    ┌─────────┐
 │ Scanner │ →  │ Parser  │ →  │ Binder  │   词法 → 语法树 → 符号表
 └─────────┘    └─────────┘    └─────────┘
                                    │
                                    ▼
                             ┌─────────────┐
                             │   Checker   │   类型检查（报错都在这一步）
                             └─────────────┘
                                    │
                                    ▼
                             ┌─────────────┐
                             │ Transformer │   语法降级到 target（类→函数等）
                             └─────────────┘
                                    │
                                    ▼
                             ┌─────────────┐
                             │   Printer   │   输出 .js / .d.ts / .js.map
                             └─────────────┘
```

★ **关键认知**：tsc 的**类型检查（Checker）与代码生成（Transformer + Printer）是两个独立阶段**。这带来两个工程上的关键能力：一是可以只检查不产出（`npx tsc --noEmit`，CI 常用）；二是可以让别的工具（esbuild、swc）只做产出不做检查——现代前端工程正是这样把"类型检查"和"转译"拆给两个工具的（见第 17 篇）。

## 2.2 语法降级（downlevel）与 API 的区别

```typescript
// 源码用了类字段与可选链（ES2020 语法）
class User {
  name?: string;
  getName() {
    return this.name?.toUpperCase();
  }
}

// target: "ES5" 时的产出：类被降级成"函数 + 原型方法"，可选链被展开
var User = /** @class */ (function () {
  function User() {}
  User.prototype.getName = function () {
    var _a;
    return (_a = this.name) === null || _a === void 0
      ? void 0 : _a.toUpperCase();
  };
  return User;
}());
```

★ **关键认知**：`target` 决定的是**语法降级**的下限——TS 会把新语法（类、async/await、可选链、展开运算符）翻译成目标版本支持的等价旧写法。⚠ 但**新 API 绝不会被自动补上**：`target: "ES5"` 下用 `Promise`、`Array.prototype.flat` 照样在旧引擎上崩，需要自己引 polyfill。注意 `lib` 选项只影响**类型检查时认不认这些 API**，它不注入任何运行时代码——这是新手最常误解的一点。

## 2.3 一次编译，多种产出

| 产出文件 | 开关选项 | 用途 |
|---------|---------|------|
| `.js` | 默认产出 | 交给 Node/浏览器运行 |
| `.d.ts` | `declaration` | 给库的使用者提供类型（发布 npm 包必备） |
| `.js.map` | `sourceMap` | 调试时把报错栈映射回 `.ts` 源码行 |
| `.d.ts.map` | `declarationMap` | 使用者"跳转到定义"能回到你的 `.ts` 源码 |

# 3. 开发环境搭建

## 3.1 可照抄执行的命令清单

```powershell
# ===== 1. 切换 npm 到国内镜像（npmmirror），下载提速 =====
npm config set registry https://registry.npmmirror.com
npm config get registry        # 确认输出 https://registry.npmmirror.com/

# ===== 2. 初始化项目并安装 TypeScript（装进 devDependencies） =====
npm init -y
npm install -D typescript

# ===== 3. 生成 tsconfig.json（自带全部选项的注释说明） =====
npx tsc --init

# ===== 4. 编译一次：按 tsconfig 的 outDir 产出 .js =====
npx tsc

# ===== 5. 监听模式：保存即重新编译（开发时常用） =====
npx tsc --watch

# ===== 6. 只做类型检查、不产出文件（CI / 提交前检查必用） =====
npx tsc --noEmit

# ===== 7. 直接运行 .ts（tsx：即时转译执行，免去先编译一步） =====
npm install -D tsx
npx tsx src/index.ts
```

★ **关键认知**：`npx tsc` 执行的是**项目本地**安装的 TypeScript（`node_modules/.bin/tsc`），而不是全局版本——这保证了团队成员与 CI 用同一版本编译器，避免"我本地能过"的经典纠纷。⚠ 如果你全局装过 `typescript`，`tsc -v` 与 `npx tsc -v` 可能显示不同版本，排查编译器行为差异时一律以 `npx tsc -v` 为准。

## 3.2 第一个 TypeScript 程序

```typescript
// src/index.ts
// 带完整类型注解的问候函数：参数类型 + 默认值 + 返回值类型
function greet(name: string, times: number = 1): string {
  return `Hello, ${name}!`.repeat(times);
}

// 调用处无需任何注解，返回值类型自动推断为 string
const message = greet("TypeScript", 2);
console.log(message);

// greet(42);
// ✗ 编译错误 TS2345: Argument of type 'number' is not assignable
//   to parameter of type 'string'.
```

★ **关键认知**：TS 的价值在**调用处**兑现——函数签名就是契约，传错类型在编译期就被拦下，而不是运行时抛出 `undefined.toUpperCase()` 这种深夜告警。与 Java 相比，TS 的契约更轻：返回类型可推断、参数可选可默认，样板代码少一个量级。

## 3.3 三种运行方式对比

| 方式 | 命令 | 类型检查 | 速度 | 适用场景 |
|------|------|---------|------|---------|
| 先编译后运行 | `npx tsc && node dist/index.js` | 有 | 慢（全量编译） | 发布产物、需要 sourcemap 调试 |
| 即时转译运行 | `npx tsx src/index.ts` | 无（仅转译） | 快 | 日常脚本、本地开发 |
| Node 原生剥离 | `node --experimental-strip-types src/index.ts` | 无 | 快 | Node ≥ 22.6 的试验特性 |

⚠ tsx、esbuild 这类工具**不做类型检查**——它们只把类型擦掉就执行。类型错误必须靠 `npx tsc --noEmit` 把关，建议把它塞进 CI 与 git pre-commit hook，否则类型系统等于白开。

# 4. tsconfig 初识

## 4.1 覆盖 90% 场景的最小配置

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",          // 产出 JS 的语法下限
    "module": "ESNext",          // 模块格式
    "moduleResolution": "bundler", // 模块解析策略（详见第 16 篇）
    "strict": true,              // 打开全部严格检查（强烈建议）
    "outDir": "dist",            // 产出目录
    "rootDir": "src",            // 源码根目录
    "sourceMap": true,           // 产出 sourcemap，便于调试
    "skipLibCheck": true         // 跳过 .d.ts 检查，大项目显著提速
  },
  "include": ["src"]             // 参与编译的文件范围
}
```

★ **关键认知**：tsconfig 里最重要的一个决定不是 target 也不是 module，而是 **`strict: true`**。它不是单个开关，而是一组严格检查的"总闸"（`strictNullChecks`、`noImplicitAny` 等，详见第 16、21 篇）。⚠ 项目成型之后再补开 strict 的成本极高——存量代码可能一次冒出成百上千个报错。**新项目第一天就该打开**，这是 TS 项目少有的"不可逆决策"。

## 4.2 为什么必须开 strict：一个真实翻车现场

```typescript
// ===== strictNullChecks 关闭时：看起来毫无问题 =====
function getUserName(user: { name?: string }) {
  return user.name.toUpperCase();   // 编译通过！
}
getUserName({});                    // 运行时炸：Cannot read properties of undefined

// ===== strictNullChecks 打开后：编译期就拦住 =====
// ✗ 编译错误 TS2532: Object is possibly 'undefined'.
```

★ **关键认知**：JavaScript 里最高频的两类崩溃——`xxx is not a function` 与 `Cannot read properties of undefined`——后者正是 `strictNullChecks` 的靶子。可选属性 `name?: string` 的**真实类型是 `string | undefined`**，只有 strict 家族开启时这个 `undefined` 才被认真对待；不开的话它被悄悄抹掉，等于把地雷埋到运行时。

# 5. 学什么、怎么学

## 5.1 TypeScript 的收益与代价

| 收益 | 代价 |
|------|------|
| 编译期拦截大量低级错误 | 类型系统本身有学习门槛 |
| 重构安全：改一处签名，全项目报错点自动浮现 | 复杂类型的报错信息晦涩难读 |
| 代码即文档：接口与函数签名自描述 | 类型体操过度会拖累可维护性 |
| IDE 补全/跳转/重命名精准可靠 | 无类型的 JS 库需要补声明文件 |

## 5.2 本系列阅读路径

| 阶段 | 篇目 | 目标 |
|------|------|------|
| 基础 | 2 ~ 6 | 类型注解与推断、收窄、函数/对象/数组类型 |
| OOP 与泛型 | 7 ~ 13 | 类、枚举、泛型、条件/映射类型、兼容性规则 |
| 工程化 | 14 ~ 18 | 模块、声明文件、tsconfig、构建发布、异步类型 |
| 高级 | 19 ~ 23 | 类型体操、装饰器、严格模式陷阱、类型测试、面试 |

★ **关键认知**：TypeScript 的学习曲线"两头陡、中间平"——基础语法一周可上手，strict 报错排解与类型体操才是深水区。建议严格按上表顺序推进，**每篇的代码示例亲手敲一遍**：类型系统的直觉只能从一条条报错里练出来，看懂和会用之间隔着一百次编译失败。