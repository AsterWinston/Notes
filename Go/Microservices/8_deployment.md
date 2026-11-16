代码能跑起来只是开始，能不能安全上线、出问题能不能快速回滚，才是微服务的分水岭。本篇讲部署链路上的五个关键点：配置怎么注入、健康检查怎么区分、进程怎么优雅退出、镜像怎么构建、Kubernetes 怎么滚动发布。这些细节做错，发布会变成事故。

# 1. 部署前要定的事

## 1.1 构建产物

Go 的部署优势是**单一静态二进制**。构建时固定三件事：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
go build -trimpath -ldflags="-s -w" -o out/server ./cmd/server
```

| 参数 | 作用 |
|------|------|
| `CGO_ENABLED=0` | 产出静态链接，能跑在 scratch/distroless |
| `GOOS/GOARCH` | 交叉编译，避免本机能跑线上不能跑 |
| `-trimpath` | 去掉本机路径，构建可复现 |

## 1.2 配置注入

| 方式 | 适用 | 注意 |
|------|------|------|
| 镜像内文件 | 默认值、本地开发 | 改配置要重新构建 |
| ConfigMap 挂载 | 普通配置 | 更新有延迟，需重启或热加载 |
| 配置中心 | 需要热更新 | 增加外部依赖 |

★ **关键认知**：**镜像不可变，配置可注入**。同一个镜像应该能部署到测试和生产，差异只来自配置。把环境相关的东西烤进镜像，等于每个环境都要单独构建，回滚时也无法确认这个镜像到底跑的什么配置。

## 1.3 端口与信号

```text
HTTP   :8000   业务接口 + /healthz + /readyz + /metrics
gRPC   :9000   内部服务调用
SIGTERM        优雅关闭信号
```

# 2. 健康检查

## 2.1 liveness 与 readiness 的区别

| 探针 | 回答的问题 | 失败后果 |
|------|-----------|----------|
| liveness | 进程还活着吗 | 重启容器 |
| readiness | 能接流量吗 | 从 Service 摘除 |
| startup | 启动完成了吗 | 暂缓其他探针 |

```text
启动中：startup 未过 → liveness/readiness 不执行
就绪后：readiness 通过 → 加入 Endpoints
运行中：readiness 失败 → 摘除但不重启；liveness 失败 → 重启
```

★ **关键认知**：**liveness 不要检查下游依赖**。如果数据库挂了，所有实例的 liveness 都失败，Kubernetes 会把它们全部重启，故障从"数据库不可用"升级为"服务全灭"。liveness 只检查进程自身是否卡死，依赖检查放 readiness。

## 2.2 实现两个端点

```go
func HealthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK) // 只证明进程能响应
}

func ReadyHandler(db *sql.DB) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx, cancel := context.WithTimeout(r.Context(), time.Second)
		defer cancel()
		if err := db.PingContext(ctx); err != nil {
			w.WriteHeader(http.StatusServiceUnavailable) // 依赖不可用，摘流量
			return
		}
		w.WriteHeader(http.StatusOK)
	}
}
```

## 2.3 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| liveness 查数据库 | 数据库故障导致全量重启 | 只查进程 |
| readiness 永远返回 200 | 未就绪也接流量 | 检查关键依赖与预热状态 |
| 没有 startupProbe | 慢启动被 liveness 杀死 | 加 startupProbe |

⚠ **关键认知**：readiness 失败只是"暂时不接流量"，不是"重启"。数据库短暂不可用时，让 Pod 留在集群里等待恢复，比反复重启更健康。很多线上雪崩就是"readiness 查依赖 + 依赖抖动 + 全量摘除"叠加出来的。

# 3. 优雅关闭

## 3.1 正确顺序

```text
收到 SIGTERM → 1. 摘除 → 2. 排空 → 3. 停新连接、等在途请求
             → 4. 关闭数据库/消息队列/追踪 → 5. 退出
```

顺序反了就会 502：先关进程，调用方还在往这个地址发请求。

## 3.2 Kratos 的做法

```go
app := kratos.New(
	kratos.Name("order-service"),
	kratos.Server(httpSrv, grpcSrv),
	kratos.Registrar(registrar), // 退出时自动摘除
)
if err := app.Run(); err != nil { // 内部处理 SIGTERM/SIGINT
	log.Fatal(err)
}
```

`app.Run()` 会阻塞，收到信号后依次停止各 Server。不要在外面再写一套 `signal.Notify`，否则会和框架抢信号。

## 3.3 手写 net/http 的关闭

```go
srv := &http.Server{Addr: ":8000", Handler: mux}
// 先 go srv.ListenAndServe()，再等待信号
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit
_ = registrar.Deregister()   // 1. 先摘除
time.Sleep(10 * time.Second) // 2. 排空窗口
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
_ = srv.Shutdown(ctx) // 3. 关闭服务器
```

★ **关键认知**：排空窗口不是"浪费 10 秒"，而是**补偿服务发现与负载均衡的传播延迟**。Kubernetes 更新 Endpoints、客户端刷新地址都需要时间；不等待就关闭，仍会有请求打到已关闭的端口。

# 4. Docker

## 4.1 多阶段构建

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /src
ENV CGO_ENABLED=0 GOOS=linux
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/server /server
COPY --from=builder /src/configs /configs
USER nonroot:nonroot
EXPOSE 8000 9000
ENTRYPOINT ["/server", "-conf", "/configs"]
```

builder 阶段包含 Go 工具链，体积大；runtime 阶段只有二进制和证书，通常几十 MB。

## 4.2 镜像安全

- 用**非 root** 用户运行。
- 基础镜像选 distroless 或 alpine，减少攻击面。
- 不把 `.git`、测试文件、密钥拷进镜像，用 `.dockerignore` 排除。
- 镜像 tag 用版本号或 commit sha，**不要用 `latest`**。

## 4.3 本地联调

```yaml
services:
  order:
    build: .
    ports: ["8000:8000", "9000:9000"]
    environment: [ENV=dev, ETCD_ENDPOINTS=etcd:2379]
  etcd:
    image: bitnami/etcd:3.5
    environment: [ALLOW_NONE_AUTHENTICATION=yes]
    ports: ["2379:2379"]
```

# 5. Kubernetes

## 5.1 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: order-service}
spec:
  replicas: 3
  strategy:
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  selector:
    matchLabels: {app: order-service}
  template:
    metadata:
      labels: {app: order-service}
    spec:
      terminationGracePeriodSeconds: 40
      containers:
        - name: order
          image: registry.example.com/order-service:v1.2.0
          ports:
            - {containerPort: 8000, name: http}
            - {containerPort: 9000, name: grpc}
          resources:
            requests: {cpu: 200m, memory: 256Mi}
            limits: {cpu: "1", memory: 512Mi}
          readinessProbe:
            httpGet: {path: /readyz, port: http}
            periodSeconds: 5
          livenessProbe:
            httpGet: {path: /healthz, port: http}
            initialDelaySeconds: 10
          lifecycle:
            preStop:
              exec: {command: ["sh", "-c", "sleep 5"]}
```

## 5.2 Service

```yaml
apiVersion: v1
kind: Service
metadata: {name: order-service}
spec:
  selector: {app: order-service}
  ports:
    - {name: http, port: 80, targetPort: http}
    - {name: grpc, port: 9000, targetPort: grpc}
```

`targetPort: http` 引用容器端口名，改端口时只需改一处。

## 5.3 资源与弹性

HPA 用 CPU 利用率 70% 作为扩缩容指标，配合 `minReplicas: 3`、`maxReplicas: 20`，避免流量突增时容量不足。更精细的场景可以接入 QPS 或自定义指标。

★ **关键认知**：`requests` 决定调度，`limits` 决定上限。**不设 requests，调度器会以为 Pod 不需要资源，导致节点超卖**；`limits` 设得太低，Go 进程会因内存超限被 OOMKill。生产上 requests 按 P95 实际用量给，limits 留 1.5 到 2 倍余量。

# 6. 滚动发布

## 6.1 参数含义

| 参数 | 含义 | 建议 |
|------|------|------|
| `maxSurge` | 最多超出期望副本数多少 | 1 或 25% |
| `maxUnavailable` | 最多允许多少副本不可用 | 0（要求不中断） |

```text
滚动过程（replicas=3, maxSurge=1, maxUnavailable=0）
[v1][v1][v1] → [v1][v1][v1][v2] → [v1][v1][v2]
             → [v1][v2][v2] → [v2][v2][v2]
```

## 6.2 发布流程

```text
1. CI 构建镜像，tag 为 commit sha，推送
2. helm upgrade / kubectl set image
3. 新 Pod readiness 通过后加入 Endpoints
4. 旧 Pod 摘除 → 排空 → 退出；观察指标，异常回滚
```

## 6.3 回滚

```bash
kubectl rollout undo deployment/order-service
kubectl rollout undo deployment/order-service --to-revision=3
kubectl rollout status deployment/order-service
```

回滚的前提是**镜像可追溯**。用 `latest` 或不带版本号的镜像，回滚时根本不知道要回到哪个版本。

⚠ **关键认知**：回滚不只是"换回旧镜像"。如果新版本执行了数据库迁移，旧代码可能读不懂新表结构。**数据库变更必须向前兼容**：先加字段、再双写、最后删旧字段，让新旧版本能同时运行。

# 7. 配置与密钥

用 ConfigMap 存 `config.yaml`，挂载到 `/configs`，并在 Deployment 里声明 `volumeMounts` 与 `volumes`。密钥用 Secret，但 Secret 默认只是 base64 编码，不是加密；生产上要开启 etcd 加密、RBAC 限制读取，或使用 Vault/云厂商 KMS。

# 8. 常见坑

| 现象 | 原因 | 解决 |
|------|------|------|
| 发布时 502 | 先停进程后摘除 | preStop + 排空窗口 |
| 数据库抖动导致全量重启 | liveness 查依赖 | liveness 只查进程 |
| Pod 反复重启 | 启动慢于 liveness 超时 | 加 startupProbe |
| 配置改了没生效 | ConfigMap 挂载未更新 | 重启或热加载 |
| 节点内存被打爆 | 没设 requests | 设置 requests/limits |

# 9. 练习

1. 给一个 Go 服务写多阶段 Dockerfile，把镜像从 800MB 降到 50MB 以内，并用非 root 用户运行。
2. 实现 `/healthz` 与 `/readyz`，用 `docker stop` 观察进程是否在宽限期内优雅退出。
3. 写一份 Deployment + Service + HPA，设置 `maxUnavailable=0`，用 `kubectl rollout` 观察滚动过程。

# 10. 小结

| 主题 | 要点 |
|------|------|
| 构建 | 静态二进制、可复现、镜像不可变 |
| 配置 | 镜像与配置分离，环境靠注入 |
| 探针 | liveness 只查进程，readiness 查依赖 |
| 关闭 | 摘除 → 排空 → 关闭 → 退出 |
| 发布 | maxUnavailable=0，回滚要可追溯 |
| 数据库 | 迁移向前兼容，保证新旧共存 |

下一篇进入测试，讲单元测试、集成测试、契约测试，以及本地多服务联调的几种方式。
