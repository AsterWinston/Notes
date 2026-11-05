自动埋点能让你在半小时内看到 Gin 的入口 span、GORM 的 SQL span 和 gRPC 的每次调用，但它只记录框架知道的事。订单号、业务分支、失败原因这些真正用于排障的信息，仍然要手写。本篇讲清三层接入、手动 span 的加法、数据库脱敏，以及上下文丢失与重复埋点这两类高频事故。

# 1. 自动埋点与手动埋点的分工

| 框架 | 包 | 自动产生的内容 |
|------|----|---------------|
| Gin | `otelgin` | 入口 Server span、方法/路由/状态码 |
| GORM | `gorm.io/plugin/opentelemetry/tracing` | 每次查询的 Client span、SQL 模板、耗时 |
| gRPC | `otelgrpc` | 每次 RPC 的 Server/Client span、状态码、metadata 透传 |

自动埋点只覆盖框架边界，能回答"调了哪个接口、执行了什么 SQL"，回答不了"这笔订单为什么失败"。

★ **关键认知**：自动埋点是骨架，手动 span 是关节。骨架撑起链路形状，业务语义必须手写补充；反过来，手写也不该重复框架已经记录的 HTTP/gRPC 基础属性。

# 2. Gin 接入

## 2.1 最小接入

```go
func newRouter(tp trace.TracerProvider) *gin.Engine {
	r := gin.New()

	// 必须在注册路由之前 Use，否则路由不会经过中间件
	r.Use(otelgin.Middleware("order-api", otelgin.WithTracerProvider(tp)))
	r.Use(gin.Recovery())

	r.GET("/orders/:id", getOrder)
	return r
}
```

中间件会自动 Extract 父上下文、创建 `SpanKindServer` 的 span、把新 ctx 放进 `c.Request`、记录状态码，并在请求结束时 End。

## 2.2 顺序、路由名与基数

```go
// 正确：otelgin 在外层，Recovery 在内层
r.Use(otelgin.Middleware("order-api"))
r.Use(gin.Recovery())
```

★ **关键认知**：Gin 的中间件只对注册之后的路由生效，顺序写反不会报错，只会安静地少掉入口 span。`otelgin` 放在 `Recovery` 外层，panic 被恢复并写入 500 后，外层才能读到正确状态码。若先 `r.GET` 再 `r.Use`，入口 span 直接消失。

`otelgin` 默认用路由模板 `/orders/:id` 作为 span 名，这是正确的低基数写法。

⚠ **关键认知**：把 `c.Param("id")` 拼进 span 名或指标标签会让后端瞬间产生海量时间线。ID 放属性可以，放名字和标签不行。健康检查建议用 `otelgin.WithFilter` 过滤掉。

## 2.3 手动补充业务 span

```go
func createOrder(c *gin.Context) {
	reqCtx := c.Request.Context() // 入口 span 所在 ctx

	vCtx, vSpan := tracer.Start(reqCtx, "order.validate")
	vSpan.SetAttributes(attribute.String("order.channel", c.GetHeader("X-Channel")))
	if err := validate(vCtx, req); err != nil {
		vSpan.RecordError(err)
		vSpan.SetStatus(codes.Error, "validate failed")
		vSpan.End()
		c.JSON(http.StatusBadRequest, gin.H{"error": "invalid order"})
		return
	}
	vSpan.End()
	// 两个阶段都从 reqCtx 派生，才是入口 span 下的兄弟节点
	payCtx, paySpan := tracer.Start(reqCtx, "order.payment")
	defer paySpan.End()

	if err := charge(payCtx, req); err != nil {
		paySpan.RecordError(err)
		paySpan.SetStatus(codes.Error, err.Error())
		c.JSON(http.StatusInternalServerError, gin.H{"error": "payment failed"})
		return
	}
}
```

★ **关键认知**：`tracer.Start` 返回的新 ctx 必须继续往下传。写 `_, span := tracer.Start(ctx, ...)` 然后继续用旧 ctx 调子函数，子 span 会全部平级挂在入口下，业务阶段的父子关系完全丢失。

## 2.4 异步任务的上下文

```go
// 短任务：保留 trace 上下文，去掉请求取消信号
base := context.WithoutCancel(c.Request.Context())
go func() {
	_, span := tracer.Start(base, "async.send-email")
	defer span.End()
	_ = sendEmail(base, userID)
}()

// 长任务：请求 span 已结束，新建 trace 并用 Link 关联
link := trace.LinkFromContext(c.Request.Context())
go func() {
	_, span := tracer.Start(context.Background(), "async.rebuild-index",
		trace.WithNewRoot(), trace.WithLinks(link))
	defer span.End()
	_ = rebuildIndex()
}()
```

⚠ **关键认知**：不要把 `*gin.Context` 丢进 goroutine。它来自 `sync.Pool`，请求结束会被复用，异步代码读到的可能是下一个请求的数据。需要请求数据时先复制必要字段，或用 `c.Copy()`。

# 3. GORM 接入

## 3.1 官方插件

```go
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{Logger: newGormLogger()})
if err != nil {
	return nil, err
}

// 记录 SQL 模板而不是变量值，避免参数进入 span
if err := db.Use(tracing.NewPlugin(
	tracing.WithoutMetrics(),
	tracing.WithoutQueryVariables(),
	tracing.WithDBSystem("postgresql"),
)); err != nil {
	return nil, err
}
```

插件为每次查询创建 `SpanKindClient` 的 span，属性包括 `db.system`、`db.statement`、`db.operation` 与影响行数。不同版本的选项名可能微调，升级前先看插件文档与编译结果。

## 3.2 必须传 WithContext

```go
// 正确：db 绑定请求 ctx，SQL span 挂在入口 span 下
func (r *OrderRepo) Find(ctx context.Context, id string) (*Order, error) {
	var o Order
	err := r.db.WithContext(ctx).Where("id = ?", id).First(&o).Error
	return &o, err
}
// 错误示范：r.db.Where(...) 没有 ctx，SQL span 会变成根 span
```

★ **关键认知**：GORM 的 `db` 对象本身不携带请求上下文，每个查询都必须 `WithContext(ctx)`。漏掉这一步，SQL 依然有 span，但 trace_id 是新的，后端里表现为"每个查询一条独立链路"，比完全没埋点更难排查。

事务里同样要显式带 ctx：

```go
func (r *OrderRepo) CreateOrder(ctx context.Context, o *Order, items []Item) error {
	return r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		if err := tx.WithContext(ctx).Create(o).Error; err != nil {
			return err
		}
		return tx.WithContext(ctx).Create(&items).Error
	})
}
```

## 3.3 SQL 语句脱敏

| 风险点 | 例子 | 处理 |
|--------|------|------|
| 参数值进 span | `WHERE email = 'a@b.com'` | 只记录带 `?` 的模板 |
| 手工拼 SQL | `fmt.Sprintf("... '%s'", input)` | 改用参数化查询 |
| GORM 日志 | 打印完整 SQL | 开启 `ParameterizedQueries` |
| Collector 兜底 | 上游已带明文 | 删除或哈希 `db.statement` |

```go
func newGormLogger() logger.Interface {
	return logger.New(log.New(os.Stdout, "", log.LstdFlags), logger.Config{
		SlowThreshold:             200 * time.Millisecond,
		LogLevel:                  logger.Warn,
		IgnoreRecordNotFoundError: true,
		ParameterizedQueries:      true, // 日志里只留 ? 占位符
	})
}
```

对强敏感表，直接不记录语句，只记录操作与表名：

```go
ctx, span := tracer.Start(ctx, "db.users.find_by_email")
defer span.End()
span.SetAttributes(
	attribute.String("db.system", "postgresql"),
	attribute.String("db.operation.name", "SELECT"),
	attribute.String("db.collection.name", "users"),
	// 不记录 email，也不记录完整 SQL
)
```

⚠ **关键认知**：参数化查询只保证数据库层不被注入，不保证遥测层不泄露。要检查三处：span 的 `db.statement`、应用日志里的 SQL、Collector 收到的原始属性。三处都处理，才算脱敏闭环。

## 3.4 N+1 与批量操作

```text
1 次查询订单
  └── N 次查询订单项（每条一个 SQL span）
        └── 后端看到 1 + N 个 span，P99 被数据库往返拖垮
```

用 `Preload` 或 `Joins` 合并查询；批量插入用 `CreateInBatches`。不要为了"链路好看"关闭 GORM 埋点：span 暴增往往正是 N+1 的报警信号。

# 4. gRPC 接入

## 4.1 stats handler 是唯一入口

```go
srv := grpc.NewServer(
	grpc.StatsHandler(otelgrpc.NewServerHandler()),
	grpc.ChainUnaryInterceptor(Recovery(), Auth()),
)
conn, err := grpc.NewClient(addr, grpc.WithStatsHandler(otelgrpc.NewClientHandler()))
```

stats handler 负责创建 RPC span、注入/提取 metadata、记录状态码。**不要再同时注册 otelgrpc 拦截器**，两者会各建一个 span。

## 4.2 手写业务 span

```go
func (s *OrderServer) Create(ctx context.Context, req *pb.CreateRequest) (*pb.CreateResponse, error) {
	ctx, span := tracer.Start(ctx, "OrderService.Create")
	defer span.End()
	span.SetAttributes(attribute.Int("order.item_count", len(req.Items)))

	order, err := s.svc.Create(ctx, req)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, "create order failed")
		return nil, status.Error(codes.Internal, "internal error") // 细节留在 span
	}
	span.SetAttributes(attribute.String("order.id", order.ID))
	return &pb.CreateResponse{Id: order.ID}, nil
}
```

区分业务性失败与技术性失败时，把状态码记成属性：

```go
st, _ := status.FromError(err)
span.SetAttributes(attribute.Int("rpc.grpc.status_code", int(st.Code())))

// 约定：NotFound 属于业务结果，不拉高错误率；其余非 OK 标 Error
if st.Code() != codes.OK && st.Code() != codes.NotFound {
	span.SetStatus(codes.Error, st.Message())
}
```

★ **关键认知**：是否把 `NotFound`、`InvalidArgument` 标成 Error，必须团队统一。有人全标，错误率里全是用户输入错误；有人全不标，真正的 500 又被淹没。先定义"什么是错误"，再写代码，否则看板无法横向比较。

## 4.3 流式调用

长连接的流会产生一个持续很久的 span，不要为每条消息建子 span 并永久保留。消息级信息用 `span.AddEvent` 记录关键节点，或只对异常消息建 span；流结束务必 `End`；双向流记录收发条数，便于判断是上游不推还是下游不收。

# 5. 全链路透传

```text
[gin server] POST /orders                         trace_id=T1
  ├── [manual] order.validate
  ├── [gorm client] SELECT users WHERE id = ?
  ├── [manual] order.payment
  │     └── [grpc client] payment.Charge
  │           └── [grpc server] payment.Charge    trace_id 仍是 T1
  │                 └── [gorm client] INSERT payments ...
  └── [gorm client] INSERT orders ...
```

★ **关键认知**：跨进程的父子关系靠 header，进程内的父子关系靠 ctx，两者缺一不可。ctx 断了，同一进程内的 span 会平级；header 断了，跨服务的 trace_id 会变。排查时先看 trace_id 是否一致，再看 parent 是否正确。

| 层 | 检查点 | 常见错误 |
|----|--------|---------|
| Gin 入口 | 中间件在路由前注册 | 健康检查也埋点，噪声大 |
| Handler | 用 `c.Request.Context()` | 用了 `context.Background()` |
| GORM | 每个查询 `WithContext` | SQL 变根 span |
| 异步 | 复制 ctx 或建 Link | 把 `*gin.Context` 丢进 goroutine |

# 6. 常见错误与排错

| 现象 | 原因 | 解决 |
|------|------|------|
| Gin 没有任何 span | 中间件注册在路由之后 | 移到路由注册之前 |
| SQL span 是独立 trace | 查询没 `WithContext(ctx)` | 每个查询显式带 ctx |
| 一条 RPC 两个 span | stats handler 与拦截器同时启用 | 只保留 stats handler |
| 子 span 和父 span 平级 | 没接住 `Start` 返回的 ctx | 用新 ctx 调下游 |
| 异步任务读到别人的请求 | 把 `*gin.Context` 传进 goroutine | 复制字段或用 `c.Copy()` |
| SQL 出现明文手机号 | 记录了参数或完整语句 | 参数化 + 删 `db.statement` |
| gRPC 下游 trace_id 变了 | 客户端没挂 stats handler | 加 `WithStatsHandler` |

★ **关键认知**：排查埋点问题先验证链路形状，再看属性细节。先确认入口 span 有没有、子 span 挂没挂对、trace_id 跨服务是否一致，这三件事对了，再去纠结属性名和采样率。

# 7. 练习与自检

1. 用 `otelgin` + GORM 插件写一个查询接口，故意在 repo 层去掉 `WithContext`，对比 SQL span 与入口 span 的 trace_id；再补上，确认父子关系恢复。
2. 给下单 Handler 加三个手动 span（校验、扣库存、落单），用 `RecordError` 与 `SetStatus` 标记一次库存不足，观察后端里的 Error 状态。
3. 分别开启和关闭 `tracing.WithoutQueryVariables()` 与 `ParameterizedQueries`，对比 span 和日志里是否出现邮箱明文，写出一份脱敏清单。

# 8. 小结

| 主题 | 要点 |
|------|------|
| 分工 | 自动埋点搭骨架，手动 span 补业务语义 |
| Gin | 中间件在路由前注册，otelgin 在 Recovery 外层 |
| GORM | 每个查询 `WithContext`，事务内也要显式传 |
| 脱敏 | 参数化查询 + 不记变量 + 日志脱敏 + Collector 兜底 |
| gRPC | 只挂 stats handler，业务 span 用 `RecordError`/`SetStatus` |
| 异步 | 短任务 `WithoutCancel`，长任务新 trace + Link |

下一篇进入后端与收尾：Jaeger、Tempo、Prometheus、Loki 各自怎么接、容量怎么估算，以及把整个模块串起来的高频自检题。
