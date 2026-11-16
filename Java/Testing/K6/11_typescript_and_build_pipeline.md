# 1. 为什么要用 TypeScript

k6 的脚本语言是 JavaScript，但自 v0.34 起 k6 **原生支持直接运行 `.ts` 文件**——运行时内置了一个 TypeScript 转译器，在执行前把类型注解剥离（strip types），再交给 JS 引擎（Goja/Sobek）解释执行。这个"原生支持"的边界必须先划清楚：**k6 只做类型剥离，不做类型检查**。写错了类型，`k6 run script.ts` 照样能跑起来，错误只会在运行时以另一种形式暴露。因此 TypeScript 在 k6 工程里的正确定位是"**开发期防线**"：类型检查由本地的 `tsc` 和编辑器的 language server 完成，k6 运行时只消费剥离类型后的产物。

## 1.1 类型约束带来的收益

**类型约束的核心收益是把运行时错误前移到编写期**。k6 脚本一旦进入 CI 就是"无人值守"执行，一个 `undefined is not a function` 要到压测跑起来才会炸，浪费的是一整轮执行时间和窗口期。配合官方类型包 `@types/k6`，编辑器能对 `http.post` 的参数、`check()` 的返回值、`options` 的每个字段给出精确提示与校验：

```bash
npm install --save-dev typescript @types/k6
```

```typescript
// script.ts —— @types/k6 提供全部内置模块的类型声明
import http from "k6/http";
import { check, sleep } from "k6";
import { Trend } from "k6/metrics";

// 自定义指标同样有类型：Trend 只接受 number 值，传 string 会在 tsc 阶段报错
const loginDuration = new Trend("login_duration", true);

export const options = {
  vus: 10,          // 拼成 vusss 会被 tsc 直接标红，运行时则静默忽略
  duration: "30s",
};

export default function () {
  const res = http.get("https://test.k6.io");
  // res 的类型是 Response | null，强迫你处理空值分支
  check(res, { "status is 200": (r) => r !== null && r.status === 200 });
  loginDuration.add(res ? res.timings.duration : 0);
  sleep(1);
}
```

除减少运行时错误外，类型还带来三项工程收益：其一，**大型脚本的可维护性**——当 `utils/` 下有十几个模块、options 对象上百行时，接口签名就是文档，重构时 tsc 会列出全部受影响调用点；其二，**团队协作的一致性**——新人不需要通读代码就能从类型签名理解一个辅助函数的输入输出契约；其三，**options 的静态校验**——k6 的 `options` 字段极多（scenarios、thresholds、TLS 配置……），拼写错误在运行时往往只是"配置未生效"这种最难察觉的静默失败，而 `@types/k6` 的 `Options` 类型能当场拦截。

> **注意**：类型是**编译期产物，运行期为零**。k6 不会因为你标注了 `res: Response` 就保证 `res` 不是 `null`；空指针、越界、协议错误该发生还是发生。类型系统降低的是"低级错误"的概率，不能替代 `check()` 和 thresholds 这些运行时断言。

## 1.2 适合 TS 的场景

**是否引入 TypeScript，取决于脚本的规模与生命周期**。k6 官方对 TS 的支持是"有限支持"——没有完整的类型检查管线、npm 生态兼容性受限（见第 3 节），所以 TS 的成本收益比在不同场景下差异很大：

**适合用 TS 的场景**：

- **场景复杂、模块多**：多 scenario 编排、共享大量辅助函数、数据参数化逻辑独立成模块的项目，模块间的类型契约价值最大
- **团队协作与代码规范**：多人维护同一套压测脚本库，类型 + ESLint 能把 review 中大量"这里会不会是 null"的讨论消灭在编译期
- **脚本作为长期资产**：性能回归测试套件跟随业务迭代数年，类型是抵抗腐化最便宜的手段

**可以直接用 JS 的场景**：

- 一次性的冒烟压测、临时验证某个接口的极限吞吐
- 脚本总量在几百行以内、单人维护
- 依赖大量浏览器端 npm 库做复杂模拟（TS 帮不上忙，瓶颈在运行时兼容性）

# 2. 构建流程设计

## 2.1 基本思路

**k6 的 TS 工程化有两条路线，选择标准是"是否需要 npm 依赖"**。路线 A 是零构建：k6 直接吃 `.ts` 文件，类型剥离由运行时完成，编辑器提示由 `tsconfig.json` + `@types/k6` 提供，类型检查交给 CI 里的 `tsc --noEmit`。路线 B 是打包构建：当脚本需要引用 npm 上的第三方库（如 `ajv` 做 JSON Schema 校验、`lodash` 做数据处理）时，k6 运行时无法解析 `node_modules` 的完整语义，必须先用 webpack/esbuild 把所有依赖打成**单文件 bundle**，再让 k6 执行这个产物：

```text
路线 A（零构建，推荐起步）：
  script.ts ──────────────────────────────► k6 run script.ts
       │                                      （运行时剥离类型）
       └──► tsc --noEmit（CI 中仅做类型检查）

路线 B（打包构建，需要 npm 依赖时）：
  src/*.ts + node_modules
       │
       ▼
  esbuild/webpack ──► dist/bundle.js（单文件、ESM、无外部依赖）
       │                    │
       ▼                    ▼
  tsc --noEmit          k6 run dist/bundle.js
```

两条路线的共同点是：**类型检查永远独立于 k6 执行**。无论哪条路线，CI 里都应该有一条 `tsc --noEmit` 关卡，否则"能跑"和"类型正确"会逐渐脱节。

## 2.2 构建脚本示例

一套可直接使用的路线 B 工程配置如下。`tsconfig.json` 的关键选项：`target` 设为 `es2017` 或更高（k6 的 JS 引擎支持 ES2015+ 的绝大部分特性，不要编译到 ES5，那会引入 k6 不需要的 helper 代码）；`module` 设为 `esnext`（k6 只认 ESM 的 `import`/`export`）；`moduleResolution` 用 `bundler`（交给 esbuild 解析）；`types` 显式声明 `k6`，避免把 `@types/node` 的浏览器不存在的全局类型（`process`、`Buffer`）泄漏进脚本：

```json
{
  "compilerOptions": {
    "target": "es2017",              // 不要降到 ES5，k6 引擎原生支持 ES2015+
    "module": "esnext",              // k6 只支持 ESM，产物必须保留 import/export
    "moduleResolution": "bundler",   // 由 esbuild 负责实际的模块解析
    "strict": true,                  // 压测脚本同样值得严格模式
    "noEmit": true,                  // tsc 只做类型检查，产物由 esbuild 生成
    "types": ["k6"],                 // 只引入 k6 类型，屏蔽 @types/node 的误导
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*.ts"]
}
```

esbuild 是目前社区主推的打包器（webpack 也可行但配置繁琐得多），一条命令完成打包，`k6` 与 `k6/*` 内置模块必须标记为 external——它们是运行时注入的，打进 bundle 反而报错：

```bash
esbuild src/main.ts \
  --bundle \
  --format=esm \
  --target=es2017 \
  --platform=neutral \
  '--external:k6' \
  '--external:k6/*' \
  --outfile=dist/bundle.js
```

`package.json` 里固化为脚本：

```json
{
  "scripts": {
    "build": "esbuild src/main.ts --bundle --format=esm --target=es2017 --platform=neutral --external:k6 '--external:k6/*' --outfile=dist/bundle.js",
    "typecheck": "tsc --noEmit",
    "test": "npm run typecheck && npm run build && k6 run dist/bundle.js --vus 1 --iterations 1"
  }
}
```

**npm 依赖被打进 bundle 的原理**是：esbuild 从入口文件出发做静态依赖图分析，把每个模块的源码内联进单一文件，并将 `import` 重写为内部引用——k6 最终看到的是一个不依赖 `node_modules` 的普通 JS 文件。这带来两个硬性限制：其一，**只有纯 JS 计算类的 npm 包能用**，任何依赖 Node 内置模块（`fs`、`net`、`child_process`）或浏览器 API（`window`、`document`）的包，打包能通过、运行必失败；其二，产物体积随依赖线性增长，而 k6 启动时要把整个 bundle 解析进**每一个 VU 的独立 JS 运行时**，超大 bundle 会显著推高内存占用（见 5.1）。

> **行业实践**：多数团队从路线 A 起步——`@types/k6` + `tsc --noEmit` 已经覆盖了 TS 的绝大部分收益，且没有构建环节就没有构建环节的坑。只有当脚本确实需要 npm 上的纯逻辑库时才升级到路线 B；k6 官方也提供了 jslib（`https://jslib.k6.io`）托管常用工具库，可以直接远程 import 而无需本地打包。

# 3. 兼容性要点

## 3.1 ES 语法限制

**k6 的 JS 引擎是嵌入式的 Go 实现（早期为 Goja，v0.5x 已迁移到其分支 Sobek），不是 V8**，这决定了两类不兼容。第一类是语法与内置对象覆盖不全：ES2015+ 的核心语法（class、解构、模板字符串、async/await、generator）都支持，但部分较新的内置方法与对象缺失或行为有差异（例如某些 `Intl` 能力、较新的数组方法），esbuild 的 `--target=es2017` 正是为了把产物约束在引擎稳定支持的子集内。第二类是环境 API 的缺失：k6 运行时**没有 DOM、没有 `window`、没有 `document`、没有 `localStorage`**，网络能力只通过 `k6/http`、`k6/net` 等内置模块暴露，任何假设自己跑在浏览器或 Node 里的代码都无法存活。

k6 提供 `--compatibility-mode` 开关在两种转译策略间选择：

| 维度 | base（默认） | extended |
|------|-------------|----------|
| 转译器 | 内置轻量转译（含 TS 类型剥离） | 内嵌 Babel |
| 性能 | 启动快、内存低 | 启动慢、每个 VU 内存更高 |
| require() | 不支持 | 支持（CommonJS 互操作） |
| 动态特性 | 仅静态 import/export | 可处理部分 Babel 能转译的写法 |
| 适用 | 绝大多数场景 | 依赖 CommonJS npm 包且无法用 esbuild 预打包时 |

## 3.2 模块解析

**k6 的模块系统只认静态 `import`/`export`，动态 `import()` 不被支持**——调用会直接抛错。原因是 k6 在执行前需要对脚本做完整的依赖图分析，把本地文件读入并编译，每个 VU 拿到的是编译好的不可变程序；运行期再按变量值去解析模块路径，与这个模型根本冲突。同理，`base` 模式下没有 `require`，条件加载模块（"根据环境变量 import 不同实现"）只能在构建期用 esbuild 的条件编译或多个入口解决，或退化为运行时 `if` 分支调用已静态导入的两套实现。

本地模块解析用**相对路径最稳定**（`import { login } from "./utils/auth.js"`，注意 k6 的 import 路径**需要写扩展名**，这与 Node 的习惯不同）；绝对路径基于脚本所在目录解析。v0.43 之后 k6 引入了实验性的 npm 模块解析，可以直接 `import` `node_modules` 里的部分包，但它不做打包、不注入 Node polyfill，兼容面窄，生产工程仍建议走 esbuild 预打包路线，把不确定性消灭在构建期。

> **注意**：从浏览器/Node 项目迁移代码到 k6 时，最常见的三类翻车依次是：动态 `import()`、`process.env` 读环境变量（k6 应使用 `__ENV.MY_VAR`）、以及依赖 `Buffer`/`TextEncoder` 的部分行为差异。这三者 tsc 都查不出来（尤其当 tsconfig 误引了 `@types/node` 时反而"看起来合法"），只能靠 code review 清单和一次 `--vus 1 --iterations 1` 的冒烟执行兜底。

# 4. 规范与质量

## 4.1 类型定义

**公共模块必须导出类型，类型与实现同文件维护**。当 `utils/` 里的辅助函数被十几个场景脚本引用时，类型声明就是模块的公共 API。一个规范的公共模块长这样——函数签名、参数接口、返回值类型全部显式导出，调用方无需阅读实现：

```typescript
// utils/auth.ts —— 类型与实现放在一起，随实现同步演进
import http from "k6/http";

// 登录所需的配置契约：调用方一眼看清要传什么
export interface LoginConfig {
  baseUrl: string;      // 被测服务根地址
  username: string;     // 压测账号
  password: string;     // 密码（从 __ENV 注入，禁止硬编码）
}

// 登录产物：后续请求要携带的凭证
export interface Session {
  token: string;        // Bearer token
  expiresAt: number;    // 过期时间戳（毫秒）
}

export function login(cfg: LoginConfig): Session {
  const res = http.post(`${cfg.baseUrl}/api/login`, JSON.stringify({
    username: cfg.username,
    password: cfg.password,
  }), { headers: { "Content-Type": "application/json" } });

  if (res === null || res.status !== 200) {
    // 快速失败：登录不通就不要继续压，避免产生海量无意义指标
    throw new Error(`login failed: status=${res?.status}`);
  }
  const body = res.json() as { token: string; expiresIn: number };
  return { token: body.token, expiresAt: Date.now() + body.expiresIn * 1000 };
}
```

**保持类型与实现一致**依赖两条纪律：类型定义不允许 `any` 兜底（ESLint 的 `@typescript-eslint/no-explicit-any` 开 warn 以上），以及 `tsc --noEmit` 进 CI 阻断合并。对确实无类型的第三方 JS 库，用 `declare module "some-lib"` 写最小化的 `.d.ts` 声明，而不是全局关掉检查。

## 4.2 Lint 与格式化

**静态检查在 CI 中执行，风格统一靠工具而不是靠 review**。ESLint（配 `@typescript-eslint`）负责抓类型之外的逻辑隐患——未处理的 Promise、误用 `==`、声明未使用的变量；Prettier 负责格式，消除团队成员的风格差异。CI 关卡的典型编排：

```yaml
# .github/workflows/k6-check.yml
name: k6-static-check
on: [pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run typecheck        # tsc --noEmit：类型关卡
      - run: npx eslint src --ext .ts # lint：逻辑与规范关卡
      - run: npm run build            # esbuild：确认能打出 bundle
      - name: 冒烟执行（1 VU × 1 次迭代）
        run: k6 run dist/bundle.js --vus 1 --iterations 1
```

这套管线的意义在于分层拦截：类型错误在 `typecheck` 死掉，未定义行为和坏味道在 `eslint` 死掉，模块解析和引擎兼容性问题在冒烟执行死掉——真正到压测窗口期才发现的问题，只剩业务语义层面的（参数化数据过期、被测环境变更）。

# 5. 常见问题

## 5.1 打包体积过大

**bundle 过大的直接代价是内存与启动时间，因为 k6 为每个 VU 创建独立的 JS 运行时实例**。虽然编译后的程序体在 VU 间有共享优化，但每个 VU 的堆、闭包环境仍是独立的，超大脚本会系统性抬高基线内存，压缩单机能承载的 VU 上限（参见第 15 篇的资源模型分析）。控制手段按优先级排列：**只引入必要依赖**——压测脚本里引 `moment` 这种带全量 locale 的库是典型反模式，日期格式化手写十行足够；**避免浏览器端大型库**——它们不仅体积大，还大概率因依赖 DOM 而无法运行；用 esbuild 的 `--analyze` 查看依赖图，把意外被传递引入的重型包揪出来；确需大数据文件（参数化 CSV）时走 `open()` 或 `SharedArray`，**不要把数据内联进 bundle**。

## 5.2 运行时错误

**"tsc 全绿、k6 一跑就错"几乎总是编译目标与 k6 运行时不匹配**。排查清单按命中率排序：

1. **target 过高或过低**：`--target=esnext` 可能产出 k6 引擎尚不支持的新语法（如某些新内置方法）；`--target=es5` 则引入 regenerator 等 helper，在 base 兼容模式下行为异常。统一钉在 `es2017` 附近最稳
2. **format 错误**：esbuild 产出必须是 `--format=esm`，`cjs` 产物里的 `module.exports` 在 base 模式下直接报 `require is not defined`
3. **external 漏标**：`k6`、`k6/http` 等内置模块被打进 bundle，运行时因找不到被内联的"假模块"而报错
4. **环境 API 缺失**：`window`、`process`、`Buffer` 在运行时不存在——这类错误 tsc 若误配了 `@types/node` 是查不出来的
5. **动态 import**：运行到该分支时抛 `SyntaxError`，冒烟执行的迭代路径若未覆盖到，会漏到正式压测才爆炸

> **设计考量**：k6 团队选择"类型剥离而非完整类型检查"、"静态模块图而非动态加载"，本质都是在为**性能与确定性**让步——每个 VU 都要独立初始化 JS 运行时，转译越轻、模块图越静态，万级 VU 场景下的启动风暴和资源占用就越可控。理解了这一点，就能理解为什么 TS 支持是"有限"的：这不是功能欠账，而是负载生成器这一角色下的主动取舍。

# 6. 小结

k6 对 TypeScript 的支持是**编译期类型剥离 + `@types/k6` 类型声明**的组合，类型检查必须由外部 `tsc --noEmit` 独立完成。工程化分两条路线：无 npm 依赖时零构建直跑 `.ts`；有依赖时用 esbuild 打成 `--format=esm --target=es2017`、k6 模块标 external 的单文件 bundle。运行时约束要牢记三条：只有静态 `import`/`export`（动态 `import()` 不支持）、没有 DOM/Node 环境 API、npm 包仅限纯 JS 计算类。CI 中按"typecheck → lint → build → 1 VU 冒烟"分层设卡，把问题拦截在压测窗口之前。
