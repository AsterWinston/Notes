请求处理里总有一些东西要跨请求、跨 worker 共享：配置、连接池、缓存、计数器。Actix Web 用 `web::Data` 把这件事做得很轻，但轻背后是 `Arc`、多 worker 和异步锁三件事叠在一起，稍不注意就会写出"每个 worker 各有一份状态"或"编译报 future 不 Send"。

# 1. 为什么需要共享状态

| 状态 | 生命周期 | 存放方式 |
|------|----------|----------|
| 请求级 | 一次请求 | Handler 局部变量 |
| 应用级 | 整个进程 | `web::Data` + `app_data` |
| 全局级 | 进程 + 跨测试 | `OnceLock`、`static`（谨慎） |

绝大多数需求属于第二类：配置、数据库连接池、缓存客户端。它们应该在启动时创建一次，再注入每个 worker。

```rust
HttpServer::new(|| {
    let state = web::Data::new(AppState::new()); // 错误：每个 worker 各建一份
    App::new().app_data(state)
})
```

★ **关键认知**：`HttpServer::new` 的闭包会被**每个 worker 各调用一次**，上面写法等于每个 worker 都有独立状态。计数器看起来能加，但只在一个 worker 内加；连接池会被创建多份，直接打爆数据库。`web::Data` 只保证同一份 `Data` 的所有 clone 共享底层数据，不保证闭包里新建的 `Data` 会跨 worker 共享。

# 2. web::Data 的本质

## 2.1 它就是 Arc

```text
web::Data<T>
┌────────────────────────────┐
│  Arc<T> 的指针 + 原子计数   │
└────────────────────────────┘
        │
        ▼
   堆上的 T（唯一一份）
```

```rust
let a = web::Data::new(String::from("shared"));
let b = a.clone();                        // 只是 Arc::clone，开销极小
assert!(std::ptr::eq(a.get(), b.get()));  // 指向同一份数据
```

★ **关键认知**：`web::Data<T>` 要求 `T: 'static`，因为它要跨线程、跨请求存活。`T` 里不能持有指向栈上数据的引用；需要共享 `String`、`Vec`、连接池这类拥有所有权的数据。

如果某个库已经用 `Arc` 管理资源，直接 `web::Data::from(arc)` 即可，不必再套一层 `Arc<Arc<T>>`。

# 3. 注册与提取

## 3.1 标准姿势

```rust
use actix_web::{web, App, HttpServer};
use std::sync::atomic::AtomicUsize;

struct AppState {
    app_name: String,
    counter: AtomicUsize,
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // 1. 在闭包外创建，只创建一次
    let state = web::Data::new(AppState {
        app_name: "actix-demo".into(),
        counter: AtomicUsize::new(0),
    });

    HttpServer::new(move || {
        // 2. 每个 worker 克隆 Data，仍指向同一份底层数据
        App::new()
            .app_data(state.clone())
            .route("/name", web::get().to(get_name))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}

async fn get_name(state: web::Data<AppState>) -> impl actix_web::Responder {
    state.app_name.clone()
}
```

⚠ 闭包必须 `move`，否则 `state` 借用自 `main` 的栈帧，而 `HttpServer` 要求 `'static`。`move` 之后闭包捕获 `Data`，每次调用再 `clone`，这是正确且惯用的。

## 3.2 提取类型要完全一致

```rust
// 注册 Data<AppState>，提取也必须是 Data<AppState>
.app_data(web::Data::new(AppState::new()))

async fn ok(state: web::Data<AppState>) { }
async fn bad(state: web::Data<std::sync::Arc<AppState>>) { } // 运行期 500
```

Actix Web 按类型查找 `app_data`，注册错、提取错都不会在编译期报，而是请求时返回 500 "app data is not configured"。

## 3.3 app_data 的作用域

```rust
App::new()
    .app_data(web::Data::new(GlobalConfig::default()))
    .service(
        web::scope("/admin")
            .app_data(web::Data::new(AdminConfig::default())) // 只对 /admin 可见
            .route("/stats", web::get().to(stats)),
    )
```

内层 `scope` 注册的同类型 `app_data` 会**遮蔽**外层。用得好可以做权限隔离，用得不好会出现"同一类型在不同 scope 里值不一样"的困惑。另外，Actix Web 3 的 `App::data` 在 4.x 已移除，统一用 `app_data(web::Data::new(...))`。

# 4. 可变共享状态

## 4.1 Atomic 适合简单计数

```rust
use std::sync::atomic::{AtomicUsize, Ordering};

async fn hit(state: web::Data<AppState>) -> impl actix_web::Responder {
    let n = state.counter.fetch_add(1, Ordering::Relaxed) + 1;
    format!("第 {} 次访问", n)
}
```

无锁、最快，适合计数器、开关、统计量，不支持复合操作。

## 4.2 std::sync::Mutex

```rust
use std::sync::Mutex;

struct State {
    users: Mutex<Vec<String>>,
}

async fn add(
    state: web::Data<State>,
    name: web::Path<String>,
) -> impl actix_web::Responder {
    let mut users = state.users.lock().unwrap(); // 同步锁，瞬间完成
    users.push(name.into_inner());
    format!("total = {}", users.len())
} // guard 在这里释放
```

★ **关键认知**：`std::sync::Mutex` 是**同步锁**，拿锁会阻塞当前线程，适合"锁内只做内存操作、绝不做 I/O"的场景。⚠ 绝不能让 `MutexGuard` 跨 `.await` 存活：

```rust
// 编译失败：future 不是 Send
async fn bad(state: web::Data<State>) {
    let guard = state.users.lock().unwrap();
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
    drop(guard);
}
```

报错通常是 "future cannot be sent between threads safely"，根因是 `MutexGuard` 不是 `Send`。正确做法是缩小锁范围：

```rust
async fn good(state: web::Data<State>) {
    {
        let mut users = state.users.lock().unwrap();
        users.push("x".into());
    } // 先释放
    tokio::time::sleep(std::time::Duration::from_secs(1)).await;
}
```

## 4.3 tokio::sync::Mutex

如果锁内必须 `.await`（先查缓存再写），用 `tokio::sync::Mutex`：`lock().await` 异步等待，不阻塞线程（需在依赖中显式加入 `tokio`）。即使能编译，也建议**不要在锁内做慢 I/O**，否则所有请求排队；更好的模式是"锁内只读写 HashMap，加载放锁外，再二次检查"。

## 4.4 Mutex 还是 RwLock

| 场景 | 选择 |
|------|------|
| 简单计数 | `AtomicUsize` |
| 写多、临界区短 | `Mutex` |
| 读多写少 | `RwLock` |
| 锁内需要 `.await` | `tokio::sync::Mutex` / `RwLock` |
| 只读配置 | 不要锁，`web::Data<Config>` 即可 |

```rust
use std::sync::RwLock;

struct State { config: RwLock<Config> }

async fn read_config(state: web::Data<State>) -> impl actix_web::Responder {
    let cfg = state.config.read().unwrap(); // 多读并发
    cfg.name.clone()
}
```

⚠ `RwLock` 并非永远更快。临界区极短、竞争激烈时，读写锁的额外记账可能比 `Mutex` 更慢；没有测量就不要想当然。

# 5. 连接池

## 5.1 池本身就是共享资源

以 SQLx 为例，`Pool` 内部已经是 `Arc`，克隆成本很低：

```rust
use sqlx::{sqlite::SqlitePoolOptions, SqlitePool};

#[derive(Clone)]
struct AppState { db: SqlitePool }

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let db = SqlitePoolOptions::new()
        .max_connections(10) // 明确上限，别让数据库被打爆
        .connect("sqlite://app.db")
        .await
        .expect("数据库连接失败");

    let state = web::Data::new(AppState { db });

    HttpServer::new(move || {
        App::new().app_data(state.clone()).route("/users", web::get().to(list_users))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}

async fn list_users(state: web::Data<AppState>) -> impl actix_web::Responder {
    let rows = sqlx::query("SELECT id FROM users")
        .fetch_all(&state.db)
        .await
        .unwrap_or_default();
    format!("{} rows", rows.len())
}
```

★ **关键认知**：不要在 Handler 里 `connect` 新连接，也不要在每个 worker 里新建池。池必须在 `main` 里建一次，通过 `Data` 共享；`max_connections` 要结合数据库上限和实例数设置，通常是"数据库允许连接数 / 实例数"再留余量。worker 多不等于连接多，所有 worker 共用同一个池。

# 6. 生命周期与清理

## 6.1 什么时候释放

```text
Data::new(T) 引用计数 = 1
  ├── worker 0 clone → 2
  ├── worker 1 clone → 3
  └── 所有 worker 与请求结束、计数归零 → T 被 drop
```

`T` 的 `Drop` 在最后一个 `Data` 释放时执行。连接池、后台任务句柄适合放进 `Data`，随进程退出统一清理。

## 6.2 状态结构体要 Send + Sync

```rust
struct AppState {
    config: Config,             // 需 Send + Sync
    cache: Mutex<HashMap<..>>,  // Mutex<T>: Sync 当 T: Send
}
```

编译报 "`Rc<...>` cannot be shared between threads safely" 时，检查状态里是否混进了 `Rc`、`RefCell`、裸指针。多线程共享要用 `Arc`、`Mutex`、`RwLock`。

# 7. 常见错误

| 现象 | 原因 | 解决 |
|------|------|------|
| 每个 worker 状态不同 | 在 `HttpServer::new` 闭包里 `Data::new` | 闭包外创建，闭包内 clone |
| 提取返回 500 | 注册类型与提取类型不一致 | 对齐 `Data<T>` |
| future 不 Send | `std::MutexGuard` 跨 `.await` | 缩小锁范围或换异步锁 |
| 连接数暴涨 | 每个 worker 新建池 | 池只建一次 |
| 锁竞争严重 | 锁内做慢 I/O | 缩小临界区 |
| 测试互相影响 | 用了全局 static | 改为 `Data` 注入 |

⚠ `Mutex` 中毒是 Rust 特有的坑：持锁线程 panic 后，锁被标记为 poisoned，后续 `lock()` 返回 `Err`。示例里的 `.unwrap()` 在 demo 没问题，生产代码要考虑恢复策略或换 `parking_lot::Mutex`。

# 8. 动手练习

1. 用 `AtomicUsize` 实现全进程访问计数器，验证并发请求下数值连续。
2. 用 `web::Data<RwLock<HashMap<String, String>>>` 实现内存 KV，支持 `GET /kv/{key}` 与 `PUT /kv/{key}`。
3. 把数据库连接池接入 `AppState`，要求 `main.rs` 只创建一次池，并确认 `max_connections` 生效。

验收标准：并发压测时计数器不丢数，KV 读写无 panic，数据库连接数不超过设定上限。

# 9. 小结

| 概念 | 要点 |
|------|------|
| `web::Data<T>` | `Arc<T>` 的封装，clone 廉价 |
| 创建位置 | 闭包外创建，闭包内 clone |
| 类型匹配 | 注册与提取的 `T` 必须完全一致 |
| 可变状态 | `Atomic` / `Mutex` / `RwLock` 按场景选 |
| 异步锁 | 锁内要 `.await` 时用 `tokio::sync` |
| 连接池 | 进程级建一次，通过 `Data` 共享 |
| 生命周期 | 最后一个 `Data` 释放时 drop |

下一篇进入中间件，讲清楚 Actix Web 的 `Transform` + `Service` 模型、为什么"最后 wrap 的最先执行"、日志与鉴权中间件怎么写，以及 CORS 和常见顺序坑。
