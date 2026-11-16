微服务的测试比单体难在一个地方：**你测的不只是函数，还有服务之间的契约**。单元测试保证逻辑正确，集成测试保证组件能拼起来，契约测试保证上下游不会因为一次字段修改就翻车。本篇按测试金字塔从下往上讲，最后给出本地多服务联调的几种务实方案。

# 1. 测试金字塔

## 1.1 分层

| 层级 | 速度 | 依赖 | 主要目的 |
|------|------|------|----------|
| 单元 | 毫秒 | 无 | 业务规则、边界条件 |
| 集成 | 秒级 | DB/Redis/MQ | 组件协作、SQL 正确性 |
| 契约 | 秒级 | 契约文件 | 接口兼容性 |
| E2E | 分钟 | 多服务 | 关键用户路径 |

测试投入应该跟**故障成本**匹配：核心支付链路值得写 E2E，内部工具接口写好单元测试即可。不要追求 100% 覆盖率，要追求"关键路径有覆盖、边界条件有覆盖、历史 bug 有回归"。

## 1.2 微服务的特殊性

单体里一个函数调用就是一次测试；微服务里，一次业务操作可能跨三个服务、两个数据库、一个消息队列。所以要额外关注：

- **接口契约**：字段增删是否兼容。
- **错误传播**：下游超时，上游返回什么。
- **幂等**：重复请求是否产生重复数据。
- **超时与重试**：会不会放大流量。

★ **关键认知**：微服务测试的重点不是"每个函数都测到"，而是**把跨进程的假设固化下来**。契约测试固化接口形状，集成测试固化组件协作，端到端测试固化关键路径。三者缺一，问题就会从测试环境漏到线上。

# 2. 单元测试

## 2.1 表驱动

Go 最常用的写法是表驱动，把输入输出列成表：

```go
func TestCalcDiscount(t *testing.T) {
	tests := []struct {
		name    string
		amount  int64
		coupon  int64
		want    int64
		wantErr bool
	}{
		{name: "无优惠", amount: 100, coupon: 0, want: 100},
		{name: "满减", amount: 100, coupon: 20, want: 80},
		{name: "优惠超过金额", amount: 10, coupon: 20, wantErr: true},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := CalcDiscount(tt.amount, tt.coupon)
			if (err != nil) != tt.wantErr {
				t.Fatalf("err = %v, wantErr = %v", err, tt.wantErr)
			}
			if got != tt.want {
				t.Errorf("got = %d, want = %d", got, tt.want)
			}
		})
	}
}
```


## 2.2 用接口隔离依赖

`biz` 层依赖的是接口，不是数据库：

```go
type OrderRepo interface {
	Find(ctx context.Context, id int64) (*Order, error)
}

func (uc *OrderUsecase) Get(ctx context.Context, id int64) (*Order, error) {
	return uc.repo.Find(ctx, id) // 业务规则省略
}
```

测试时用 `memRepo` 这样的内存实现替换，不需要数据库。

## 2.3 gomock

依赖多了以后，手写 fake 会很啰嗦，可以用 `mockgen` 生成 mock：测试里用 `EXPECT()` 声明调用与返回值，未声明的调用会让测试失败。

★ **关键认知**：mock 的边界应该画在**领域接口**上，而不是数据库方法上。mock `OrderRepo.Find` 比 mock `sql.Rows.Scan` 稳定得多，因为前者是业务语义，后者是实现细节。

## 2.4 该测什么

| 值得测 | 不值得测 |
|--------|----------|
| 业务规则、金额计算 | getter/setter |
| 历史 bug 的回归 | 常量定义 |

⚠ **关键认知**：不要为了覆盖率写"调用一下不断言"的测试。这种测试既不能发现问题，又会在重构时增加负担。覆盖率是结果，不是目标。

# 3. 集成测试

## 3.1 范围

集成测试验证"代码 + 真实组件"能否协作，典型对象是 Repository 对真实 MySQL 的 SQL、缓存读写与序列化、消息生产与消费，以及 HTTP/gRPC 的完整处理链。

## 3.2 testcontainers

`testcontainers-go` 在测试时临时拉起容器，跑完销毁：

```go
req := testcontainers.ContainerRequest{
	Image:        "mysql:8.0",
	ExposedPorts: []string{"3306/tcp"},
	Env:          map[string]string{"MYSQL_ROOT_PASSWORD": "root", "MYSQL_DATABASE": "test"},
	WaitingFor:   wait.ForListeningPort("3306/tcp"),
}
c, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
	ContainerRequest: req, Started: true,
})
if err != nil {
	t.Fatal(err)
}
t.Cleanup(func() { _ = c.Terminate(ctx) })
host, _ := c.Host(ctx)
port, _ := c.MappedPort(ctx, "3306")
dsn := fmt.Sprintf("root:root@tcp(%s:%s)/test", host, port.Port())
```

## 3.3 构建标签隔离

集成测试默认不应该拖慢 `go test ./...`，用构建标签隔离：

```go
//go:build integration

package data

// 这个文件只在 go test -tags=integration 时编译
```

```bash
go test ./...                      # 只跑单元测试
go test -tags=integration ./...    # 加上集成测试
```

## 3.4 数据隔离

| 做法 | 优点 | 缺点 |
|------|------|------|
| 每个测试独立 schema | 完全隔离 | 建库慢 |
| 唯一前缀 + 清理 | 简单 | 可能残留 |

推荐"每个测试用唯一前缀造数据 + `t.Cleanup` 删除"，既快又不会互相污染。

★ **关键认知**：集成测试最怕**依赖执行顺序**。测试 A 插入的数据被测试 B 用到，单独跑就失败。每个测试必须自己准备数据、自己清理，能独立运行。

# 4. 契约测试

## 4.1 为什么需要

提供者把 `sku_id` 从 `int64` 改成 `string`，自己测试全过，消费者上线后直接崩。契约测试就是把这个"隐式约定"显式化：**消费者声明自己需要什么，提供者验证自己是否满足**。

## 4.2 proto 兼容性检查

gRPC 场景最实用的契约测试是**编译期兼容检查**，用 buf：

```yaml
# buf.yaml
version: v2
breaking:
  use:
    - FILE
```

```bash
buf breaking --against '.git#branch=main'
```

它能提前发现：删字段、改类型、改编号、删 RPC 等破坏性变更。把它放进 CI，比任何运行时测试都便宜。

| 变更 | buf 判定 | 处理 |
|------|----------|------|
| 新增字段 | 兼容 | 直接发 |
| 删除字段 | 不兼容 | 先 `reserved`，再删 |

## 4.3 Pact 思路

REST 场景常用 Pact：消费者测试生成契约文件，提供者拉取契约并验证。核心是让提供者在 CI 中跑消费者的契约，而不是自己写一份"我觉得没问题"的测试。

★ **关键认知**：契约测试的核心不是"测接口能不能通"，而是"**提供者的变更会不会破坏消费者**"。所以契约要由消费者定义、由提供者在 CI 中验证，而不是提供者自己写一份"我觉得没问题"的测试。

# 5. 服务内测试

## 5.1 HTTP 用 httptest

```go
func TestGetOrderHandler(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/v1/orders/1", nil)
	rec := httptest.NewRecorder()
	NewOrderHandler(memSvc).ServeHTTP(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want 200", rec.Code)
	}
	assert.Contains(t, rec.Body.String(), `"id":1`)
}
```


## 5.2 gRPC 用 bufconn

```go
lis := bufconn.Listen(1024 * 1024) // 内存连接，不走网络
srv := grpc.NewServer()
orderv1.RegisterOrderServer(srv, &orderServer{})
go func() { _ = srv.Serve(lis) }()
t.Cleanup(srv.Stop)

conn, err := grpc.NewClient("passthrough:///bufnet",
	grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
		return lis.DialContext(ctx)
	}),
	grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
	t.Fatal(err)
}
resp, _ := orderv1.NewOrderClient(conn).Get(context.Background(), &orderv1.GetRequest{Id: 1})
assert.Equal(t, int64(1), resp.Id)
```

## 5.3 替换外部依赖

| 依赖 | 测试替代 |
|------|----------|
| MySQL | testcontainers / sqlite（注意方言差异） |
| 下游 gRPC | bufconn + 假实现 |

★ **关键认知**：**时间是最隐蔽的不稳定源**。业务里直接调 `time.Now()`，测试就无法构造"过期前 1 秒"这种边界。把时钟抽象成接口，测试时注入固定时间，能让大量偶发失败消失。

# 6. 本地多服务联调

## 6.1 方案对比

| 方案 | 优点 | 缺点 | 适用 |
|------|------|------|------|
| docker compose | 简单、可复现 | 手动维护依赖 | 日常联调 |
| 共享开发环境 | 零配置 | 互相干扰 | 小团队 |

## 6.2 compose 编排

compose 里把 order、inventory、etcd、mysql、redis 各起一个 service，用 `depends_on` 声明启动顺序，环境变量统一指向 `etcd:2379`。这样一条 `docker compose up` 就能拉起完整依赖。

## 6.3 本地进程 + 容器依赖

最常见的开发姿势是：**只把当前服务跑在 IDE 里，其余全部容器化**。

```text
IDE 里的 order-service（可断点）
   ├──▶ localhost:2379   etcd 容器
   ├──▶ localhost:3306   mysql 容器
   └──▶ localhost:9000   inventory 容器
```

好处是调试体验最好；代价是要处理"容器里的服务如何访问宿主机"。macOS/Windows 的 Docker Desktop 原生支持 `host.docker.internal`，Linux 需要额外配置或直接用宿主机 IP。

⚠ **关键认知**：本地联调失败，八成不是业务代码问题，而是**注册地址不可达**。容器里注册的是容器 IP，宿主机进程连不上；宿主机注册的是 `127.0.0.1`，容器里也连不上。统一用可被双方解析的地址，或者在本地关掉注册、直接连固定地址。

# 7. 测试稳定性

| 不稳定源 | 表现 | 解决 |
|----------|------|------|
| 时间 | 跨天、时区、过期边界 | 注入 Clock |
| 网络 | 偶发超时 | 本地 mock / bufconn |
| 端口 | 并行冲突 | 用 `:0` 让系统分配 |

```bash
go test -race ./...        # 检测数据竞争
go test -count=10 ./...    # 重复跑，暴露偶发失败
go test -shuffle=on ./...  # 打乱顺序，暴露依赖
```

# 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 本地全过，CI 失败 | 依赖本地环境 | testcontainers / 固定版本 |
| 测试互相影响 | 共享数据库 | 独立数据 + 清理 |
| 改字段后消费者崩 | 没有契约检查 | buf breaking + Pact |
| 覆盖率很高但 bug 多 | 只调用不断言 | 关注断言质量 |

# 9. 练习

1. 给一个 `biz` 用例写表驱动测试，覆盖正常、边界、错误三类分支，要求不依赖数据库。
2. 用 testcontainers 启动 MySQL，为 Repository 写一个集成测试，用构建标签隔离。
3. 配置 `buf breaking`，故意删掉一个 proto 字段，观察 CI 是否拦截。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 金字塔 | 单元多、集成中、E2E 少 |
| 单元 | 表驱动 + 接口 mock，不碰真实依赖 |
| 契约 | buf 检查 proto，Pact 检查 REST |
| 服务内 | httptest、bufconn、依赖替换 |
| 联调 | 本地进程 + 容器依赖，注意地址可达 |

下一篇是全模块收尾，讲服务边界、数据一致性、团队协作、常见反模式，并把前面九篇串成一张可执行的学习路线图。
