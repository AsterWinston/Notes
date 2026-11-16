GORM 是 Go 生态里使用最广的 ORM 之一。本篇先讲清它到底替你做了什么、和标准库 `database/sql` 是什么关系，再从零跑通连接、建表和第一个 CRUD，最后集中处理初学者最常卡住的连接报错。

# 1. GORM 是什么

## 1.1 定位：对 database/sql 的增强

Go 标准库 `database/sql` 只提供连接池和 `Query`/`Exec` 这类最底层能力，它不负责拼 SQL、不负责把行映射成结构体。你写一个查询通常要这样：

```go
rows, err := db.Query("SELECT id, name, age FROM users WHERE age > ?", 18)
if err != nil {
    return err
}
defer rows.Close()

var users []User
for rows.Next() {
    var u User
    if err := rows.Scan(&u.ID, &u.Name, &u.Age); err != nil { // 顺序错一个就全错
        return err
    }
    users = append(users, u)
}
return rows.Err()
```

GORM 把这套重复劳动封装成了链式 API：

```go
var users []User
err := db.Where("age > ?", 18).Find(&users).Error // 扫描、拼装、错误统一
```

★ **关键认知**：GORM 不是数据库驱动，也不是 `database/sql` 的替代品。它**建立在 `database/sql` 之上**，最终仍然调用驱动执行 SQL。理解这一点，才能理解"为什么 GORM 的报错常常来自下层驱动"。

## 1.2 与 database/sql 的关系

```text
┌──────────────────────────────────────────┐
│              你的业务代码                 │
└───────────────────┬──────────────────────┘
                    ▼
┌──────────────────────────────────────────┐
│        GORM（模型映射、SQL 生成、钩子）    │
└───────────────────┬──────────────────────┘
                    ▼
┌──────────────────────────────────────────┐
│      database/sql（连接池、Stmt、事务）    │
└───────────────────┬──────────────────────┘
                    ▼
┌──────────────────────────────────────────┐
│   驱动 go-sql-driver/mysql、pgx 等        │
└───────────────────┬──────────────────────┘
                    ▼
                 Database
```

| 能力 | database/sql | GORM |
|------|--------------|------|
| 连接池 | 提供 | 复用同一套，可取出 `*sql.DB` 调参 |
| SQL 手写 | 必须 | 可选，`Raw`/`Exec` 仍能写原生 SQL |
| 行到结构体映射 | 手写 `Scan` | 自动按字段/标签映射 |
| 关联查询 | 手写 JOIN 与二次查询 | `Preload`/`Joins` |
| 钩子与软删除 | 无 | 内建 |
| 性能可控度 | 最高 | 高，但需要知道它生成了什么 SQL |

★ **关键认知**：ORM 的价值在"减少样板代码"，代价是"多一层看不见的 SQL"。用 GORM 的正确姿势不是不学 SQL，而是**始终能说出这条链式调用最终会生成什么 SQL**。

# 2. 安装与连接

## 2.1 安装

```bash
go mod init example.com/gorm-demo
go get -u gorm.io/gorm
go get -u gorm.io/driver/mysql   # 按你使用的数据库选驱动
```

驱动是独立的模块，GORM 主库本身不含任何数据库协议实现：

| 数据库 | 驱动包 |
|--------|--------|
| MySQL / MariaDB | `gorm.io/driver/mysql` |
| PostgreSQL | `gorm.io/driver/postgres` |
| SQLite | `gorm.io/driver/sqlite`（需要 CGO）或 `github.com/glebarez/sqlite`（纯 Go） |
| SQL Server | `gorm.io/driver/sqlserver` |

## 2.2 连接 MySQL

```go
package main

import (
    "log"
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func main() {
    // parseTime=True 让驱动把 DATETIME 解析成 time.Time，否则扫描到结构体会报错
    dsn := "root:123456@tcp(127.0.0.1:3306)/demo?charset=utf8mb4&parseTime=True&loc=Local"

    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info), // 打印所有 SQL，学习期必开
    })
    if err != nil {
        log.Fatalf("连接数据库失败: %v", err)
    }

    sqlDB, err := db.DB() // 取回底层 *sql.DB，才能设置连接池
    if err != nil {
        log.Fatalf("获取底层连接失败: %v", err)
    }
    sqlDB.SetMaxOpenConns(100)                 // 最大打开连接数
    sqlDB.SetMaxIdleConns(10)                  // 最大空闲连接数
    sqlDB.SetConnMaxLifetime(time.Hour)        // 连接最长存活时间，避免被数据库主动断开
    sqlDB.SetConnMaxIdleTime(10 * time.Minute) // 空闲多久后关闭
}
```

⚠ `gorm.Open` 返回 nil error **不代表连接已经成功**。多数驱动是惰性连接，第一次真正执行 SQL 时才会握手。想启动即校验，调用 `sqlDB.Ping()`。

```go
if err := sqlDB.Ping(); err != nil {
    log.Fatalf("Ping 失败: %v", err)
}
```

## 2.3 连接 SQLite 与 PostgreSQL

```go
// SQLite：适合本地练习和单元测试，零外部依赖
import "github.com/glebarez/sqlite"

db, err := gorm.Open(sqlite.Open("demo.db"), &gorm.Config{})
```

```go
// PostgreSQL
import "gorm.io/driver/postgres"

dsn := "host=127.0.0.1 user=postgres password=123456 dbname=demo port=5432 sslmode=disable TimeZone=Asia/Shanghai"
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
```

## 2.4 常用 Config 选项

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    SkipDefaultTransaction: true,   // 关闭单条写操作的默认事务，写入性能明显提升
    PrepareStmt:            true,   // 缓存预编译语句，减少解析开销
    Logger:                 logger.Default.LogMode(logger.Warn),
    NowFunc: func() time.Time {     // 统一时区，避免 CreatedAt 差 8 小时
        return time.Now().Local()
    },
})
```

⚠ `SkipDefaultTransaction: true` 会失去"单条 Create 内部多语句的原子性"，但单条 INSERT 本身在数据库层就是原子的，因此绝大多数场景安全。真正需要多条语句原子性时用的是显式事务，不受此选项影响。

# 3. 第一个 CRUD

## 3.1 定义模型

```go
package main

import (
    "time"

    "gorm.io/gorm"
)

type User struct {
    ID        uint           `gorm:"primaryKey"`          // 主键，默认自增
    Name      string         `gorm:"size:64;not null"`    // varchar(64)
    Email     string         `gorm:"size:128;uniqueIndex"` // 唯一索引
    Age       int            `gorm:"default:0"`
    CreatedAt time.Time      // GORM 自动写入创建时间
    UpdatedAt time.Time      // GORM 自动维护更新时间
    DeletedAt gorm.DeletedAt `gorm:"index"` // 软删除标记
}
```

## 3.2 自动建表

```go
if err := db.AutoMigrate(&User{}); err != nil {
    log.Fatalf("迁移失败: %v", err)
}
```

## 3.3 增删改查

```go
// 增：Create 后主键会回填到 user.ID
user := User{Name: "alice", Email: "alice@example.com", Age: 18}
if err := db.Create(&user).Error; err != nil {
    log.Printf("创建失败: %v", err)
}
fmt.Println(user.ID) // 例如 1

// 查：主键查询；查不到返回 gorm.ErrRecordNotFound
var found User
err := db.First(&found, user.ID).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
    log.Println("用户不存在")
}

// 查多条：Find 查不到不报错，返回空切片
var users []User
db.Where("age >= ?", 18).Find(&users)

// 改：只更新指定字段
db.Model(&user).Update("age", 19)

// 删：有 DeletedAt 字段时为软删除，只写 deleted_at
db.Delete(&user)
```

★ **关键认知**：`First`、`Take`、`Last` 在未找到时会返回 `gorm.ErrRecordNotFound`，而 `Find` 不会。判断"不存在"用 `errors.Is(err, gorm.ErrRecordNotFound)`，不要用字符串比较，也不要靠 `RowsAffected` 猜。

# 4. 项目结构建议

## 4.1 学习期单文件

```text
gorm-demo/
├── go.mod
├── go.sum
└── main.go        # 连接、建表、CRUD 都写在这里
```

先把一条链路跑通，不要一上来就分层。

## 4.2 工程化结构

```text
gorm-demo/
├── cmd/server/main.go          # 入口：读配置、连库、启动
├── internal/
│   ├── model/                  # 数据模型，只放结构体与标签
│   ├── repository/             # 数据访问，封装 GORM 调用
│   └── service/                # 业务逻辑，不直接碰 *gorm.DB
├── config/
└── migrations/                 # 生产环境的 SQL 迁移脚本
```

★ **关键认知**：让 `*gorm.DB` 只出现在 repository 层。业务层一旦直接持有 `*gorm.DB`，换 ORM、写单测、加多租户过滤都会变得极其困难。这一条比目录本身重要得多。

# 5. 常见连接错误

| 报错关键字 | 原因 | 解决 |
|------------|------|------|
| `Access denied for user` | 账号密码错、来源 IP 未授权 | 检查 DSN；确认用户有 `'%'` 或对应 host 权限 |
| `Unknown database 'demo'` | 库不存在 | 先 `CREATE DATABASE demo`，或 DSN 里加 `?` 参数前先建库 |
| `dial tcp 127.0.0.1:3306: connectex: No connection could be made` | 服务没起、端口错、容器网络隔离 | `ping`/`telnet` 端口；容器内用服务名而不是 `localhost` |
| `parseTime=True` 相关 | 未开启时间解析，`time.Time` 扫描失败 | DSN 加 `parseTime=True` |
| `Error 1071: Specified key was too long` | MySQL 老版本索引长度限制 | 缩短 `varchar` 或设 `utf8mb4` 前缀索引 |
| `invalid connection` | 连接被 MySQL `wait_timeout` 断开 | 设置 `SetConnMaxLifetime` 小于数据库超时 |

⚠ 最常见的误判：把容器里的 `127.0.0.1` 当成宿主机。在 docker-compose 中，服务之间要用**服务名**连接，`localhost` 指向容器自身。

# 6. 动手练习

1. 用 SQLite 建一个 `demo.db`，定义 `Product` 模型（名称、价格、库存），完成增删改查，并打开 `logger.Info` 观察每条 SQL。
2. 把 MySQL DSN 故意写错密码，分别观察 `gorm.Open` 和 `Ping` 的返回时机，理解"惰性连接"。
3. 给连接池设置 `SetMaxOpenConns(1)`，起 10 个 goroutine 并发查询，观察串行执行的耗时。

# 7. 小结

| 主题 | 要点 |
|------|------|
| GORM 定位 | 基于 `database/sql` 的 ORM，负责映射与 SQL 生成 |
| 连接 | `gorm.Open(driver.Open(dsn), &gorm.Config{})`，惰性建连 |
| 连接池 | 通过 `db.DB()` 取回 `*sql.DB` 后设置 |
| 最小 CRUD | `Create`/`First`/`Update`/`Delete`，错误统一看 `.Error` |
| 未找到 | 只有 `First`/`Take`/`Last` 返回 `ErrRecordNotFound` |
| 分层 | `*gorm.DB` 收敛在 repository 层 |

下一篇进入模型与迁移，讲清字段标签、主键策略、时间戳约定、表名规则，以及 `AutoMigrate` 在生产环境到底能用到什么程度。
