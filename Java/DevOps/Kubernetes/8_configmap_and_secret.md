# 1.ConfigMap概述

## 1.1 为什么需要ConfigMap

在任何软件工程实践中，将配置从代码中分离都是被反复验证过的核心原则（The Twelve-Factor App 第三原则：Store config in the environment）。Kubernetes 作为容器编排平台，天然面临这个问题：一个容器镜像被构建出来后，如何在不同的环境中使用不同的配置运行？如何在不重新构建镜像的情况下修改配置？

ConfigMap 的设计目标就是解决以下三个核心诉求：

**配置与代码分离**

传统做法中，配置文件往往和应用代码放在同一个代码仓库，甚至直接打包进镜像。这种做法带来几个严重问题：第一，任何一个配置项的修改（比如数据库连接串从测试库换成生产库）都需要重新经过 CI/CD 流水线构建新镜像，增加发布周期和成本；第二，包含生产环境配置的镜像如果被推送到公共镜像仓库，会造成配置泄露；第三，不同环境需要维护不同版本的镜像，镜像版本数量爆炸式增长。

ConfigMap 将配置信息以 Kubernetes API 对象的形式存储在集群中，与应用镜像解耦。应用镜像只包含应用程序本身，不包含环境特定配置。在 Pod 启动时，Kubernetes 将 ConfigMap 中的数据注入到容器中（通过环境变量或文件挂载）。这意味着同样的镜像可以安全地在开发、测试、生产环境之间流转，只需要绑定不同的 ConfigMap 即可。

```yaml
# 示例：同一个镜像在不同 Namespace 使用不同 ConfigMap
# dev namespace
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  DB_HOST: "dev-db.local"
  LOG_LEVEL: "DEBUG"
---
# prod namespace
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  DB_HOST: "prod-db.internal"
  LOG_LEVEL: "WARN"
```

**环境差异化**

微服务架构中，同一个服务可能以多实例形式部署在不同环境中。这些环境之间的差异主要体现在配置上：数据库地址、缓存集群地址、消息队列端点、日志级别、第三方 API 密钥等。ConfigMap 使得我们可以用同一套应用部署清单（Deployment YAML），通过引用不同 Namespace 下的同名 ConfigMap，或通过 Kustomize/Helm 的覆盖机制，自动适配不同环境的配置差异。

更重要的是，ConfigMap 天然支持 Namespace 级别的隔离。不同 Namespace 下的 ConfigMap 互不可见，这意味着你可以为开发、测试、生产创建完全独立的配置空间，避免配置串扰。结合 RBAC，你可以控制哪些团队成员有权修改生产环境的 ConfigMap，哪些人只能查看开发环境的配置。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:v1.2.3  # 同一个镜像
    envFrom:
    - configMapRef:
        name: app-config  # 不同 Namespace 下自动获取不同配置
```

**动态更新**

现代微服务架构中，配置变更不应总是要求重启服务。例如，日志级别从 INFO 调整为 DEBUG 以排查生产问题，或者限流阈值需要临时下调，这些场景都希望配置能快速生效。当 ConfigMap 以 Volume 方式挂载到 Pod 时，kubelet 会定期（默认每 60 秒，可通过 kubelet 的 `--sync-frequency` 参数调整）从 API Server 同步最新的 ConfigMap 内容到 Pod 的文件系统中。应用如果能够 watch 文件变化并自动重新加载配置，就能实现不重启 Pod 的配置热更新。

但需要注意的是，这种热更新机制有明确的前提条件和限制：必须以文件挂载方式使用 ConfigMap，如果以环境变量方式注入，Pod 启动后环境变量就固定了，不会再变化；挂载的文件实际上是通过符号链接（symlink）机制实现的，kubelet 会将新版本的 ConfigMap 数据写入新的目录，然后原子地切换符号链接；应用程序本身需要支持配置热加载，否则文件变化后依然不会生效。

## 1.2 配置管理演进

要理解 ConfigMap 的设计价值，需要回顾配置管理在软件工程中的演进历程。每一个阶段的方案都是在解决前一个阶段的痛点，而 ConfigMap 是这个演进过程中的重要里程碑。

**第一阶段：硬编码**

最原始的做法是将配置直接写在代码中。例如，在 Java 代码中直接 `String dbUrl = "jdbc:mysql://localhost:3306/mydb"`。这种做法的问题显而易见：任何配置变更都需要修改代码并重新编译部署；不同环境需要维护不同的代码分支；敏感信息（如密码）暴露在代码仓库中。即使在今天，仍然能在一些遗留系统中看到这种反模式。

```java
// 硬编码反模式 - 绝不推荐
public class DatabaseConfig {
    public static final String URL = "jdbc:mysql://192.168.1.100:3306/proddb";
    public static final String USER = "admin";
    public static final String PASSWORD = "MySecret123";  // 密码明文写在代码中
}
```

**第二阶段：配置文件**

将配置提取到外部配置文件中（如 `application.properties`、`config.yml`、`.env` 文件），代码运行时读取这些文件。这解决了硬编码的部分问题，配置文件可以独立于代码进行修改。但新的问题也随之而来：配置文件通常和代码在同一个仓库中，依然存在敏感信息泄露风险；配置文件需要被打包进部署制品（JAR/WAR/Docker 镜像），修改配置仍然需要重新构建；多环境配置需要通过 profile 机制（如 `application-dev.properties`、`application-prod.properties`）管理，配置文件数量快速增长。

```properties
# application.properties - 传统配置文件方式
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
server.port=8080
logging.level.root=INFO
```

**第三阶段：环境变量**

将配置通过操作系统环境变量传递给应用程序。这是 The Twelve-Factor App 推荐的做法。Docker 原生支持通过 `-e` 参数或 `--env-file` 传入环境变量。环境变量的优势在于：完全与代码和构建制品解耦；容器运行时可动态指定；不会意外提交到代码仓库。但环境变量也有天然的限制：环境变量是扁平的键值对，不支持层次化结构；大量配置通过环境变量管理会变得杂乱；修改环境变量通常需要重启容器；在 Docker Compose 或脚本中管理上百个环境变量非常痛苦。

```bash
# Docker 环境变量方式
docker run -d \
  -e DB_HOST=prod-db.internal \
  -e DB_PORT=5432 \
  -e LOG_LEVEL=info \
  -e REDIS_HOST=redis.internal \
  myapp:v1.2.3
```

**第四阶段：配置中心**

随着微服务架构的兴起，集中式配置中心应运而生，如 Spring Cloud Config、Apollo（携程）、Nacos（阿里）、Consul KV、etcd 等。配置中心提供了集中管理、版本控制、动态刷新、灰度发布等高级特性。但配置中心本身也是需要运维的基础设施，引入了额外的复杂度；应用需要在代码层面集成配置中心的客户端 SDK；配置中心可能成为单点故障（如果配置中心不可用，新启动的服务实例可能无法获取配置）。

```yaml
# Spring Cloud Config 配置中心架构
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo
          search-paths: '{application}'
  application:
    name: my-service
```

**第五阶段：Kubernetes ConfigMap**

ConfigMap 是 Kubernetes 原生的配置管理方案。它直接集成在 Kubernetes API 中，不需要部署额外的配置中心服务；通过 Kubernetes 的声明式 API 管理配置，支持 kubectl、GitOps 工具（如 ArgoCD、Flux）等标准工作流；天然支持 Namespace 隔离和 RBAC 权限控制；通过 Volume 挂载实现配置热更新；与 Secret 配合，覆盖敏感和非敏感两种配置场景。

ConfigMap 不是要取代配置中心，而是提供了另一种轻量级的选择。对于已经深度使用配置中心的组织，可以在 Kubernetes 中继续使用配置中心；对于追求简约和 Kubernetes 原生体验的团队，ConfigMap + Secret 完全够用。实际工程中，也有团队采用混合模式：非敏感配置用 ConfigMap，敏感配置用配置中心的高级加密特性。

```
配置管理演进总结：

硬编码 → 配置文件 → 环境变量 → 配置中心 → K8s ConfigMap/Secret
  ↓        ↓          ↓          ↓              ↓
 不可维护  需重新构建   扁平结构   额外基础设施    K8s原生集成
 不安全    仓库耦合    需重启容器  单点故障风险   声明式API管理
```

## 1.3 ConfigMap的设计哲学与限制

ConfigMap 的设计体现了 Kubernetes 的设计哲学：简单、声明式、API 驱动、关注点分离。但简单也意味着存在明确的边界和限制，理解这些限制对于正确使用 ConfigMap 至关重要。

**设计哲学**

第一个设计原则是"一切皆 API 对象"。ConfigMap 是一个标准的 Kubernetes API 资源，拥有完整的 API 版本（`v1`），支持标准的 CRUD 操作，可以被 watch、可以被 RBAC 控制、可以参与声明式管理。这意味着你可以像管理 Deployment、Service 一样管理 ConfigMap，使用 kubectl、client-go、各类 GitOps 工具。

第二个设计原则是"声明式而非命令式"。你声明期望的配置状态（`data` 字段中的键值对），Kubernetes 负责将这个状态传递到使用它的 Pod 中。你不必关心配置是如何同步到节点的，如何写入容器文件系统的，只需声明"这个 Pod 需要用到那个 ConfigMap"。

第三个设计原则是"纯数据载体"。ConfigMap 本身不包含任何业务逻辑，它只是一个键值对集合。它不解释配置的含义，不验证配置的合法性，不做配置转换。这些职责留给应用本身或专门的配置管理工具。

**只能存储明文数据**

ConfigMap 的数据存储在 etcd 中默认以明文形式存在（除非开启了 etcd 静态加密）。这意味着任何有权限读取 ConfigMap API 的人都能看到完整内容。ConfigMap 的设计意图是存储非敏感配置信息——数据库连接串（不含密码）、日志级别、端口号、功能开关等。对于密码、Token、证书、私钥等敏感信息，必须使用 Secret 资源。

这是一个有意为之的设计决策，而非安全缺陷。ConfigMap 和 Secret 的分离体现了"关注点分离"原则：ConfigMap 专注于配置管理的便利性（明文、易编辑、易查看），Secret 专注于安全性（base64 编码、独立权限控制、可加密存储）。将两者混为一谈反而会导致安全实践恶化——开发者可能为了"方便"而将密码放进 ConfigMap。

```yaml
# ❌ 错误：密码放在 ConfigMap 中
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_USER: admin
  DB_PASSWORD: SuperSecret123  # 绝对不要这样做！

# ✅ 正确：密码放在 Secret 中
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: SuperSecret123
```

**1MB 大小限制**

每个 ConfigMap 对象的 `data` 字段内容总和不能超过 1MB（准确说是 1048576 字节，即 etcd 的请求大小限制）。这个限制来自 etcd 的 `--max-request-bytes` 参数，默认是 1.5MB，Kubernetes 对 ConfigMap 做了更保守的 1MB 限制。

为什么有这个限制？etcd 是分布式一致性存储系统，每个写请求都需要通过 Raft 协议在多数节点间达成共识。单个对象过大会导致 Raft 提案延迟增加，影响 etcd 集群的整体性能。如果允许存储 10MB 甚至更大的 ConfigMap，不仅会拖慢 ConfigMap 自身的更新速度，还会影响其他 API 对象（Node、Pod、Service）的创建和更新，因为所有对象共享同一个 etcd 集群。

另外，ConfigMap 在存储时，每个键值对都会创建一个独立的文件或环境变量。一个超大 ConfigMap 会导致 Pod 启动时需要写入大量文件或设置大量环境变量，增加 Pod 启动延迟。

对于超过 1MB 的配置场景，有以下替代方案：
- 将配置拆分为多个 ConfigMap，每个不超过 1MB
- 将大型配置文件（如 Nginx 的 GeoIP 数据库、机器学习模型文件）存储在 PersistentVolume 中
- 使用 Init Container 从对象存储（S3、OSS）下载大型配置
- 如果是静态资源，考虑直接打包进镜像

```bash
# 检查 ConfigMap 大小
kubectl get configmap large-config -o json | Measure-Object -Property Length
# 或者查看 etcd 中存储的近似大小
kubectl get configmap large-config -o yaml | Measure-Object -Character
```

**etcd 存储压力**

对于生产集群，ConfigMap 数量和大小需要被监控和管理。etcd 是 Kubernetes 的"大脑"，所有 API 对象都存储在 etcd 中。如果创建了大量的 ConfigMap（比如几千个），etcd 的数据文件会显著增长，影响备份恢复速度和集群故障恢复时间。

每个 Node 上的 kubelet 会 watch 本节点所有 Pod 引用的 ConfigMap。如果一个 ConfigMap 被大量 Pod 使用（数百个），每次 ConfigMap 更新都会触发所有相关 Pod 的 Volume 刷新，造成 API Server 和 kubelet 的压力。这也是后面要讨论的 Immutable ConfigMap 的动机之一。

**不可直接跨 Namespace 引用**

ConfigMap 是 Namespace 级别的资源，Pod 只能引用同一 Namespace 下的 ConfigMap。这是 Kubernetes 安全模型的一部分——配置数据不应该跨 Namespace 边界自由流动。如果确实需要在多个 Namespace 共享配置，需要手动将 ConfigMap 复制到每个目标 Namespace，或者使用更高级的工具（如 Kyverno 的复制策略、Crossplane 等）。

```bash
# 将 ConfigMap 从一个 Namespace 复制到另一个
kubectl get configmap my-config -n source-ns -o yaml |
  Select-String -NotMatch "namespace:|resourceVersion:|uid:|creationTimestamp:" |
  ForEach-Object { $_ -replace '^\s+', '' } |
  kubectl apply -n target-ns -f -
```


# 2.ConfigMap的创建方式

## 2.1 YAML定义创建

YAML 定义创建是生产环境中最推荐的方式，因为它支持声明式管理、版本控制和 GitOps 工作流。通过 YAML 文件定义 ConfigMap，你可以将配置纳入 Git 仓库，通过 CI/CD 流水线自动应用到集群。

ConfigMap 的基本结构非常简单：包含 `apiVersion`、`kind`、`metadata` 和 `data` 四个核心字段。`data` 字段是一个字符串键值对映射，键名必须符合 DNS 子域名规则（只能包含小写字母、数字、`-` 和 `.`，且以字母数字开头结尾），值可以是任意文本内容。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
  namespace: default
  labels:
    app: game
    version: v1
  annotations:
    description: "游戏服务配置"
data:
  # 简单键值对
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30

  # UI 配置
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice

  # 整数配置（注意：所有值都是字符串）
  max-players: "100"
  min-players: "2"

  # JSON 格式配置
  app-settings.json: |
    {
      "logLevel": "INFO",
      "maxConnections": 1000,
      "timeoutMs": 30000,
      "features": {
        "newUI": true,
        "betaSearch": false
      }
    }

binaryData:
  # 如果需要存储二进制数据，使用 binaryData（base64 编码）
  # 注意：binaryData 和 data 的键不能重复
  logo.png: iVBORw0KGgoAAAANSUhEUgAA...
```

`data` 字段中的所有值都是字符串。即使你写的是数字 `100`，Kubernetes 也会将其作为字符串 `"100"` 存储和处理。这意味着你的应用在读取这些配置时需要自行进行类型转换。这是 ConfigMap 的一个设计选择——保持简单和通用性，不做类型推断。

对于多行配置内容，使用 YAML 的 `|`（literal block scalar）可以保留换行和缩进，非常适合存储配置文件内容（如 `.properties`、`.yaml`、`.json`、`.toml` 等）。

`binaryData` 字段用于存储二进制数据，值必须是 base64 编码的字符串。`binaryData` 中的键和 `data` 中的键不能重复。实际使用中，`binaryData` 很少用到，因为 ConfigMap 的设计目标就是存储文本配置，二进制数据建议用其他方式（如打包进镜像、使用 PersistentVolume 或对象存储）。

**生产环境中的 YAML 管理实践**

在实际项目中，ConfigMap 的 YAML 文件应该被纳入版本控制，和应用代码放在一起（或统一的配置仓库中）。典型的目录结构如下：

```
my-service/
├── src/                    # 应用源代码
├── Dockerfile
└── k8s/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── configmap.yaml   # 基础配置
    └── overlays/
        ├── dev/
        │   └── configmap-patch.yaml  # 开发环境覆盖
        ├── staging/
        │   └── configmap-patch.yaml  # 预发布环境覆盖
        └── prod/
            └── configmap-patch.yaml  # 生产环境覆盖
```

## 2.2 kubectl create configmap --from-literal

`--from-literal` 方式适合快速创建包含少量键值对的 ConfigMap，在开发和调试阶段非常方便。它直接从命令行参数中读取键值对，使用 `key=value` 的格式。

```bash
# 创建包含单个键值对的 ConfigMap
kubectl create configmap my-config \
  --from-literal=DB_HOST=mysql.default.svc.cluster.local

# 创建包含多个键值对的 ConfigMap
kubectl create configmap app-config \
  --from-literal=DB_HOST=mysql.default.svc.cluster.local \
  --from-literal=DB_PORT=3306 \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# 查看创建的 ConfigMap
kubectl get configmap app-config -o yaml
```

输出结果大致如下：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  DB_HOST: mysql.default.svc.cluster.local
  DB_PORT: "3306"
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
```

`--from-literal` 的底层实现是 kubectl 将这些键值对构造为 ConfigMap YAML，然后通过 API 发送给 API Server。和直接编写 YAML 没有本质区别。需要注意的是，每个 `--from-literal` 参数只能包含一个键值对，大量配置时命令行会变得很长。

一个常见的调试技巧是使用 `--dry-run=client -o yaml` 预览生成的 YAML，然后再决定是否实际创建：

```bash
# 预览生成的 YAML，不实际创建
kubectl create configmap my-config \
  --from-literal=key1=value1 \
  --from-literal=key2=value2 \
  --dry-run=client -o yaml
```

这种方式最适合的场景是：临时测试配置、快速验证、演示用途。在生产环境中，应该使用 YAML 文件声明式管理。

## 2.3 kubectl create configmap --from-file

`--from-file` 是从文件内容创建 ConfigMap 最常用和强大的方式。它会读取一个或多个文件，以文件名（或指定键名）作为 ConfigMap 的键，文件内容作为值。

**按文件名创建（文件名自动成为键名）**

```bash
# 假设存在以下文件：
# - nginx.conf (Nginx 配置文件)
# - app.properties (应用配置文件)
# - logging.xml (日志配置文件)

# 逐个文件创建
kubectl create configmap nginx-config --from-file=nginx.conf

# 从目录中所有文件创建（递归读取）
kubectl create configmap app-config --from-file=config-files/

# 同时从多个来源创建
kubectl create configmap combined-config \
  --from-file=nginx.conf \
  --from-file=app.properties \
  --from-literal=ENV=production
```

当指定目录时，kubectl 会递归读取目录中的所有文件（不包括子目录，如需包含子目录需要使用扁平化脚本），每个文件成为一个数据项。

**自定义键名**

如果不希望使用文件名作为键名，可以使用 `<key>=<file-path>` 语法自定义键名：

```bash
# 自定义键名
kubectl create configmap nginx-config \
  --from-file=nginx-main.conf=nginx.conf \
  --from-file=nginx-default.conf=default.conf

# 这样在 ConfigMap 中，键名是 nginx-main.conf 和 nginx-default.conf
# 而不是原始的 nginx.conf 和 default.conf
```

**底层机制**

当使用 `--from-file` 时，kubectl 将文件内容直接读取为字节流，存储在 ConfigMap 的 `data` 字段中。文件内容会保持原样，包括换行符、特殊字符等。如果文件很大（接近 1MB 限制），kubectl 会读取并传输整个文件内容。

对于二进制文件，kubectl 会自动将其 base64 编码后存储在 `binaryData` 字段中。kubectl 通过检查文件内容是否包含 null 字节来判断是否为二进制文件。这意味着文本文件中的 null 字节可能导致其被错误识别为二进制文件。

```bash
# kubectl 底层做的事情大致等价于：
# 1. 读取文件内容
# 2. 构建 ConfigMap 对象
# 3. 检查内容是否二进制（包含 null 字节）
# 4. 如果是二进制，存入 binaryData；否则存入 data
# 5. 通过 REST API 发送给 API Server

# 可以用以下命令确认二进制检测规则：
# 如果文件包含 \x00 字节，kubectl 会将其识别为二进制
```

**大量文件的处理策略**

当配置文件数量非常多时（如有几十个 nginx vhost 配置文件），逐个指定 `--from-file` 会很繁琐。推荐的做法是将所有配置文件组织在一个目录中，然后使用 `--from-file=directory/` 一次性导入。更好的做法是使用 Kustomize 的 configMapGenerator，支持通配符和自动哈希。

## 2.4 kubectl create configmap --from-env-file

`--from-env-file` 用于从 `.env` 格式的文件创建 ConfigMap。`.env` 文件格式是 `KEY=VALUE` 每行一个，支持空行和 `#` 开头的注释行。

```bash
# 创建 .env 文件
@"
DB_HOST=mysql.default.svc.cluster.local
DB_PORT=3306
DB_NAME=myapp
DB_USER=admin
# 这是注释行，会被忽略
LOG_LEVEL=info

MAX_CONNECTIONS=100
"@ | Out-File -Encoding UTF8 config.env

# 从 .env 文件创建 ConfigMap
kubectl create configmap db-config --from-env-file=config.env

# 查看结果
kubectl get configmap db-config -o yaml
```

生成的 ConfigMap 会将 `.env` 文件中的每一行解析为一个独立的键值对：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: mysql.default.svc.cluster.local
  DB_PORT: "3306"
  DB_NAME: myapp
  DB_USER: admin
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
```

`.env` 文件解析规则：
- 空行自动忽略
- `#` 开头的行被当作注释忽略
- 没有 `=` 的行可能被忽略或报错（取决于 kubectl 版本）
- 值中的引号不会自动去除（`KEY="value"` 会变成键为 `KEY`、值为 `"value"`）
- 值中的空格会保留（`KEY = value` 的键是 `KEY ` 带尾部空格）

`--from-env-file` 和 `--from-literal` 的组合使用：

```bash
kubectl create configmap full-config \
  --from-env-file=base-config.env \
  --from-literal=ENV_OVERRIDE=custom-value
```

`--from-env-file` 适合的场景：已有 `.env` 文件的项目迁移到 Kubernetes；使用 Docker Compose（天然使用 `.env` 文件）迁移到 Kubernetes 时保持配置格式一致；团队习惯使用 `.env` 文件管理环境变量。

## 2.5 生成器 (kustomize configMapGenerator)

Kustomize 是 Kubernetes 原生的配置管理工具（从 1.14 版本开始集成在 kubectl 中 `kubectl apply -k`）。`configMapGenerator` 是 Kustomize 提供的声明式 ConfigMap 生成器，解决了手动管理 ConfigMap 时的两个核心痛点：

**痛点一：名字冲突和回滚**

当你修改 ConfigMap 内容并重新 apply 时，旧 Pod 仍然挂载着旧版本的 ConfigMap。只有当 Pod 重启时，才会加载新版本。这导致了配置和实际生效之间的不一致窗口。更糟糕的是，如果你需要回滚，你找不到一个明确的"上一版本配置"。

Kustomize 的 configMapGenerator 通过在 ConfigMap 名称后追加内容哈希来解决这个问题。每次配置内容变化，哈希值就变化，生成的新 ConfigMap 就有新名字。Deployment 引用新名字的 ConfigMap（通过 `replacements` 或手动更新），触发 Pod 滚动更新，新 Pod 自动加载新配置。

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

configMapGenerator:
- name: app-config
  files:
  - application.properties
  - logback.xml
  literals:
  - JAVA_OPTS=-Xms512m -Xmx1024m
  - ENV=production
  # 生成的 ConfigMap 名称类似：app-config-7k8m2f
  # 每次内容变化，后缀自动变化
```

执行 `kubectl apply -k .` 后，实际创建的 ConfigMap 名称会是 `app-config-<hash>`，例如 `app-config-g5f8k2m`。

**痛点二：多环境覆盖**

同一个应用在不同环境需要不同配置。Kustomize 通过 overlay 机制天然支持这个场景：

```yaml
# base/kustomization.yaml - 基础配置
configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=info
  - MAX_CONNECTIONS=100

# overlays/prod/kustomization.yaml - 生产环境覆盖
bases:
- ../../base
configMapGenerator:
- name: app-config
  behavior: merge  # merge：合并；replace：替换；create：新建
  literals:
  - LOG_LEVEL=warn  # 覆盖 base 中的 LOG_LEVEL
  - MONITORING_ENABLED=true  # 新增配置项
```

`behavior` 字段控制合并策略：
- `merge`（默认）：将 overlay 中的配置合并到 base 的 ConfigMap 中，同名键会覆盖
- `replace`：用 overlay 中定义的 ConfigMap 完全替换 base 中的同名 ConfigMap
- `create`：如果已存在同名 ConfigMap 则报错，否则新建

获取应用最终配置的命令：

```bash
# 查看 Kustomize 生成的最终清单（包括哈希后的 ConfigMap 名称）
kubectl kustomize overlays/prod/

# 直接应用
kubectl apply -k overlays/prod/
```

Kustomize 的 configMapGenerator 不会自动触发 Deployment 的滚动更新（因为 Deployment 需要引用新 ConfigMap 名称才能触发）。解决这个问题的常见方案：

1. 在 Deployment 中使用 Kustomize 的 `nameSuffix` 或 `namePrefix` 让 Deployment 名也带哈希
2. 使用 Kustomize 的 `replacements` 特性自动更新 Deployment 中的 ConfigMap 引用
3. 配合 ArgoCD 等 GitOps 工具的自动同步和滚动更新
4. 使用 `kubectl rollout restart deployment/app` 手动触发重启

```yaml
# 自动更新 Deployment 引用的 ConfigMap 名称
# kustomization.yaml
configMapGenerator:
- name: app-config
  files:
  - config.properties

# 使用 replacements 自动更新 Deployment 中的引用
replacements:
- source:
    kind: ConfigMap
    name: app-config
    fieldPath: metadata.name
  targets:
  - select:
      kind: Deployment
      name: my-app
    fieldPaths:
    - spec.template.spec.volumes.[name=config].configMap.name
```



# 3.ConfigMap的使用方式

## 3.1 环境变量注入

环境变量注入是最简单的 ConfigMap 使用方式，适合配置项数量适中（几十个以内）且应用原生支持从环境变量读取配置的场景。Kubernetes 支持两种环境变量注入方式：全部注入和单项注入。

**全部注入 (`envFrom.configMapRef`)**

这种方式将 ConfigMap 中的所有键值对全部注入为容器的环境变量，键名直接成为环境变量名，值成为环境变量值。这是最简便的方式，无需逐个枚举配置项。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config         # 引用整个 ConfigMap
        optional: false          # 默认 false，ConfigMap 不存在则 Pod 无法启动
    - configMapRef:
        name: common-config      # 可以引用多个 ConfigMap
        optional: true           # true 表示 ConfigMap 不存在时跳过，Pod 仍可启动
```

`optional` 参数的语义：当设置为 `false`（默认值）时，如果引用的 ConfigMap 不存在，Pod 将无法启动，会一直处于 `ContainerCreating` 状态，kubelet 会不断重试。当设置为 `true` 时，即使 ConfigMap 不存在，Pod 也会正常启动（只是缺少了那些环境变量）。`optional: true` 适合以下场景：ConfigMap 由另一个控制器（如 Operator）动态创建的；可选的调试/监控配置；向后兼容的配置项。

`envFrom` 注入的环境变量命名规则：ConfigMap 的键名直接作为环境变量名。如果键名包含不符合 POSIX 环境变量命名规范的字符（如包含 `.`、`-`），它们会被保留但可能导致应用解析环境变量时出问题。最佳实践是 ConfigMap 和 Secret 的键名只使用大写字母、数字和下划线。

当多个 `envFrom` 源包含同名的键时，后出现的源会覆盖先出现的源。对于同一个 `envFrom` 中的 `configMapRef` 和 `secretRef`，secretRef 的优先级更高（因为它们后处理）。

**单项注入 (`env.valueFrom.configMapKeyRef`)**

这种方式允许你精确选择 ConfigMap 中的某个键值对，并将其映射为环境变量。环境变量名可以和 ConfigMap 中的键名不同。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # 直接映射，键名相同
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log.level        # ConfigMap 中的键名
          optional: false

    # 映射到不同的环境变量名
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: db-config
          key: jdbc.url
          optional: true

    # 组合 ConfigMap 和直接赋值
    - name: APP_NAME
      value: "my-application"
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: env-config
          key: environment
```

当 ConfigMap 中存在不符合环境变量命名规范的键时（如 `log.level` 包含 `.`），直接使用 `envFrom` 注入可能导致不可预期的行为。使用 `env.valueFrom.configMapKeyRef` 可以将任意格式的键重命名为符合规范的环境变量名。

**注入顺序和覆盖关系**

环境变量的优先级从高到低依次是：
1. 容器 `.env` 字段中的直接赋值（`value` 字段）
2. `envFrom` 中后出现的源
3. `envFrom` 中先出现的源
4. 镜像中定义的环境变量（Dockerfile 中的 `ENV` 指令）
5. 容器运行时的默认环境变量

```yaml
# 优先级示例
containers:
- name: app
  image: myapp:latest
  env:
  - name: LOG_LEVEL
    value: "ERROR"              # 最高优先级：直接赋值
  envFrom:
  - configMapRef:
      name: config-2            # 次高优先级：后出现的源
  - configMapRef:
      name: config-1            # 较低优先级：先出现的源
  # 镜像中的 ENV LOG_LEVEL=INFO 优先级最低
```

**环境变量注入的底层实现**

当 kubelet 创建容器时，它会调用容器运行时（如 containerd、CRI-O）的 `CreateContainer` 接口，传递一个容器配置结构。这个结构中包含 `Env` 字段，是一个 `key=value` 字符串数组。kubelet 会解析 Pod Spec 中的 `env` 和 `envFrom` 配置，读取对应的 ConfigMap 内容，构建这个环境变量数组，然后传给容器运行时。

整个流程：

1. kubelet 的 Pod Sync Loop 检测到需要创建新 Pod
2. kubelet 调用 `makeEnvironmentVariables` 函数，遍历容器的 `env` 和 `envFrom` 配置
3. 对于 `envFrom.configMapRef`，kubelet 通过 API Server 获取 ConfigMap 内容（如果在缓存中则直接读取）
4. 对于 `env.valueFrom.configMapKeyRef`，kubelet 获取 ConfigMap 中指定的键
5. 按优先级规则合并所有环境变量
6. 将最终的环境变量列表传递给容器运行时
7. 容器运行时在启动容器时设置这些环境变量

一旦容器启动，环境变量就固定了。后续 ConfigMap 的修改不会影响已运行容器的环境变量。这是环境变量注入方式不支持热更新的根本原因——进程的环境变量在进程启动时由操作系统内核拷贝到进程地址空间，内核没有提供修改另一进程环境变量的机制。

## 3.2 文件挂载

文件挂载是 ConfigMap 最强大和灵活的使用方式。它将 ConfigMap 中的每个键值对渲染为容器文件系统中的一个文件，文件名就是键名，文件内容就是值。这种方式支持热更新，适合配置文件、模板文件等场景。

**基本挂载方式**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config   # 挂载点
      readOnly: true               # 推荐始终设为 true
  volumes:
  - name: config-volume
    configMap:
      name: app-config             # ConfigMap 名称
      # defaultMode: 0644          # 文件权限，默认 0644
      # optional: false            # 是否可选
```

在这个例子中，`app-config` ConfigMap 中的每个键值对都会在 `/etc/app/config/` 目录下创建一个文件。例如 ConfigMap 中有键 `application.properties`，则容器内会有文件 `/etc/app/config/application.properties`。

**注意事项：挂载点覆盖**

当 ConfigMap 挂载到某个目录时，该目录下的原有内容（如果存在）会被隐藏。这与 Linux 的 mount 行为一致——mount 操作会覆盖目标目录的原有内容。如果你的应用依赖容器镜像中自带的配置文件（如 `/etc/nginx/nginx.conf`），挂载 ConfigMap 到 `/etc/nginx/` 会导致整个目录被替换，原有文件全部丢失。

解决方案有两种：

方案一：挂载到子目录，应用同时读取两个目录：

```yaml
volumeMounts:
- name: nginx-config
  mountPath: /etc/nginx/conf.d    # 挂载到子目录
  # 原有的 /etc/nginx/nginx.conf 不受影响
volumes:
- name: nginx-config
  configMap:
    name: nginx-extra-config
```

方案二：使用 `subPath` 挂载单个文件（但需注意后文提到的热更新陷阱）。

**细粒度控制：items 字段**

`items` 字段允许你精确控制哪些键被挂载以及挂载后的文件名：

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    items:
    - key: application.properties        # ConfigMap 中的键
      path: app.properties               # 挂载后的文件名
      mode: 0644                         # 文件权限（覆盖 defaultMode）
    - key: logback.xml
      path: logback.xml
      mode: 0600                         # 只允许应用用户读写
    - key: nginx.conf
      path: nginx/site.conf              # 支持子目录路径
      mode: 0444
```

如果不使用 `items`，ConfigMap 中的所有键都会被挂载。使用 `items` 后，只有列表中指定的键会被挂载，其他键被忽略。

**文件权限控制**

ConfigMap Volume 的文件权限通过两层控制：

1. `defaultMode`：Volume 级别的默认权限，作用于 ConfigMap 中的所有文件。默认值是 `0644`（所有者可读写，同组可读，其他人可读）。必须是 0 到 0777 之间的八进制数。

2. `mode`：`items` 中每个文件级别权限，覆盖 `defaultMode`。如果不指定，使用 `defaultMode` 的值。

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    defaultMode: 0440        # 所有文件默认：r--r-----
    items:
    - key: public.conf
      path: public.conf
      mode: 0444             # 单独覆盖：r--r--r--
    - key: secret.conf
      path: secret.conf
      mode: 0400             # 单独覆盖：r--------
```

注意：ConfigMap 挂载的文件，其所有者和组始终是容器运行用户（由 Dockerfile 的 `USER` 指令或 Pod 的 `securityContext.runAsUser` 决定）。`mode` 和 `defaultMode` 只控制权限位，不控制所有者。

**subPath 挂载（及其陷阱）**

`subPath` 允许将 ConfigMap 中的单个文件挂载到容器中的指定路径，而不是挂载整个 Volume：

```yaml
containers:
- name: app
  image: nginx:latest
  volumeMounts:
  - name: config-volume
    mountPath: /etc/nginx/nginx.conf    # 目标文件路径
    subPath: nginx.conf                 # ConfigMap 中的键名
volumes:
- name: config-volume
  configMap:
    name: nginx-config
```

`subPath` 的好处是不会覆盖目标目录中的其他文件。但 `subPath` 有一个重要的陷阱：**使用 subPath 挂载的文件不支持热更新**。这是因为 subPath 挂载使用的是 bind mount 机制，kubelet 直接将文件绑定到目标路径，而不是通过 symlink 机制。当 ConfigMap 更新时，kubelet 不会重新绑定 subPath 挂载的文件。

此外，使用 subPath 挂载 ConfigMap 时，如果 ConfigMap 中的键在 Pod 创建时不存在，Pod 将无法启动。这是因为 subPath 需要在容器创建时就确定挂载源。

## 3.3 命令行参数引用

Kubernetes 支持在容器的 `command` 和 `args` 字段中引用环境变量，使用 `$(VAR_NAME)` 语法。这允许你将 ConfigMap 的值注入到容器的启动命令中：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: command-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    command: ["/bin/sh", "-c"]
    args:
    - |
      echo "Starting app with following config:"
      echo "Database Host: $(DB_HOST)"
      echo "Log Level: $(LOG_LEVEL)"
      ./start-app.sh --db-host=$(DB_HOST) --log-level=$(LOG_LEVEL)
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: db-config
          key: host
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log.level
```

`$(VAR_NAME)` 语法的规则：
- 只能引用同一个容器中定义的 `env` 环境变量（包括 `envFrom` 注入的）
- 不能引用其他容器的环境变量
- 不能引用容器内的文件系统中的变量
- 如果变量未定义，`$(VAR_NAME)` 会保持字面值，不会展开
- 使用 `$$` 转义 `$` 字面值

```yaml
# 转义示例
args:
- "echo $$(VAR_NAME)"  # 输出 $(VAR_NAME) 字面值，不进行展开
```

这个功能在以下场景特别有用：
- 需要在启动时动态指定 JVM 参数（如 `-Xmx$(MEMORY_LIMIT)`）
- 需要将配置通过命令行参数传递给不支持环境变量的程序
- 需要在 Init Container 或 Sidecar 中使用不同配置

## 3.4 三种方式的使用场景对比

| 维度 | 环境变量注入 | 文件挂载 | 命令行参数 |
|------|-------------|---------|-----------|
| 热更新 | 不支持 | 支持（subPath 除外） | 不支持 |
| 大型配置 | 不适合（环境变量有大小限制） | 适合（支持大文件） | 不适合 |
| 结构化配置 | 不支持（扁平的 key=value） | 支持（YAML/JSON/XML 文件） | 部分支持 |
| 配置数量 | 适合几十个配置项 | 适合任意数量 | 适合少量配置项 |
| 应用改动 | 最小（读环境变量即可） | 需要读文件 | 需要命令行解析 |
| 安全性 | 环境变量可被其他进程读取 | 文件权限可控 | 命令行可被他人看到 |
| 容器镜像影响 | 无 | 需注意挂载点覆盖 | 无 |

**选择决策树**

1. 应用是否已经支持从文件读取配置（如 Spring Boot 的 `application.yml`、Nginx 的 `nginx.conf`）？ → 使用文件挂载
2. 配置项是否需要在运行时动态变更而不重启 Pod？ → 使用文件挂载（配合应用的热加载机制）
3. 配置是否只是几个简单的键值对（如 `LOG_LEVEL`、`ENV`）？ → 使用环境变量注入
4. 应用是否无法修改代码，只能通过命令行参数接收配置？ → 使用命令行参数引用
5. 应用是否遵循 Twelve-Factor App，天然从环境变量读取配置？ → 使用环境变量注入

实际项目中，三种方式经常组合使用：

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    command: ["java"]
    args:
    - "-Xmx$(JAVA_XMX)"
    - "-jar"
    - "/app/myapp.jar"
    - "--spring.config.location=/etc/app/config/"
    env:
    - name: JAVA_XMX          # 命令行参数中用到的
      valueFrom:
        configMapKeyRef:
          name: jvm-config
          key: heap.size
    envFrom:
    - configMapRef:            # 简单环境变量
        name: common-env
    volumeMounts:
    - name: app-config         # 结构化配置文件
      mountPath: /etc/app/config
      readOnly: true
  volumes:
  - name: app-config
    configMap:
      name: app-files-config
```



# 4.ConfigMap热更新

## 4.1 文件挂载方式的热更新

ConfigMap 以 Volume 形式挂载时，支持自动热更新。这是 ConfigMap 最具价值的特性之一，但它的底层机制经常被误解。

**kubelet 的同步机制**

kubelet 是每个 Node 上的核心代理进程，负责维护本节点上 Pod 的状态。kubelet 内部有一个 ConfigMap Manager，它会 watch 本节点上所有 Pod 引用的 ConfigMap。当 kubelet 从 API Server 收到 ConfigMap 变更事件时（通过 etcd watch 机制），ConfigMap Manager 会将新内容写入到节点本地文件系统中。

同步频率由两个因素控制：
1. kubelet 的 `--sync-frequency` 参数（默认 1 分钟），控制 kubelet 与 API Server 定期同步的间隔
2. API Server 的 watch 机制，ConfigMap 变更会通过 watch 连接实时推送给 kubelet

实际上，ConfigMap 变更通常会在 1 分钟内反映到 Pod 的文件系统中。但官方文档明确说明这不是实时同步，TTL（Time To Live）由 kubelet 的 sync frequency 决定，默认约 1 分钟。因此你不应该期望 ConfigMap 更新后 Pod 能立即看到变化。

**symlink（符号链接）更新机制**

这是 ConfigMap 热更新实现的关键技术，理解它对排查更新问题非常有帮助。

当 kubelet 为 Pod 挂载 ConfigMap Volume 时，实际的目录结构是这样的：

```
容器内的挂载点 /etc/app/config/
      ↓ (挂载传播)
节点上的 Volume 路径 /var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~configmap/<volume-name>/
      ↓
实际的 ConfigMap 数据存储在：
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~configmap/<volume-name>/..data/
      ↑ 这是一个 symlink，指向当前版本的数据目录
```

具体步骤：
1. kubelet 为每次 ConfigMap 更新创建一个新的时间戳目录，如 `..2023_10_15_12_30_45.123456789`
2. 将新版本的 ConfigMap 数据写入这个新目录（每个键写入对应的文件）
3. 原子性地将 `..data` symlink 从旧目录切换到新目录
4. kubelet 清理旧的版本目录（通常保留最近两个版本）

```bash
# 进入容器查看实际文件结构
kubectl exec -it my-pod -- ls -la /etc/app/config/

# 你会看到类似这样的输出：
# drwxr-xr-x  3 root root  120 Oct 15 12:30 .
# drwxr-xr-x  1 root root   20 Oct 15 12:00 ..
# drwxr-xr-x  2 root root   80 Oct 15 12:30 ..2023_10_15_12_30_45.123456789
# lrwxrwxrwx  1 root root   31 Oct 15 12:30 ..data -> ..2023_10_15_12_30_45.123456789
# lrwxrwxrwx  1 root root   18 Oct 15 12:30 app.properties -> ..data/app.properties
# lrwxrwxrwx  1 root root   16 Oct 15 12:30 logback.xml -> ..data/logback.xml
```

每次文件都是以 symlink 形式存在的，指向 `..data/` 下的同名文件。而 `..data` 本身是一个 symlink，指向当前版本的时间戳目录。

当 ConfigMap 更新时：
1. kubelet 创建新的时间戳目录并写入新内容
2. 原子性地将 `..data` symlink 切换到新目录
3. 由于 `..data` 的 inode 不变（只是目标变了），已打开文件的进程不会受到影响（继续读取旧版本）
4. 进程需要重新打开文件（`open()` 系统调用）才能读取到新内容

**为什么需要应用配合**

symlink 机制确保了文件系统的原子更新，但应用程序不一定会自动感知文件变化并重新加载配置。Linux 内核不会主动通知进程"你打开的文件内容变了"。因此，应用程序需要实现配置刷新机制，常见的做法包括：

1. **轮询检查**：应用定期（如每 10 秒）检查配置文件的修改时间（mtime）或内容哈希，发现变化后重新加载
2. **文件系统事件**：使用 inotify（Linux）或类似机制 watch 配置文件的变化事件，收到事件后重新加载
3. **信号触发**：应用提供一个 HTTP 端点或信号处理，由外部通知触发配置重载（如 `kill -HUP <pid>` 触发 nginx reload）
4. **框架支持**：Spring Cloud Kubernetes 自动 watch ConfigMap 变化并刷新 `@ConfigurationProperties`

不依赖应用改动就能实现热更新的常见模式是使用 Sidecar 容器监控配置变化并触发主容器重载：

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
  - name: config-reloader
    image: nginx:alpine
    command:
    - /bin/sh
    - -c
    - |
      while true; do
        inotifywait -e modify /etc/nginx/conf.d/
        nginx -s reload
      done
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
  volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
```

## 4.2 环境变量方式不支持热更新

通过环境变量注入的配置在 Pod 创建时就已经固定，后续修改 ConfigMap 不会影响已运行容器的环境变量值。这是由操作系统层面的限制决定的。

**为什么不能动态更新环境变量**

从操作系统角度，进程的环境变量是在进程创建时由内核从父进程（容器运行时）复制到子进程的地址空间的。一旦进程开始运行，它的环境变量存储在其地址空间的栈上方（`/proc/<pid>/environ`），没有任何系统调用可以安全地修改另一个进程的环境变量。

即使理论上可以通过 `ptrace` 等调试接口修改目标进程的地址空间（实际上 Kubernetes 也不支持这种操作），这样做也是极度危险的——应用程序可能已经缓存了环境变量的值（比如在 `static` 变量中），修改原始环境变量可能导致应用状态不一致。

**解决方案**

需要动态变更配置且使用了环境变量注入的 Pod，唯一的办法是重启 Pod：

```bash
# 手动重启 Deployment
kubectl rollout restart deployment/my-app

# 或删除 Pod 让 ReplicaSet 重建
kubectl delete pod -l app=my-app

# 或修改一个无关的 annotation 触发滚动更新
kubectl patch deployment my-app -p \
  '{"spec":{"template":{"metadata":{"annotations":{"restartedAt":"'$(Get-Date -Format o)'"}}}}}'
```

## 4.3 应用的配置reload机制

生产环境中实现对 ConfigMap 变更的自动感知和配置重载，有多种方案可选。

**Spring Cloud Kubernetes**

对于 Java 生态（Spring Boot）应用，Spring Cloud Kubernetes 是最原生的方案。它通过 fabric8 Kubernetes Client watch ConfigMap 的变化，当检测到变更时，触发 Spring 的 `RefreshScope` 或 `@RefreshScope` 注解的 Bean 重新初始化。

```xml
<!-- pom.xml 依赖 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yaml
spring:
  cloud:
    kubernetes:
      config:
        sources:
        - name: my-app-config
          namespace: default
      reload:
        enabled: true
        monitoring-config-maps: true
        strategy: refresh           # refresh / restart_context / shutdown
        mode: event                 # event（事件驱动）/ polling（轮询）
        period: 5000                # 轮询模式下的检测间隔（毫秒）
```

重载策略（`strategy`）说明：
- `refresh`：仅刷新 `@ConfigurationProperties` 和 `@RefreshScope` 的 Bean，最温和
- `restart_context`：重启 Spring ApplicationContext，所有 Bean 重新初始化
- `shutdown`：关闭应用容器，依赖 Kubernetes 的自动重启机制恢复

Spring Cloud Kubernetes 的 watch 机制：它使用 Kubernetes API Watch 接口，建立一个长连接监听 ConfigMap 的变更事件。当有 MODIFIED 事件时，触发配置刷新。与 polling（定期轮询 API Server）相比，事件驱动的延迟更低，对 API Server 的压力也更小。

**confd**

confd 是一个轻量级的配置管理守护进程，广泛用于非 Java 生态。它支持从 etcd、Consul、Vault、环境变量、文件等后端读取配置，使用 Go 模板语法渲染配置文件，并在配置变化时执行重载命令。

```bash
# confd 配置文件
# /etc/confd/conf.d/nginx.toml
[template]
src = "nginx.conf.tmpl"       # 模板文件
dest = "/etc/nginx/nginx.conf" # 目标配置文件
keys = [
    "/config/nginx/worker_processes",
    "/config/nginx/upstream_host",
]
check_cmd = "nginx -t"        # 重载前检查配置语法
reload_cmd = "nginx -s reload" # 重载命令
```

在 Kubernetes 中，confd 通常以 Sidecar 或 Init Container 形式运行：

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
  - name: confd
    image: confd:latest
    env:
    - name: CONFD_BACKEND
      value: "file"
    - name: CONFD_FILE_DIR
      value: "/etc/confd/templates"
    volumeMounts:
    - name: nginx-config
      mountPath: /etc/nginx/conf.d
    - name: confd-config
      mountPath: /etc/confd
  volumes:
  - name: nginx-config
    emptyDir: {}
  - name: confd-config
    configMap:
      name: confd-config
```

**自定义 Sidecar / Init Container 模式**

对于更定制化的需求，可以编写简单的 Sidecar 脚本或使用专门的 ConfigMap Reloader 工具：

```yaml
# 使用 stakater/Reloader 自动检测 ConfigMap 变化并触发滚动更新
# 这是一个 Kubernetes Operator，无需改动应用代码

# 安装 Reloader
# kubectl apply -f https://raw.githubusercontent.com/stakater/Reloader/master/deployments/kubernetes/reloader.yaml

# 在 Deployment 中使用 annotation 指定要 watch 的 ConfigMap
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  annotations:
    configmap.reloader.stakater.com/reload: "app-config"
    secret.reloader.stakater.com/reload: "db-secret"
spec:
  template:
    metadata:
      annotations:
        configmap.reloader.stakater.com/reload: "app-config"
    spec:
      containers:
      - name: app
        image: myapp:latest
```

Reloader 的工作方式：它 watch 指定的 ConfigMap/Secret 变化，当检测到变更时，自动对关联的 Deployment/DaemonSet/StatefulSet 执行滚动更新。这与热更新不同——它会重启 Pod，但优点是应用无需任何代码改动。

## 4.4 subPath挂载不更新的陷阱

使用 `subPath` 挂载单个 ConfigMap 文件时，该文件不支持热更新。这是 Kubernetes 用户最常遇到的问题之一。

**原因分析**

`subPath` 挂载的实现与普通 Volume 挂载不同：

- 普通 Volume 挂载：kubelet 在节点的 Volume 目录中创建 symlink 结构，通过原子切换 `..data` symlink 实现更新。容器内的应用通过重新打开文件获取新内容。

- `subPath` 挂载：kubelet 使用 bind mount 将 ConfigMap 中的单个文件直接绑定到容器的目标路径。bind mount 在容器创建时建立，将一个文件或目录挂载到另一个位置。一旦建立，bind mount 的源和目标关系就固定了。

当 ConfigMap 更新时，kubelet 创建新的版本目录并更新了 `..data` symlink，但容器内的 bind mount 仍然指向旧文件（bind mount 是在 mount namespace 级别建立的，不经过 symlink 解析）。因此容器内的文件内容不会变化。

**这个行为不是 bug**

Kubernetes 官方将这种行为明确标记为"不支持 subPath 热更新"而非 bug。原因在于：
1. 如果不重新创建 bind mount，无法安全地让容器看到新文件（需要考虑文件锁、正在读写等并发问题）
2. 重建 bind mount 意味着需要重启容器（至少需要重新执行 mount 系统调用）
3. 如果允许自动重建 bind mount，可能导致数据不一致或其他不可预期的行为

**解决方案**

1. 不使用 subPath，改为挂载整个 Volume 到子目录：

```yaml
# ❌ 使用 subPath（不支持热更新）
volumeMounts:
- name: config
  mountPath: /etc/nginx/nginx.conf
  subPath: nginx.conf

# ✅ 挂载到子目录（支持热更新）
volumeMounts:
- name: config
  mountPath: /etc/nginx/conf.d
volumes:
- name: config
  configMap:
    name: nginx-config
    items:
    - key: nginx.conf
      path: site.conf   # 改个名字，让 nginx include 这个文件
```

2. 使用 Init Container 将文件复制到 emptyDir Volume：

```yaml
spec:
  initContainers:
  - name: copy-config
    image: busybox
    command: ['sh', '-c', 'cp /tmp/config/nginx.conf /etc/nginx/nginx.conf']
    volumeMounts:
    - name: config-source
      mountPath: /tmp/config
    - name: config-dest
      mountPath: /etc/nginx
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: config-dest
      mountPath: /etc/nginx
  volumes:
  - name: config-source
    configMap:
      name: nginx-config
  - name: config-dest
    emptyDir: {}    # 注意：emptyDir 的内容不会被自动更新
```

3. 使用专门的配置重载工具（如 Reloader）自动重启 Pod。

# 5.不可变ConfigMap (Immutable ConfigMap)

## 5.1 为什么需要不可变性

从 Kubernetes 1.21 开始，ConfigMap 和 Secret 支持 `immutable` 字段，设置为 `true` 后，对象内容不可修改（只能删除重建）。这个特性解决了一个长久存在的性能和运维问题。

**kubelet watch 压力问题**

在大型 Kubernetes 集群中（数百个 Node、数千个 Pod），kubelet 需要对每个 Node 上所有 Pod 引用的 ConfigMap 和 Secret 建立 watch。每当这些对象发生变化时，kubelet 需要更新节点上的缓存，并将新内容同步到对应 Pod 的文件系统中。

如果一个 ConfigMap 被设置为不可变，kubelet 知道它永远不会变化，因此不需要为其建立 watch。这会显著减少 kubelet 与 API Server 之间的 watch 连接数，降低 API Server 的内存和 CPU 消耗。

**性能数据**

根据 Kubernetes 社区的测试，在 5000 个 Node 的集群中，将所有 ConfigMap 设为不可变可以：
- 减少 API Server 约 40% 的 watch 连接数
- 减少 kubelet 约 30% 的内存使用
- 降低 etcd 的 I/O 压力（因为不需要处理 ConfigMap 更新）

对于小集群（几十个 Node），这个优化几乎感觉不到。但对于大规模集群，这是非常重要的性能优化手段。

**防止意外修改**

从运维角度，将 ConfigMap 标记为不可变也是一种安全措施——防止未经授权或无意的配置变更。在 GitOps 工作流中，配置变更应该通过 Git 提交和 PR 审核，而非直接 `kubectl edit`。不可变 ConfigMap 强制了这种实践——修改配置必须创建新的 ConfigMap 并更新 Deployment 引用，这个过程中的每一步都可以被审计和回滚。

**滚动更新的触发**

当 ConfigMap 内容变化时，Pod 不会自动重启（除非 Deployment 的 `spec.template` 也变化）。这导致了一种常见的运维失误：运维人员修改了 ConfigMap，以为配置已生效，但实际上旧的 Pod 仍在使用旧配置。

使用不可变 ConfigMap + 名称带版本号（或哈希）的模式，每次配置变更都需要创建一个新的 ConfigMap 并更新 Deployment 的引用。Deployment 的 `spec.template` 变化自动触发滚动更新，确保新配置被所有 Pod 加载。

## 5.2 immutable字段的使用

设置 ConfigMap 为不可变非常简单，在 metadata 同级添加 `immutable: true`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
  namespace: default
immutable: true
data:
  app.name: my-application
  app.env: production
  log.level: info
  db.host: mysql.prod.svc.cluster.local
```

一旦创建，任何修改 `data` 字段的尝试都会失败：

```bash
# 尝试修改不可变 ConfigMap
kubectl edit configmap app-config-v1
# 保存时会收到错误：
# "configmap 'app-config-v1' is immutable and cannot be updated"
# "immutable configmaps cannot be modified"

# 直接 patch 也会失败
kubectl patch configmap app-config-v1 -p '{"data":{"log.level":"debug"}}'
# Error from server: configmap "app-config-v1" is immutable

# 唯一的选择是删除重建
kubectl delete configmap app-config-v1
kubectl create configmap app-config-v2 --from-literal=log.level=debug
```

注意事项：
- `immutable` 只能在创建时设置，不能事后修改（将一个已存在的可变 ConfigMap 改为不可变是不允许的）
- 不可变的 ConfigMap 仍然可以被删除
- `immutable` 作用于整个 ConfigMap 对象，不能按键设置
- `binaryData` 中的内容也同样不可变

**与 Kustomize configMapGenerator 的集成**

结合 Kustomize 的 configMapGenerator，每次内容变化自动生成带哈希后缀的新名称，天然适配不可变模式：

```yaml
# kustomization.yaml
configMapGenerator:
- name: app-config
  options:
    disableNameSuffixHash: false   # 默认 true（从 1.23+ 开始默认启用哈希）
    immutable: true                # 生成不可变 ConfigMap
  files:
  - application.properties
```

## 5.3 不可变ConfigMap的更新策略

由于不可变 ConfigMap 无法直接修改，需要采用"改名+更新引用"的策略。

**策略一：版本号命名**

```yaml
# 创建 ConfigMap 时使用带版本号的名字
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
immutable: true
data:
  log.level: "info"
  max.connections: "100"
---
# 需要更新时，创建新版本
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
immutable: true
data:
  log.level: "debug"          # 修改了
  max.connections: "200"      # 修改了
---
# 更新 Deployment 引用
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        volumeMounts:
        - name: config
          mountPath: /etc/app/config
      volumes:
      - name: config
        configMap:
          name: app-config-v2   # 从 v1 更新到 v2
```

版本号命名的优点是语义清晰，人为可理解当前使用的是哪个版本。缺点是需要手动管理版本号，不适合自动化。

**策略二：内容哈希命名**

使用 Kustomize 的 configMapGenerator，它会自动根据内容计算哈希值并附加到名称后：

```bash
# kustomization.yaml
configMapGenerator:
- name: app-config
  files:
  - application.properties

# 执行 kustomize build 查看生成的结果
kubectl kustomize .
# 生成的 ConfigMap 名称类似：app-config-7g4k2m
# 当 application.properties 内容变化，哈希值也变化，名字自动更新
```

在 Deployment 中引用 ConfigMap 时，不能写死名称，需要通过 Kustomize 的变量替换：

```yaml
# deployment.yaml
volumes:
- name: config
  configMap:
    name: app-config  # Kustomize 会自动替换为带哈希的名称
```

**策略三：使用 Operator/Controller 自动管理**

对于复杂的更新场景，可以编写自定义 Operator 监听 ConfigMap 的变更事件并自动更新所有引用它的 Deployment。社区中已有现成的 Controller 如 stakater/Reloader 可以完成这个工作。



# 6.Secret概述

## 6.1 为什么需要Secret

Secret 是 Kubernetes 中专用于存储和管理敏感信息的 API 对象。在企业级应用部署中，敏感信息的保护是安全体系的基石。这些敏感信息包括但不限于：

**数据库凭证**：生产数据库的用户名和密码是应用访问数据的核心凭证。一旦泄露，攻击者可以绕过应用层直接访问、篡改或删除数据。在微服务架构中，每个服务可能拥有独立的数据库凭证，管理数十甚至数百组凭证。

**API 密钥和 Token**：第三方服务（如支付网关、短信服务、地图服务）的 API 密钥，云服务商的 Access Key/Secret Key，内部的 JWT 签名密钥等。这些密钥一旦泄露，轻则造成服务滥用和账单飙升，重则导致用户数据泄露和合规违规。

**TLS/SSL 证书和私钥**：HTTPS 通信依赖 TLS 证书。如果私钥泄露，攻击者可以实施中间人攻击，解密通信内容；或者伪造合法的服务，进行钓鱼攻击。

**镜像仓库认证信息**：私有镜像仓库（Harbor、ACR、ECR、GCR 等）的 pull 凭证。如果泄露，攻击者可以拉取你的私有镜像进行逆向分析或二次利用。

**SSH 密钥**：用于 Git 仓库访问、服务器远程管理、自动化部署的 SSH 私钥。

为什么不用 ConfigMap 存储敏感信息？因为 ConfigMap 的明文存储特性使其完全不适合存储敏感数据：
- ConfigMap 的数据存储在 etcd 中默认明文（不加密）
- ConfigMap 通常有更宽松的 RBAC 权限设置（开发者可以查看以排查问题）
- ConfigMap 的内容可能被输出到日志、终端、监控系统、审计系统
- ConfigMap 的设计目标就是"非敏感数据的配置管理"

Secret 提供了额外的保护层：
- 独立于 ConfigMap 的 RBAC 权限模型，默认只有管理员可以访问 Secret
- etcd 静态加密支持，Secret 数据在 etcd 存储时可以加密
- Secret 在使用时才会被传递到节点，且存储在 tmpfs（内存文件系统）而非磁盘
- 部分 Kubernetes 发行版（如 OpenShift、GKE）提供了额外的 Secret 加密层

## 6.2 Secret vs ConfigMap

| 特性 | ConfigMap | Secret |
|------|-----------|--------|
| 存储内容 | 非敏感配置数据 | 敏感数据（密码、密钥、证书等） |
| 数据编码 | 明文（YAML 中直接可读） | base64 编码（`data` 字段）/ 明文（`stringData` 字段） |
| 默认存储 | etcd 明文 | etcd 明文（可配置静态加密） |
| RBAC | 通常授予开发者 list/get | 通常限制为管理员，或按需最小授权 |
| 节点存储 | 磁盘（kubelet 目录） | tmpfs 内存文件系统（1.9+） |
| API 响应 | 完整内容返回 | 完整内容返回（corev1 无自动隐藏机制） |
| Pod 内访问方式 | 环境变量 / 文件挂载 | 同 ConfigMap（但建议文件挂载优先） |
| kubectl describe | 显示完整数据 | 不显示 `data` 内容（仅显示键名） |
| 加密支持 | 无原生加密 | etcd 静态加密 / 第三方工具集成 |

**kubectl describe 的保护机制**

`kubectl describe configmap` 会显示 ConfigMap 的完整数据内容，而 `kubectl describe secret` 出于安全考虑不会显示 `data` 字段的内容（只显示键名和大小）。这是 kubectl 客户端的保护措施，但需要注意：`kubectl get secret -o yaml` 仍然会显示完整的 base64 编码数据。

**使用建议**

作为开发者或运维人员，一个简单的决策规则是：
- 如果这个配置项的值在团队内部应该是公开的（如端口号、数据库主机名、日志级别），放在 ConfigMap 中
- 如果这个配置项的值应该只有特定人员知道（如密码、密钥、Token），放在 Secret 中
- 如果自己无法判断，默认放 Secret 中——安全性宁可过度也不要不足

## 6.3 Secret的安全边界

理解 Secret 的安全边界，是正确使用 Secret 的前提。Secret 提供的是一种"纵深防御"（Defense in Depth）的策略，而非绝对安全。

**base64 编码不是加密**

这是 Kubernetes 新手最容易产生的误解。Secret 的 `data` 字段使用 base64 编码存储数据，但 base64 是一种编码算法（encoding），不是加密算法（encryption）。编码的目的是将二进制数据转换为可打印的 ASCII 字符，方便传输和存储，但任何人都可以轻松地将 base64 解码回原始数据。

```bash
# "加密" - 实际上只是编码
echo -n "MyPassword123" | base64
# 输出：TXlQYXNzd29yZDEyMw==

# "解密" - 任何人都能轻易解码
echo "TXlQYXNzd29yZDEyMw==" | base64 -d
# 输出：MyPassword123
```

base64 编码在 Secret 中的作用：
1. 允许二进制数据（如证书文件、密钥文件）以文本形式存储在 YAML 中
2. 防止在 YAML 输出中因为特殊字符（如换行符、引号）导致的格式问题
3. 提供微弱的"防窥屏"保护（经过编码的字符串肉眼不可读原始内容，但这绝不能视为安全措施）

`stringData` 字段允许你以明文编写 Secret 数据，kubectl 在提交给 API Server 时会自动进行 base64 编码：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:            # 明文编写
  username: admin
  password: MySecret123
# 提交后自动编码为：
# data:
#   username: YWRtaW4=
#   password: TXlTZWNyZXQxMjM=
```

**etcd 静态加密**

Secret 数据默认以明文形式存储在 etcd 中。对于生产环境，强烈建议启用 etcd 静态加密（Encryption at Rest）。这是通过 API Server 的 `--encryption-provider-config` 参数配置的，将在第 9 章详细讨论。

**RBAC 访问控制**

Secret 的安全很大程度上依赖 RBAC。一个基本原则是：不要给任何人或服务账户授予超出其需要范围的 Secret 访问权限。

```yaml
# 最小权限原则：只授予特定 Namespace 下特定 Secret 的读取权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: my-app
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-secret", "api-key-secret"]  # 只能访问指定的 Secret
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: my-app
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: my-app
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**节点上的安全**

当 Secret 被挂载到 Pod 时，kubelet 会将 Secret 数据写入 tmpfs（基于内存的临时文件系统），而不是磁盘。这意味着：
- Secret 数据不会写入节点的磁盘持久层
- 节点重启后，tmpfs 中的数据自动消失
- 恶意用户无法通过分析磁盘块来恢复 Secret 数据

但需要注意：具有 root 权限访问节点的用户仍然可以从 `/proc/<pid>/environ` 或容器的 tmpfs 挂载点读取 Secret 数据。

**API 层的暴露风险**

获得 Pod/exec 权限的攻击者可以直接进入容器读取挂载的 Secret 文件。获得 Pod 创建权限的攻击者可以创建一个 Pod 挂载目标 Secret 并提取数据。因此，集群级别的 RBAC 管理是 Secret 安全的基础。

```bash
# 获得容器 shell 权限的用户可以读取 Secret 文件
kubectl exec -it my-pod -- cat /etc/secrets/db-password

# 有权限创建 Pod 的用户可以构造 Pod 窃取 Secret
# 这就是为什么 Pod 创建权限需要严格控制
```

# 7.Secret类型详解

## 7.1 Opaque (通用Secret)

`Opaque` 是 Secret 的默认类型，也是最灵活的类型。如果不指定 `type` 字段，Secret 默认为 `Opaque`。`Opaque` 类型的 Secret 可以存储任意键值对，没有预定义的数据结构要求。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque   # 可省略，默认为 Opaque
data:
  username: YWRtaW4=           # base64("admin")
  password: cGFzc3dvcmQxMjM=   # base64("password123")
  connection-string: amRiYzpteXNxbDovL2RiLmhvc3Q6MzMwNi9teWRi  # 任意数据
stringData:   # 与 data 同时存在时，stringData 会被合并并编码
  api-key: sk-abcdef123456     # 明文书写，自动编码
```

`Opaque` 的命名来自于其语义：对 Kubernetes 系统来说，这种 Secret 的数据内容是不透明的（opaque）——系统不理解也不关心其中的数据含义，只负责存储和传递。

`Opaque` Secret 适合的场景包括：
- 数据库用户名和密码
- 第三方 API 密钥
- 应用内部的加密密钥
- 任意需要保护的键值对信息

## 7.2 kubernetes.io/service-account-token

这是 Kubernetes 自动为每个 ServiceAccount 创建的 Secret 类型（在 Kubernetes 1.24 之前）。从 1.24 开始，ServiceAccount Token 的创建机制发生了重大变化——不再自动创建长期有效的 Secret，而是通过 TokenRequest API 动态生成有时效性的 Token（Bound ServiceAccount Token）。

**传统模式（1.23 及之前）**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: default-token-xxxxx
  namespace: default
  annotations:
    kubernetes.io/service-account.name: "default"
    kubernetes.io/service-account.uid: "xxxxx-xxxxx-xxxxx"
type: kubernetes.io/service-account-token
data:
  token: ZXlKaGJHY2lPaUpTV...     # JWT Token（base64 编码）
  ca.crt: LS0tLS1CRUdJTi...          # 集群 CA 证书（base64 编码）
  namespace: ZGVmYXVsdA==             # Namespace 名称
```

传统 ServiceAccount Token 的特点：
- 是一个没有过期时间的 JWT Token（长期有效）
- 自动挂载到使用该 ServiceAccount 的每个 Pod 的 `/var/run/secrets/kubernetes.io/serviceaccount/` 目录
- 包含三个文件：`token`（JWT）、`ca.crt`（CA 证书）、`namespace`（命名空间）
- 一旦签发，除非删除 Secret，Token 永久有效

**Bound ServiceAccount Token（1.22+ 推荐）**

从 Kubernetes 1.22 开始，Bound ServiceAccount Token 成为 GA。它通过 `TokenRequest` API 动态生成，具有以下特性：
- 有时间限制（默认 1 小时，可配置）
- 绑定到特定的 Pod（audience 和 bound object 机制）
- 不再自动创建 Secret 对象，而是在 Pod 创建时通过 Projected Volume 注入
- 支持 Token 轮换，kubelet 会自动刷新即将过期的 Token

```yaml
# Pod 中配置 Bound ServiceAccount Token
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: token
      mountPath: /var/run/secrets/tokens
  volumes:
  - name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600     # 1小时过期
          audience: "my-audience"     # 指定 audience
```

## 7.3 kubernetes.io/dockerconfigjson

用于存储私有 Docker 镜像仓库的认证信息，替代了旧版本的 `kubernetes.io/dockercfg` 类型。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwczovL...  # base64 编码的 Docker config.json
```

Docker config.json 的原始格式：

```json
{
  "auths": {
    "https://index.docker.io/v1/": {
      "username": "myuser",
      "password": "mypassword",
      "email": "myuser@example.com",
      "auth": "bXl1c2VyOm15cGFzc3dvcmQ="
    },
    "harbor.internal.com": {
      "username": "admin",
      "password": "Harbor12345",
      "auth": "YWRtaW46SGFyYm9yMTIzNDU="
    }
  }
}
```

创建方式：

```bash
# 使用 kubectl 创建（推荐，自动处理 base64 转换）
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com

# 从已有的 Docker 配置文件创建
kubectl create secret generic regcred \
  --from-file=.dockerconfigjson=$env:USERPROFILE\.docker\config.json \
  --type=kubernetes.io/dockerconfigjson
```

使用方式——在 Pod 中通过 `imagePullSecrets` 引用：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
  - name: app
    image: harbor.internal.com/myapp/backend:v1.0
  imagePullSecrets:
  - name: regcred        # 引用 dockerconfigjson Secret
```

也可以将 `imagePullSecrets` 添加到 ServiceAccount 中，该 SA 创建的所有 Pod 自动获得 pull 权限：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: default
imagePullSecrets:
- name: regcred
```

**跨 Namespace 使用的问题**

`imagePullSecrets` 是 Namespace 级别资源，不能跨 Namespace 引用。如果需要在多个 Namespace 中使用同一个镜像仓库凭证，需要手动在每个 Namespace 创建同名的 Secret。这个问题在社区中讨论了很多年，至今没有原生的跨 Namespace 传播机制。

解决方案：
1. 使用脚本批量复制 Secret 到多个 Namespace
2. 使用 Kubernetes Operator（如 `kubernetes-replicator`）自动同步 Secret 到指定 Namespace
3. 使用 Kyverno/OPA 等策略引擎自动创建

## 7.4 kubernetes.io/tls

用于存储 TLS 证书和私钥。这是最常用的 Secret 类型之一，Ingress Controller（如 Nginx Ingress Controller）默认使用这种类型的 Secret 来提供 HTTPS 服务。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0...  # base64 编码的证书（PEM 格式）
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVk...  # base64 编码的私钥（PEM 格式）
```

创建方式：

```bash
# 方式一：从证书文件创建
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem

# 方式二：YAML 手动定义
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
type: kubernetes.io/tls
stringData:
  tls.crt: |
    -----BEGIN CERTIFICATE-----
    MIIDRTCCAi2gAwIBAgIJAJ...
    -----END CERTIFICATE-----
  tls.key: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEpAIBAAKCAQEA...
    -----END RSA PRIVATE KEY-----
```

注意事项：
- `tls.crt` 和 `tls.key` 这两个键名是固定的，不能自定义
- 证书格式必须是 PEM（Base64 编码的 DER，带有 `-----BEGIN CERTIFICATE-----` 头尾标记）
- 私钥不能有密码保护（Passwordless），因为 Nginx Ingress Controller 等工具不支持加密私钥
- 证书应在 Secret 创建前就准备好，建议通过 cert-manager 自动管理

**在 Ingress 中使用**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-tls-secret    # 引用 TLS Secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

## 7.5 kubernetes.io/ssh-auth

用于存储 SSH 私钥，常用于 Git 仓库访问、GitOps 工具（如 ArgoCD、Flux）的仓库认证等场景。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ssh-key-secret
type: kubernetes.io/ssh-auth
data:
  ssh-privatekey: LS0tLS1CRUdJTiBPUEV...  # base64 编码的 SSH 私钥
```

创建方式：

```bash
# 从 SSH 私钥文件创建
kubectl create secret generic ssh-key-secret \
  --from-file=ssh-privatekey=$env:USERPROFILE\.ssh\id_rsa \
  --type=kubernetes.io/ssh-auth
```

注意事项：
- 键名必须是 `ssh-privatekey`（固定值）
- 私钥文件必须是 PEM 格式（`-----BEGIN OPENSSH PRIVATE KEY-----` 或 `-----BEGIN RSA PRIVATE KEY-----`）
- 不推荐存储密码保护的私钥（大部分工具不支持交互式密码输入）

## 7.6 kubernetes.io/basic-auth

用于存储 HTTP Basic Authentication 的用户名和密码。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth-secret
type: kubernetes.io/basic-auth
data:
  username: YWRtaW4=       # base64("admin")
  password: cGFzc3dvcmQ=   # base64("password")
```

创建方式：

```bash
kubectl create secret generic basic-auth-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123 \
  --type=kubernetes.io/basic-auth
```

这种类型的 Secret 常用于 Nginx Ingress 的 Basic Authentication 功能。

## 7.7 bootstrap.kubernetes.io/token

Bootstrap Token 是一种特殊的 Secret，用于新节点加入集群时的认证。它由 `kubeadm` 自动创建和管理。

```bash
# 查看现有的 Bootstrap Token
kubectl get secrets -n kube-system | grep bootstrap-token

# Bootstrap Token Secret 的结构
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-abcdef
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  token-id: YWJjZGVm               # base64 编码的 Token ID
  token-secret: MTIzNDU2Nzg5MA==    # base64 编码的 Token Secret
  usage-bootstrap-authentication: dHJ1ZQ==   # base64("true")
  usage-bootstrap-signing: dHJ1ZQ==           # base64("true")
  auth-extra-groups: c3lzdGVtOmJvb3RzdHJhcHBlcnM6ZGVmYXVsdC1ub2RlLXRva2Vu
  # 还有其他字段：expiration, description 等
```

`kubeadm token create` 命令会自动创建这种类型的 Secret。`kubelet` 在加入集群时，使用 Bootstrap Token 向 API Server 认证，然后获取客户端证书。

```bash
# 创建 Bootstrap Token
kubeadm token create --print-join-command

# 列出当前所有 Bootstrap Token
kubeadm token list

# 删除过期的 Bootstrap Token
kubeadm token delete <token-id>
```



# 8.Secret的创建与使用

## 8.1 YAML定义创建

与 ConfigMap 类似，YAML 声明式创建是生产环境最推荐的 Secret 创建方式。Secret 的 YAML 结构比 ConfigMap 多了一个 `type` 字段。

**使用 data 字段（base64 编码）**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
  labels:
    app: myapp
  annotations:
    description: "数据库连接凭证"
type: Opaque
data:
  # 所有值必须是 base64 编码后的字符串
  username: YWRtaW4=              # echo -n 'admin' | base64
  password: U3VwZXJTZWNyZXQhMjM=  # echo -n 'SuperSecret!23' | base64
  host: bXlzcWwucHJvZC5zdmMuY2x1c3Rlci5sb2NhbA==   # echo -n 'mysql.prod.svc.cluster.local' | base64
  port: MzMwNg==                  # echo -n '3306' | base64
```

手动对每个值进行 base64 编码非常繁琐且容易出错。在实际工作中，大多数人使用 `stringData` 字段。

**使用 stringData 字段（明文，自动编码）**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: SuperSecret!23
  host: mysql.prod.svc.cluster.local
  port: "3306"
```

`stringData` 是写入时专用的字段，它只存在于你的本地 YAML 文件中。当你通过 `kubectl apply` 提交给 API Server 时，kubectl 会将 `stringData` 中的值进行 base64 编码，合并到 `data` 字段中，然后提交。API Server 存储的只有 `data` 字段，不保存 `stringData`。

如果你从集群中获取一个 Secret：

```bash
kubectl get secret db-secret -o yaml
```

你会看到输出中没有 `stringData`，只有 `data`（base64 编码的值）。

**data 和 stringData 同时存在时的行为**

如果同时指定了 `data` 和 `stringData`，kubectl 会先处理 `data` 中的值，然后将 `stringData` 中的值编码后合并。如果两者有相同的键，`stringData` 中的值会覆盖 `data` 中的值。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hybrid-secret
type: Opaque
data:
  key1: dmFsdWUx     # base64("value1")
  key2: dmFsdWUy     # base64("value2")
stringData:
  key2: overridden   # 覆盖 data 中的 key2
  key3: value3       # 新增 key3
# 最终 data 中：
# key1: dmFsdWUx
# key2: b3ZlcnJpZGRlbg==  (base64("overridden"))
# key3: dmFsdWUz           (base64("value3"))
```

**生产环境的最佳实践**

绝不将含有 `stringData` 的 YAML 文件提交到 Git 仓库（除非配合加密工具如 Sealed Secrets）。正确的做法是：

1. 在本地编写 YAML 用于开发和测试
2. 对于 Git 仓库中的文件，使用占位符并通过 CI/CD 系统注入实际值
3. 或使用 Sealed Secrets / External Secrets 等加密/外部引用方案

## 8.2 kubectl create secret

kubectl 提供了便捷的命令行方式来创建不同类型的 Secret，适合手动操作和脚本化。

**generic（通用 Secret / Opaque）**

```bash
# --from-literal: 从命令行键值对创建
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='SuperSecret!23' \
  --from-literal=host=mysql.default.svc.cluster.local

# --from-file: 从文件创建，文件名作为键
kubectl create secret generic cert-secret \
  --from-file=./tls.crt \
  --from-file=./tls.key

# --from-file 自定义键名
kubectl create secret generic cert-secret \
  --from-file=ca.crt=./ca-cert.pem \
  --from-file=client.crt=./client-cert.pem

# --from-env-file: 从 .env 文件创建
kubectl create secret generic env-secret \
  --from-env-file=./secrets.env
```

**docker-registry（镜像仓库认证）**

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com \
  --namespace=default

# 针对私有仓库
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.internal.com \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --namespace=prod
```

**tls（TLS 证书）**

```bash
# 从 PEM 文件创建 TLS Secret
kubectl create secret tls my-tls \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem

# 自签名证书示例（测试用）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.example.com/O=myapp"
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key
```

**预览生成的 YAML（dry-run）**

```bash
# 不实际创建，只是预览 YAML
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml

# 输出：
# apiVersion: v1
# kind: Secret
# metadata:
#   name: db-secret
# data:
#   password: c2VjcmV0MTIz
#   username: YWRtaW4=
# type: Opaque
```

## 8.3 环境变量注入Secret

Secret 可以通过与 ConfigMap 完全相同的方式注入为环境变量，支持全部注入和单项注入。

**全部注入 (`envFrom.secretRef`)**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - secretRef:
        name: db-secret
        optional: false
    - secretRef:
        name: api-secret
        optional: true   # Secret 不存在时 Pod 仍可启动
```

**单项注入 (`env.valueFrom.secretKeyRef`)**

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username       # Secret 中的键名
          optional: false
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

**环境变量注入 Secret 的安全注意事项**

将 Secret 作为环境变量注入存在一些安全隐患：

1. 环境变量对所有能访问 `/proc/<pid>/environ` 的进程可见。如果应用存在命令注入漏洞，攻击者可能通过 `cat /proc/1/environ` 读取所有环境变量。

2. 许多应用会将环境变量输出到日志或错误信息中。例如，Java 应用在启动失败时可能打印完整的 `JAVA_TOOL_OPTIONS` 环境变量，某些监控 agent 会收集所有环境变量。

3. 如果应用程序崩溃并生成 core dump，环境变量可能被包含在 core dump 文件中。

4. 子进程会继承父进程的环境变量，如果一个被 compromise 的子进程能读取环境变量，Secret 就泄露了。

因此，安全最佳实践是优先使用文件挂载方式注入 Secret，而非环境变量。文件挂载允许更精细的权限控制，并且 Secret 数据不会出现在进程列表中。

## 8.4 文件挂载Secret

文件挂载是推荐的使用 Secret 的方式。与 ConfigMap 的文件挂载几乎相同，但有一些 Secret 特有的安全特性。

**基本挂载方式**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets-volume
      mountPath: /etc/secrets    # 挂载点
      readOnly: true             # 强制只读
  volumes:
  - name: secrets-volume
    secret:
      secretName: db-secret
      # defaultMode: 0400        # 默认 0644，建议设为更严格的权限
      # optional: false
      items:                     # 选择性地挂载部分键
      - key: username
        path: db-username
        mode: 0400               # 只有所有者可读
      - key: password
        path: db-password
        mode: 0400
```

**文件权限控制（defaultMode / mode）**

Secret 的文件权限控制尤为重要，因为敏感数据不应该被容器内的其他用户（如 `nobody`、其他 Sidecar）读取。

权限值的计算与 Linux 文件系统一致，是一个三位的八进制数：
- 第一位：所有者权限
- 第二位：同组用户权限
- 第三位：其他用户权限
- 4 = 可读，2 = 可写，1 = 可执行

```yaml
volumes:
- name: secrets
  secret:
    secretName: db-secret
    defaultMode: 0400     # 推荐：只有文件所有者可读
    items:
    - key: tls.crt
      path: cert.pem
      mode: 0444          # 证书文件其他用户可以读
    - key: tls.key
      path: key.pem
      mode: 0400          # 私钥只有所有者可读
```

**tmpfs 存储机制**

与 ConfigMap 不同，Secret 被挂载为 Volume 时，kubelet 使用 tmpfs（临时文件系统）而非磁盘存储 Secret 数据。这意味着：
- Secret 数据只存在于内存中，不会写入磁盘
- tmpfs 中的数据在 Pod 删除或节点重启后自动消失
- 即使攻击者获得节点的磁盘访问权限，也无法从磁盘中恢复 Secret 数据

这是 Secret 相对于 ConfigMap 的一个重要安全改进。

**原子性更新**

与 ConfigMap 类似，Secret 的文件挂载也支持通过 symlink 机制的原子性更新。当 Secret 内容变化时，kubelet 将新内容写入新的时间戳目录，然后原子性地切换 `..data` symlink。应用在下次打开文件时就能看到新内容。

注意：`subPath` 挂载 Secret 同样不支持热更新（与 ConfigMap 相同的问题）。

## 8.5 imagePullSecrets

`imagePullSecrets` 是 Kubernetes 中用于认证私有镜像仓库的专用机制。它与普通的 Secret 使用方式不同，是 Pod Spec 的一个独立字段。

**基本使用**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-image-pod
spec:
  containers:
  - name: app
    image: harbor.internal.com/team/myapp:v1.0  # 私有仓库镜像
  imagePullSecrets:
  - name: harbor-regcred    # 引用 dockerconfigjson 类型的 Secret
```

`imagePullSecrets` 可以指定多个（如果镜像来自不同的私有仓库）：

```yaml
imagePullSecrets:
- name: dockerhub-secret
- name: harbor-secret
- name: ecr-secret
```

**kubelet 的处理流程**

当 kubelet 需要拉取镜像时，它会依次尝试所有 `imagePullSecrets` 中指定的 Secret，找到与镜像仓库地址匹配的认证信息。具体步骤：

1. kubelet 解析镜像地址，提取 registry 部分（如 `harbor.internal.com`）
2. 遍历 `imagePullSecrets` 列表，获取每个 Secret
3. 解码 Secret 的 `.dockerconfigjson` 数据，解析出 `auths` 映射
4. 在 `auths` 中查找匹配的 registry 地址
5. 使用找到的认证信息 `Authorization: Basic <base64>` 请求镜像仓库

如果某个 registry 在多个 `imagePullSecrets` 中都有匹配的认证信息，kubelet 使用列表中第一个匹配的 Secret。

**跨 Namespace 的问题与解决方案**

`imagePullSecrets` 是 Namespace 级别的资源，Pod 只能引用同一 Namespace 下的 Secret。这意味着每创建一个新的 Namespace，都需要重新创建 `imagePullSecrets`，这在多租户环境中是一个显著的运维负担。

原生解决方案——通过 ServiceAccount 绑定：

```yaml
# 在 Namespace 中创建 Secret
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: team-a
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded>

---
# 在 ServiceAccount 中添加 imagePullSecrets
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: team-a
imagePullSecrets:
- name: regcred

# 之后该 ServiceAccount 创建的所有 Pod 自动拥有 imagePullSecrets
```

但这仍然需要在每个 Namespace 手动操作。社区解决方案：

1. **kubernetes-replicator**：一个 Kubernetes Operator，自动将指定的 Secret 复制到新创建的 Namespace 中。

2. **Kyverno ClusterPolicy**：策略引擎，可以在 Namespace 创建时自动生成 imagePullSecrets。

```yaml
# Kyverno ClusterPolicy 示例
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-image-pull-secret
spec:
  rules:
  - name: sync-image-pull-secret
    match:
      resources:
        kinds:
        - Namespace
    generate:
      kind: Secret
      name: regcred
      namespace: "{{request.object.metadata.name}}"
      clone:
        namespace: default
        name: regcred
```



# 9.Secret安全增强

## 9.1 etcd静态加密

默认情况下，Kubernetes 将所有 API 对象（包括 Secret）以明文形式存储在 etcd 中。任何有权限访问 etcd 的人（集群管理员、etcd 备份文件持有者）都能直接读取 Secret 的内容。

启用 etcd 静态加密后，API Server 在将数据写入 etcd 前进行加密，在从 etcd 读取数据后进行解密。这意味着 etcd 中存储的是密文，即使 etcd 数据文件泄露，Secret 数据也不会直接暴露。

**EncryptionConfiguration 配置文件**

API Server 通过 `--encryption-provider-config` 参数指定加密配置文件：

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets                # 对 Secret 启用加密
    - configmaps             # 也可以对 ConfigMap 启用加密（非必需）
    providers:
    - aescbc:                # 使用 AES-CBC 加密
        keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>   # 32 字节 AES-256 密钥
    - identity: {}           # 回退方案：明文（不加密）
```

可用的加密 Provider：

| Provider | 安全性 | 性能 | 说明 |
|----------|--------|------|------|
| `identity` | 无加密 | 最高 | 明文存储，用于回退或过渡期 |
| `aescbc` | 中高 | 中 | AES-256-CBC 加密，2006 年提出的方案，目前认为不够安全 |
| `aesgcm` | 高 | 中 | AES-256-GCM，推荐使用，带认证加密（AEAD） |
| `secretbox` | 高 | 中 | XSalsa20 + Poly1305，NaCl 库的 AEAD 方案 |
| `kms` | 最高 | 低 | 使用外部 Key Management Service（AWS KMS/GCP KMS/Azure Key Vault），密钥不存储在集群中 |

**Provider 列表的优先级**

API Server 使用 Provider 列表中的第一个 Provider 对新写入数据进行加密。解密时，API Server 会尝试列表中的所有 Provider（通过尝试解密或检查元数据中的 Provider 标识），找到匹配的 Provider 进行解密。

这意味着：
- 列表中的第一个 Provider 决定了新数据的加密方式
- `identity: {}` 放在最后作为回退，确保即使加密密钥丢失，旧数据仍可读取
- 添加新 Provider 时，放在列表最前面，旧数据会逐步被重写（新 Provider 加密）

**最佳实践配置**

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    # 第一优先级：使用 KMS 加密（推荐生产环境）
    - kms:
        name: myKmsPlugin
        endpoint: unix:///var/run/kmsplugin/socket.sock
        cachesize: 1000
        timeout: 3s
    # 第二优先级：使用 AES-GCM 加密
    - aesgcm:
        keys:
        - name: key1
          secret: <base64-encoded-key>
    # 最后回退：明文
    - identity: {}
```

**密钥轮换**

定期轮换加密密钥是安全最佳实践。Kubernetes 支持多密钥配置以实现无缝轮换：

```yaml
providers:
- aesgcm:
    keys:
    - name: key2    # 新密钥（第一优先级，用于加密新数据）
      secret: <new-base64-key>
    - name: key1    # 旧密钥（第二优先级，用于解密旧数据）
      secret: <old-base64-key>
- identity: {}
```

轮换步骤：
1. 在配置文件中添加新密钥（放在最前面）
2. 重启 API Server 加载新配置
3. 运行 `kubectl get secrets --all-namespaces -o json | kubectl replace -f -` 重新加密所有 Secret
4. 确认所有 Secret 已用新密钥加密后，移除旧密钥配置
5. 再次重启 API Server

**启用加密后的注意事项**

- 加密只保护 etcd 中的数据，不保护传输中的数据或节点上的数据
- API Server 重启时需要读取 EncryptionConfiguration，确保配置文件本身安全（权限 600）
- 加密密钥的安全存储和备份至关重要——密钥丢失意味着数据永久不可恢复
- 启用加密会增加 API Server 的 CPU 消耗（特别是 KMS Provider，因为涉及外部 I/O）

## 9.2 RBAC访问控制

RBAC 是 Secret 安全的第一道防线。正确的 RBAC 配置可以防止未授权的 Secret 访问。

**最小权限原则**

只授予每个用户/ServiceAccount 完成任务所必需的最小 Secret 访问权限。

```yaml
# ❌ 危险：授予某个 SA 读取所有 Secret 的权限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader-all
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch"]    # 可以读取所有 Namespace 的所有 Secret

# ✅ 安全：只授予特定 Namespace 下特定 Secret 的只读权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: db-secret-reader
  namespace: app-prod
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-secret"]       # 只能访问指定的 Secret
  verbs: ["get"]                     # 只能 get，不能 list/watch
```

**禁止 ClusterRole 访问 Secret**

禁止创建或使用能跨 Namespace 访问 Secret 的 ClusterRole。Secret 的访问应该始终限制在特定 Namespace 内，使用 Role/RoleBinding 而非 ClusterRole/ClusterRoleBinding。

```yaml
# ❌ 不应存在的 ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["*"]    # 跨所有 Namespace 访问 Secret——极度危险

# ✅ 正确：使用 Role 限制在特定 Namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-secret-manager
  namespace: app-prod
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret-1", "app-secret-2"]
  verbs: ["get", "update"]
```

**审计日志监控**

启用 Kubernetes 审计日志（Audit Logging），监控所有 Secret 的访问行为。配置审计策略记录 Secret 的 get/list/watch 操作：

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# 记录所有 Secret 和 ConfigMap 的操作
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
  verbs: ["create", "update", "patch", "delete"]

# 记录 Secret 的读取操作（可能表示信息收集行为）
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets"]
  verbs: ["get", "list", "watch"]
```

## 9.3 外部密钥管理集成概述

对于生产环境，仅依赖 Kubernetes 原生的 Secret 机制往往不够。核心痛点包括：
- Secret 的 `data` 数据在 Git 仓库中仍需保护（不能明文提交）
- 多集群间 Secret 同步困难
- 缺乏自动密钥轮换机制
- 缺少与外部密钥管理服务（AWS Secrets Manager、Azure Key Vault、HashiCorp Vault 等）的集成

以下三节将详细介绍社区中最主流的解决方案。

## 9.4 Sealed Secrets (Bitnami)

Sealed Secrets 是 Bitnami 开发的开源工具，解决了"如何将 Secret 安全地存储在 Git 中"这一核心问题。它的设计理念是：将 Secret 加密（seal）为一个 `SealedSecret` CR（Custom Resource），SealedSecret 可以安全地提交到 Git 仓库。在集群中，Sealed Secrets Controller 自动将 SealedSecret 解密，生成原生的 Kubernetes Secret。

**工作原理**

```
开发者                    Git 仓库                    Kubernetes 集群
  │                         │                              │
  ├─ 1. 编写 Secret YAML    │                              │
  │                         │                              │
  ├─ 2. kubeseal 加密       │                              │
  │   生成 SealedSecret ──→ 3. 提交到 Git ──→ 4. ArgoCD/Flux 同步 ──→ 5. Controller 解密
  │                         │                              │    生成原生 Secret
  │                         │                              │
  └──────────────────────────────────────────────────────────── Pod 挂载 Secret
```

SealedSecret 使用非对称加密（RSA 或类似方案）：Controller 持有私钥，开发者使用公钥加密 Secret。只有能访问 Controller 私钥的人（集群管理员）才能解密 SealedSecret。加密后的内容包含在 SealedSecret 的 `spec.encryptedData` 字段中。

```yaml
# 原始 Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
data:
  password: cGFzc3dvcmQxMjM=

---
# 加密后的 SealedSecret（可安全存入 Git）
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
  namespace: default
spec:
  encryptedData:
    password: AgB4hKV5D5zCqM3k...  # 加密后的密文
  template:
    metadata:
      name: db-secret
      namespace: default
    type: Opaque
```

**操作流程**

```bash
# 1. 在集群中安装 Sealed Secrets Controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# 2. 获取公钥（用于本地加密）
kubeseal --fetch-cert > pubkey.pem

# 3. 编写 Secret YAML，使用 kubeseal 加密
kubectl create secret generic db-secret \
  --from-literal=password=secret123 \
  --dry-run=client -o yaml |
  kubeseal --cert=pubkey.pem -o yaml > sealed-secret.yaml

# 4. 提交 sealed-secret.yaml 到 Git
# 5. Controller 检测到新的 SealedSecret 后自动解密创建原生 Secret
# 6. 查看自动生成的 Secret
kubectl get secret db-secret -o yaml
```

**优点与限制**

优点：
- SealedSecret YAML 可以安全地存储在 Git 仓库中
- 单向加密——只有集群中的 Controller 能解密
- 支持密钥轮换（Controller 管理多版本密钥）
- 与 GitOps 工作流完美集成
- 对开发者友好，不需要额外的密钥管理服务

限制：
- 仍然需要在集群中管理密钥（Controller 的私钥）
- Controller 私钥丢失后，所有 SealedSecret 无法被解密
- 多集群场景需要为每个集群单独加密（每个集群的 Controller 密钥对不同）
- 不支持动态密钥轮换（轮换需要重新加密所有 Secret）

## 9.5 External Secrets Operator (ESO)

External Secrets Operator（ESO）是 CNCF 项目，它将外部密钥管理服务（如 AWS Secrets Manager、GCP Secret Manager、Azure Key Vault、HashiCorp Vault、1Password 等）与 Kubernetes 集成。它的核心概念是：外部密钥管理服务是"真实来源"（Source of Truth），Kubernetes 中的 Secret 是"镜像副本"。

**架构和工作原理**

```
AWS Secrets Manager ──────┐
GCP Secret Manager ──────┤
Azure Key Vault ─────────┤  ← External Secrets Operator 定期同步
HashiCorp Vault ─────────┤     (默认每 1 分钟)
1Password ───────────────┘
        │
        ▼
┌──────────────────────────────────┐
│   ExternalSecret (CR)            │
│   - secretStoreRef               │
│   - target (name, type)          │
│   - data[] (remoteRef, key)      │
└──────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────┐
│   Kubernetes Secret              │
│   (自动生成并保持同步)              │
└──────────────────────────────────┘
```

**安装**

```bash
# 使用 Helm 安装
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace

# 验证安装
kubectl get pods -n external-secrets-system
```

**配置 Secret Store（以 AWS Secrets Manager 为例）**

```yaml
# 1. ClusterSecretStore（集群级别）或 SecretStore（Namespace 级别）
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:              # 或使用 secretRef / controllerRef
          serviceAccountRef:
            name: eso-service-account
            namespace: external-secrets-system
---
# 2. ExternalSecret：定义从外部密钥服务同步的具体数据
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: default
spec:
  refreshInterval: 1h               # 同步间隔
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret                  # 生成的 Kubernetes Secret 名称
    creationPolicy: Owner            # Owner: ESO 管理生命周期; Orphan: 独立
    deletionPolicy: Retain           # Retain / Delete / Merge
  data:
  - secretKey: username              # Kubernetes Secret 中的键
    remoteRef:
      key: prod/database/credentials # AWS Secrets Manager 中的 Secret 名称
      property: username             # JSON 中的具体字段
  - secretKey: password
    remoteRef:
      key: prod/database/credentials
      property: password
```

**支持的外部 Provider**

ESO 支持非常丰富的外部密钥管理服务：

| Provider | 说明 |
|----------|------|
| AWS Secrets Manager | Amazon 的密钥管理服务 |
| AWS Parameter Store | AWS SSM 参数存储 |
| GCP Secret Manager | Google Cloud 密钥管理 |
| Azure Key Vault | Azure 密钥保管库 |
| HashiCorp Vault | 企业级密钥管理平台 |
| 1Password | 团队密码管理器 |
| Akeyless | 密钥管理 SaaS 平台 |
| IBM Cloud Secrets Manager | IBM 云密钥管理 |
| Oracle Vault | Oracle 云密钥管理 |
| GitLab Variables | GitLab CI/CD 变量 |

**高级特性**

ESO 不仅支持简单的键值对同步，还支持：
- **JSON 解析**：将 JSON 格式的外部 Secret 解析为多个 Kubernetes Secret 键
- **模板化**：使用 Go 模板语法对 Secret 值进行转换
- **自动轮换**：按 `refreshInterval` 定期同步，外部密钥轮换后 K8s Secret 自动更新
- **多集群管理**：同一个 `ExternalSecret` 可以在多个集群中应用（配合 GitOps）
- **密钥标记**：生成 Kubernetes Secret 时自动添加 OwnerReference，方便清理

```yaml
# 高级示例：JSON 解析 + 模板
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: complex-secret
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secret
    template:                         # 模板化
      data:
        config.json: |
          {
            "db": {
              "host": "{{ .dbHost }}",
              "port": {{ .dbPort }}
            },
            "api": {
              "key": "{{ .apiKey }}"
            }
          }
  data:
  - secretKey: dbHost
    remoteRef:
      key: secret/data/prod/database
      property: host
  - secretKey: dbPort
    remoteRef:
      key: secret/data/prod/database
      property: port
  - secretKey: apiKey
    remoteRef:
      key: secret/data/prod/api
      property: key
```

## 9.6 Vault Sidecar Injector (HashiCorp Vault)

HashiCorp Vault 是企业级密钥管理平台，提供动态密钥生成、密钥租约管理、审计日志等高级特性。Vault Sidecar Injector 是 Vault 的 Kubernetes 集成方案，通过 Sidecar 模式将密钥注入到应用容器中。

**架构**

```
┌─────────────────────────────────────────────┐
│ Pod                                         │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │   App Container │  │ Vault Agent      │  │
│  │                 │  │ (Sidecar)        │  │
│  │ 读取 /vault/    │◄─│ 从 Vault 获取密钥  │  │
│  │ secrets/db-creds│  │ 写入共享 Volume    │  │
│  └────────┬────────┘  └────────┬─────────┘  │
│           │                    │             │
│           └────────┬───────────┘             │
│                    ▼                         │
│  ┌─────────────────────────────────────────┐│
│  │ shared volume (/vault/secrets)          ││
│  │ 内存 tmpfs，不写入磁盘                      ││
│  └─────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
           ▲
           │ Kubernetes Auth Method
           ▼
┌──────────────────────┐
│  HashiCorp Vault     │
│  - 动态数据库凭证      │
│  - PKI 证书颁发       │
│  - 加密即服务          │
│  - 审计日志           │
└──────────────────────┘
```

**Mutating Webhook 注入机制**

Vault Sidecar Injector 通过 Kubernetes Mutating Admission Webhook 工作。当 Pod 创建时带有特定的 annotation（如 `vault.hashicorp.com/agent-inject: "true"`），webhook 会自动修改 Pod Spec：
1. 添加 Vault Agent Sidecar 容器
2. 添加共享 Volume（内存 tmpfs）
3. 配置 Vault Agent 的认证和密钥获取模板
4. Vault Agent 在启动时认证到 Vault，获取密钥，写入共享 Volume
5. 应用容器从共享 Volume 中读取密钥文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-demo
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "my-app"           # Vault 中的角色名
    vault.hashicorp.com/agent-inject-secret-db-creds: "database/creds/my-role"  # Vault 路径
    vault.hashicorp.com/agent-inject-template-db-creds: |
      {{- with secret "database/creds/my-role" -}}
      DB_USERNAME={{ .Data.username }}
      DB_PASSWORD={{ .Data.password }}
      {{- end }}
spec:
  serviceAccountName: my-app-sa                   # 用于 Vault 认证的 SA
  containers:
  - name: app
    image: myapp:latest
```

**Vault Kubernetes Auth Method**

Vault 通过 Kubernetes Auth Method 验证 Pod 的身份。Pod 使用其 ServiceAccount Token 向 Vault 认证，Vault 验证 Token 的合法性（通过 TokenReview API）后，根据 Role 返回对应的 Secret。

```bash
# 在 Vault 中配置 Kubernetes Auth Method
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"

vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app-sa \
  bound_service_account_namespaces=default \
  policies=my-app-policy \
  ttl=1h
```

**对比 ESO**

| 特性 | Vault Sidecar Injector | External Secrets Operator |
|------|----------------------|--------------------------|
| 架构 | Sidecar 模式 | Operator 模式 |
| 密钥存储 | 应用运行时注入，不创建 K8s Secret | 创建 K8s Secret 对象 |
| 密钥更新 | Vault Agent 自动续租 | ESO 定期同步 |
| 依赖 | 需要 Vault 集群 | 支持多种 Provider |
| 运维复杂度 | 高（需要维护 Vault） | 中（ESO 轻量级） |
| 动态密钥 | 支持（数据库临时凭证） | 有限 |
| 适用场景 | 大型企业、严格合规需求 | 中小型团队、多云环境 |

## 9.7 CSI Secret Store Driver

Secrets Store CSI Driver 是 Kubernetes SIG-Auth 开发的官方方案，通过 CSI（Container Storage Interface）将外部密钥存储中的密钥挂载为 Volume。从 Kubernetes 1.27 开始，部分特性升级为 GA 或 Beta。

**架构**

```
外部密钥管理服务 (Vault / AWS / GCP / Azure)
        │
        ▼
┌───────────────────────────────────────┐
│  SecretProviderClass (CR)             │
│  - provider: aws / vault / gcp       │
│  - parameters: 密钥路径、版本等       │
└───────────────────────────────────────┘
        │            ▲
        │ 引用       │ 管理
        ▼            │
┌───────────────────────────────────────┐  ┌───────────────────┐
│  Pod                                  │  │ CSI Driver Daemon │
│  volumeMounts:                        │  │ Set (每个 Node)    │
│    /mnt/secrets-store/                │  │                   │
│      db-password                      │  │ 挂载到 Pod 的     │
│      api-key                          │  │ Volume 中         │
└───────────────────────────────────────┘  └───────────────────┘
```

**安装**

```bash
# 使用 Helm 安装 CSI Driver
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true          # 启用 K8s Secret 同步

# 安装 AWS Provider（以 AWS 为例）
helm install aws-provider \
  secrets-store-csi-driver/secrets-store-csi-driver-provider-aws \
  --namespace kube-system
```

**SecretProviderClass 配置**

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
  namespace: default
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/database/credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: "username"
            objectAlias: "db-username"
          - path: "password"
            objectAlias: "db-password"
      - objectName: "prod/api/keys"
        objectType: "secretsmanager"
        jmesPath:
          - path: "api_key"
            objectAlias: "api-key"
  secretObjects:                     # 可选：同步为 K8s Secret
  - secretName: db-creds-synced
    type: Opaque
    data:
    - objectName: db-username
      key: username
    - objectName: db-password
      key: password
```

**Pod 中使用**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: csi-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets-store
      mountPath: /mnt/secrets-store
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: aws-secrets
```

挂载后，`/mnt/secrets-store/` 目录下会包含 `db-username`、`db-password`、`api-key` 三个文件，内容即是对应的密钥值。

**与 ESO 的对比和选择**

- CSI Driver 的优势：密钥不创建 K8s Secret 对象（如果不需要 secretObjects），密钥只存在于 Pod 的 tmpfs 挂载中；支持自动密钥轮换（Provider 原生支持）；CSI 是 Kubernetes 的标准接口。

- ESO 的优势：管理平面更简洁（不需要每个 Node 的 DaemonSet）；自动创建和管理 K8s Secret 对象，兼容传统应用；支持更多 Provider 和更灵活的模板化。

选择建议：
- 如果你的应用可以直接读取文件中的密钥，且你希望密钥不产生 K8s Secret 对象 → 使用 CSI Driver
- 如果你的应用已经依赖 K8s Secret（环境变量、已有部署模式），或你需要跨 Namespace 同步 → 使用 ESO
- 两者可以共存——CSI Driver 用于文件注入，ESO 用于 Secret 对象管理



# 10.Downward API

## 10.1 概念与用途

Downward API 是 Kubernetes 提供的一个特殊机制，它允许容器内的应用程序获取自身 Pod 和容器的元数据信息（如 Pod 名称、Namespace、UID、标签、注解、资源限制等），而不需要调用 Kubernetes API。

为什么叫 "Downward" API？因为这个信息流是从 Kubernetes 控制平面"向下"流向 Pod 内的容器，与通常的应用"向上"调用 API Server 方向相反。

**要解决的问题**

在很多场景中，应用需要知道自己的运行环境信息：
- 日志中需要记录所属的 Pod 名称和 Namespace，方便追踪
- 需要知道自己的 CPU/内存限制来决定线程池大小或 JVM 堆大小
- 需要将自己的 Pod 标签注册到服务发现系统
- 需要 Pod 的 UID 作为分布式锁的持有者标识
- 需要 Node 名称来感知自己所在的物理位置（多可用区部署）

如果应用直接调用 Kubernetes API 来获取这些信息，需要：
1. 在容器内集成 Kubernetes Client SDK
2. 配置 ServiceAccount 和 RBAC 权限
3. 处理 API Server 的网络延迟和可用性

Downward API 提供了更简单、更轻量的替代方案——这些信息在 Pod 启动时就已经确定，直接通过环境变量或文件提供给容器，无需任何 API 调用。

**可暴露的信息**

| 类别 | 字段 | 说明 |
|------|------|------|
| Pod 信息 | `metadata.name` | Pod 名称 |
| Pod 信息 | `metadata.namespace` | Pod 所在 Namespace |
| Pod 信息 | `metadata.uid` | Pod 的唯一标识符 |
| Pod 信息 | `metadata.labels['key']` | Pod 的标签 |
| Pod 信息 | `metadata.annotations['key']` | Pod 的注解 |
| Pod 信息 | `spec.serviceAccountName` | Pod 使用的 ServiceAccount |
| Pod 信息 | `spec.nodeName` | Pod 所在节点名称 |
| Pod 信息 | `status.podIP` | Pod 的 IP 地址（1.9+） |
| Pod 信息 | `status.hostIP` | 宿主机 IP 地址（1.9+） |
| 容器信息 | `resource.requests.cpu` | CPU 请求量 |
| 容器信息 | `resource.requests.memory` | 内存请求量 |
| 容器信息 | `resource.limits.cpu` | CPU 限制量 |
| 容器信息 | `resource.limits.memory` | 内存限制量 |

注意：Downward API 只能暴露 Pod 创建时已确定的信息，不能用于获取动态变化的信息（如 Pod status phase）。

## 10.2 通过环境变量暴露

通过环境变量暴露 Downward API 是最常用的方式，适合应用在启动时读取这些信息。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-env-demo
  namespace: my-namespace
  labels:
    app: myapp
    version: v1
    env: production
  annotations:
    build-id: "12345"
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "env | grep POD_"]
    env:
    # Pod 元数据
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_UID
      valueFrom:
        fieldRef:
          fieldPath: metadata.uid
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: HOST_IP
      valueFrom:
        fieldRef:
          fieldPath: status.hostIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName

    # 资源信息
    - name: CPU_REQUEST
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
          divisor: "1"           # 不除以任何单位，显示为毫核
    - name: MEMORY_LIMIT_MIB
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
          divisor: "1Mi"         # 除以 1Mi 显示为 MiB 单位
    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

`divisor` 字段用于资源值单位转换：
- 对于 CPU：默认是 `1m`（毫核），设置为 `1` 可以显示为毫核数值
- 对于内存：默认是 `1`（字节），设置为 `1Ki` / `1Mi` / `1Gi` 可以转换为更大的单位

**标签和注解**

对于 label 和 annotation，语法稍有不同：

```yaml
env:
- name: APP_VERSION
  valueFrom:
    fieldRef:
      fieldPath: metadata.labels['version']    # 通过标签名获取
- name: BUILD_ID
  valueFrom:
    fieldRef:
      fieldPath: metadata.annotations['build-id']  # 通过注解名获取
```

注意事项：
- label 和 annotation 的键名中的特殊字符（如 `/` 和 `.`）在 Pod YAML 中需要转义或用方括号引用
- 如果指定的 label/annotation 在 Pod 创建时不存在，环境变量值为空字符串
- label/annotation 变化后，已启动容器的环境变量不会更新（和环境变量注入的其他场景一样）

## 10.3 通过卷文件暴露

通过 Downward API Volume 暴露信息为文件，支持更丰富的数据格式和动态更新。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-volume-demo
  labels:
    app: myapp
    tier: frontend
  annotations:
    build-number: "42"
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do cat /etc/podinfo/labels; sleep 5; done"]
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
      readOnly: true
  volumes:
  - name: podinfo
    downwardAPI:
      items:
      # Pod 元数据
      - path: "name"
        fieldRef:
          fieldPath: metadata.name
      - path: "namespace"
        fieldRef:
          fieldPath: metadata.namespace
      - path: "uid"
        fieldRef:
          fieldPath: metadata.uid
      - path: "pod-ip"
        fieldRef:
          fieldPath: status.podIP

      # 资源信息
      - path: "cpu-request"
        resourceFieldRef:
          containerName: app
          resource: requests.cpu
          divisor: 1m
      - path: "memory-limit"
        resourceFieldRef:
          containerName: app
          resource: limits.memory
          divisor: 1Mi

      # 标签和注解
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels     # 整个 labels map
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations  # 整个 annotations map
    resources:
      requests:
        cpu: "250m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

使用 Downward API Volume 的优势：
- label 和 annotation 可以整个 map 作为文件暴露，文件内容为 `key=value` 格式
- 支持热更新：当 Pod 的 label/annotation 变化时，kubelet 会在约 1 分钟内更新对应文件（与 ConfigMap 的更新机制相同）
- 可以设置文件权限（`mode` 字段）
- 避免环境变量数量过多的问题

**label/annotation 文件格式**

当暴露整个 label/annotation map 时，文件内容的格式为：

```
# /etc/podinfo/labels 文件内容
app="myapp"
tier="frontend"

# /etc/podinfo/annotations 文件内容
build-number="42"
kubernetes.io/config.seen="2023-01-01T00:00:00Z"
kubernetes.io/config.source="api"
```

## 10.4 使用场景

**场景一：日志中包含 Pod 标识**

微服务架构中，同一个应用有多个 Pod 副本。当日志汇聚到集中的日志系统（如 ELK、Loki）时，需要能追溯到具体是哪个 Pod 产生的日志。

```yaml
env:
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
```

然后在应用的日志配置中使用这些环境变量（如 Logback 的 `%X{POD_NAME}` MDC 变量）。

**场景二：JVM 内存自适应**

Java 应用可以通过 Downward API 获取容器的内存限制，自动设置 JVM 堆大小，避免 OOMKilled。

```yaml
env:
- name: MEMORY_LIMIT
  valueFrom:
    resourceFieldRef:
      containerName: app
      resource: limits.memory
      divisor: 1Mi
```

在启动命令中使用：

```dockerfile
# Dockerfile 中的启动脚本
CMD java \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -jar /app/myapp.jar
```

Java 10+ 支持 `-XX:MaxRAMPercentage`，可以直接使用容器的内存限制（从 cgroup 读取），不再需要手动计算堆大小。但某些场景下仍然需要精确控制，这时 Downward API 就派上了用场。

**场景三：分布式协调**

当应用需要参与选主（Leader Election）或分布式锁时，Pod 的唯一标识（名称+UID）可以作为竞争者的身份标识：

```yaml
env:
- name: INSTANCE_ID
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
```

**场景四：注册到服务发现**

如果使用 Consul、Eureka、Nacos 等服务发现系统，Pod 的 IP 和端口信息需要通过 Downward API 获取并注册：

```yaml
env:
- name: POD_IP
  valueFrom:
    fieldRef:
      fieldPath: status.podIP
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
```

# 11.Projected Volume

## 11.1 概念

Projected Volume 是 Kubernetes 1.11 引入的特性，它允许将多个 Volume 源聚合到一个 Volume 中，统一挂载到一个目录。支持的源包括：

- `secret`：Secret 资源
- `configMap`：ConfigMap 资源
- `downwardAPI`：Downward API 信息
- `serviceAccountToken`：ServiceAccount Token（Bound Token）

在 Projected Volume 出现之前，如果需要同时挂载 Secret 和 ConfigMap，需要分别创建两个 Volume 和两个 VolumeMount，挂载到不同的目录。Projected Volume 将多个源聚合到一个目录中，简化了 Pod Spec 并减少了挂载点数量。

```
Projected Volume: /etc/app/config/
├── [来自 ConfigMap] application.properties
├── [来自 ConfigMap] logback.xml
├── [来自 Secret] db-username
├── [来自 Secret] db-password
├── [来自 DownwardAPI] pod-name
├── [来自 DownwardAPI] pod-namespace
└── [来自 ServiceAccountToken] token
```

## 11.2 使用方式

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: projected-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: all-in-one
      mountPath: /etc/app/config
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      defaultMode: 0400          # 所有文件的默认权限
      sources:
      # 源 1：ConfigMap
      - configMap:
          name: app-config
          items:
          - key: application.properties
            path: config/app.properties
            mode: 0444
          - key: logback.xml
            path: config/logback.xml
            mode: 0444

      # 源 2：Secret
      - secret:
          name: db-secret
          items:
          - key: username
            path: secrets/db-username
            mode: 0400
          - key: password
            path: secrets/db-password
            mode: 0400

      # 源 3：Downward API
      - downwardAPI:
          items:
          - path: "info/pod-name"
            fieldRef:
              fieldPath: metadata.name
          - path: "info/namespace"
            fieldRef:
              fieldPath: metadata.namespace
          - path: "info/labels"
            fieldRef:
              fieldPath: metadata.labels

      # 源 4：ServiceAccount Token
      - serviceAccountToken:
          path: token/sa-token
          expirationSeconds: 3600
          audience: api
```

生成的目录结构：

```
/etc/app/config/
├── config/
│   ├── app.properties
│   └── logback.xml
├── secrets/
│   ├── db-username
│   └── db-password
├── info/
│   ├── pod-name
│   ├── namespace
│   └── labels
└── token/
    └── sa-token
```

每个源的文件都通过 symlink 机制映射到目标目录，因此 ConfigMap 和 Secret 的热更新特性在 Projected Volume 中仍然生效（前提是不使用 subPath）。

## 11.3 ServiceAccountToken的自动投影

从 Kubernetes 1.22 开始，Bound ServiceAccount Token API 成为 GA。kubelet 会自动为每个 Pod 投影 ServiceAccount Token，无需手动配置 Volume。

**自动投影的机制**

当 Pod 的 ServiceAccount 包含 `kubernetes.io/enforce-mountable-secrets` 注解或使用新版 API（`TokenRequest`）时，kubelet 自动注入一个 Projected Volume，包含：
1. 有时效性的 JWT Token（默认 1 小时过期）
2. Token 过期前自动刷新（kubelet 负责）
3. 不创建长期有效的 Secret 对象

```yaml
# 新建的 Pod 自动获得如下 Projected Volume
# （由 kubelet 的 ServiceAccount admission controller 自动添加）
volumes:
- name: kube-api-access-xxxxx
  projected:
    defaultMode: 0420
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3607
    - configMap:
        name: kube-root-ca.crt
        items:
        - key: ca.crt
          path: ca.crt
    - downwardAPI:
        items:
        - path: namespace
          fieldRef:
            fieldPath: metadata.namespace
```

挂载点：`/var/run/secrets/kubernetes.io/serviceaccount/`

文件列表：
- `token`：有时效性的 JWT Token
- `ca.crt`：集群 CA 证书（1.20+ 来自 `kube-root-ca.crt` ConfigMap，而非从 Secret 获取）
- `namespace`：当前 Pod 的 Namespace

**手动配置更长的 Token 有效期**

```yaml
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-sa
  containers:
  - name: app
    volumeMounts:
    - name: sa-token
      mountPath: /var/run/secrets/custom
  volumes:
  - name: sa-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 7200    # 2 小时
          audience: my-api-audience  # 自定义 audience
```

**禁用自动挂载**

如果应用不需要 ServiceAccount Token，可以显式禁用：

```yaml
apiVersion: v1
kind: Pod
spec:
  automountServiceAccountToken: false   # 禁用自动挂载
  # 或者
  serviceAccountName: my-sa
  containers:
  - name: app
```

这一行简单的配置不仅减少了不必要的安全暴露面，还减少了 kubelet 的工作量。

# 12.最佳实践

## 12.1 ConfigMap命名与组织

**命名约定**

一个清晰一致的命名约定能显著提高可维护性。推荐的命名模式：

```
<application-name>-<config-type>
<application-name>-<config-type>-<version>

# 示例：
web-app-config        # 通用配置
web-app-env           # 环境配置
web-app-nginx         # Nginx 配置
web-app-jvm           # JVM 配置
```

对于不可变 ConfigMap，版本号或哈希后缀是必需的：

```
web-app-config-v1
web-app-config-v2
web-app-config-7g4k2m  # Kustomize 自动生成
```

**标签管理**

所有 ConfigMap 和 Secret 都应该有适当的标签用于管理和筛选：

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-app
    app.kubernetes.io/part-of: ecommerce-platform
    app.kubernetes.io/version: v1.2.3
    app.kubernetes.io/component: backend
    app.kubernetes.io/managed-by: helm
    config-type: application
```

**按关注点分离**

不要将一个应用的所有配置放在一个巨大的 ConfigMap 中。而是按关注点分离：

- `app-config`：应用业务配置（功能开关、参数）
- `app-env`：环境相关配置（服务端点、数据库主机名）
- `app-jvm`：JVM/运行时配置（堆大小、GC 参数）
- `app-logging`：日志配置
- `app-thirdparty`：第三方服务端点

这种分离的好处：
- 每个 ConfigMap 更小、更聚焦、更易维护
- 不同的配置有不同的变更频率，分离后变更影响范围更小
- 不同的配置可能由不同的团队/人员管理

## 12.2 敏感信息必须用Secret

**基本原则**

- 密码、API Key、Token、私钥、证书 → Secret
- 数据库地址、端口号、日志级别、功能开关 → ConfigMap
- 不确定属于哪一类 → 优先使用 Secret

**环境变量 vs 文件挂载**

| 场景 | 推荐方式 |
|------|---------|
| 简单的用户名/密码 | 文件挂载（安全） |
| 数据库连接串（含密码） | 文件挂载，组合成连接串 |
| 需要热更新的敏感配置 | 文件挂载 + 应用热加载 |
| 少量简单的 API Key | 环境变量（读取方便） |
| TLS 证书和私钥 | 文件挂载（天然的证书文件） |
| 高安全要求场景 | 文件挂载 + 最小权限（0400） |

## 12.3 大型配置管理策略

**问题场景**

某些应用的配置文件可能非常大：
- Nginx 包含数十个 upstream 配置的完整 nginx.conf
- JVM 的完整 jvm.options
- 包含 SSL 证书链的完整配置文件
- GeoIP 数据库文件

**策略一：拆分配置文件**

将大型配置文件拆分为多个小文件，利用 ConfigMap 的 `items` 字段分别挂载，应用使用 `include` 机制加载：

```yaml
# 原始大文件 nginx.conf 被拆分为：
# - nginx-main.conf（主配置，include conf.d/*.conf）
# - conf.d/upstreams.conf（upstream 配置）
# - conf.d/server-sites.conf（server 配置）

configMap:
  name: nginx-config
  items:
  - key: nginx-main.conf
    path: nginx.conf
  - key: upstreams.conf
    path: conf.d/upstreams.conf
  - key: server-sites.conf
    path: conf.d/server-sites.conf
```

**策略二：分层配置**

利用应用框架的配置优先级机制，将配置分层：

```yaml
envFrom:
# 第一层：基础配置（ConfigMap，大但稳定）
- configMapRef:
    name: app-base-config
# 第二层：环境覆盖（ConfigMap，小且频繁变化）
- configMapRef:
    name: app-env-overlay
# 第三层：敏感信息（Secret，小）
- secretRef:
    name: app-credentials
```

**策略三：使用 PersistentVolume 存储超大配置**

对于确实超过 1MB 限制的配置（如 GeoIP 数据库），使用 PersistentVolume：

```yaml
spec:
  initContainers:
  - name: download-config
    image: busybox
    command: ['sh', '-c', 'wget -O /data/GeoLite2-City.mmdb https://config-server/geoip.mmdb']
    volumeMounts:
    - name: config-data
      mountPath: /data
  containers:
  - name: app
    volumeMounts:
    - name: config-data
      mountPath: /app/data
  volumes:
  - name: config-data
    persistentVolumeClaim:
      claimName: config-pvc
```

## 12.4 多环境配置管理方案

**方案一：Kustomize Overlay（原生推荐）**

```yaml
# 目录结构
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── configmap.yaml        # 使用 configMapGenerator
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── config-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── config-patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── config-patch.yaml
```

优点：原生支持、无需额外工具、声明式

**方案二：Helm Values（模板化）**

```yaml
# values/dev.yaml
app:
  config:
    logLevel: DEBUG
    dbHost: dev-db.internal
    dbPort: 5432
  secrets:
    dbPassword: dev-password

# values/prod.yaml
app:
  config:
    logLevel: WARN
    dbHost: prod-db.internal
    dbPort: 5432
  secrets:
    dbPassword: prod-password
```

优点：模板灵活、社区支持丰富、Chart 分发方便

**方案三：命名空间隔离 + 同名 ConfigMap**

```yaml
# 每个 Namespace 创建同名 ConfigMap，内容不同
# kubectl apply -f configmap.yaml -n dev
# kubectl apply -f configmap.yaml -n prod

# Deployment 引用时不需要知道在哪个 Namespace
volumes:
- name: config
  configMap:
    name: app-config  # 在 dev ns 取 dev 配置，在 prod ns 取 prod 配置
```

优点：简单直接、适合小型项目

**选择建议**

| 方案 | 适合场景 |
|------|---------|
| Kustomize Overlay | Kubernetes 原生团队，中小规模，追求简单 |
| Helm | 需要模板化，需要 Chart 分发和版本管理 |
| 命名空间隔离 | 极简场景，少量环境 |

## 12.5 GitOps中的Secret管理

GitOps 的核心理念是将 Git 仓库作为声明式基础设施和应用的单一事实来源（Source of Truth）。但 Secret 的敏感特性与 Git 的明文存储天然矛盾。以下是几种主流的 GitOps + Secret 管理方案：

**方案一：Sealed Secrets + Git**

```bash
# Git 仓库中存储加密后的 SealedSecret
git-repo/
├── apps/
│   └── my-app/
│       ├── deployment.yaml
│       ├── sealed-secret.yaml      # 加密的 Secret（安全）
│       └── kustomization.yaml      # 引用上述资源
```

适用于：中小团队，希望简单可靠且不需要外部密钥管理服务。

**方案二：External Secrets Operator + Git**

```bash
# Git 仓库中存储 ExternalSecret CR（不含真实密钥数据）
git-repo/
├── apps/
│   └── my-app/
│       ├── deployment.yaml
│       ├── external-secret.yaml    # 引用外部密钥管理器
│       └── kustomization.yaml
```

ESO 在集群中自动从 AWS/Azure/GCP/Vault 同步真实密钥并创建 Secret。

适用于：已使用云厂商密钥管理服务，或需要多集群密钥同步。

**方案三：SOPS + Flux/ArgoCD**

```bash
# 使用 Mozilla SOPS 加密整个 YAML 文件
# Flux 和 ArgoCD 原生支持 SOPS 解密

# .sops.yaml
creation_rules:
  - path_regex: .*.enc.yaml$
    kms: arn:aws:kms:us-east-1:123456789012:key/xxx

# secrets.enc.yaml（加密后提交）
apiVersion: v1
kind: Secret
metadata:
    name: db-secret
data:
    password: ENC[AES256_GCM,data:...]
sops:
    kms:
        - arn: arn:aws:kms:...
```

适用于：已使用 SOPS 的团队，或需要文件级加密。

**方案四：Vault + Git（零 Secret 在 Git 中）**

Git 仓库中不存储任何 Secret 相关内容（甚至加密后的也不存）。所有 Secret 通过 Vault Sidecar Injector 或 ESO + Vault 在运行时注入。

适用于：严格合规要求（如 PCI-DSS、HIPAA），大型企业。

**Git 仓库中 Secret 文件的安全检查**

无论使用哪种方案，都应该在 Git 仓库中启用以下安全检查：

```yaml
# .pre-commit-config.yaml（pre-commit 钩子配置）
repos:
- repo: https://github.com/Yelp/detect-secrets
  rev: v1.4.0
  hooks:
  - id: detect-secrets
    args: ['--baseline', '.secrets.baseline']
- repo: https://github.com/gitleaks/gitleaks
  rev: v8.18.0
  hooks:
  - id: gitleaks
```

CI/CD 流水线中也应包含 Secret 扫描步骤，防止意外提交明文密钥。

**总结：各方案的决策矩阵**

| 方案 | Git 中的内容 | 额外的集群组件 | 适用场景 |
|------|-------------|--------------|---------|
| Sealed Secrets | 加密的 SealedSecret | Sealed Secrets Controller | 中小团队，简单可靠 |
| ESO | ExternalSecret CR（无敏感数据） | ESO + 外部 Key Manager | 云原生，多集群 |
| SOPS | 文件级加密的 Secret | Flux/ArgoCD + KMS | 已有 SOPS 实践 |
| Vault Injector | 无任何 Secret | Vault + Sidecar Injector | 高安全合规 |
| CSI Driver | SecretProviderClass（无敏感数据） | CSI Driver + Provider | 精细控制密钥挂载 |
| 混合方案 | 根据敏感级别选择 | 多个组件 | 大型企业 |

最终选择取决于团队规模、安全合规要求、已有的基础设施投资和运维能力。没有银弹，只有适合当前阶段的最佳实践。
