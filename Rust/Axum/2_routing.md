路由决定"这个请求交给谁"。Axum 的路由写法不复杂，但 0.8 换了路径语法、匹配交给 matchit 的 radix tree，和 Actix Web 的"注册顺序即匹配顺序"完全不同。把路径参数、嵌套、合并和兜底这四件事理清，路由就不会再出意外。

# 1. 路由的基本写法

## 1.1 方法路由

```rust
use axum::{
    routing::{get, post},
    Router,
};

let app = Router::new()
    .route("/", get(index))
    .route("/users", get(list_users).post(create_user));
```

`get(handler)` 返回一个 `MethodRouter`，可以继续链式挂 `.post()`、`.put()`、`.delete()`。同一个路径的多个方法写成一条链，阅读时一眼能看出"这个路径支持哪些方法"。

## 1.2 分开注册与重叠规则

```rust
Router::new()
    .route("/users", get(list_users))
    .route("/users", post(create_user)); // 不同方法，可以分开写
```

同一路径、不同方法可以分成两次 `route`；但同一路径、同一方法注册两次，会在**构建 Router 时 panic**，而不是等到请求进来才报错。

```rust
Router::new()
    .route("/users", get(list_users))
    .route("/users", get(get_user)); // 运行期 panic：同路径同方法
```

★ **关键认知**：Axum 把路由冲突放在启动期暴露，这是好事——错误在 `cargo run` 时就炸，而不是线上下一个请求才 500。⚠ 但 panic 信息只告诉你"哪条路径冲突"，不告诉你"哪次调用重复"，路径多时要用二分法定位。

## 1.3 any 与显式方法

```rust
use axum::routing::any;

Router::new().route("/echo", any(echo_handler)); // 所有方法都进同一个 Handler
```

`any` 适合代理、调试端点、Webhook 入口。业务接口不建议用它，因为方法语义会丢失，405 也无从谈起。

# 2. 路径参数

## 2.1 0.8 的 {id} 语法

```rust
use axum::{
    extract::Path,
    routing::get,
};

async fn get_user(Path(id): Path<i64>) -> String {
    format!("user id = {id}")
}

let app = Router::new().route("/users/{id}", get(get_user));
```

请求 `/users/42` 得到 `42i64`；请求 `/users/abc` 返回 **400**，因为 `Path<i64>` 在进入 Handler 前就解析失败。⚠ 0.8 不再支持 `:id`。写成 `:id` 时它会被当成普通字面量路径，请求永远匹配不上，表现为"路由明明注册了却 404"。

★ **关键认知**：路径参数的类型转换由提取器完成，失败自动返回 400，Handler 根本不会执行。这与"拿到字符串自己 parse"是两种心智模型：Axum 把校验前移到了提取器层。

## 2.2 多个参数

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct OrderPath {
    uid: i64,
    oid: i64,
}

async fn get_order(Path(params): Path<OrderPath>) -> String {
    format!("user {} order {}", params.uid, params.oid)
}

let app = Router::new().route("/users/{uid}/orders/{oid}", get(get_order));
```

也可以用元组 `Path<(i64, i64)>`，按声明位置解构。⚠ 元组按位置、结构体按字段名；参数名改了却忘了改结构体字段，运行期会得到 400，而不是编译错误。

## 2.3 通配符

```rust
async fn serve_file(Path(path): Path<String>) -> String {
    format!("file = {path}")
}

let app = Router::new().route("/files/{*path}", get(serve_file));
```

请求 `/files/a/b.txt` 时，`path` 是 `a/b.txt`，不带前导斜杠。通配符通常用于静态文件、代理转发，业务路由尽量用明确段数，否则很容易把不该匹配的请求吞掉。

# 3. 嵌套与合并

## 3.1 nest

```rust
let api = Router::new()
    .route("/users", get(list_users))
    .route("/users/{id}", get(get_user));

let app = Router::new().nest("/api/v1", api);
```

最终路径是 `/api/v1/users`、`/api/v1/users/{id}`。`nest` 会保留前缀，适合版本化 API 和按业务域拆分。

```text
app
 └── nest("/api/v1")
      └── api
           ├── /users
           └── /users/{id}
```

★ **关键认知**：`nest` 不只是字符串拼接，它还划定了一个子 Router 的边界：子 Router 的 fallback、中间件、状态都可以独立配置。⚠ 嵌套层数不是越多越好，每多一层，排查完整路径就多一次心算。

## 3.2 merge

```rust
let users = Router::new().route("/users", get(list_users));
let orders = Router::new().route("/orders", get(list_orders));

let app = users.merge(orders);
```

`merge` 把两个 Router 的路由表平铺到一起，不添加前缀。它适合"每个业务模块自己定义完整路径"的组织方式。

## 3.3 nest 与 merge 的选择

| 方式 | 是否加前缀 | 适合场景 | 冲突行为 |
|------|-----------|----------|----------|
| `nest` | 是 | 版本前缀、业务域隔离 | 子 Router 内部冲突照样 panic |
| `merge` | 否 | 模块各自定义完整路径 | 同路径同方法 panic |

★ **关键认知**：`merge` 之后两个 Router 变成同一个路由表，**状态类型必须一致**；`nest` 也是同样要求。⚠ 常见错误是一个子 Router 返回 `Router<AppState>`、另一个已经 `with_state` 成 `Router<()>`，合并时类型对不上，编译错误会指向 `merge` 的泛型参数。

# 4. fallback 与 405

## 4.1 fallback

```rust
use axum::{
    http::StatusCode,
    response::IntoResponse,
    routing::get,
    Json, Router,
};
use serde_json::json;

async fn not_found() -> impl IntoResponse {
    (
        StatusCode::NOT_FOUND,
        Json(json!({ "code": "NOT_FOUND", "message": "接口不存在" })),
    )
}

let app = Router::new()
    .route("/", get(index))
    .fallback(not_found);
```

`fallback` 兜住所有"路径没匹配上"的请求。统一错误结构、返回 JSON 而不是空 404，都靠它。

## 4.2 method_not_allowed_fallback

```rust
async fn method_not_allowed() -> impl IntoResponse {
    (StatusCode::METHOD_NOT_ALLOWED, "方法不支持")
}

let app = Router::new()
    .route("/users", get(list_users))
    .method_not_allowed_fallback(method_not_allowed);
```

路径存在但方法不匹配时，Axum 默认返回 405；想自定义响应体就用这个 API。⚠ `fallback` 管 404，`method_not_allowed_fallback` 管 405，两者不要混用。

## 4.3 fallback_service

```rust
use tower_http::services::ServeDir;

let app = Router::new()
    .nest("/api", api)
    .fallback_service(ServeDir::new("static"));
```

`fallback_service` 接收一个 Tower Service，适合把未匹配请求交给静态文件服务或反向代理。注意顺序：API 路由先匹配，剩下的才轮到静态文件。

# 5. 匹配顺序

## 5.1 静态优先

Axum 底层用 matchit 的 radix tree，匹配优先级是**静态段 > 路径参数 > 通配符**，与注册顺序无关。

```rust
let app = Router::new()
    .route("/users/{id}", get(get_user)) // 参数路由
    .route("/users/me", get(get_me));    // 静态路由
```

请求 `/users/me` 会进入 `get_me`，不会像 Actix Web 那样被前面的 `{id}` 截走。请求 `/users/42` 才进入 `get_user`。

★ **关键认知**：Axum 的匹配由路由树结构决定，不是按注册顺序逐条尝试。所以"把静态路由写在参数路由前面"在 Axum 里不是必需的；真正要避免的是同一位置出现两个形状相同的参数路由。

## 5.2 冲突在启动时 panic

```rust
Router::new()
    .route("/users/{id}", get(a))
    .route("/users/{uid}", get(b)); // panic：同一位置两个参数名
```

两条路由的形状完全一样，matchit 无法决定谁优先，构建 Router 时直接 panic。⚠ 这类冲突不会在编译期发现，必须跑一次启动或加一个构造 Router 的测试。

## 5.3 尾斜杠

`/users` 与 `/users/` 是两条不同路径，Axum 默认不做重定向。团队要约定统一风格，别让一部分接口带斜杠、一部分不带。需要兼容时可以用 `tower_http::normalize_path::NormalizePathLayer`，但它会改写 URI，影响日志与缓存键，生产更推荐从路由定义上统一。

# 6. 常见错误

| 现象 | 常见原因 | 排查方式 |
|------|----------|----------|
| 路由 404 | 用了 `:id` 旧语法 | 改成 `{id}` |
| 路径参数 400 | 类型不匹配或字段名写错 | 对照 `Path<T>` 与路由 |
| 启动直接 panic | 同路径同方法或同形状参数路由 | 检查重复注册 |
| `/users/me` 进错 Handler | 用了 Actix Web 的旧心智 | Axum 静态优先，确认路径形状 |
| 405 | 路径对但方法没注册 | 检查 `get/post` 链 |
| 嵌套后 404 | `nest` 前缀与内部路径重复 | 打印完整路径核对 |

⚠ 调试路由时，先列出"方法 + 完整路径 + Handler"三列，比盯着代码猜快得多。线上 404 优先看反向代理是否改写了路径。

# 7. 动手练习

1. 实现 `/api/v1/users` 的列表与创建，列表支持 `page`、`size`（缺省 1 和 20）。
2. 实现 `/api/v1/users/{id}` 的详情与删除，`id` 用 `i64` 提取，非数字返回 400。
3. 额外实现 `/api/v1/users/me`，验证静态路由是否优先于 `{id}`，再故意写两条同形状参数路由观察启动 panic。

验收标准：`/users/me` 返回当前用户，`/users/abc` 返回 400，重复路由在启动时立即暴露。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 方法路由 | `get(handler).post(handler)` 链式注册 |
| 路径参数 | `{id}`，类型解析失败返回 400 |
| 通配符 | `{*path}`，捕获值不带前导斜杠 |
| `nest` | 加前缀，划分子 Router 边界 |
| `merge` | 平铺路由表，不加前缀 |
| `fallback` | 兜住 404 |
| 405 | 路径存在但方法不支持 |
| 匹配顺序 | 静态 > 参数 > 通配符，与注册顺序无关 |

下一篇进入提取器，讲清楚 `Path`、`Query`、`Json`、`Form`、`State` 的解析规则，为什么请求体提取器必须放在参数列表最后，以及如何实现自己的 `FromRequestParts`。
