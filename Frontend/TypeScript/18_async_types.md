异步代码是类型系统最容易失守的地方：Promise 的嵌套包装、catch 的无类型变量、网络返回的未知数据，处处是 `any` 的渗透点。本篇讲清 `Promise<T>` / `Awaited<T>` 的类型机制，并实战手写 Result 模式、类型化 fetch 封装与类型安全事件表，把"运行时数据"这一静态类型的盲区补上。

# 1. Promise<T> 与 async 函数的类型

## 1.1 Promise<T> 的类型结构

```typescript
// lib 里 Promise 接口的骨架（简化版，只看类型形状）
interface Promise<T> {
  then<TResult1 = T, TResult2 = never>(
    onfulfilled?: ((value: T) => TResult1 | PromiseLike<TResult1>) | null,   // 成功回调拿 T
    onrejected?: ((reason: any) => TResult2 | PromiseLike<TResult2>) | null  // ⚠ 失败回调是 any
  ): Promise<TResult1 | TResult2>;
  catch<TResult = never>(
    onrejected?: ((reason: any) => TResult | PromiseLike<TResult>) | null    // ⚠ 同样是 any
  ): Promise<T | TResult>;
  finally(onfinally?: (() => void) | null): Promise<T>;                      // 不改变 T
}

const p: Promise<number> = Promise.resolve(42);   // 泛型参数 = 成功值的类型
```

★ **关键认知**：`Promise<T>` 的 `T` 只描述**成功通道**，失败通道在内建类型里是 `any`（历史遗留设计）。⚠ 所以 `.catch(e => ...)` 的 `e` 默认是 `any`，滥用会绕过全部类型检查——实践上应把 `e` 当 `unknown` 处理（见第 3 章）。这与 Rust 的 `Result<T, E>` 把成功/失败**都**类型化形成鲜明对比：TS 的 Promise 天生是"半边类型"的。

## 1.2 async 函数的返回值自动包装

```typescript
// async 函数声明返回 Promise<number>，实际 return 的是 number
async function loadAge(): Promise<number> {
  return 18;                         // return 值自动包装成 Promise<number>
}

// ✗ 常见错误：注解写成"展开后"的类型
async function bad(): string {       // TS2322: 'Promise<string>' is not assignable to 'string'
  return "x";                        // 自动包装后是 Promise<string>，与注解冲突
}

// ✓ 正确：注解一律写 Promise<T>
async function good(): Promise<string> {
  return "x";                        // 匹配
}

// 非 async 函数也能返回 Promise（此时必须手动包）
function manual(): Promise<string> {
  return Promise.resolve("x");       // 没有 async 就没有自动包装
}
```

★ **关键认知**：`async` 是一层**返回值类型变换**——函数返回类型 `Promise<T>` 中的 `T` 是"return 表达式 await 之后的类型"，编译器自动完成 `T → Promise<T>` 的包装。⚠ 由此推论：`async` 函数永远返回 Promise（`Promise<void>` 也不能写成 `void`），且两个 `async` 函数返回类型相同不等价于内部实现一致——包装发生在编译器与运行时约定的边界上。

## 1.3 与 JS/Java/Go 的异步类型对照

| 维度 | TypeScript | JavaScript | Java | Go |
|------|-----------|-----------|------|-----|
| 异步返回值类型 | `Promise<T>`（类型化） | `Promise`（无类型） | `CompletableFuture<T>` | `(T, error)` 多返回值 |
| 失败通道类型 | 无（`any`/`unknown`） | 无 | `Exception`（有层级） | `error` 接口（显式） |
| 阻塞/非阻塞 | 纯非阻塞（微任务） | 同左 | 可阻塞 | goroutine + channel |
| 类型检查覆盖 | 成功值全覆盖 | — | 全覆盖 | 全覆盖 |

★ **关键认知**：Go 的 `(T, error)` 把失败**值化**、Java 把失败**类型化**（异常层级）、TS 的 Promise 原生只管成功——所以 TS 社区才会演化出 Result 模式（第 3 章）来补课。⚠ 别用 try/catch 模拟 Go 风格却不收窄 `catch` 变量，那只是把 `any` 从一个通道挪到另一个通道。

# 2. Awaited<T> 与 Promise 组合子的类型

## 2.1 Awaited<T>：展开多层 Promise

```typescript
// 手写 Awaited 的核心（简化自 lib 定义）：递归剥掉 thenable 的壳
type MyAwaited<T> =
  T extends null | undefined ? T :                              // null/undefined 原样保留
  T extends { then(onfulfilled: (value: infer V) => any): any } // 能 .then 就是 thenable
    ? MyAwaited<V>                                              // 递归展开
    : T;                                                        // 裸值直接返回

// 实际效果
type A = Awaited<Promise<string>>;               // string
type B = Awaited<Promise<Promise<number>>>;      // number：递归展开多层
type C = Awaited<Promise<string> | number>;      // string | number：联合逐分支展开
```

★ **关键认知**：`Awaited<T>` 是**递归条件类型 + `infer`** 的经典应用——它不只剥 Promise，而是识别一切 thenable（有 `then` 方法的对象），并且对联合类型**分布式展开**。⚠ 递归类型有实例化深度上限（约 50 层），对超深嵌套的 Promise 链会报 TS2589（Type instantiation is excessively deep）。

## 2.2 Promise.all / allSettled / race 的推断细节

```typescript
const pNum = Promise.resolve(1);                  // Promise<number>
const pStr = Promise.resolve("a");                // Promise<string>

// Promise.all：元组入参 → 逐位置精确推断
const [n, s] = await Promise.all([pNum, pStr]);   // n: number, s: string（不是联合！）

// 元组丢失的场景：数组变量（非字面量）退化为联合数组
const arr = [pNum, pStr];                         // (Promise<number> | Promise<string>)[]
const results = await Promise.all(arr);           // (number | string)[]：位置信息丢失

// Promise.allSettled：结果是可辨识联合，靠 status 收窄
const settled = await Promise.allSettled([pNum, pStr]);
for (const r of settled) {
  if (r.status === "fulfilled") {
    r.value;                                      // 收窄到成功值（number | string）
  } else {
    r.reason;                                     // ⚠ 类型是 any，取用时先落 unknown
  }
}

// Promise.race：返回最快的那个 → 成功值类型是联合
const winner = await Promise.race([pNum, pStr]);  // number | string
```

★ **关键认知**：`Promise.all` 的精确元组推断依赖**数组字面量的位置信息**——TS 对字面量数组做元组推断再逐元素 `Awaited`，而一旦先存成变量（宽化为数组）位置信息就丢了。⚠ `PromiseSettledResult` 是教科书级的**可辨识联合**（`status` 是字面量判别字段），但 `reason: any` 的定义提醒我们：内建类型也会留 `any` 的口子，收窄后第一件事是转 `unknown`。

# 3. 错误处理的类型设计

## 3.1 catch 变量默认 unknown

```typescript
try {
  throw new Error("boom");
} catch (e) {                        // useUnknownInCatchVariables 开启时：e: unknown
  // console.log(e.message);         // ✗ TS18046: 'e' is of type 'unknown'
  if (e instanceof Error) {          // ✓ 类型收窄：instanceof 是可靠的收窄手段
    console.log(e.message);          // 收窄到 Error，属性可用
  }
}
```

★ **关键认知**：`useUnknownInCatchVariables`（strict 家族成员）把 catch 变量从 `any` 改成 `unknown`，逼你显式**类型收窄**后才能用。⚠ `throw` 可以抛任何值（JS 允许 `throw "字符串"`），所以 `unknown` 才是诚实的类型——Java 的 `catch (Exception e)` 至少保证了异常对象的形状，TS 的 catch 什么承诺都没有。

## 3.2 自定义错误层级

```typescript
// 应用错误基类：保留 instanceof 与堆栈（两个经典坑一起修）
class AppError extends Error {
  constructor(message: string, readonly code: string) {
    super(message);                                  // 初始化基类
    this.name = new.target.name;                     // name 反映实际子类（AppError/NetworkError）
    Object.setPrototypeOf(this, new.target.prototype); // ⚠ target 低于 ES6 时修复原型链
  }
}

class NetworkError extends AppError {
  constructor(readonly url: string) {
    super(`fetch failed: ${url}`, "NETWORK");        // 子类补充领域信息
  }
}

class ParseError extends AppError {
  constructor(readonly raw: unknown) {
    super("response body does not match schema", "PARSE");
  }
}

// 消费处：按层级分层捕获、按 code 分支处理
try {
  risky();
} catch (e) {
  if (e instanceof NetworkError) { retry(e.url); }          // 精确类型 + 精确字段
  else if (e instanceof AppError) { report(e.code); }       // 基类兜底
  else { throw e; }                                          // 未知错误原样上抛
}
```

★ **关键认知**：⚠ `extends Error` 在 `target` 低于 ES6 时有著名陷阱——降级后的原型链断裂导致 `instanceof AppError` 返回 `false`，`Object.setPrototypeOf(this, new.target.prototype)` 是标准修复。错误层级的设计哲学与 Java 异常体系一致（基类 + 领域子类 + 分层捕获），差别是 TS 的 `throw` 无签名约束，全靠纪律。

## 3.3 Result 模式：Ok / Err 可辨识联合

```typescript
// 手写 Result：把"失败"从异常通道搬进返回值通道（Go 风格的类型化版本）
type Result<T, E = AppError> =
  | { ok: true; value: T }        // 成功分支携带 value
  | { ok: false; error: E };      // 失败分支携带 error，判别字段是 ok

// 构造器：字面量收窄让返回值类型精确
function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };     // ok 字段是字面量 true，不是 boolean
}
function err<E>(error: E): Result<never, E> {
  return { ok: false, error };
}

// 实战：返回 Result 的解析函数——不 throw，把错误显式交给调用方
function parsePort(input: string): Result<number, AppError> {
  const n = Number(input);
  if (!Number.isInteger(n) || n < 1 || n > 65535) {
    return err(new AppError(`invalid port: ${input}`, "PARSE"));   // 失败：类型上可见
  }
  return ok(n);                                                   // 成功：类型上可见
}

// 消费处：一次解构 + 一次判别，穷尽收窄
const r = parsePort("8080");
if (r.ok) {
  const port: number = r.value;    // 收窄到成功分支
} else {
  const code: string = r.error.code;   // 收窄到失败分支，error 类型精确
}
```

★ **关键认知**：Result 模式的核心价值是**让失败出现在签名里**——`Result<number, AppError>` 的调用者不可能"忘记处理错误"，类型系统强制分支。⚠ 实现要点：构造器用 `{ ok: true }` 字面量（写 `ok: boolean` 判别字段就收窄不了）；联合分支的 `ok` 类型必须是字面量类型才能**类型收窄**。这正是 Go 显式 error 检查的 TS 版本，比 try/catch 的优势是错误类型可枚举、可组合。

# 4. 类型化 fetch 封装实战

## 4.1 泛型响应封装 + 类型守卫衔接

```typescript
// 类型守卫：运行时校验"未知数据是否符合 User 形状"
function isUser(v: unknown): v is User {
  if (typeof v !== "object" || v === null) return false;     // 先排掉非对象与 null
  const o = v as Record<string, unknown>;                    // 类型断言取属性（收窄起点）
  return typeof o.id === "number" && typeof o.name === "string";
}

// 泛型封装：T 由调用方指定，但必须提供校验函数才能拿到 T
async function getJson<T>(
  url: string,
  isT: (v: unknown) => v is T,        // 校验器作为参数：类型与运行时绑定
  signal?: AbortSignal                 // 可取消（见 5.3）
): Promise<T> {
  const res = await fetch(url, { signal });            // fetch: Promise<Response>
  if (!res.ok) throw new NetworkError(url);            // HTTP 错误走异常通道
  const body: unknown = await res.json();              // ⚠ res.json() 返回 any，先落 unknown
  if (!isT(body)) throw new ParseError(body);          // 校验失败即抛，返回值类型仍是 T
  return body;                                         // 类型守卫收窄：unknown → T
}

// 使用处：泛型 + 校验器一一对应
const user = await getJson("/api/user", isUser);       // user: User，且确实校验过
```

★ **关键认知**：⚠ **静态类型从不校验运行时数据**——`getJson<User>` 若不带 `isUser`，`User` 只是"作者的口头承诺"，服务端返回垃圾照样通过编译、运行时才炸。把类型守卫作为参数塞进封装（或返回 `Result<T>`），是让"类型承诺"与"运行时事实"闭环的最小手段。这与 Java 反序列化（框架按 Class 元数据校验）、Go 的 `json.Unmarshal` 后手动断言是同一问题的三种解法。

## 4.2 静态类型 ≠ 运行时校验：对照表

| 数据来源 | TS 视角 | 真实情况 | 必要动作 |
|---------|---------|---------|---------|
| 自己构造的对象 | 类型可信 | 可信 | 无需校验 |
| `res.json()` | `any`（默认） | 未知 | 类型守卫/校验函数 |
| `JSON.parse(str)` | `any` | 未知 | 同上 |
| 用户输入 / 配置文件 | 注解什么就是什么 | 未知 | 同上 |
| `as T` 类型断言 | 强转为 T | 未验证 | ⚠ 断言是"说服编译器"不是"验证数据" |

★ **关键认知**：`as` 是**类型断言**不是类型转换——它零运行时成本、零校验，只把"说服编译器"的责任转嫁给作者。⚠ 用 `as User` 绕过校验得到的类型安全是幻觉；正确姿势是 `unknown` 起步、守卫收窄、断言只留在守卫内部（如 `isUser` 里的 `Record<string, unknown>` 一步）。

# 5. 事件与异步迭代的类型

## 5.1 类型安全 EventEmitter：泛型事件映射表

```typescript
// 事件映射表：事件名 → 参数元组（替代"全 any 参数"的裸回调）
interface UserEvents {
  login: [user: User, at: Date];           // login 事件：两个参数，类型精确
  logout: [];                              // 无参事件
  error: [e: AppError];                    // 带错误对象
}

class TypedEmitter<E extends Record<string, unknown[]>> {
  // 内部存储刻意放宽为 any[]（类内边界），对外 API 全程严格
  private handlers = new Map<keyof E, Array<(...args: any[]) => void>>();

  on<K extends keyof E>(event: K, cb: (...args: E[K]) => void): this {
    const list = this.handlers.get(event) ?? [];
    list.push(cb);                         // 存储（内部 any[] 兜住）
    this.handlers.set(event, list);
    return this;
  }

  emit<K extends keyof E>(event: K, ...args: E[K]): boolean {
    const list = this.handlers.get(event);
    if (!list) return false;
    for (const cb of list) cb(...args);    // 参数类型由 E[K] 保证
    return true;
  }
}

const em = new TypedEmitter<UserEvents>();
em.on("login", (user, at) => console.log(user.name, at));   // ✓ 回调参数自动推断
em.emit("login", { id: 1, name: "Alice" }, new Date());     // ✓ 参数个数/类型都查
// em.emit("login", new Date());                            // ✗ TS2345：参数不匹配
```

★ **关键认知**：泛型事件映射把"事件名字符串 + 松散参数"建模为**映射类型**（第 11 篇的手法），回调参数通过 `E[K]` 的元组展开获得精确推断——`K` 与 `E[K]` 的联动是**依赖式泛型**（参数类型随另一个参数收窄）。⚠ 内外类型的边界（内部 `any[]`）要集中在类内一处，一旦泄漏到 API 签名上类型安全就破防。

## 5.2 AsyncIterable 与 Generator 的类型

```typescript
// 同步生成器：Generator<产出值, 返回值, next 传入值>
function* counter(limit: number): Generator<number, string, undefined> {
  for (let i = 0; i < limit; i++) {
    yield i;                          // yield 出去的类型是 number（第一个参数）
  }
  return "done";                      // 结束值类型是 string（第二个参数）
}

// 异步生成器：AsyncGenerator<产出值, 返回值, next 传入值>
async function* fetchPages(url: string): AsyncGenerator<unknown, void, undefined> {
  let page = 1;
  while (true) {
    const body: unknown = await (await fetch(`${url}?page=${page}`)).json();
    if (!Array.isArray(body) || body.length === 0) return;   // 结束（TReturn = void）
    yield body;                       // 产出一页数据
    page++;
  }
}

// 消费：for await 按 AsyncIterable 协议取值
for await (const page of fetchPages("/api/items")) {
  console.log(page);                  // page: unknown（产出值类型）
}
```

★ **关键认知**：`Generator<TOut, TReturn, TNext>` 的三个参数分别对应 `yield`/`return`/`next()` 三个通道——这与 `Promise<T>` 只管一个成功通道的建模形成对比：生成器是**双向**值传递（产出值 + 回传值）。⚠ 与 JS 无类型时"随便 yield"不同，TS 下三个通道都要显式交代，漏写会退化成 `Generator<any, void, unknown>`。

## 5.3 AbortSignal：可取消异步的类型

```typescript
// 可取消模式：信号对象作为可选参数（与 fetch 的签名风格一致）
async function fetchWithTimeout<T>(
  url: string,
  isT: (v: unknown) => v is T
): Promise<T> {
  const ctrl = new AbortController();           // 创建控制器
  const timer = setTimeout(() => ctrl.abort(), 3000);   // 3 秒后触发取消
  try {
    const res = await fetch(url, { signal: ctrl.signal });   // 传信号，取消即抛 AbortError
    const body: unknown = await res.json();
    if (!isT(body)) throw new ParseError(body);
    return body;
  } catch (e) {
    if (e instanceof DOMException && e.name === "AbortError") {
      throw new AppError(`timeout: ${url}`, "TIMEOUT");     // 取消转成领域错误
    }
    throw e;                                                 // 其余错误原样上抛
  } finally {
    clearTimeout(timer);                                     // 清理定时器
  }
}
```

★ **关键认知**：`AbortSignal` / `AbortController` 是"取消"这件事的**标准化类型接口**——信号（signal）交给被调方、控制器（controller）留在调用方，职责分离且类型内建。⚠ 取消在 TS 类型上无迹可寻（签名只是 `Promise<T>`），区分"超时取消"与"真实网络错误"必须靠 `AbortError` 的运行时判断——又一次印证：静态类型描述不了的维度，只能靠运行时代码补。
