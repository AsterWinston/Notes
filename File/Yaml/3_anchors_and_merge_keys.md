配置文件写到一定规模，就会遇到大量重复：几个服务共享同一段超时配置，几个 Job 共享同一段重试策略，几个环境共享同一组标签。复制粘贴能跑，但改一处要改十处，迟早漏掉。YAML 用**锚点（anchor）**、**别名（alias）**和**合并键（merge key）**解决复用问题。本篇讲清三者的语法、覆盖规则和常见误用。

# 1. 为什么需要复用

先看一个没有复用的例子：

```yaml
serviceA:
  timeout: 3s
  retries: 2
  url: https://a.example.com

serviceB:
  timeout: 3s
  retries: 2
  url: https://b.example.com

serviceC:
  timeout: 3s
  retries: 2
  url: https://c.example.com
```

`timeout` 和 `retries` 重复了三遍。想统一改成 `5s`，就得改三处，还可能漏。理想写法是把公共部分定义一次，其他位置引用它：

```yaml
defaults: &defaults
  timeout: 3s
  retries: 2

serviceA:
  <<: *defaults
  url: https://a.example.com
```

★ **关键认知**：YAML 的复用分两步——**锚点 `&` 打标签，别名 `*` 引用标签**，再把引用的映射用合并键 `<<` 展开成当前映射的字段。锚点是"定义",别名是"引用",合并键是"展开并允许覆盖"。

# 2. 锚点与别名

## 2.1 基本语法

```yaml
name: &my_name alice      # & 定义锚点，名字是 my_name
other: *my_name           # * 引用锚点，值变成 alice
```

解析后：

```yaml
name: alice
other: alice
```

锚点名可以自己取，`&` 后紧跟名字，别名用 `*` 加同名引用。

## 2.2 复用映射

锚点最常见用在映射上：

```yaml
base: &base
  timeout: 3s
  retries: 2

serviceA: *base
```

解析后 `serviceA` 得到：

```yaml
serviceA:
  timeout: 3s
  retries: 2
```

注意 `*base` 是**整体引用**，它把整个映射复制过来，不涉及覆盖。

## 2.3 复用序列与标量

锚点也能用在序列和标量上：

```yaml
ports: &common_ports
  - 80
  - 443

web:
  ports: *common_ports
```

```yaml
image: &img nginx:1.25

frontend:
  image: *img

backend:
  image: *img
```

★ **关键认知**：别名不是"指针"，而是**值的展开**。解析完成后，所有引用位置都拿到独立的值副本（概念上）。因此修改其中一处不会影响另一处——YAML 没有运行时的"共享对象"语义。

⚠ **关键认知**：锚点必须在**使用之前定义**，且只在**同一个文档内**有效。跨文档（`---` 之后）引用会报 "unknown anchor" 错误。

# 3. 合并键 `<<`

## 3.1 语法

合并键 `<<` 出现在映射内部，值是一个别名（或别名列表）：

```yaml
defaults: &defaults
  timeout: 3s
  retries: 2

serviceA:
  <<: *defaults
  url: https://a.example.com
```

解析后：

```yaml
serviceA:
  timeout: 3s
  retries: 2
  url: https://a.example.com
```

`<<` 把 `*defaults` 指向的映射里的键**合并进当前映射**。

## 3.2 覆盖规则

当前映射里的显式键会覆盖合并来的同名键，**无论 `<<` 写在前还是后**：

```yaml
defaults: &defaults
  timeout: 3s
  retries: 2

serviceA:
  <<: *defaults
  timeout: 5s          # 覆盖合并来的 3s
```

解析后 `timeout` 是 `5s`，`retries` 仍是 `2`。

```yaml
serviceB:
  timeout: 5s          # 写在 << 之前也能覆盖
  <<: *defaults
```

★ **关键认知**：合并键的优先级规则是"**本地显式键 > 合并来的键**"，与书写顺序无关。这跟很多人的直觉相反——通常以为后写的会覆盖先写的，但 YAML 规范明确本地键优先。

⚠ **关键认知**：`<<` 只对**映射**有效。合并的是"键值对",不是整个节点。用它去合并序列或标量会报错。

## 3.3 合并多个映射

`<<` 的值也可以是别名列表：

```yaml
base: &base
  timeout: 3s
  retries: 2

extra: &extra
  log: debug
  timeout: 10s

service:
  <<: [*base, *extra]
  url: https://example.com
```

多个映射合并时，**列表中靠前的优先**：

```yaml
service:          # 解析结果
  timeout: 3s     # base 的 3s 赢过 extra 的 10s（靠前优先）
  retries: 2
  log: debug
  url: https://example.com
```

★ **关键认知**：多映射合并的优先级是"**列表中越靠前优先级越高**"，与后覆盖前的直觉相反。这与单映射"本地键覆盖"规则叠加后，完整的优先级是：本地显式键 > 列表靠前的映射 > 列表靠后的映射。

## 3.4 多级合并

合并可以层层叠加，形成配置继承：

```yaml
base: &base
  timeout: 3s
  retries: 2

prod_base: &prod_base
  <<: *base
  replicas: 3

api:
  <<: *prod_base
  name: api
```

解析后 `api`：

```yaml
api:
  timeout: 3s
  retries: 2
  replicas: 3
  name: api
```

# 4. 实战：CI 配置复用

GitLab CI 里锚点非常常用：

```yaml
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
  script:
    - npm test

build:
  <<: *defaults
  script:
    - npm run build
```

两个 Job 共享镜像、安装步骤和缓存配置，只写各自不同的 `script`。

再看一个 Kubernetes 场景：

```yaml
common:
  resources: &resources
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi

deployment:
  containers:
    - name: app
      resources: *resources
```

★ **关键认知**：复用的边界是"**结构相同、值可能不同**"。如果两段配置只是长得像，但字段含义不同，强行合并会让后来人看不懂。复用要为了"同一份语义",不是为了"少写几行"。

⚠ **关键认知**：锚点与合并键在**转成 JSON 后会消失**。如果下游系统需要 JSON，合并会在转换时展开成最终值，你得到的是一份"没有复用痕迹"的完整数据。这是正常的，但要知道原始文件与转换结果不再一一对应。

# 5. 常见误用与限制

| 误用 | 结果 | 正确做法 |
|------|------|----------|
| 先 `*` 后 `&` | 报 unknown anchor | 锚点先定义再引用 |
| 跨 `---` 引用 | 报 unknown anchor | 每个文档内单独定义 |
| 用 `<<` 合并序列 | 报错或意外结构 | `<<` 只用于映射 |
| 以为后写覆盖先写 | 值不符合预期 | 记住本地键优先 |
| 别名引用后改一处 | 以为会影响另一处 | 别名是值展开，互不影响 |
| 用锚点复用不同语义 | 后期维护困难 | 只在语义相同时复用 |

还有一个需要注意的限制：**合并键是 YAML 1.1 的扩展**。YAML 1.2 规范本身没有把 `<<` 列为标准语法，但主流解析器（PyYAML、SnakeYAML、js-yaml）都支持。个别严格实现可能不认。

⚠ **关键认知**：`<<` 不是"合并任意节点"的操作符。它只把一个或多个**映射**的键并入当前映射。理解这一点，就不会出现"为什么我合并列表失败"的困惑。

# 6. 动手练习

1. 用锚点定义一份 `defaults`（含 `timeout`、`retries`），让 `serviceA` 和 `serviceB` 通过 `<<` 复用它，并让 `serviceB` 把 `timeout` 覆盖成 `10s`。

2. 定义 `base` 和 `extra` 两个映射，用 `<<: [*base, *extra]` 合并，验证同名键最终取哪个值。

3. 在 GitLab CI 里，用 `.defaults` 锚点让 `test` 和 `lint` 两个 Job 共享 `image` 与 `before_script`，只保留各自的 `script`。

# 7. 小结

| 语法 | 作用 | 关键点 |
|------|------|--------|
| `&name` | 定义锚点 | 必须先定义 |
| `*name` | 引用锚点 | 同文档内有效 |
| `<<: *name` | 合并映射 | 只对映射有效 |
| `<<: [*a, *b]` | 合并多个映射 | 靠前优先 |
| 本地键 | 覆盖合并值 | 与书写顺序无关 |

一句话原则：**锚点复用"同一份语义",合并键负责"展开并允许局部覆盖"**。下一篇进入多文档与 Schema，讲 `---`/`...` 的边界、如何用 JSON Schema 校验 YAML，以及 YAML 1.1 与 1.2 的版本差异。
