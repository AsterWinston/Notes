前面几篇讲的是通用语法，这一篇把它们落到最真实的战场：Kubernetes、GitHub Actions、GitLab CI。这些平台的配置几乎全是 YAML，也几乎所有的"部署失败"最终都能追溯到某个缩进、某个类型或某个冒号空格。本篇用可直接运行的示例，串起资源定义、ConfigMap/Secret、流水线配置和缩进排错。

# 1. 为什么这些平台都用 YAML

| 平台 | YAML 用途 | 特点 |
|------|-----------|------|
| Kubernetes | 描述期望状态 | 声明式、嵌套深、字段多 |
| GitHub Actions | 定义流水线 | `on`/`jobs`/`steps` 三层结构 |
| GitLab CI | 定义流水线 | 大量用锚点和 `extends` 复用 |
| Docker Compose | 本地多容器编排 | 服务、网络、卷 |

共同点是**配置即代码**：文件进 Git，变更走评审，部署靠 apply。这让 YAML 的可读性和可校验性变得关键。

★ **关键认知**：这些平台的 YAML 不是"随便写的配置",而是有严格 schema 的**契约**。字段名拼错、层级放错、类型不对，平台会拒绝，而且往往只给一句模糊报错。会看 schema、会缩进对齐，比背字段更重要。

# 2. Kubernetes 资源

## 2.1 最小 Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  labels:
    app: demo
spec:
  containers:
    - name: demo
      image: nginx:1.25
      ports:
        - containerPort: 80
```

四个顶层字段是 K8s 资源的骨架：

| 字段 | 作用 |
|------|------|
| `apiVersion` | API 版本，决定用哪套 schema |
| `kind` | 资源类型 |
| `metadata` | 名称、标签、命名空间等元信息 |
| `spec` | 期望状态，具体字段由 kind 决定 |

## 2.2 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: nginx:1.25
          ports:
            - containerPort: 80
```

★ **关键认知**：`spec.selector.matchLabels` 必须与 `spec.template.metadata.labels` **匹配**，否则 Deployment 无法管理自己创建的 Pod。这是"创建成功但一直不 Ready"的常见原因。

## 2.3 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

`selector.app: api` 要能匹配到 Pod 的标签，Service 才能把流量转发过去。

⚠ **关键认知**：K8s 里三处 `app: api`（Deployment 的 selector、Pod 模板的 labels、Service 的 selector）必须一致。少改一处，最常见的结果是"Service 起来了但访问不通"。

# 3. ConfigMap 与 Secret

## 3.1 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  MODE: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
```

`data` 下的值**必须是字符串**：

```yaml
data:
  PORT: 8080          # 错误：数字，K8s 要求字符串
  PORT: "8080"        # 正确
  ENABLED: "true"     # 布尔也要加引号
```

★ **关键认知**：ConfigMap 的 `data` 只接受字符串。写了裸数字或布尔，`kubectl apply` 会报 `cannot unmarshal number into Go struct field ... of type string`。这是 K8s 配置最高频的类型错误。

## 3.2 Secret

Secret 有两种写法：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  password: c2VjcmV0MTIz      # base64 编码
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  password: secret123         # 明文，写入时自动编码
```

| 字段 | 值格式 | 说明 |
|------|--------|------|
| `data` | base64 | 需要先编码 |
| `stringData` | 明文 | 便于手写，apply 时转成 base64 |

⚠ **关键认知**：`data` 里的值必须是**合法 base64**，不是随便写的密文。用 `echo -n 'secret123' | base64` 生成。另外，Secret 只是"编码"不是"加密"，提交到 Git 前应配合 Sealed Secrets、SOPS 或外部密钥管理。

# 4. CI 示例

## 4.1 GitHub Actions

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npm test
```

★ **关键认知**：GitHub Actions 的 `on` 是触发键，但在 YAML 1.1 里 `on` 是布尔 `true`。GitHub 的解析器专门处理了这一点，所以 `on:` 能正常工作；但很多通用 linter 会警告，编辑器也可能高亮异常。这是历史遗留的兼容特例。

## 4.2 GitLab CI

```yaml
stages:
  - test
  - build

.defaults: &defaults
  image: node:20
  before_script:
    - npm ci
  cache:
    key: node-cache
    paths:
      - node_modules/

test:
  <<: *defaults
  stage: test
  script:
    - npm test

build:
  <<: *defaults
  stage: build
  script:
    - npm run build
```

GitLab CI 的复用手段：

| 手段 | 写法 | 说明 |
|------|------|------|
| 锚点+合并 | `<<: *defaults` | 复用映射 |
| `extends` | `extends: .defaults` | 官方推荐的复用方式 |
| `!reference` | `script: !reference [.defaults, script]` | 引用特定字段 |

⚠ **关键认知**：GitLab CI 里点号开头的 Job（如 `.base`）是**隐藏 Job**，不会被执行，专门用来做模板。忘记加点，模板会被当成真实 Job 跑起来。

# 5. 常见缩进错误

缩进错误在 K8s/CI 里后果最直接，因为平台不会告诉你"第几行的语义错了",只会给结构错误。

## 5.1 列表项对齐错误

```yaml
# 错误：image 与 name 没对齐到同一层
containers:
  - name: app
   image: nginx:1.25
```

```yaml
# 正确：name 和 image 左对齐
containers:
  - name: app
    image: nginx:1.25
```

## 5.2 冒号后空格缺失

```yaml
metadata:
  name:demo          # 错误：变成字符串
```

```yaml
metadata:
  name: demo         # 正确
```

## 5.3 层级放错

```yaml
# 错误：ports 放到了 metadata 下
metadata:
  name: app
  ports:
    - containerPort: 80
```

```yaml
# 正确：ports 属于容器
spec:
  containers:
    - name: app
      ports:
        - containerPort: 80
```

## 5.4 Tab 与空格混用

```text
Error from server (BadRequest): error when parsing: yaml: line 6: found character that cannot start any token
```

看到这类报错，先检查是否混入 Tab。

★ **关键认知**：排错顺序应该是"**先语法、后结构、再语义**"。先跑 `yamllint` 或 `kubectl apply --dry-run=client` 排除缩进和语法问题，再用 `--dry-run=server` 或 `kubeconform` 查 schema，最后才怀疑业务字段。

# 6. 动手练习

1. 写一个 Deployment + Service 的 YAML，要求 Service 能正确选中 Pod，然后故意改错 selector 观察 `kubectl get endpoints` 的变化。

2. 用 ConfigMap 把一个 `nginx.conf` 挂载进容器，注意 `data` 的值必须是字符串、配置文件用 `|` 保留换行。

3. 用 GitLab CI 的 `.defaults` 锚点，让 `test` 和 `lint` 共享 `image` 与 `before_script`，并解释为什么 `.defaults` 不会被执行。

# 7. 小结

| 场景 | 关键点 |
|------|--------|
| K8s 资源 | `apiVersion`/`kind`/`metadata`/`spec` 四件套 |
| Deployment | selector 与 Pod 标签必须匹配 |
| ConfigMap | `data` 的值一律字符串 |
| Secret | `data` 是 base64，`stringData` 是明文 |
| GitHub Actions | `on`/`jobs`/`steps` 三层 |
| GitLab CI | `.` 开头是隐藏模板，`extends` 复用 |
| 排错 | 先语法、后结构、再语义 |

一句话原则：**K8s 和 CI 的 YAML 错误，八成不是"字段写错",而是"层级放错"或"类型没加引号"**。下一篇进入解析器与工具链，讲 Java/Python/JS 怎么安全加载 YAML，以及 yq、格式化、schema 校验工具怎么用。
