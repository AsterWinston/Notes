Rust 把测试作为一等公民内建在 cargo 中，无需第三方框架。测试分为**单元测试**（与被测代码同文件）和**集成测试**（独立 tests 目录），配合丰富的断言宏和运行选项。

# 1. cargo test 和 #[test]

## 1.1 基本写法

```rust
#[test]
fn it_works() {
    assert_eq!(2 + 2, 4);
}
```

```
运行：cargo test

cargo test 会自动：
1. 编译成测试模式（带 test harness）
2. 运行所有 #[test] 标注的函数
3. 输出通过/失败统计
```

★ **关键认知**：`#[test]` 是**属性宏**，标注测试函数。`cargo test` 会为每个 `#[test]` 函数启动一个线程运行（可用 `--test-threads` 控制并发）。测试函数失败 = panic。

## 1.2 测试失败的机制

★ **关键认知**：测试函数**通过 = 不 panic**；**失败 = panic**。所以任何断言失败（断言宏内部就是 panic）或 `panic!` 都会导致测试失败。这比返回错误码的方式更简洁。

# 2. assert —— 断言宏

```
2.1 assert!(condition)               // 条件为 false 则 panic
2.2 assert_eq!(left, right)          // 相等断言，要求类型实现 PartialEq + Debug
2.3 assert_ne!(left, right)          // 不等断言
```

```rust
#[test]
fn test_assertions() {
    assert!(true);
    assert_eq!(2 + 2, 4);
    assert_ne!(3, 4);
}
```

★ **关键认知**：`assert_eq!`/`assert_ne!` 要求操作数实现 `PartialEq`（比较）和 `Debug`（失败时打印）。它们比手写 `assert!(a == b)` 好在**失败时会打印两边的值**，便于定位。自定义类型可用 `#[derive(PartialEq, Debug)]` 满足要求。

# 3. 自定义错误信息

```rust
#[test]
fn test_with_message() {
    let result = 2 + 2;
    assert_eq!(result, 4, "计算结果错了：期望 4，实际 {}", result);
}
```

★ **关键认知**：所有断言宏都支持**追加自定义消息**（剩余参数是 `format!` 风格的参数）。当断言失败时，会一并打印这条消息，帮助快速定位问题。

# 4. #[should_panic]

## 4.1 期望 panic

```rust
#[test]
#[should_panic]
fn test_panic() {
    panic!("this test expects a panic");
}
```

★ **关键认知**：`#[should_panic]` 表示"**该测试期望 panic**"——如果没 panic 反而失败。用于测试"非法输入应该 panic"的场景。

## 4.2 expected 子串

```rust
#[test]
#[should_panic(expected = "division by zero")]
fn test_division_by_zero() {
    panic!("division by zero");
}
```

★ **关键认知**：`expected` 参数要求 panic 消息**包含指定子串**，避免"任何 panic 都算通过"的假阳性。这让测试更精确。

# 5. Result<T, E> 作为测试返回

```rust
#[test]
fn test_result() -> Result<(), String> {
    if 2 + 2 == 4 {
        Ok(())
    } else {
        Err(String::from("two plus two does not equal four"))
    }
}
```

★ **关键认知**：测试函数可返回 `Result<(), E>`——返回 `Ok(())` 表示通过，`Err(e)` 表示失败（并打印 e）。这样可以在测试里使用 `?` 运算符，适合需要传播错误的测试。⚠ 用 Result 返回时**不能同时用 `#[should_panic]`**。

# 6. cargo test 常用选项

```
cargo test --help          # 查看 cargo test 的选项
cargo test -- --help       # 查看测试二进制的选项（-- 后面的参数传给测试运行器）

cargo test -- --test-threads=1     # 单线程运行（避免并发干扰）
cargo test -- --nocapture          # 显示测试中的 println! 输出（默认被捕获）
cargo test -- --show-output        # 显示成功测试的输出
```

★ **关键认知**：`cargo test -- --xxx` 中的 `--` 用于把参数**透传给测试运行器**（test harness）。默认情况下测试的 `println!` 输出会被捕获，只有失败时才显示；用 `--nocapture` 可强制显示。

# 7. 按名称运行测试

```bash
cargo test add            # 只运行名字包含 "add" 的测试
cargo test tests::add     # 用模块路径精确过滤
cargo test -- --exact foo # 精确匹配名字为 foo 的测试
cargo test some_module::  # 运行某模块下的所有测试
```

★ **关键认知**：`cargo test <filter>` 会让测试运行器只跑**名字包含该子串**的测试。这是快速迭代单个测试的主要方式。

# 8. 忽略测试：#[ignore]

```rust
#[test]
#[ignore]
fn expensive_test() {
    // 耗时测试，默认跳过
}
```

```bash
cargo test -- --ignored          # 只运行被忽略的测试
cargo test -- --include-ignored  # 运行所有（含被忽略的）
```

★ **关键认知**：`#[ignore]` 标记"默认不运行"的测试（如耗时、需要特殊环境）。用 `--ignored` 单独运行它们，或用 `--include-ignored` 全部运行。

# 9. 组织测试

## 9.1 单元测试：#[cfg(test)]

```rust
// 在被测代码同一文件底部
pub fn add(a: i32, b: i32) -> i32 { a + b }

#[cfg(test)]
mod tests {
    use super::*;         // 引入父模块（被测代码）

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
}
```

★ **关键认知**：单元测试放在源代码文件底部的 `#[cfg(test)] mod tests` 中。`#[cfg(test)]` 表示**只在测试编译时包含**这段代码（正常构建时被剔除）。`use super::*` 让测试能访问父模块的私有项——这是私有函数可被测试的关键。

## 9.2 集成测试：tests 目录

```
项目结构：
├── src/
│   └── lib.rs
└── tests/                 // 集成测试目录
    ├── integration_test.rs
    └── common/
        └── mod.rs
```

```rust
// tests/integration_test.rs
use my_crate;              // 像外部用户一样引用 crate

#[test]
fn test_public_api() {
    assert_eq!(my_crate::add(2, 3), 5);
}
```

★ **关键认知**：`tests/` 目录下每个文件是**独立的 crate**，只能访问库的**公开 API**（像真实用户）。这适合测试"整体行为与公共接口"。`tests/common/mod.rs` 这类子目录不会被当独立测试文件，可作为共享辅助代码。

# 10. 小结

| 概念 | 要点 |
|------|------|
| #[test] | 标注测试函数 |
| assert!/eq!/ne! | 断言宏，可加自定义消息 |
| #[should_panic] | 期望 panic，可加 expected |
| Result 返回 | 测试可用 ? 传播错误 |
| --nocapture | 显示 println 输出 |
| 名称过滤 | cargo test <substr> |
| #[ignore] | 默认跳过的测试 |
| #[cfg(test)] | 单元测试模块 |
| tests/ | 集成测试目录 |

下一篇进入闭包与迭代器，讲清楚闭包的三种捕获方式、`Fn`/`FnMut`/`FnOnce` trait、`move` 关键字，以及强大的迭代器链式操作。
