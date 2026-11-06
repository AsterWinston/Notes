路由决定"这个请求交给谁"。Actix Web 的路由写法不难，但它有一个和 Gin、Axum 都不太一样的脾气：**匹配顺序基本等于注册顺序**，而不是自动让静态路径优先。不理解这一点，很容易写出"明明注册了却总是进错 Handler"的代码。

# 1. 路由的两种写法

## 1.1 宏路由

```rust
use actix_web::{get, post, HttpResponse, Responder};

#[get("/users")]
async fn list_users() -> impl Responder {
    HttpResponse::Ok().body("user list")
}

#[post("/users")]
async fn create_user() -> impl Responder {
    HttpResponse::Created().finish()
}
```

用 `.service(list_users)` 注册。支持 `#[get]`、`#[post]`、`#[put]`、`#[patch]`、`#[delete]` 等方法宏，多方法可用 `#[route("/path", method = "GET")]`。

## 1.2 Builder 路由

```rust
use actix_web::{web, App};

App::new()
    .route("/users", web::get().to(list_users))
    .route("/users", web::post().to(create_user));
```

同一路径挂多个方法时，用 `web::resource` 更清晰：

```rust
App::new().service(
    web::resource("/users")
        .route(web::get().to(list_users))
        .route(web::post().to(create_user)),
);
```

★ **关键认知**：`route` 是"往当前路径追加一个方法"，`resource` 是"先声明路径，再往上挂方法"。宏路由的路径必须写在属性里，不能运行时拼接；需要动态前缀时只能用 `scope` 或 builder。

# 2. 路径参数

## 2.1 基本语法

Actix Web 4.x 用花括号声明路径参数：

```rust
use actix_web::{get, web, Responder};

#[get("/users/{id}")]
async fn get_user(id: web::Path<i64>) -> impl Responder {
    format!("user id = {}", id.into_inner())
}
```

请求 `/users/42` 得到 `42i64`；请求 `/users/abc` 直接返回 **400**，因为提取器解析失败，Handler 不会执行。

★ **关键认知**：路径参数的类型转换由提取器完成。`web::Path<i64>` 在进入 Handler 前就完成解析，失败时框架自动返回 400。这与 Gin 里"拿到字符串自己 parse"是两种心智模型。

## 2.2 多个参数

```rust
#[get("/users/{uid}/orders/{oid}")]
async fn get_order(path: web::Path<(i64, i64)>) -> impl Responder {
    let (uid, oid) = path.into_inner(); // 按声明位置解构
    format!("user {} order {}", uid, oid)
}
```

用元组时按位置对应，参数名只影响路由匹配。用具名结构体则按字段名对应：

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct OrderPath {
    uid: i64,
    oid: i64,
}
```

## 2.3 正则约束

```rust
#[get("/users/{id:[0-9]+}")]
async fn get_user(id: web::Path<u64>) -> impl Responder {
    format!("numeric id = {}", id.into_inner())
}
```

正则写在参数名后的 `:` 里，适合"只允许数字 ID"这类约束。⚠ 正则越复杂匹配越慢，多数时候用类型约束就够，不必把业务校验塞进路由。

# 3. 查询参数

## 3.1 web::Query

```rust
use serde::Deserialize;

#[derive(Deserialize)]
struct PageQuery {
    page: Option<u32>,          // 可选参数必须用 Option
    size: Option<u32>,
    keyword: Option<String>,
}

#[get("/search")]
async fn search(q: web::Query<PageQuery>) -> impl Responder {
    let page = q.page.unwrap_or(1);
    let size = q.size.unwrap_or(20).min(100); // 顺手做上限保护
    format!("page={} size={} keyword={:?}", page, size, q.keyword)
}
```

请求 `/search?page=2&size=10` 会被解析进结构体。

★ **关键认知**：`web::Query<T>` 依赖 `serde_urlencoded`，字段类型决定解析规则。**非 `Option` 字段缺失时直接返回 400**，所以可选参数必须显式写成 `Option<T>`，不能靠默认值兜底。

## 3.2 路径参数与查询参数的分工

路径参数用于**标识资源**（`/users/42`），查询参数用于**过滤、分页、排序**（`/users?page=2`）。前者通常参与缓存键，后者视业务而定。

# 4. 作用域与资源

## 4.1 web::scope

`scope` 给一组路由加统一前缀，同时划定中间件和 `app_data` 的作用范围：

```rust
App::new().service(
    web::scope("/api/v1")
        .service(web::resource("/users").route(web::get().to(list_users)))
        .service(web::resource("/orders").route(web::get().to(list_orders))),
);
```

实际路径是 `/api/v1/users`、`/api/v1/orders`。

```rust
web::scope("/admin")
    .wrap(AuthMiddleware) // 只作用于 /admin 下的路由
    .service(web::resource("/stats").route(web::get().to(stats)))
```

## 4.2 web::resource

`resource` 表示"一个路径 + 一组方法"：

```rust
web::resource("/users/{id}")
    .name("user-detail")            // 给路由起名，便于反向生成 URL
    .route(web::get().to(get_user))
    .route(web::delete().to(delete_user))
```

## 4.3 三者关系

```text
App
 └── scope("/api")                统一前缀 + 可挂中间件
      └── resource("/users/{id}") 一个具体路径
           ├── route(GET)        方法 + Handler
           └── route(DELETE)     方法 + Handler
```

| 组合 | 结果路径 |
|------|----------|
| `scope("/api")` + `route("/users")` | `/api/users` |
| `resource("/users")` + `route("/{id}")` | `/users/{id}` |

★ **关键认知**：`scope` 是"前缀 + 中间件作用域"，`resource` 是"具体路径 + 方法集合"，`route` 是"单个方法绑定"。不要把 `scope` 只当字符串拼接工具，它同时影响中间件和数据的可见范围。

# 5. 路由匹配顺序

## 5.1 注册顺序就是匹配顺序

这是 Actix Web 最容易踩的点：

```rust
App::new()
    .route("/users/{id}", web::get().to(get_user))   // 先注册
    .route("/users/me", web::get().to(get_me))       // 后注册
```

请求 `/users/me` 会进入 **`get_user`**，并把 `"me"` 当成 `{id}` 解析。因为 `web::Path<i64>` 失败，最终返回 400，而不是你期待的 `get_me`。调换顺序即可：

```rust
App::new()
    .route("/users/me", web::get().to(get_me))       // 静态路径放前面
    .route("/users/{id}", web::get().to(get_user))   // 参数路径放后面
```

★ **关键认知**：Actix Web 按**注册顺序**逐条尝试匹配，先命中先返回。所以"静态路径优先"不是框架自动保证的，而是靠你把静态路由写在参数路由前面。⚠ 这和 Gin 的 radix tree 自动优先静态路径不同，从 Gin 迁过来的人特别容易中招。

## 5.2 匹配流程

```text
请求 GET /users/me
      │
      ▼
┌──────────────────────────┐
│ 取第 1 条 /users/{id}    │
│ 路径匹配？ 是            │
│ 方法匹配？ 是            │
└───────────┬──────────────┘
            ▼
      进入 get_user
      Path<i64> 解析 "me" 失败
            │
            ▼
        返回 400
   （不会回头尝试第 2 条）
```

⚠ 一旦路径和方法都匹配，框架就不会继续找"更合适"的路由。参数解析失败属于 Handler 阶段，不会触发重新匹配。

## 5.3 用 guard 精确控制

```rust
use actix_web::{guard, web};

web::resource("/ping")
    .route(web::route().guard(guard::Get()).to(ping_get))
    .route(web::route().guard(guard::Post()).to(ping_post))
```

| guard | 含义 |
|-------|------|
| `guard::Get()` | 方法为 GET |
| `guard::Header("X-Key", "secret")` | 请求头精确匹配 |
| `guard::fn_guard(|req| ...)` | 自定义判断 |

★ **关键认知**：`guard` 是让重叠路由按条件分流的正规手段。比起在 Handler 里写 `if method == ...`，用 guard 能让路由表本身表达清楚匹配规则，也便于 405 的语义正确。

## 5.4 405 与尾斜杠

路径匹配了 `resource` 但没有任何 `route` 的方法匹配时，返回 **405 Method Not Allowed**；完全没有路径匹配才是 **404**。

默认 `/users` 与 `/users/` 是两条不同路径。要统一行为可加 `NormalizePath`：

```rust
use actix_web::middleware::NormalizePath;

App::new().wrap(NormalizePath::trim()) // 去掉尾部斜杠后再匹配
```

⚠ 它会改写 URI，影响日志与缓存键。生产更推荐"约定统一风格"，而不是依赖中间件兜底。

## 5.5 自定义 404

```rust
use actix_web::{web, App, HttpResponse};

async fn not_found() -> HttpResponse {
    HttpResponse::NotFound().json(serde_json::json!({
        "code": "NOT_FOUND",
        "message": "接口不存在"
    }))
}

App::new().service(/* ... */).default_service(web::to(not_found))
```

`default_service` 兜住所有未匹配请求，适合统一错误结构。

# 6. 常见错误与排错

| 现象 | 常见原因 | 排查方式 |
|------|----------|----------|
| `/users/me` 返回 400 | 参数路由注册在静态路由之前 | 调整注册顺序 |
| 路径参数拿不到 | 花括号名字与结构体字段不一致 | 对照路由与字段 |
| 405 | 路径对了但方法没注册 | 检查 `web::get/post` |
| 路由被前面的通配吃掉 | 前缀路由注册过早 | 精确路由前置 |

# 7. 动手练习

1. 实现 `/api/v1/users` 的列表与创建接口，列表支持 `page`、`size`（缺省 1 和 20）。
2. 实现 `/api/v1/users/{id}` 的详情与删除，`id` 用 `i64` 提取，非数字返回 400。
3. 额外实现 `/api/v1/users/me`，验证你是否把静态路由放在了参数路由之前。

验收标准：`/users/me` 返回当前用户而不是 400，`/users/abc` 返回 400，`page` 超过 100 时被截断。

# 8. 小结

| 概念 | 要点 |
|------|------|
| 宏路由 | `#[get]` 等，注册用 `.service()` |
| Builder 路由 | `route` 挂单方法，`resource` 挂多方法 |
| 路径参数 | `{id}`，类型由提取器解析，失败 400 |
| 查询参数 | `web::Query<T>`，可选字段必须 `Option` |
| `scope` | 前缀 + 中间件 + 数据作用域 |
| 匹配顺序 | 按注册顺序，静态路由写在参数路由前 |
| guard | 条件分流，避免重叠路由互相遮挡 |
| 404/405 | 无路径 vs 路径存在但方法不支持 |

下一篇进入 Handler 与提取器，讲清楚 `web::Json`、`web::Form`、`web::Path`、`web::Query` 的解析规则，为什么请求体只能读一次，以及如何实现自己的 `FromRequest`。
