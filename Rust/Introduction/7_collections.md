Rust 标准库提供了几种常用集合，它们的数据都分配在**堆上**。最常用的三种是：`Vec<T>`（动态数组）、`String`（可变字符串）、`HashMap<K, V>`（哈希表）。

# 1. Vector<T>

## 1.1 创建

```rust
let v: Vec<i32> = Vec::new();          // 空 Vec，需标注类型
let v = vec![1, 2, 3];                 // vec! 宏，自动推断类型
let v = vec![0; 5];                    // [0, 0, 0, 0, 0]
```

★ **关键认知**：`Vec<T>` 是**可增长的动态数组**，栈上存"指针 + 长度 + 容量"，堆上存元素。`Vec::new()` 空 Vec 无法推断类型时必须标注，用 `vec!` 宏则能从元素推断。

## 1.2 访问元素

```rust
let v = vec![1, 2, 3];

// 方式1：下标（越界 panic）
let third = v[2];

// 方式2：get（越界返回 Option，安全）
match v.get(2) {
    Some(third) => println!("{}", third),
    None => println!("no element"),
}

// 遍历
for i in &v {           // 不可变借用
    println!("{}", i);
}
for i in &mut v {       // 可变借用，可修改
    *i += 1;
}
```

★ **关键认知**：`v[i]` 越界会 **panic**（运行时错误）；`v.get(i)` 返回 `Option<&T>`（安全）。首选 `get`，除非你确定下标安全。这与 Go 的切片越界 panic 类似，但 Rust 提供了 `get` 这个安全替代。

## 1.3 增删改

```rust
let mut v = Vec::new();
v.push(1);              // 追加
v.push(2);
v.pop();                // 弹出末尾，返回 Option<T>
v.insert(0, 0);         // 在索引处插入
v.remove(0);            // 移除索引处元素
v.len(); v.is_empty();  // 长度/是否为空
v.clear();              // 清空
```

## 1.4 底层特点

★ **关键认知**：`Vec<T>` 的扩容策略是**翻倍增长**（容量不足时分配 2 倍空间并拷贝）。若已知元素数量，用 `Vec::with_capacity(n)` 预分配，避免多次扩容。⚠ 对 Vec 的任何"可能扩容"的操作（push）后，之前获取的元素引用会失效（借用检查器会阻止你同时持有引用和修改）。

# 2. String 和 &str

## 2.1 创建与转换

```rust
let s = String::new();                      // 空 String
let s = String::from("hello");              // 从 &str 创建
let s = "hello".to_string();                // 等价写法
```

★ **关键认知**：`&str` 是**字符串切片**（对 UTF-8 字节序列的不可变引用，可能是字面量或 String 的一部分）；`String` 是**堆上可增长**的字符串。`&str` 和 `&String` 可通过 deref coercion 互转。

## 2.2 修改

```rust
let mut s = String::from("hello");
s.push_str(" world");       // 追加 &str
s.push('!');                // 追加单个 char
s += "!!";                  // 用 + 拼接（会移动左操作数）
let s3 = format!("{}-{}", s1, s2);   // format! 拼接，不移动
```

★ **关键认知**：`+` 运算符会**移动**左操作数（`s1 + &s2` 会消耗 s1），因为其底层签名是 `fn add(self, s: &str) -> String`。多字符串拼接用 `format!` 宏（不移动任何操作数，返回新 String）。

## 2.3 访问与切片

```rust
let s = String::from("hello");
s.len();                    // 字节长度
s.chars().nth(0);           // 取第 0 个字符（返回 Option<char>）
let slice = &s[0..2];       // 字符串切片（按字节）
```

★ **关键认知**：Rust **不允许用下标访问字符串的"字符"**（`s[0]` 编译错误），因为 UTF-8 变长编码下"第 n 个字符"与"第 n 个字节"不同。要按字符访问用 `s.chars()`，要按字节用 `s.bytes()`。切片 `&s[a..b]` 按字节，⚠ 切在字符中间会 panic。

## 2.4 遍历

```rust
for c in s.chars() { }      // 遍历 Unicode 字符
for b in s.bytes() { }      // 遍历字节
```

# 3. HashMap<K, V>

## 3.1 创建与插入

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

★ **关键认知**：`HashMap<K, V>` 默认使用 `SipHash` 哈希算法（安全性高、抗 DoS 攻击，但比普通哈希慢）。它要求键 `K` 实现 `Eq + Hash` trait。⚠ 与 Go map 不同，Rust 的 HashMap **默认不做随机化遍历**（遍历顺序取决于哈希，但非刻意随机）。

## 3.2 从迭代器收集

```rust
let teams = vec![String::from("Blue"), String::from("Yellow")];
let initial = vec![10, 50];

let scores: HashMap<_, _> = teams.into_iter().zip(initial.into_iter()).collect();
```

## 3.3 访问

```rust
let team = String::from("Blue");
let score = scores.get(&team);      // 返回 Option<&V>
match score {
    Some(s) => println!("{}", s),
    None => println!("no score"),
}

for (key, value) in &scores {        // 遍历（顺序不定）
    println!("{}: {}", key, value);
}
```

## 3.4 entry 与 or_insert

★ **关键认知**：`entry` API 是 HashMap 的精华，用于"**不存在则插入**"的原子操作，避免多次查找：

```rust
scores.entry(String::from("Blue")).or_insert(50);       // 不存在才插入
scores.entry(String::from("Blue")).or_insert(0);        // 已存在，不修改

// 经典用法：词频统计
let text = "hello world hello";
let mut map = HashMap::new();
for word in text.split_whitespace() {
    let count = map.entry(word).or_insert(0);
    *count += 1;            // or_insert 返回 &mut V
}
```

★ **关键认知**：`entry(key)` 返回一个 `Entry` 枚举（Occupied/Vacant），`or_insert(default)` 在 Vacant 时插入并返回 `&mut V`。这一套 API 比 Go 的"`v, ok := m[k]; if !ok {...}`"更简洁，且只查找一次。

## 3.5 所有权

★ **关键认知**：对于实现了 `Copy` 的类型（如 i32），值会被复制进 HashMap；对于 `String` 等堆类型，**所有权会被移动**进 HashMap：

```rust
let field_name = String::from("Favorite color");
let field_value = String::from("Blue");
let mut map = HashMap::new();
map.insert(field_name, field_value);
// field_name 和 field_value 已被移动，不能再用
```

# 4. 其他集合

```
HashSet<T>      // 无序集合，去重（HashMap 的键集合）
BTreeMap<K,V>   // 有序 map（基于 B 树，键需 Ord，遍历有序）
BTreeSet<T>     // 有序集合
VecDeque<T>     // 双端队列
LinkedList<T>   // 双向链表（少用，性能通常不如 Vec）
```

★ **关键认知**：需要**有序遍历**时用 `BTreeMap`（基于 B 树，`O(log n)`，遍历有序）；需要 `O(1)` 查找时用 `HashMap`（遍历无序）。这是 Rust 与 Go 的一个差异——Go 只有 map，Rust 提供了有序/无序两种选择。

# 5. 小结

| 集合 | 要点 |
|------|------|
| Vec<T> | 动态数组，堆分配，可增长 |
| String / &str | 堆字符串 / 字符串切片 |
| HashMap | 哈希表，键需 Eq+Hash |
| entry().or_insert() | 不存在则插入的惯用法 |
| BTreeMap | 有序 map |
| get vs [] | get 返回 Option（安全） |

下一篇进入错误处理，讲清楚 Rust 的 `panic!` 与 `Result<T, E>`、`?` 传播运算符、`unwrap`/`expect` 的使用边界。
