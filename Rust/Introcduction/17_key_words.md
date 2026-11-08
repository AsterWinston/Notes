Rust 的关键字分三类：**保留关键字**（当前使用）、**保留未使用**（为将来预留）和**上下文关键字**（只在特定位置有特殊含义）。本篇除分类清单外，重点给出按用途分组的速查表，并把易混点（as/dyn/impl、loop/while/for 返回值、ref 位置语义、pub(crate) 可见性、where 子句）一次讲清。

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

## 1.1 按用途分组速查

```
多态与抽象   dyn（运行时 vtable）/ impl（编译期单态）/ trait / auto trait（Send/Sync）
并发与异步   async / await / move / gen（已重新保留，见第 2 节）
内存与契约   unsafe / extern / static / const
生命周期     'a（参数位置）/ 'static（永远有效，或 T: 'static "不含短引用"）
可见性       pub / pub(crate) / pub(super) / pub(in path)
模式与绑定   ref / ref mut / let-else / if let / while let
约束表达     where / 冒号 bound（T: Trait）/ dyn + Send 组合 bound
```

```rust
fn spawn_ok<T>(v: T) where T: Send + 'static {      // where 放签名与函数体之间，比冒号 bound 更易读
    std::thread::spawn(move || drop(v));            // move：所有权进闭包；T: 'static 才允许离开本栈帧
}

trait Animal { fn name(&self) -> &'static str; }
struct Dog;
impl Animal for Dog { fn name(&self) -> &'static str { "dog" } }

let a: &dyn Animal = &Dog;                          // dyn：运行期查 vtable，可异构收集
let c: Box<dyn Animal + Send> = Box::new(Dog);      // trait 对象 + auto trait bound
```

```
&dyn Animal 是胖指针（16 字节）：
┌──────────────┬──────────────┐
│   数据指针   │  vtable 指针 │
└──────────────┴──────────────┘
```

★ **关键认知**：`impl Trait` 与 `dyn Trait` 是同一多态问题的两种答案——前者编译期**单态化**（零开销、类型必须固定），后者运行期**vtable 分发**（可异构收集、有装箱与间接调用成本）。⚠ `dyn Animal + Send` 的 `Send` 不是装饰：跨线程搬运 trait 对象缺它直接编译失败。

## 1.2 易混关键字对照：as vs dyn vs impl

| 关键字 | 所在层 | 作用 | 典型误用 |
|--------|--------|------|----------|
| `as` | 表达式/路径 | 显式数值转换；use 重命名 | `x as u8` 截断值域不报错 |
| `dyn` | 类型位置 | trait 对象，运行期分发 | 无脑 `&dyn` 付出 vtable 开销 |
| `impl` | 项/参数位置 | 实现块；编译期单态参数 | 作返回类型却想返回多种类型 |

```rust
fn demo(v: i64) -> u8 {
    let n = v as u8;                                // as：按位截断，值损失自担
    let b: Box<dyn std::fmt::Display> = Box::new(n);// dyn：装箱 trait 对象
    n
}
```

## 1.3 loop / while / for：返回值与 let 家族

```rust
fn demo(stack: &mut Vec<i32>) {
    let x = loop { break 42; };                 // loop 是表达式：break 带值即"返回值"
    let _ = x;

    let mut i = 0;
    while i < 3 { i += 1; }                     // while / for 永远返回 ()
    for e in [1, 2, 3] { let _ = e; }

    let Some(v) = Some(1) else { return };      // let-else（1.65 稳定）：不匹配走发散分支
    let _ = v;
    if let Ok(n) = "42".parse::<i32>() { let _ = n; }   // 单一模式的 match 简写
    while let Some(t) = stack.pop() { let _ = t; }      // 配合栈/迭代器的惯用循环
}
```

★ **关键认知**：三个循环关键字的**类型不同**——只有 `loop` 能通过 `break 值` 产出任意类型，`while`/`for` 固定返回 `()`。"重试直到成功"用 `loop` 最顺手：成功 `break ok`、失败 `continue`。`if let`/`while let`/`let-else` 是 match 的三种裁剪。

## 1.4 ref / ref mut 的位置语义

```rust
let p = Some(String::from("hi"));

if let Some(ref s) = p {        // ref 在模式侧：绑定 &String，p 不被移动
    println!("{s}");
}
println!("{p:?}");              // p 仍可用

let mut q = Some(1);
if let Some(ref mut v) = q { *v += 1; }   // ref mut：按可变引用绑定
```

★ **关键认知**：`ref`/`ref mut` 只出现在**模式位置**，决定解构时"按引用绑定还是按值移动"——默认规则是"能 Copy 就复制、否则移动"。⚠ 现代代码可用 `match &x` 自动取引用（match ergonomics），但读老代码必须认识它。

## 1.5 pub 与路径可见性

```rust
mod inner {
    pub fn exported() {}                        // 任何 crate 可见
    pub(crate) fn crate_only() {}               // 仅本 crate
    mod deeper {
        pub(super) fn parent_can_call() {}      // 仅父模块
        pub(in crate::inner) fn scoped() {}     // 指定模块
    }
}
```

★ **关键认知**：Rust 默认可见性是**私有**，`pub` 可带路径上界：`pub(crate)` 是库作者最常用的——内部工具全 crate 可用、不出公共 API；`pub(super)` 只暴露给父模块。⚠ 结构体本身私有则 `pub` 字段外部也摸不到。

# 2. 保留未使用

这些关键字被 Rust 保留但**当前未使用**，为将来的语法特性预留，同样不能用作标识符。

| abstract | become | box | do | final |
|:--------:|:------:|:---:|:--:|:-----:|
| macro | override | priv | try | typeof |
| unsized | virtual | yield | | |

★ **关键认知**：保留未使用关键字体现了 Rust 对**向后兼容**的重视——先把名字占住，避免将来新特性与用户代码冲突。`try` 曾用于旧的 `try!` 宏，`become` 为尾调用优化预留，`yield`/`box` 对应协程与堆分配语法方向；`abstract`/`final`/`override`/`virtual` 是早期面向对象草案的遗迹——Rust 最终选择 trait + dyn 而非继承。

## 2.1 关键字演进时间线（2015 → 2024）

```
2015 edition   初版清单；become/virtual/override/abstract 等早期设计残留被保留未用
2018 edition   async / await 升格保留；dyn 从上下文关键字升为保留字；
               try 块只走 unstable 路线（? 运算符定于一尊）
2021 edition   无新保留字；闭包按字段捕获（disjoint capture）改变 move 的实际语义
2024 起        gen 重新保留：gen { yield ... } 与 async gen 推进中，
               rustc 开始对 `let gen = ...` 发保留字警告，与 yield 正好配套
```

★ **关键认知**：保留字清单是**活的**——`dyn`/`async` 从上下文升保留、`try` 路线被 `?` 取代、`gen` 2024 年重新入列。跨 edition 升级时"某个变量名突然告警"，八成是它被将来特性征用了；此时要么改名，要么用 `r#gen` 原始标识符过渡。

# 3. 上下文关键字

这些词**只在特定上下文中**被当作关键字，平时可以作为普通标识符使用。

| 关键字 | 出现位置 | 含义 |
|--------|----------|------|
| `union` | `union Foo {}` | 联合体定义 |
| `macro_rules` | `macro_rules! name` | 声明宏 |
| `dyn` | `dyn Trait` | 动态 trait 对象 |
| `async` | `async fn` | 异步 |
| `await` | `expr.await` | 等待 |
| `try` | `try { }`（预留） | 错误传播块 |
| `'static` | 生命周期位置 | 静态生命周期 |

```rust
let union = 1;      // 合法：不在 union Foo {} 语法位置就是普通标识符
// async / await 在 2018+ edition 已升为保留字，直接作标识符需要 r# 前缀
```

★ **关键认知**：上下文关键字让 Rust 能在**不破坏已有代码**的前提下扩展语言——比如 `union` 引入前，很多代码用它作变量名；设为上下文关键字后，只有出现在 `union Foo` 的语法位置才当关键字。对照 2.1 时间线可见这类关键字会随 edition **升降级**：`dyn`/`async`/`await` 原本是上下文关键字，2018 升为保留字。

# 4. 弱关键字与标识符规则

## 4.1 弱关键字

```
safe      只在特定语境有含义（如未来 safe extern fn 的标注）
default   只在 impl 中（specialization）有含义
```

## 4.2 原始标识符 raw identifier

```rust
// 用 r# 前缀可以把关键字当标识符用（如与老代码/FFI 兼容）
let r#match = 3;
fn r#async() {}
```

★ **关键认知**：`r#` 前缀（raw identifier）允许把关键字当普通标识符，主要用于**与旧版本代码或外部接口交互**——2015 代码里叫 `async` 的函数迁到 2018 edition 时，迁移工具正是自动改写成 `r#async`。⚠ 引用处也必须带 `r#`，两边写法不一致会编译不过。

# 5. 小结

| 分类 | 能否作标识符 | 示例 |
|------|--------------|------|
| 保留关键字 | 否 | fn, let, match, async |
| 保留未使用 | 否（预留） | try, yield, box, become |
| 上下文关键字 | 视位置而定 | union, dyn, macro_rules |
| 弱关键字 | 是 | safe, default |
| 原始标识符 | 用 r# 可绕过 | r#match |

| 易混点 | 一句话结论 |
|--------|------------|
| as / dyn / impl | 显式转换 / 运行期 vtable / 编译期单态 |
| loop vs while/for | 只有 loop 能 break 出值 |
| ref vs & | 模式侧取引用 vs 表达式侧构造引用 |
| 'static 两方向 | 引用永存 vs T: 'static 不含短引用 |
| gen / yield | 2024 重新保留，async gen 在路上 |

本系列笔记至此结束。从工具链、变量、所有权，到 trait、并发、智能指针、面向对象、unsafe 与宏，覆盖了 Rust 从入门到核心特性的完整脉络。后续可继续深入：异步编程（async/await/Future）、`tokio` 运行时、FFI 与 `no_std` 等方向。
