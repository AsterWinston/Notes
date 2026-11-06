只要项目超过三个文件，就绕不开模块。Node.js 同时支持 CommonJS 与 ES Module 两套体系，它们语法不同、加载时机不同、互相引用时还有一堆细节。本篇把两套机制拆开讲清楚，再说明 `package.json` 的 `type` 字段如何决定一个 `.js` 文件的身份。

# 1. 为什么需要模块系统

## 1.1 全局变量时代的痛点

早期浏览器脚本靠全局变量协作，文件一多就会撞名：

```javascript
// a.js
var config = { debug: true };

// b.js
var config = { debug: false }; // 覆盖了 a.js 的 config
```

模块系统要解决三件事：**作用域隔离**、**显式导出**、**依赖声明**。每个文件有自己的作用域，只有明确导出的内容才能被外部使用，依赖关系写在文件顶部，一眼可见。

## 1.2 Node 的两套体系

| 体系 | 语法 | 加载时机 | 典型文件 |
|------|------|---------|---------|
| CommonJS (CJS) | `require` / `module.exports` | 运行时同步加载 | `.cjs`，默认 `.js` |
| ES Module (ESM) | `import` / `export` | 解析阶段静态分析 | `.mjs`，`type: module` 下的 `.js` |

★ **关键认知**：CommonJS 是为服务端设计的，**同步加载**——`require` 会阻塞到模块执行完毕。ESM 是语言标准，加载过程分"解析依赖图"和"执行"两个阶段，支持静态分析和 tree-shaking。

# 2. CommonJS

## 2.1 require 与 module.exports

```javascript
// math.js
function add(a, b) {
  return a + b;
}

const PI = 3.14159;

module.exports = { add, PI };
```

```javascript
// main.js
const math = require("./math.js");
console.log(math.add(1, 2)); // 3
console.log(math.PI);        // 3.14159
```

`require` 做的事情可以概括为：

```text
1. 把传入的路径解析成绝对路径
2. 检查 require.cache 是否已有该模块
3. 没有则读取文件,包进一个函数里执行
4. 把 module.exports 存入缓存并返回
```

## 2.2 exports 与 module.exports 的关系

Node 在包装模块时，做了这样一件事：

```javascript
const exports = module.exports; // exports 只是 module.exports 的引用
```

所以：

```javascript
// 正确:给同一个对象挂属性
exports.add = (a, b) => a + b;

// 正确:整体替换
module.exports = { add: (a, b) => a + b };

// 错误:exports 指向了新对象,和 module.exports 断开
exports = { add: (a, b) => a + b };
```

★ **关键认知**：`exports` 只是 `module.exports` 的**初始引用**。一旦你写 `exports = ...`，就等于把局部变量指向了新对象，而 `require` 返回的仍是旧的 `module.exports`。⚠ 想整体导出时，永远用 `module.exports = ...`；想挂属性时，两者都可以。

## 2.3 模块包装函数

每个 CommonJS 模块在执行前都会被包进一个函数：

```javascript
(function (exports, require, module, __filename, __dirname) {
  // 你的模块代码
});
```

这解释了四件事：

- 为什么模块里的变量不会污染全局——它们在函数作用域内；
- 为什么可以直接用 `require`、`module`、`__filename`、`__dirname`——它们是参数；
- 为什么 `this` 在模块顶层指向 `module.exports`——普通函数调用；
- 为什么模块代码是同步执行的——函数调用本身同步。

## 2.4 模块缓存

`require` 第一次加载模块后，会把结果放进 `require.cache`，键是解析后的绝对路径。之后再 `require` 同一个文件，直接返回缓存对象，不会重新执行。

```javascript
// counter.js
let count = 0;
module.exports = () => ++count;
```

```javascript
// main.js
const next = require("./counter.js");
console.log(next()); // 1
console.log(next()); // 2
console.log(require("./counter.js") === require("./counter.js")); // true
```

★ **关键认知**：模块缓存是**按解析后的绝对路径**区分的。`./math` 和 `./math.js` 通常解析到同一文件，共享缓存；但大小写不同的路径在 Linux 上可能被当成两个模块。⚠ 缓存也意味着**模块级状态会被共享**，别在模块顶层放"每个请求独立"的数据。

需要强制重新加载时：

```javascript
delete require.cache[require.resolve("./config.js")];
const freshConfig = require("./config.js");
```

这只适合开发热重载等少数场景，生产代码不应依赖它。

## 2.5 循环依赖

假设两个文件互相引用：

```javascript
// a.js
console.log("a 开始");
exports.done = false;
const b = require("./b.js");
console.log("在 a 中,b.done =", b.done);
exports.done = true;
console.log("a 结束");
```

```javascript
// b.js
console.log("b 开始");
exports.done = false;
const a = require("./a.js");
console.log("在 b 中,a.done =", a.done);
exports.done = true;
console.log("b 结束");
```

```javascript
// main.js
const a = require("./a.js");
console.log("在 main 中,a.done =", a.done);
```

输出：

```text
a 开始
b 开始
在 b 中,a.done = false
b 结束
在 a 中,b.done = true
a 结束
在 main 中,a.done = true
```

原因是：`a.js` 执行到 `require("./b.js")` 时，自己还没执行完，`exports.done` 仍是 `false`。Node 会把**当前已产生的部分导出**交给 `b.js`，而不是等 `a.js` 全部完成。

★ **关键认知**：CommonJS 的循环依赖拿到的是**不完整的中间态**。⚠ 解决方法不是"记住输出顺序"，而是**消除循环**：把共享部分抽到第三个模块，或改为在函数内部延迟 `require`。

# 3. ES Module

## 3.1 import 与 export

```javascript
// math.mjs
export function add(a, b) {
  return a + b;
}

export const PI = 3.14159;

export default function multiply(a, b) {
  return a * b;
}
```

```javascript
// main.mjs
import multiply, { add, PI } from "./math.mjs";

console.log(add(1, 2));       // 3
console.log(PI);              // 3.14159
console.log(multiply(2, 3));  // 6
```

命名导出用花括号，默认导出不用。一个模块可以有多个命名导出，但只能有一个默认导出。

## 3.2 静态解析与实时绑定

ESM 的 `import` 必须写在顶层，路径必须是字符串字面量：

```javascript
// 错误:不能条件导入
if (process.env.NODE_ENV === "production") {
  import { logger } from "./prod-logger.js";
}
```

需要条件加载时用动态导入：

```javascript
const { logger } = await import("./prod-logger.js");
```

ESM 的导出是**实时绑定**，不是值的拷贝：

```javascript
// counter.mjs
export let count = 0;
export function increment() {
  count++;
}
```

```javascript
// main.mjs
import { count, increment } from "./counter.mjs";
console.log(count); // 0
increment();
console.log(count); // 1,能读到变化
```

★ **关键认知**：CommonJS 导出的是**值的快照**(对象引用)，ESM 导出的是**活的绑定**。这让 ESM 更适合做状态共享，也让循环依赖的表现与 CommonJS 不同——ESM 会先建立绑定关系，再执行模块体。

## 3.3 顶层 await

ESM 支持在模块顶层直接使用 `await`：

```javascript
// config.mjs
const response = await fetch("https://example.com/config.json");
export const config = await response.json();
```

引入这个模块的代码会等待它执行完成。这让"加载配置再启动"变得自然，但也会拖慢依赖链上的启动时间。

⚠ **关键认知**：顶层 `await` 只存在于 ESM，CommonJS 不支持。它会让整个模块图变成异步加载，不要在库的入口随意使用，否则调用方的启动时机变得难以预测。

## 3.4 ESM 与 CommonJS 互操作

**ESM 引入 CommonJS**：

```javascript
// main.mjs
import pkg from "./legacy.cjs";        // 默认导入拿到 module.exports
import { readFileSync } from "node:fs"; // Node 内置模块可命名导入
```

Node 会用静态分析尽量识别 CommonJS 的命名导出，但并非所有写法都能识别。稳妥做法是**默认导入后再解构**：

```javascript
import legacy from "./legacy.cjs";
const { add } = legacy;
```

**CommonJS 引入 ESM**：不能直接 `require`，只能用动态 `import()`：

```javascript
// main.cjs
async function load() {
  const { add } = await import("./math.mjs");
  console.log(add(1, 2));
}
load();
```

★ **关键认知**：互操作的方向是**单向友好**的——ESM 可以较自然地引入 CommonJS，CommonJS 引入 ESM 必须走异步 `import()`。⚠ 混用两套体系会让加载顺序和缓存行为变复杂，新项目应统一选一套。

# 4. package.json 的 type 与双格式

## 4.1 type 字段

`package.json` 里的 `type` 决定 `.js` 文件按哪种格式解析：

| `type` 值 | `.js` 解析为 | `.cjs` | `.mjs` |
|-----------|-------------|--------|--------|
| 缺省或 `"commonjs"` | CommonJS | CommonJS | ESM |
| `"module"` | ESM | CommonJS | ESM |

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "type": "module"
}
```

★ **关键认知**：扩展名优先级高于 `type`。`.cjs` 永远是 CommonJS，`.mjs` 永远是 ESM，只有 `.js` 才由 `type` 决定。⚠ 从一个项目复制 `.js` 文件到另一个 `type` 不同的项目，可能直接报 `Cannot use import statement outside a module`。

## 4.2 条件导出 exports

`exports` 字段可以针对不同加载方式给出不同入口：

```json
{
  "name": "my-lib",
  "type": "module",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./utils": "./dist/utils.mjs"
  }
}
```

这样 ESM 用户拿到 `index.mjs`，CommonJS 用户拿到 `index.cjs`。条件是**从上到下匹配**，`types` 通常要放在最前面。

⚠ **关键认知**：配置了 `exports` 后，包内未被列出的路径将**无法被外部导入**，这既是封装也是常见的"升级后突然不能深引"的原因。需要暴露子路径时，记得在 `exports` 里显式声明。

## 4.3 双包陷阱

同一个包同时提供 ESM 与 CJS 入口时，如果两份代码各自持有模块级状态，就会出现**双实例**：

```text
ESM 入口 ──► 实例 A(有自己的缓存与单例)
CJS 入口 ──► 实例 B(有自己的缓存与单例)
```

如果这个包管理连接池、配置或事件总线，两份实例互不相认，就会出现"配置了却没生效""事件发了没人收"的怪现象。缓解办法是尽量让两种入口共享同一份核心实现，或干脆只发布一种格式。

# 5. 对比与选型

| 维度 | CommonJS | ES Module |
|------|----------|-----------|
| 加载方式 | 运行时同步 | 解析阶段静态分析 |
| 顶层 await | 不支持 | 支持 |
| 导出绑定 | 值的快照 | 实时绑定 |
| 循环依赖 | 拿到中间态 | 先建绑定再执行 |
| 动态导入 | `require` 可在任意位置 | `import()` 返回 Promise |
| tree-shaking | 较难 | 天然支持 |
| 浏览器原生 | 不支持 | 支持 |

选型建议：

- **新项目**：优先 ESM，和浏览器、打包工具一致；
- **老项目**：保持 CommonJS，逐步迁移，不要一次性全改；
- **发布库**：优先只发 ESM；必须兼容旧环境时再提供 CJS 入口，并处理好双包问题。

# 6. 常见错误与练习

## 6.1 常见错误

**错误一：`exports =` 整体赋值**

```javascript
exports = { add }; // 错误:require 拿不到
module.exports = { add }; // 正确
```

**错误二：在 ESM 里用 `__dirname`**

```javascript
// ESM 中没有 __dirname
import { fileURLToPath } from "node:url";
import { dirname } from "node:path";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```

**错误三：混用 `require` 与 `import` 而不看文件类型**

```javascript
// 在 type: module 的 .js 文件里
const fs = require("node:fs"); // 报错:require is not defined
```

改用 `import fs from "node:fs"` 或 `import { readFile } from "node:fs/promises"`。

## 6.2 练习

1. 写一个 `logger.cjs`，导出 `info` 和 `error` 两个函数，再在 `main.mjs` 中导入并使用，分别尝试默认导入与命名导入。

2. 构造两个互相 `require` 的模块，打印各自的导出，解释为什么其中一个拿到的是不完整对象。

3. 给一个包配置 `exports`，要求 `import` 走 ESM 入口、`require` 走 CJS 入口，并说明为什么不能省略 `require` 条件。

# 7. 小结

| 概念 | 要点 |
|------|------|
| CommonJS | 同步加载，`module.exports` 是真正的出口 |
| exports | 只是 `module.exports` 的初始引用，重赋值会断开 |
| 模块缓存 | 按绝对路径缓存，模块级状态共享 |
| 循环依赖 | CJS 拿到中间态，应尽量消除循环 |
| ESM | 静态分析、实时绑定、支持顶层 await |
| type | 只决定 `.js`，`.cjs` / `.mjs` 由扩展名决定 |
| 互操作 | ESM 可引 CJS，CJS 只能 `import()` 引 ESM |

下一篇进入 npm 与包管理，讲清楚 `package.json` 各字段、语义化版本与版本范围、锁文件的作用，以及 npm 与 pnpm 在安装结构上的差异。
