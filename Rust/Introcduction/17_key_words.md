Rust 的关键字分三类：**保留关键字**（当前使用）、**保留未使用**（为将来预留）和**上下文关键字**（只在特定位置有特殊含义）。了解它们有助于读懂语言的全貌边界。

# 1. 保留关键字

这些是 Rust 当前正在使用的关键字，不能用作标识符（变量/函数/类型名）。

```
as       类型转换 / 重命名导入
break    跳出循环
const    常量 / 常量指针
continue 继续下一次循环
crate    当前 crate 的根路径
else     if 的否则分支
enum     枚举定义
extern   外部函数接口（FFI）
false    布尔假值
fn       函数定义
for      for 循环
if       条件分支
impl     实现块
in       for 循环中 / 范围
let      变量绑定
loop     无限循环
match    模式匹配
mod      模块声明
move     move 闭包 / 转移所有权
mut      可变性标记
pub      公开可见性
ref      引用绑定（模式中）
return   返回值
self     方法接收者 / 当前模块
Self     当前类型（impl 中）
static   静态变量
struct   结构体定义
super    父模块路径
trait    trait 定义
true     布尔真值
type     类型别名 / 关联类型
unsafe   unsafe 块 / 函数
use      引入路径
where    trait 约束子句
while    while 循环
async    异步函数/块（2018+）
await    等待 Future（2018+）
dyn      动态分发 trait 对象（2018+）
```

| as | break | const | continue | crate |
|:--:|:-----:|:-----:|:--------:|:-----:|
| else | enum | extern | false | fn |
| for | if | impl | in | let |
| loop | match | mod | move | mut |
| pub | ref | return | self | Self |
| static | struct | super | trait | true |
| type | unsafe | use | where | while |
| async | await | dyn | | |

★ **关键认知**：其中 `async`/`await`/`dyn` 是 2018 edition 引入的关键字；`try` 被保留给未来的错误处理（现在用 `?`）。`self`/`Self`/`super`/`crate` 是特殊的**路径关键字**。

# 2. 保留未使用

这些关键字被 Rust 保留但**当前未使用**，为将来可能的语法特性预留，现阶段同样不能用作标识符。

| abstract | become | box | do | final |
|:--------:|:------:|:---:|:--:|:-----:|
| macro | override | priv | try | typeof |
| unsized | virtual | yield | | |

★ **关键认知**：保留未使用关键字体现了 Rust 对**向后兼容**的重视——先把名字占住，避免将来引入新特性时与用户代码冲突。`try` 曾用于旧的 `try!` 宏，`yield`/`box` 等则是未来可能引入的特性预留。

# 3. 上下文关键字

这些词**只在特定上下文中**被当作关键字，平时可以作为普通标识符使用（如变量名）。

| 关键字 | 出现位置 | 含义 |
|--------|----------|------|
| `union` | `union Foo {}` | 联合体定义 |
| `macro_rules` | `macro_rules! name` | 声明宏 |
| `dyn` | `dyn Trait` | 动态 trait 对象 |
| `async` | `async fn` | 异步 |
| `await` | `expr.await` | 等待 |
| `try` | `try { }`（预留） | 错误传播块 |
| `'static` | 生命周期位置 | 静态生命周期 |

★ **关键认知**：上下文关键字让 Rust 能在**不破坏已有代码**的前提下扩展语言——比如 `union` 引入前，很多代码用 `union` 作变量名；把它设为上下文关键字，只有出现在 `union Foo` 的语法位置时才当关键字。

# 4. 弱关键字与标识符规则

## 4.1 弱关键字

```
safe      只在特定语境有含义
default   只在 impl 中（specialization）有含义
```

## 4.2 原始标识符 raw identifier

```rust
// 用 r# 前缀可以把关键字当标识符用（如与老代码/FFI 兼容）
let r#match = 3;
fn r#async() {}
```

★ **关键认知**：`r#` 前缀（raw identifier）允许把关键字当作普通标识符，主要用于**与旧版本代码或外部接口交互**的场景，避免因新增关键字导致已有标识符失效。

# 5. 小结

| 分类 | 能否作标识符 | 示例 |
|------|--------------|------|
| 保留关键字 | 否 | fn, let, match, async |
| 保留未使用 | 否（预留） | try, yield, box, become |
| 上下文关键字 | 视位置而定 | union, dyn, macro_rules |
| 原始标识符 | 用 r# 可绕过 | r#match |

本系列笔记至此结束。从工具链、变量、所有权，到 trait、并发、智能指针、面向对象、unsafe 与宏，覆盖了 Rust 语言从入门到核心特性的完整脉络。后续可继续深入：异步编程（async/await/Future）、`tokio` 运行时、FFI 与 `no_std` 等方向。
