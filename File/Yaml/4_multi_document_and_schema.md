一个 YAML 文件可以承载多个文档，`kubectl apply -f all.yaml` 一次部署整套资源就是靠这个能力。但多文档的边界、Schema 校验、版本兼容这三件事，恰恰是"本地能跑、线上报错"的高发区。本篇讲清 `---` 与 `...` 的语义、如何用 JSON Schema 约束 YAML、以及 YAML 1.1 与 1.2 的兼容性问题。

# 1. 多文档

## 1.1 流与文档

YAML 规范把输入叫 **流（stream）**，流里可以包含零个或多个 **文档（document）**。每个文档是一棵完整的节点树。

```text
流（stream）
├── 文档 1
├── 文档 2
└── 文档 3
```

物理上，它就是同一个文件里的多段内容。

## 1.2 `---` 与 `...`

`---` 是**文档开始**标记，`...` 是**文档结束**标记：

```yaml
kind: ConfigMap
metadata:
  name: app-config

---
kind: Secret
metadata:
  name: app-secret
...
```

规则：

| 标记 | 含义 | 是否必须 |
|------|------|----------|
| `---` | 开始一个新文档 | 多文档之间的分隔必需 |
| `...` | 结束当前文档 | 通常可省略 |

★ **关键认知**：`---` 用在**两个文档之间**，它的作用是"结束上一个、开始下一个"。文件开头的第一个 `---` 是可选的，写了也无害。`...` 主要用于显式收尾，或者在后面接非 YAML 内容（如日志）时划清边界。

单文档里也可能看到 `---`：

```yaml
---
kind: Pod
metadata:
  name: demo
```

这里的 `---` 只是"文档开始",不影响解析。

⚠ **关键认知**：`---` 必须**独占一行**（前面可有空白）。写成 `---kind: Pod` 会把整行当字符串。同理，`...` 后面不能再跟同一行的内容。

## 1.3 空文档与多文档解析

两个 `---` 之间如果没有内容，会产生一个**空文档**（解析为 `null`）：

```yaml
---
---
kind: Pod
```

第一个文档是 `null`，第二个才是 Pod。有些解析器会把空文档也计入结果，处理时要过滤掉。

不同 API 的解析行为：

```python
import yaml

with open("all.yaml") as f:
    docs = list(yaml.safe_load_all(f))   # 返回一个可迭代对象，逐个文档
print(len(docs))
```

★ **关键认知**：单文档用 `load`，多文档用 `loadAll`/`safe_load_all`/`load_all`。用单文档 API 读多文档文件，多数实现只取第一个文档，后面的被静默忽略——这是"资源少创建了却没人报错"的典型原因。

# 2. Schema 与校验

## 2.1 为什么需要 schema

YAML 只保证**语法合法**，不保证**结构正确**。下面这段完全合法：

```yaml
replicas: "3"
image: 123
```

但业务上 `replicas` 应该是整数、`image` 应该是字符串。语法校验器不会报错，问题要等到运行时才暴露。

Schema 的作用是声明"字段名、类型、必填、取值范围",把错误提前到提交阶段。

## 2.2 用 JSON Schema 描述 YAML

YAML 没有独立的 Schema 语言，业界普遍用 **JSON Schema**：先把 YAML 转成数据模型，再用 JSON Schema 约束。因为 YAML 兼容 JSON，两者模型可以互通。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["image", "replicas"],
  "properties": {
    "image": { "type": "string" },
    "replicas": { "type": "integer", "minimum": 1 },
    "env": {
      "type": "object",
      "additionalProperties": { "type": "string" }
    }
  },
  "additionalProperties": false
}
```

对应合法 YAML：

```yaml
image: nginx:1.25
replicas: 3
env:
  MODE: production
```

非法 YAML（`replicas` 是字符串、多了未知字段）：

```yaml
image: nginx:1.25
replicas: "3"
unknown: true
```

## 2.3 常见校验工具

| 工具 | 场景 | 说明 |
|------|------|------|
| `ajv` | JSON/YAML 数据 | JS 生态最常用的 JSON Schema 校验器 |
| `check-jsonschema` | 文件校验 | Python 命令行，支持 YAML |
| `kubeconform` | K8s 资源 | 用官方 schema 校验，CI 常客 |
| `datree` / `kubectl --dry-run` | K8s | 结合集群校验 |
| VS Code YAML 插件 | 编辑时 | 支持 `$schema` 提示与实时校验 |
| `yamllint` | 风格/语法 | 检查缩进、重复键、行宽 |

在 YAML 里声明 schema 关联（供编辑器识别）：

```yaml
# yaml-language-server: $schema=./schema.json
image: nginx:1.25
replicas: 3
```

★ **关键认知**：schema 校验和语法校验是两层。`yamllint` 管"写得规不规范",JSON Schema 管"结构对不对",K8s 的 admission 管"集群是否接受"。三层各司其职，不能互相替代。

## 2.4 Kubernetes 的 schema

K8s 的 schema 不在 YAML 文件里，而是由 API Server 通过 OpenAPI 提供。常用命令：

```bash
# 让服务端做 dry-run 校验，不真正创建
kubectl apply -f deployment.yaml --dry-run=server

# 用 kubeconform 离线校验
kubeconform -strict -summary deployment.yaml
```

⚠ **关键认知**：`--dry-run=server` 会走完整的认证、准入和 schema 校验，比客户端 `--dry-run=client` 更严格。CI 里建议两者都用：client 快速失败，server 做最终把关。

# 3. 版本兼容

## 3.1 YAML 1.1 与 1.2 的差异

YAML 有两个广泛使用的规范版本，差异集中在类型解析：

| 特性 | YAML 1.1 | YAML 1.2 |
|------|----------|----------|
| 布尔 | `yes/no/on/off/y/n` 是布尔 | 只有 `true/false` |
| 八进制 | `0777` | `0o777` |
| 六十进制 | `1:30` 是数字 90 | 不再支持 |
| 合并键 `<<` | 扩展支持 | 非核心规范，但解析器普遍支持 |
| null | `~`/`null`/空 | 相同 |

Norway problem 的根源就是 1.1 的宽泛布尔规则。

## 3.2 解析器的差异

同一个文件，不同库可能读出不同结果：

| 库 | 语言 | 默认规范倾向 |
|----|------|--------------|
| PyYAML | Python | 接近 YAML 1.1 |
| SnakeYAML | Java | 可配置，需注意版本 |
| js-yaml | JS | 默认更接近 1.2 core schema |
| ruamel.yaml | Python | 可切换 1.1/1.2 |
| Go yaml.v3 | Go | 大体遵循 1.2，保留部分兼容 |

★ **关键认知**：不存在"所有解析器都一致"的保证。跨语言配置的兼容策略是**主动加引号**，把所有可能有歧义的值固定成字符串，让隐式类型规则失效。

# 4. 多文档实战

一份包含多个 K8s 资源的文件：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: demo
spec:
  replicas: 2
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
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: demo
spec:
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 80
```

处理流程：

```text
读取文件
  │
  ▼
按 --- 切分成多个文档
  │
  ▼
逐个文档做 schema 校验
  │
  ▼
逐个 apply / 部署
  │
  ▼
任一失败 → 停止或回滚
```

★ **关键认知**：多文档天然是"半结构化的事务单元"。K8s apply 多文档时，前面的资源成功、后面的失败，不会自动回滚前面的。需要原子性时，用 Helm 或 GitOps 工具统一管理。

# 5. 常见错误

| 错误 | 表现 | 修复 |
|------|------|------|
| `---` 未独占一行 | `---kind` 变成字符串 | 换行写 `---` |
| 单文档 API 读多文档 | 只解析第一个 | 用 `loadAll`/`safe_load_all` |
| 空文档未过滤 | 得到 `null` 元素 | 结果里跳过 `None` |
| schema 用错 draft | 关键字不生效 | 对齐 `$schema` 与校验器支持 |
| 以为 1.1 和 1.2 一致 | 布尔/八进制结果不同 | 有歧义的值加引号 |
| `%YAML` 写在中间 | 被当普通文本 | 放到流开头 |

# 6. 动手练习

1. 写一个包含三个文档的 YAML（两个 ConfigMap、一个 Secret），用 Python `safe_load_all` 读取并打印文档数量。

2. 用 JSON Schema 约束一个对象：`name` 必填且为字符串、`port` 必填且为 1-65535 的整数，然后用 `ajv` 或 `check-jsonschema` 校验一份合法和一份非法文件。

3. 分别在 PyYAML 和 js-yaml 里解析 `a: NO` 与 `b: 0777`，比较两者结果，解释差异来自哪个规范版本。

# 7. 小结

| 概念 | 要点 |
|------|------|
| `---` | 文档开始/分隔，独占一行 |
| `...` | 文档结束，通常可省 |
| 多文档读取 | 用 `loadAll`/`safe_load_all` |
| Schema | YAML 校验用 JSON Schema |
| 校验工具 | yamllint 管风格，schema 管结构 |
| 版本 | 1.1/1.2 类型规则不同，有歧义就加引号 |

一句话原则：**多文档靠标记切分，结构正确靠 schema 兜底，版本差异靠显式引号消除**。下一篇进入 Kubernetes 与 CI 实战，把前面学的语法、类型、复用全部用真实配置串起来，并集中演示缩进错误。
