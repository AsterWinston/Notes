请求处理里总有一些东西要跨请求、跨连接共享：配置、连接池、缓存、计数器。Axum 用 `State` 把这件事做得类型安全，但背后仍然是 `Arc`、`Send + Sync` 和异步锁几件事叠在一起。稍不注意就会写出"状态每请求一份"或"future 不 Send"的代码。

# 1. 为什么需要共享状态

## 1.1 请求级与应用级

| 状态 | 生命周期 | 存放方式 |
|------|----------|----------|
| 请求级 | 一次请求 | Handler 局部变量 |
| 应用级 | 整个进程 | `State` + `with_state` |
| 全局级 | 进程 + 跨测试 | `OnceLock`、`static`（谨慎） |

绝大多数需求属于第二类：配置、数据库连接池、缓存客户端。它们应该在启动时创建一次，再注入 Router。

## 1.2 State 的本质

`State<T>` 不复制数据本身，它复制的是一个**廉价的句柄**。通常 `T` 里放的是 `Arc`、连接池或字符串：

```rust
use std::sync::{
    atomic::{AtomicUsize, Ordering},
    Arc,
};

#[derive(Clone)]
struct AppState {
    app_name: String,              // 小数据，克隆成本低
    counter: Arc<AtomicUsize>,     // 计数器，跨请求共享
}
```

★ **关键认知**：`State<T>` 要求 `T: Clone + Send + Sync + 'static`。Clone 是为了每个请求能拿一份句柄；Send + Sync 是因为请求可能跑在不同线程；'static 是因为状态活得比任何一次请求都久。⚠ 把 `Vec<User>` 直接放进 `AppState` 也能编译，但每次提取都会克隆整个 Vec，请求一多就是灾难。共享大对象要放进 `Arc`。

# 2. 注册与提取

## 2.1 with_state

```rust
use axum::{extract::State, routing::get, Router};

let state = AppState {
    app_name: "axum-demo".into(),
    counter: Arc::new(AtomicUsize::new(0)),
};

let app = Router::new()
    .route("/name", get(name))
    .route("/hit", get(hit))
    .with_state(state);

async fn name(State(state): State<AppState>) -> String {
    state.app_name.clone()
}

async fn hit(State(state): State<AppState>) -> String {
    let n = state.counter.fetch_add(1, Ordering::Relaxed) + 1;
    format!("第 {n} 次访问")
}
```

`with_state` 把 `Router<AppState>` 变成 `Router<()>`。⚠ 状态必须在组装 Router 时一次性注入；`Router<AppState>` 直接交给 `axum::serve` 会编译失败，这是故意的。

## 2.2 FromRef 子状态

当 Handler 只关心状态的一部分时，用 `FromRef` 做投影，避免每个 Handler 都拿整个 `AppState`：

```rust
use axum::extract::FromRef;

// 前置：AppState 中已有 db: sqlx::SqlitePool 字段
#[derive(Clone)]
struct DbPool(sqlx::SqlitePool);

impl FromRef<AppState> for DbPool {
    fn from_ref(state: &AppState) -> Self {
        Self(state.db.clone())
    }
}

async fn list(State(db): State<DbPool>) -> String {
    format!("pool size = {}", db.0.size())
}
```

★ **关键认知**：`FromRef` 让"状态结构"和"Handler 依赖"解耦。Handler 只声明自己需要的 `DbPool`，测试时可以只构造这一部分，不必拼出完整 `AppState`。⚠ 子状态的 `from_ref` 里只做克隆，不要做连接、读文件这类重操作，它会在每次提取时执行。

## 2.3 State 与 Extension

| 维度 | `State<T>` | `Extension<T>` |
|------|-----------|----------------|
| 类型检查 | 编译期 | 运行期 |
| 注册方式 | `with_state` | `.layer(Extension(value))` |
| 缺失时 | Router 无法构建 | 请求返回 500 |
| 典型用途 | 应用状态、连接池 | 中间件向 Handler 传值 |

鉴权中间件解析出的用户信息通常用 `Extension` 传递，因为它是"每次请求动态产生"的，不适合放进全局 `State`。

# 3. 可变共享状态

## 3.1 Atomic

```rust
use std::sync::atomic::Ordering;

async fn hit(State(state): State<AppState>) -> String {
    let n = state.counter.fetch_add(1, Ordering::Relaxed) + 1;
    format!("{n}")
}
```

无锁、最快，适合计数器、开关、统计量。⚠ 它不支持复合操作，"先读再判断再写"必须换锁，否则会有竞态。

## 3.2 Mutex 与 RwLock

```rust
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct AppState {
    users: Arc<Mutex<Vec<String>>>,
}

async fn add(State(state): State<AppState>, Path(name): Path<String>) -> String {
    let mut users = state.users.lock().unwrap();
    users.push(name);
    format!("total = {}", users.len())
} // guard 在这里释放
```

★ **关键认知**：`std::sync::Mutex` 是**同步锁**，拿锁会阻塞当前线程，适合"锁内只做内存操作、绝不做 I/O"的场景。⚠ 绝不能让 `MutexGuard` 跨 `.await` 存活：

```rust
async fn bad(state: State<AppState>) {
    let guard = state.users.lock().unwrap();
    tokio::time::sleep(std::time::Duration::from_secs(1)).await; // ✗ guard 不是 Send
    drop(guard);
}
```

报错是 "future cannot be sent between threads safely"，根因是 `MutexGuard` 不是 `Send`。正确做法是把锁范围缩到最小：

```rust
async fn good(state: State<AppState>) {
    {
        let mut users = state.users.lock().unwrap();
        users.push("x".into());
    } // 先释放
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
}
```

## 3.3 异步锁

如果锁内必须 `.await`（先查缓存再回填），用 `tokio::sync::Mutex`：

```rust
use tokio::sync::Mutex;

#[derive(Clone)]
struct AppState {
    cache: Arc<Mutex<Vec<String>>>,
}

async fn fill(State(state): State<AppState>) {
    let mut cache = state.cache.lock().await; // 异步等待，不阻塞线程
    cache.push("loaded".into());
}
```

| 场景 | 选择 |
|------|------|
| 简单计数 | `AtomicUsize` |
| 写多、临界区短 | `std::sync::Mutex` |
| 读多写少 | `std::sync::RwLock` |
| 锁内必须 `.await` | `tokio::sync::Mutex` / `RwLock` |
| 只读配置 | 不要锁，直接放 `State` |

⚠ 能编译不等于合理。在 `tokio::sync::Mutex` 里做慢查询，会让所有等锁的请求排队；更好的模式是"锁内只读写 HashMap，加载放锁外，再二次检查"。

# 4. 连接池与外部资源

## 4.1 池就是共享资源

```rust
#[derive(Clone)]
struct AppState {
    db: sqlx::SqlitePool,
}

let pool = sqlx::sqlite::SqlitePoolOptions::new()
    .max_connections(10) // 明确上限，别让数据库被打爆
    .connect("sqlite://app.db?mode=rwc")
    .await?;

let state = AppState { db: pool };
```

★ **关键认知**：连接池内部已经是 `Arc`，`clone()` 只增加引用计数，不会新建连接。所以把池直接放进 `AppState` 即可，不需要再套 `Arc<SqlitePool>`。⚠ `max_connections` 要结合数据库上限和实例数设置，通常是"数据库允许连接数 / 实例数"再留余量；worker 多不等于连接多，所有 worker 共用同一个池。

## 4.2 生命周期

```text
State::new(T) 引用计数 = 1
  ├── 每个请求提取 → 计数 +1
  ├── 请求结束 → 计数 -1
  └── Router 与所有请求结束、计数归零 → T 被 drop
```

`T` 的 `Drop` 在最后一个句柄释放时执行。连接池、后台任务句柄适合放进 `State`，随进程退出统一清理。

# 5. Send + Sync 边界

## 5.1 为什么要求

Axum 默认跑在多线程 Tokio 上，一个请求的 future 可能在不同 worker 线程之间迁移，所以 Handler 的 future 必须 `Send`；`State` 会被多个请求同时持有，所以必须 `Sync`。

```rust
#[derive(Clone)]
struct AppState {
    config: Config,                    // 配置本身要满足 Send + Sync
    users: Arc<Mutex<Vec<String>>>,    // 可跨线程共享（T: Send）
}
```

## 5.2 常见不满足类型

| 类型 | 问题 | 替代 |
|------|------|------|
| `Rc<T>` | 非原子引用计数，不 Send | `Arc<T>` |
| `RefCell<T>` | 非线程安全内部可变 | `Mutex<T>` / `RwLock<T>` |
| `*const T` / `*mut T` | 裸指针不 Send | 用安全抽象包起来 |
| `std::sync::MutexGuard` | 不 Send，不能跨 await | 缩小锁范围或换异步锁 |

⚠ 看到 "`Rc<...>` cannot be shared between threads safely" 时，顺着报错找 `AppState` 里混进去的非线程安全类型，通常是从单线程代码里直接搬过来的。

# 6. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 每个请求状态都重置 | 在 Handler 里新建状态 | 启动时建一次，`with_state` 注入 |
| 提取报 500 | 用了 `Extension` 但没写入 | 改用 `State` 或补中间件 |
| future 不 Send | `std::MutexGuard` 跨 `.await` | 缩小锁范围或换 `tokio::sync::Mutex` |
| 连接数暴涨 | 每个 Handler 自己建池 | 池只建一次，放 `State` |
| 锁竞争严重 | 锁内做慢 I/O | 缩小临界区，加载放锁外 |
| 测试互相影响 | 用了全局 `static` | 改为 `State` 注入，每个测试新建 |

⚠ `std::sync::Mutex` 中毒是 Rust 特有的坑：持锁线程 panic 后，锁被标记为 poisoned，后续 `lock()` 返回 `Err`。示例里的 `.unwrap()` 在 demo 没问题，生产代码要考虑恢复策略或换 `parking_lot::Mutex`。

# 7. 动手练习

1. 用 `Arc<AtomicUsize>` 实现全进程访问计数器，并发请求下验证数值连续。
2. 用 `Arc<RwLock<HashMap<String, String>>>` 实现内存 KV，支持 `GET /kv/{key}` 与 `PUT /kv/{key}`。
3. 把数据库连接池接入 `AppState`，要求只在启动时创建一次，并用 `FromRef` 给 Handler 暴露 `DbPool` 子状态。

验收标准：并发压测时计数器不丢数，KV 读写无 panic，数据库连接数不超过设定上限。

# 8. 小结

| 概念 | 要点 |
|------|------|
| `State<T>` | 编译期注入，要求 `Clone + Send + Sync + 'static` |
| `with_state` | 把 `Router<AppState>` 变成 `Router<()>` |
| `FromRef` | 把完整状态投影成 Handler 需要的小状态 |
| `Extension` | 运行期按类型查找，适合中间件传值 |
| `Atomic` | 简单计数首选，不支持复合操作 |
| `Mutex` / `RwLock` | 锁内只做内存操作，guard 不跨 `.await` |
| 连接池 | 进程级建一次，clone 廉价 |
| Send + Sync | 请求跨线程、状态跨请求的硬性要求 |

下一篇进入中间件与 Tower，讲清楚 `Service` / `Layer` 到底是什么、`layer` 与 `route_layer` 的差别、日志与 CORS 怎么写，以及"后加的中间件先执行"这个最容易写反的顺序问题。
