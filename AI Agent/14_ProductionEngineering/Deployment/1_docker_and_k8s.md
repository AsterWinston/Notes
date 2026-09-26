# 1. 为什么容器化

容器化把应用与其运行时依赖一起打包为不可变镜像（Immutable Image），从根本上解决"环境一致、依赖隔离、快速部署、弹性伸缩"四类问题。对 LLM（大语言模型，Large Language Model）应用而言，容器化还额外承担 JVM/Python 运行时版本统一、CPU 与内存隔离、镜像分层缓存复用等职责。

```
容器化解决：环境一致、依赖隔离、快速部署、弹性伸缩
```

## 1.1 容器化解决的问题

### 1.1.1 环境一致性

传统部署中，"我本地能跑"往往源于开发机与生产机的 JDK 版本、系统库、时区、字体、编码不一致。容器把操作系统用户态、运行时、依赖库与配置全部固化进镜像，任何机器上运行的字节码完全相同，从而消除"环境漂移"。

- 基础镜像锁定 JDK 主版本（如 `eclipse-temurin:21-jre`）。
- 镜像以 digest 而非 tag 部署，防止 `latest` 漂移。
- 时区、编码在镜像内显式声明，不依赖宿主机。

### 1.1.2 依赖隔离

同一台宿主机可运行多个互相冲突的应用：一个需要 JDK 17、一个需要 JDK 21；一个依赖 glibc 2.31、一个需要 2.35。容器通过命名空间（Namespace）与控制组（cgroup，Control Group）实现进程、网络、文件系统、资源的隔离，互不干扰。

### 1.1.3 快速部署与弹性伸缩

镜像一旦构建完成，部署就退化为"拉取镜像并启动进程"，秒级完成。配合编排系统（Kubernetes），可基于负载自动增减副本，实现弹性伸缩。这是应对 LLM 应用突发流量的基础。

## 1.2 容器 vs 虚拟机

| 维度 | 容器 | 虚拟机 |
|------|------|--------|
| 隔离级别 | 进程级（共享内核） | 硬件级（独立内核） |
| 启动速度 | 秒级 | 分钟级 |
| 资源开销 | 低（MB 级） | 高（GB 级） |
| 镜像大小 | 小 | 大 |
| 安全隔离 | 较弱 | 强 |
| 适用场景 | 微服务、弹性伸缩 | 强隔离、异构内核 |

> 结论：绝大多数 Web/Agent 服务用容器；需要强隔离或不同内核时用虚拟机。

## 1.3 边界与坑

- 容器不是"轻量虚拟机"，隔离强度弱于 VM，多租户场景需配合 gVisor/Kata 等安全容器。
- 容器内 PID 1 需正确处理信号（SIGTERM），否则无法优雅停机。

# 2. Dockerfile（JVM 应用）

Dockerfile 是镜像的构建脚本。对 JVM 应用，推荐"多阶段构建 + 分层提取 + 非 root 运行"三件套：构建产物与构建工具分离，依赖层与业务层分离，运行用户降权。

```dockerfile
# 多阶段构建
FROM eclipse-temurin:21-jdk AS builder
WORKDIR /build
COPY . .
RUN ./mvnw -q -DskipTests package

FROM eclipse-temurin:21-jre
WORKDIR /app

# 时区与编码
ENV TZ=Asia/Shanghai \
    LANG=C.UTF-8 \
    JAVA_OPTS="-XX:MaxRAMPercentage=75 -XX:+UseG1GC"

COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

## 2.1 多阶段构建

### 2.1.1 构建阶段（builder）

构建阶段使用完整 JDK（含 `javac`、Maven/Gradle），产出 `target/*.jar`。该阶段的所有层都不会进入最终镜像，因此可以放心安装编译依赖。

### 2.1.2 运行阶段（runtime）

运行阶段只复制构建产物，使用 JRE（不含编译器）作为基础镜像，体积可从 ~800MB 降到 ~250MB。注意 `COPY --from=builder` 是跨阶段复制的唯一通道。

## 2.2 分层构建（加快构建）

### 2.2.1 分层原理

Spring Boot 3.x 支持把 fat jar 拆成 `dependencies`、`spring-boot-loader`、`snapshot-dependencies`、`application` 四层。依赖层几乎不变，业务层频繁变更。Docker 按层缓存，因此只需重传变化的应用层，推送与拉取都更快。

### 2.2.2 分层 Dockerfile 示例

```dockerfile
# 利用 Spring Boot 分层 jar，依赖层不常变
FROM eclipse-temurin:21-jre
WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

COPY --from=builder /build/target/dependencies/ ./
COPY --from=builder /build/target/spring-boot-loader/ ./
COPY --from=builder /build/target/snapshot-dependencies/ ./
COPY --from=builder /build/target/application/ ./

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

> 顺序很重要：从最不易变到最易变，缓存命中率最高。

## 2.3 镜像瘦身与安全

### 2.3.1 基础镜像选择

| 基础镜像 | 体积 | 说明 |
|----------|------|------|
| `eclipse-temurin:21-jdk` | ~450MB | 含编译器，仅构建用 |
| `eclipse-temurin:21-jre` | ~250MB | 运行推荐 |
| `eclipse-temurin:21-jre-alpine` | ~180MB | 更小，注意 musl 兼容 |
| `distroless/java21` | ~200MB | 无 shell，最安全 |

### 2.3.2 非 root 用户运行

```dockerfile
RUN groupadd -r app && useradd -r -g app app
USER app
```

> 容器以 root 运行是常见高危项，一旦逃逸影响宿主机。

## 2.4 边界与坑

- Alpine 使用 musl libc，某些原生库（如 RocksDB、部分向量库）不兼容，需实测。
- `ENTRYPOINT` 用 `sh -c` 会导致 JVM 不是 PID 1，SIGTERM 无法直接送达；可用 `exec java ...` 或直接 exec 形式。
- 不要在生产镜像里放 `.env`、密钥、测试数据。
- 镜像 tag 不要用 `latest`，用语义化版本或 git commit。

# 3. 资源限制（LLM 应用关键）

Kubernetes 通过 `requests`（调度依据）与 `limits`（上限）控制容器资源。LLM 应用对内存尤其敏感：向量计算、Netty 堆外缓冲、模型客户端连接池都会占用大量内存，配置不当会被 OOMKilled（内存溢出被杀）。

```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

| 注意 | 说明 |
|------|------|
| 堆内存 | `MaxRAMPercentage` 配合 limit |
| 堆外内存 | Netty/向量计算会用堆外 |
| CPU | 流式与并发需要 CPU |
| 内存 limit | 太小会 OOMKilled |

```yaml
env:
  - name: JAVA_OPTS
    value: "-XX:MaxRAMPercentage=70 -XX:MaxDirectMemorySize=512m"
```

> 容器感知：JDK 10+ 默认支持 cgroup 限制，无需手动设 `-Xmx`。

## 3.1 堆内存与 limit

### 3.1.1 MaxRAMPercentage

`-XX:MaxRAMPercentage=70` 表示堆上限取容器内存 limit 的 70%。留出 30% 给堆外、线程栈、元空间（Metaspace）、JVM 自身。若手动写死 `-Xmx4g` 而 limit 也是 4Gi，JVM 会因堆外占用被 OOMKilled。

## 3.2 堆外内存

Netty 的 `DirectByteBuffer`、向量库的原生内存、gRPC/HTTP 客户端的缓冲区都属于堆外。`-XX:MaxDirectMemorySize` 只约束 `DirectByteBuffer`，不约束 JNI 分配，因此预算要留足。

```
容器内存 limit 4Gi 预算示例：
  堆（-Xmx）        ~2.8Gi  (70%)
  直接内存          ~0.5Gi
  元空间/线程栈/GC   ~0.4Gi
  系统/其他          ~0.3Gi
```

## 3.3 CPU 限制

- `requests.cpu` 影响调度与 HPA 计算，`limits.cpu` 会触发 CFS 节流（throttling），导致延迟尖刺。
- 对延迟敏感的 LLM 服务，建议 `requests` 与 `limits` 相等（Guaranteed），或干脆不设 CPU limit 只设 request。
- `1000m` = 1 核。流式输出与并发推理对 CPU 敏感，核数不足会拖慢首 token 延迟。

## 3.4 边界与坑

- 内存 limit 过小 → `OOMKilled`，退出码 137，需在事件中确认。
- `requests` 远小于实际 → 节点超卖，容器互相争抢。
- CPU limit 过低 → 频繁 throttling，P99 延迟飙升。

# 4. Deployment

Deployment 是 Kubernetes 中管理无状态应用的核心对象，负责副本数、滚动更新、自愈与版本历史。对 LLM 服务，关键是"零中断滚动 + 正确探针 + 资源限制"。

## 4.1 滚动更新策略

### 4.1.1 maxSurge 与 maxUnavailable

| 参数 | 含义 | 零中断建议 |
|------|------|-----------|
| `maxSurge` | 更新期间可超出的副本数 | 1 或 25% |
| `maxUnavailable` | 更新期间不可用副本数 | 0 |

`maxUnavailable: 0` 表示先起新 Pod 并探针通过后才杀旧 Pod，保证容量不下降。

## 4.2 探针

### 4.2.1 readinessProbe

就绪探针决定 Pod 是否加入 Service 端点。未就绪的 Pod 不接收流量。LLM 服务启动慢（加载向量库、预热模型客户端），需给足 `initialDelaySeconds`。

### 4.2.2 livenessProbe

存活探针失败会重启容器。切勿把外部依赖（模型 API、数据库）纳入存活判断，否则依赖抖动会引发全量重启雪崩。

### 4.2.3 startupProbe

启动探针用于慢启动应用：在 `failureThreshold × periodSeconds` 内允许存活探针失败，避免启动期被误杀。

```yaml
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 8080 }
  failureThreshold: 30
  periodSeconds: 5
```

## 4.3 完整 Deployment 示例

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0          # 保证零中断
  selector:
    matchLabels:
      app: ai-app
  template:
    metadata:
      labels:
        app: ai-app
    spec:
      containers:
        - name: ai-app
          image: registry/ai-app:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: ai-secrets
                  key: openai-api-key
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 40
            periodSeconds: 15
          resources:
            requests: { memory: "2Gi", cpu: "1000m" }
            limits:   { memory: "4Gi", cpu: "2000m" }
```

## 4.4 边界与坑

- 探针路径指向昂贵接口（如真实推理）会拖垮服务。
- `replicas` 单副本无法滚动更新（会中断），至少 2 副本。
- 未设 `PodDisruptionBudget` 时，节点维护可能同时驱逐多个副本。
- 镜像更新用 `kubectl set image` 会改变模板，需确保 tag 唯一。

# 5. 优雅停机

长任务不能被打断：一个 Agent 请求可能正在多轮调用工具、流式输出或写入向量库。优雅停机指收到 SIGTERM 后停止接收新请求、等待在途请求完成、再退出进程。

## 5.1 停机流程

### 5.1.1 preStop 钩子

Kubernetes 删除 Pod 时，先执行 `preStop`，同时从 Service 端点摘除。给负载均衡器与调用方留出"感知摘除"的时间窗口。

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "sleep 10"]
```

### 5.1.2 terminationGracePeriodSeconds

宽限期是 preStop 与容器进程退出的总时限，超时会收到 SIGKILL（不可捕获）。

```yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60
```

## 5.2 Java 优雅停机实现

Spring Boot 提供 `server.shutdown=graceful` 处理 Web 请求，但后台 Agent 任务需自行排空。

```java
@Component
public class GracefulShutdown implements ApplicationListener<ContextClosedEvent> {

    private final AgentTaskRegistry taskRegistry;

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        log.info("收到停机信号，等待进行中的任务完成...");
        taskRegistry.awaitCompletion(Duration.ofSeconds(50));
        log.info("任务已排空，准备停机");
    }
}
```

配合配置：

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 55s
```

> 配合 K8s 的 `preStop` 钩子，先摘流量再停机。

## 5.3 边界与坑

- `awaitCompletion` 时长必须小于 `terminationGracePeriodSeconds`，否则被 SIGKILL。
- 长轮询/SSE 连接不主动断开会占满宽限期，需主动通知客户端重连。
- 未处理的线程池（如 `ExecutorService`）不会随上下文关闭，需显式 `shutdown`。

# 6. HPA 自动扩缩容

HPA（Horizontal Pod Autoscaler，水平 Pod 自动扩缩容）根据指标调整副本数。LLM 应用的特点是"单请求成本高、延迟长"，单纯按 CPU 扩缩往往滞后，需结合并发与队列积压。

## 6.1 基于资源的扩缩容

### 6.1.1 CPU 指标

CPU 利用率是默认指标，适合计算密集型服务。目标值一般 60%~70%，为突发留余量。

## 6.2 基于自定义指标

### 6.2.1 Pods 指标（并发）

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_active_requests
        target:
          type: AverageValue
          averageValue: "50"
```

### 6.2.2 External 指标（队列积压）

对 Worker 类应用，按**队列积压**扩缩容更准确：

```yaml
metrics:
  - type: External
    external:
      metric:
        name: kafka_consumergroup_lag
      target:
        type: AverageValue
        averageValue: "100"
```

## 6.3 边界与坑

- HPA 需要 metrics-server（资源指标）与 Prometheus Adapter（自定义指标）。
- 缩容默认有 5 分钟稳定窗口，扩缩频繁会抖动，可配 `behavior` 调节。
- `minReplicas` 至少 2，避免单点与冷启动。
- LLM 服务冷启动慢，扩容后需就绪才接流量，扩容"感知延迟"较长。
- 节点资源不足时 Pod 会 Pending，HPA 需配合 Cluster Autoscaler。

# 7. 健康检查

健康检查是编排系统判断"是否接流量"与"是否重启"的依据。原则：探针要快、要轻、要能反映真实可用性，但绝不调用昂贵依赖。

## 7.1 Spring Boot Actuator 健康端点

### 7.1.1 分组配置

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when_authorized
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

### 7.1.2 liveness 与 readiness 语义

| 探针 | 失败后果 | 应包含 |
|------|----------|--------|
| liveness | 重启容器 | 仅进程内部状态 |
| readiness | 摘除流量 | 依赖就绪（DB、向量库、模型连接） |

## 7.2 自定义健康指示器

### 7.2.1 轻量模型探测

```java
@Component
public class AiHealthIndicator implements HealthIndicator {

    private final ChatModel model;

    @Override
    public Health health() {
        try {
            // 轻量探测（可用固定短语 + 小 maxTokens）
            model.chat("ping");
            return Health.up().build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

> **健康检查不要调用昂贵的模型接口**，可改为检查配置与连接。

## 7.3 边界与坑

- 把外部依赖放进 liveness → 依赖抖动引发重启风暴。
- 探针超时/阈值过紧 → 慢启动服务反复重启。
- 健康端点暴露敏感详情 → 用 `when_authorized` 限制。

# 8. 配置与密钥

配置与镜像分离是十二要素应用（12-Factor App）的核心。Kubernetes 用 ConfigMap 承载非敏感配置、Secret 承载敏感信息。

## 8.1 ConfigMap

```yaml
# ConfigMap：非敏感配置
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-config
data:
  application.yaml: |
    app:
      ai:
        rag:
          top-k: 20
```

以卷挂载可让配置热更新（应用需支持重载）；以环境变量注入则需重启。

## 8.2 Secret

```yaml
---
# Secret：敏感信息
apiVersion: v1
kind: Secret
metadata:
  name: ai-secrets
type: Opaque
stringData:
  openai-api-key: ${OPENAI_API_KEY}
```

### 8.2.1 外部密钥管理

> 生产建议用 External Secrets 从 Vault/KMS 同步，而非把密钥存 K8s Secret。通过 `ExternalSecret` 声明式引用远端密钥，由控制器定期同步为 K8s Secret。

## 8.3 边界与坑

- K8s Secret 默认仅 base64 编码，非加密，需开启 etcd 加密或外部密钥管理。
- 环境变量注入的密钥会出现在 `kubectl describe`/进程环境中，卷挂载更安全。
- 配置变更也应视为一次发布，需可灰度、可回滚。
- 不要在镜像里固化环境相关配置。

# 9. 生产注意事项

## 9.1 风险清单

| 风险 | 建议 |
|------|------|
| 内存 limit 过小 | OOMKilled，按实际调 |
| 无优雅停机 | 长任务被中断 |
| 健康检查调模型 | 昂贵且易误判 |
| 密钥明文 | 用密钥管理 |
| 单副本 | 至少 2 副本 |
| 无 HPA | 突发流量压垮 |

## 9.2 上线检查清单

- [ ] 镜像非 root 运行、tag 唯一且非 `latest`。
- [ ] requests/limits 已按压测结果设置。
- [ ] 三类探针（startup/readiness/liveness）语义正确。
- [ ] 优雅停机时长 < 宽限期。
- [ ] HPA 指标与实际瓶颈匹配。
- [ ] 密钥来自外部密钥管理，未进镜像与日志；已配置 PDB 与多副本。

# 10. 小结

- 容器化保证环境一致与快速部署。
- LLM 应用要特别注意堆内/堆外内存与 CPU。
- 长任务必须优雅停机 + 足够宽限期。
- HPA 按 CPU/并发/队列积压扩缩容。
- 密钥用 Secret + 密钥管理，不写进镜像。
