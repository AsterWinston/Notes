Rust 把错误分为两类：**不可恢复错误**用 `panic!`，**可恢复错误**用 `Result<T, E>`。这种显式区分配合 `?` 运算符，让错误处理既安全又不啰嗦。

# 1. panic!()

## 1.1 两种 panic

```rust
// 1. 显式 panic
panic!("something went wrong");

// 2. 隐式 panic（以下情况自动 panic）
let v = vec![1, 2, 3];
v[99];                    // 越界
let x = Some(5).unwrap(); // unwrap 一个 None
```

★ **关键认知**：`panic!` 表示"**不可恢复的致命错误**"。默认行为是**栈展开（unwinding）**——逐层回溯调用栈、调用各层的 drop 清理资源，然后退出程序。也可通过 `panic = "abort"` 配置为直接终止（不清理，二进制更小）。

## 1.2 展开 vs 终止

```toml
# Cargo.toml
[profile.release]
panic = "abort"    # 直接终止，不展开栈（体积更小、性能略好，但不能 catch_unwind）
```

★ **关键认知**：栈展开（unwind）会调用 `Drop` 做清理，代价是二进制更大、panic 稍慢；`abort` 直接终止，无清理。默认（debug 和 release）都是 unwind。⚠ 栈展开期间若 `Drop` 又 panic，会直接 abort。

## 1.3 何时用 panic

```
应该 panic：
  - 程序 bug（逻辑上不可能发生的状态）
  - 测试代码、原型、示例
  - 违反前置条件（如数组越界）

不应 panic（应返回 Result）：
  - 可预期的错误（文件不存在、解析失败、网络超时）
  - 库代码（把错误决定权交给调用方）
```

# 2. Result<T, E>

## 2.1 定义

```rust
// 标准库定义（简化）
enum Result<T, E> {
    Ok(T),      // 成功，携带值 T
    Err(E),     // 失败，携带错误 E
}
```

★ **关键认知**：`Result<T, E>` 是可恢复错误的载体，与 `Option<T>` 类似（都是枚举），区别是 `Err` 携带**错误信息**。它是 Rust 错误处理的核心类型——文件操作、解析、网络 I/O 都返回 `Result`。

## 2.2 基本处理

```rust
use std::fs::File;

let f = File::open("hello.txt");

let f = match f {
    Ok(file) => file,
    Err(error) => panic!("Problem opening the file: {:?}", error),
};
```

## 2.3 错误类型匹配

```rust
use std::io::ErrorKind;

let f = match File::open("hello.txt") {
    Ok(file) => file,
    Err(error) => match error.kind() {
        ErrorKind::NotFound => match File::create("hello.txt") {
            Ok(fc) => fc,
            Err(e) => panic!("Problem creating: {:?}", e),
        },
        other_error => panic!("Problem opening: {:?}", other_error),
    },
};
```

## 2.4 return 与传播

```rust
// 用 return 显式传播
fn read_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");
    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
    };
    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
}
```

## 2.5 ? 运算符与 From

★ **关键认知**：`?` 是 Rust 的**错误传播运算符**——表达 `Result`/`Option` 时，`Ok(v)` 解包为 v，`Err(e)` 则**提前 return Err(e)**。它极大简化了错误传播：

```rust
fn read_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;     // 出错直接返回
    let mut s = String::new();
    f.read_to_string(&mut s)?;                // 出错直接返回
    Ok(s)
}
```

★ **关键认知**：`?` 只能用于返回类型是 `Result`/`Option` 的函数。它还内置了 **`From` 转换**——遇到 `Err(e)` 时，会调用 `From::from(e)` 把错误类型转换成函数返回的错误类型。这使得"不同错误类型自动统一"成为可能。

## 2.6 ? 与 Option

```rust
fn first_char(s: &str) -> Option<char> {
    let c = s.chars().next()?;    // None 则提前返回 None
    Some(c)
}
```

★ **关键认知**：`?` 对 `Option` 同样有效——`None` 时提前返回 `None`。这让你能写链式的 Option 处理。

# 3. unwrap() 与 expect()

```rust
// unwrap：Ok 则取值，Err 则 panic（错误信息为 Debug 输出）
let f = File::open("hello.txt").unwrap();

// expect：Ok 则取值，Err 则 panic（自定义错误信息）
let f = File::open("hello.txt").expect("Failed to open hello.txt");
```

★ **关键认知**：`unwrap`/`expect` 会让程序在 `Err` 时 panic，**只应用于你确定不会出错或出错即致命**的场景（原型、测试、逻辑上不可能的 None）。库代码和可预期错误应避免。`expect` 比 `unwrap` 好，因为它能给出有意义的错误信息，便于定位问题。

# 4. 错误处理的最佳实践

## 4.1 自定义错误类型

```rust
#[derive(Debug)]
enum MyError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
}

// 实现 From 使 ? 能自动转换
impl From<std::io::Error> for MyError {
    fn from(e: std::io::Error) -> Self {
        MyError::Io(e)
    }
}

fn work() -> Result<i32, MyError> {
    let n: i32 = "42".parse()?;   // 自动 From<ParseIntError>
    Ok(n)
}
```

## 4.2 Box<dyn Error> 与 anyhow

```rust
use std::error::Error;

// 需要返回"任意错误类型"时用 Box<dyn Error>
fn work() -> Result<(), Box<dyn Error>> {
    let f = File::open("x.txt")?;    // 任意实现 Error 的类型都能转换
    Ok(())
}
```

★ **关键认知**：`Box<dyn Error>` 是"**错误类型擦除**"的方案——任何实现了 `Error` 的类型都能自动转成它，适合应用层（main、顶层逻辑）。库代码应定义**具体的错误枚举**（让调用方能匹配处理），应用代码可用 `anyhow` 简化。

## 4.3 thiserror 与 anyhow

```
thiserror      // 为库定义结构化错误枚举（derive 自动实现 Error/From）
anyhow         // 为应用简化错误处理（anyhow::Result<T>、context 附加信息）
```

★ **关键认知**：这是 Rust 社区的两大错误处理库——**thiserror 给库用**（定义清晰的错误类型），**anyhow 给应用用**（图方便、快速传播）。二者常配合使用。

# 5. 小结

| 概念 | 要点 |
|------|------|
| panic! | 不可恢复错误，默认栈展开 |
| Result<T,E> | 可恢复错误，Ok/Err |
| ? 运算符 | 错误传播 + From 自动转换 |
| unwrap/expect | Err 时 panic，慎用 |
| 自定义错误 | 枚举 + impl From |
| Box<dyn Error> | 错误类型擦除（应用层） |
| thiserror/anyhow | 库/应用错误处理库 |

下一篇进入泛型、trait 与生命周期，这是 Rust 类型系统的三大支柱，也是编写可复用、安全抽象的关键。
