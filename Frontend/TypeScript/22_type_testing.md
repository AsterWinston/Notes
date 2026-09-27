类型就是库的公开 API：重构一旦破坏类型契约，使用方的编译会当场变红，而运行时测试对此一无所知。本篇教你用 `@ts-expect-error`、`Equal`/`Expect` 断言工具把类型行为锁进测试，再用 .d.ts 快照与 CI 流程守住公共 API 不被悄悄破坏。

# 1. 为什么类型也要测试

## 1.1 类型即 API

```typescript
// 库的第一版：参数接受宽类型
export function debounce(fn: (...args: unknown[]) => void, ms: number) {
  return fn;                                  // 简化实现，聚焦类型
}

// 重构后"顺手收窄"了签名——运行时行为没变，类型契约却变了
export function debounceV2(fn: (...args: []) => void, ms: number) {
  return fn;                                  // 不再接受带参回调
}

// 使用方编译立刻变红：类型契约破坏等价于 API 破坏
const d = debounceV2(() => {}, 300);          // ✅ 无参回调还能用
// debounceV2((x: number) => console.log(x), 300);  // ✗ TS2345: 参数不兼容
```

★ **关键认知**：函数签名、导出类型、泛型约束都是**公开 API 的一部分**，它们的兼容性规则和运行时行为一样严格——TS 的**结构化类型**让"改个字段名""加个必填参数"都可能波及全下游。运行时测试绿灯只证明"行为没变"，类型测试才能证明"契约没变"，二者缺一不可。

## 1.2 类型测试与运行时测试的分工

| 维度 | 类型测试 | 运行时测试 |
|------|---------|-----------|
| 检测时机 | 编译期（`tsc --noEmit`） | 运行测试框架时 |
| 验证对象 | 签名、推断结果、泛型行为、报错是否如期 | 输出值、副作用、异常 |
| 失败表现 | 编译报错（TS 错误号） | 断言失败（红条） |
| 能否测出"必须报错" | 能（`@ts-expect-error`） | 不能 |
| 典型盲区 | 真实数据、边界值 | 类型收窄、推断正确性 |

★ **关键认知**：类型测试的"断言"就是**让编译通过或让编译报错**——没有测试框架也能跑：`npx tsc --noEmit` 就是类型测试运行器。这与 Go 的编译期检查文化同源（Go 靠编译器兜住大量错误），但 TS 更进一步：连"这段代码**必须**编译失败"都能写成测试（负例断言）。

# 2. 注释指令三兄弟

## 2.1 @ts-expect-error vs @ts-ignore vs @ts-nocheck

| 指令 | 作用范围 | 会验证错误真的存在吗 | 建议 |
|------|---------|-------------------|------|
| `// @ts-expect-error` | 下一行 | ✅ 会：若下一行没报错，它自己报错 | 负例测试首选 |
| `// @ts-ignore` | 下一行 | ❌ 不会：错误消失后指令变"哑弹" | 禁用（lint 拦截） |
| `// @ts-nocheck` | 整个文件 | — | 只用于逃生（第三方文件） |

```typescript
type Num = number;

// @ts-expect-error 下一行必须报错：给 number 赋 string 是负例
const a: Num = "oops";            // ✅ 报错，符合预期，测试通过

// @ts-ignore 盲目压掉下一行的一切报错（危险）
// @ts-ignore
const b: Num = "oops";            // 无提示通过——错误被静默吞掉

// ⚠ 指令失去目标时的行为差异：
// @ts-expect-error
const c: Num = 42;                // ✗ 这行没错误 → @ts-expect-error 自己报错
// @ts-ignore
const d: Num = 42;                // 什么都不发生：哑弹永远留在代码里
```

★ **关键认知**：`@ts-expect-error` 是"**负例断言**"——它不仅压住错误，还断言"错误必须存在"。当重构让原本报错的代码合法化时，`@ts-expect-error` 会立刻自爆，提醒你更新测试；`@ts-ignore` 则永远沉默。这就是二者的本质区别：一个是断言，一个是创可贴。⚠ 工程上应在 lint 里禁掉 `@ts-ignore` 与 `@ts-nocheck`，只放行 `@ts-expect-error`。

## 2.2 实战用法：把"必须报错"写成断言

```typescript
type Fn = (n: number) => string;
const call = (f: Fn) => f(1);                  // 正常调用通道

// 形态一：断言"非法调用必须被拒绝"
// @ts-expect-error  参数逆变约束下，(x: string) => string 不可当 Fn 用
call((x: string) => x);                        // 参数类型不兼容，必须报错

// 形态二：断言"非法类型表达必须被拒绝"
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
// @ts-expect-error  "z" 不是 keyof { a: 1 }，约束应当失守报错
type Bad = MyPick<{ a: 1 }, "z">;

// ⚠ @ts-expect-error 只断言"下一行有错误"，不断言"是哪个错误"
// 一行里若有多个潜在报错源，无法分辨压掉的是哪一个
```

★ **关键认知**：裸 `tsc` 的负例断言只能验证"错误存在"；dtslint 一类工具能进一步断言**错误码**（如 `^? TS2344`），原理是把指令旁注释里的错误码与编译器输出比对。设计负例时尽量让**每行只有一个错误源**，否则断言会失焦——压掉 A 错误的测试，在 A 修好后会因为 B 错误"碰巧还在"而继续通过，产生虚假的安全感。

# 3. 手写类型断言工具

## 3.1 Equal 与 Expect 的实现

```typescript
// Equal：判断两个类型是否"完全相同"（而非互相兼容）
type Equal<X, Y> =
  (<T>() => T extends X ? 1 : 2) extends        // 把 X 装进泛型条件"探针"
  (<T>() => T extends Y ? 1 : 2)                // 与 Y 的探针做函数签名比较
    ? true
    : false;

// Expect：断言参数必须是 true——约束不满足即编译报错（测试失败）
type Expect<T extends true> = T;

// 用法：类型相等时编译通过，不等时报 TS2344
type _ok = Expect<Equal<{ a: string }, { a: string }>>;   // ✅ 通过
type _bad = Expect<Equal<string, number>>;                // ✗ 约束不满足
```

## 3.2 为什么不用 `X extends Y` 判相等

```typescript
// 朴素双向 extends：会被可选属性、any 等骗过
type NaiveEq<A, B> = A extends B ? (B extends A ? true : false) : false;

type T1 = NaiveEq<{ a: string }, { a: string; b?: number }>;  // true（误判！）
type T2 = Equal<{ a: string }, { a: string; b?: number }>;    // false（正确）

type T3 = NaiveEq<any, string>;                               // true（any 污染）
type T4 = Equal<any, string>;                                 // false（正确）
```

★ **关键认知**：`Equal` 的原理是**用泛型函数签名做"探针比较"**：`<T>() => T extends X ? 1 : 2` 把类型 X 藏进条件类型的判别位，TS 比较两个函数类型时对条件类型采用**结构严格相等**判定（连可选性、readonly、联合顺序、any/never 的差异都能察觉），而不是普通的双向可赋值。这是类型编程里最著名的技巧之一（社区常称 "the ultimate type equality check"）；⚠ 它对 `any`/`never` 的极端组合仍有细微差异，做类型测试时遇到边缘情况以 `tsc` 实际输出为准。

## 3.3 补充断言原语：NotEqual 与 IsAny

```typescript
// NotEqual：断言两类型"必须不同"
type NotEqual<X, Y> = Equal<X, Y> extends true ? false : true;
type _n1 = Expect<NotEqual<string, number>>;     // ✅ 确实不同，断言通过
// @ts-expect-error  any 与 any 相同，NotEqual 应当给出 false
type _n2 = Expect<NotEqual<any, any>>;

// IsAny：探针识别 any（借用 any 的"万能交叉"怪癖）
type IsAny<T> = 0 extends 1 & T ? true : false;  // 只有 any 能让 1 & T 变 any
type _a1 = Expect<IsAny<any>>;                   // ✅ 是 any
type _a2 = Expect<Equal<IsAny<string>, false>>;  // ✅ 不是 any
```

★ **关键认知**：`1 & any` 得到 `any`、`0 extends any` 恒真，而任何普通类型都不满足 `0 extends 1 & T`——`IsAny` 正是利用这条**类型层面的怪异性质**把 any 挑出来。运行时没有等价物（不存在 `typeof x === "any"`），但类型测试里必须能识别 any，否则"被 any 污染的推断结果"会伪装成通过。这类**探针类型**（Equal/NotEqual/IsAny）是类型测试工具箱的底层零件，expect-type 等库的断言原语都由此衍生。

# 4. 类型测试实战

## 4.1 给工具类型写正例 + 负例测试

```typescript
// ===== 待测工具类型（自实现） =====
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };   // 选出部分属性
type MyOmit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;  // 剔除属性

// ===== 正例：结果必须与期望类型完全一致 =====
type _p1 = Expect<Equal<MyPick<{ a: 1; b: 2 }, "a">, { a: 1 }>>;   // ✅
type _p2 = Expect<Equal<MyOmit<{ a: 1; b: 2 }, "a">, { b: 2 }>>;   // ✅

// ===== 负例：必须报错，@ts-expect-error 验证"错误真的存在" =====
// @ts-expect-error  "z" 不是 keyof { a: 1 }，应当被拒绝
type _p3 = MyPick<{ a: 1 }, "z">;

// @ts-expect-error  类型不相等时 Expect 的约束应当失败
type _p4 = Expect<Equal<MyPick<{ a: 1 }, "a">, { a: 2 }>>;
```

```powershell
# 类型测试 = 一次"零产出编译"：断言失败都表现为编译报错
npx tsc --noEmit          # 退出码 0 = 全部类型测试通过；非 0 = 有断言失败
```

★ **关键认知**：正例用 `Expect<Equal<实际, 期望>>` 锁"必须是这个类型"，负例用 `@ts-expect-error` 锁"必须不接受这种用法"——两条腿合起来才是完整的类型行为规格。类型测试文件通常命名 `*.test-d.ts` 并排除在构建产物外，但纳入 `tsc --noEmit` 的检查范围。dtslint、expect-type 这类库本质上就是把 `Equal`/`Expect` 产品化（更好的报错文案、更多断言原语），思想与手写版完全一致，原理通了用哪个都顺手。

## 4.2 测什么：高价值断言清单

| 断言类型 | 示例 | 防住的回归 |
|---------|------|-----------|
| 推断结果 | `Expect<Equal<ReturnType<typeof f>, number>>` | 返回类型被意外拓宽 |
| 泛型约束 | `@ts-expect-error` 非法实参 | 约束被放松 |
| 可辨识联合 | `@ts-expect-error` 漏分支 | 新增成员未被处理 |
| 映射类型 | `Expect<Equal<Pick<T,"a">, {a: 1}>>` | 属性修饰符意外变化 |
| 字面量保留 | `Expect<Equal<Fn<"a">, "a">>` | as const 类信息丢失 |

★ **关键认知**：类型测试最划算的目标是**公共 API 的推断结果**——`ReturnType`/`Parameters` 这些内置工具（第 12 篇）正是"把推断结果取出来送进 `Equal`"的探针。内部实现类型的测试价值低、维护成本高，测试应集中在"用户看得见的契约"上。

## 4.3 类型测试的组织与运行

```
 src/
 ├─ index.ts          库源码
 ├─ index.test.ts     运行时测试（验证值与行为）
 └─ index.test-d.ts   类型测试（不产出、不执行，只参与类型检查）
 tsconfig.json        include: ["src"] —— test-d.ts 天然纳入 tsc 检查
```

```typescript
// index.test-d.ts：只写类型断言，不需要任何测试框架
export {};                                   // 转成模块，避免全局脚本污染

type Fn = (...args: never[]) => void;
declare function debounce<F extends Fn>(fn: F, ms: number): F;  // 待测签名

type _infer1 = Expect<
  Equal<ReturnType<typeof debounce<() => void>>, () => void>    // 推断保持原函数
>;
// @ts-expect-error  泛型约束不接受非函数实参，负例必须报错
type _infer2 = Parameters<typeof debounce<string, never>>;
```

```powershell
# 运行类型测试：没有 runner，编译器就是测试器
npx tsc --noEmit --strict        # 显式带 --strict，保证本地与 CI 检查强度一致
```

| 组织要点 | 做法 |
|---------|------|
| 文件命名 | `*.test-d.ts`（约定俗成，工具可识别） |
| 构建排除 | 发布用 tsconfig 排除 `*.test-d.ts`，避免断言混入 .d.ts |
| 与运行时测试的关系 | 并行两步：先 `tsc --noEmit`，再跑运行时测试 |
| 失败反馈 | 编译报错的文件行号就是断言失败位置 |

★ **关键认知**：类型测试的"执行"就是被 `tsc` 检查一遍——没有 setup、没有 runner、毫秒级完成，这让它能无缝塞进任何 CI，甚至 git pre-commit hook。⚠ 两个细节别踩：`*.test-d.ts` 若不导出任何东西会被当作全局脚本（多个同名断言可能冲突），加 `export {}` 转成模块最稳；用到 `declare function` 时只是"声明签名供测试"，不提供实现，绝不能被构建产物引用。

# 5. API 破坏性变更检测

## 5.1 .d.ts 快照与公共 API 报告

```
 提交代码 ──► tsc --noEmit（类型测试）──► 产出 .d.ts ──► 与基线快照 diff
                 │                              │
                 ▼                              ▼
        断言失败 = 测试红                差异 = API 变更，需人工 review
```

基本思路：把库的**声明文件**当作公共 API 的"可执行规格"，每次发版前产出一份规范化 `.d.ts`，与入库的基线快照做文本对比——签名变了 diff 就会现形。TS 工具链里的 api-extractor 能自动生成 API 报告文件（把 `.d.ts` 归一化成单文件），思想相同：**用声明文件做变更审计的锚点**。

## 5.2 CI 完整流程

```powershell
# ===== 可照抄执行：类型测试 + API 变更检测接入 CI =====
npm ci                                                    # 锁版本安装
npx tsc --noEmit                                          # ① 全量类型检查（即类型测试）
npx tsc --declaration --emitDeclarationOnly --outDir api-current   # ② 产出当前声明
git diff --no-index api-baseline api-current              # ③ 与基线对比（有差异=API 变更）
# diff 非空：若为预期变更，把 api-current 覆盖 api-baseline 并随代码一起提交
```

| CI 阶段 | 命令 | 失败含义 |
|---------|------|---------|
| 类型检查 | `npx tsc --noEmit` | 类型测试断言失败 |
| 声明产出 | `tsc --declaration --emitDeclarationOnly` | 声明生成不出来（类型不可表达） |
| 基线对比 | `git diff --no-index api-baseline api-current` | 公共 API 发生变更，需 review |

★ **关键认知**：**API 基线快照**把"破坏性变更"从 code review 的口头约定变成机器可检出的事实——尤其能抓住"无意的破坏"（顺手改了推断结果、放宽了泛型约束）。这与 Go 社区的 API 兼容性检查（如 apidiff）思路完全一致：签名即契约，契约变更必须显式过审。⚠ 基线文件要入库并随预期变更一起 review，否则快照机制会退化成摆设。

# 6. 类型层面的代码质量清单

| ☐ | 检查项 | 说明 |
|---|--------|------|
| ☐ | `strict: true` 全开 | 新项目第一天就开（见第 21 篇） |
| ☐ | lint 禁用 `@ts-ignore`/`@ts-nocheck` | 只允许 `@ts-expect-error` |
| ☐ | 公共 API 显式标注返回类型 | 推断一变就是 API 破坏 |
| ☐ | 外部数据入口用 `unknown` + 校验 | 信任边界不放 any（见第 21 篇） |
| ☐ | 公共工具类型配套类型测试 | 正例 + `@ts-expect-error` 负例 |
| ☐ | CI 常驻 `tsc --noEmit` | 本地能过 ≠ 仓库能过 |
| ☐ | `.d.ts` 基线快照入库 | API 变更必须显式提交 |
| ☐ | 类型体操复杂度封顶 | 递归层数、编译耗时有预算（见第 19 篇） |

★ **关键认知（全篇收束）**：类型测试的哲学与运行时测试完全同源——**把契约写成断言，让机器替人守门**；区别只在于断言的载体从"值"换成了"类型"，运行器从测试框架换成了 `tsc --noEmit`。一套成熟的质量清单应该是：类型测试锁契约、运行时测试锁行为、`.d.ts` 快照锁 API 面，三层齐备才敢放心重构。
