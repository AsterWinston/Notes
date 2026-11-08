
从 7.2 开始，一个 Logstash 进程可以**同时运行多条相互隔离的管道**：每条管道有自己完整的 Input→Filter→Output 拓扑、独立的 workers 线程组、独立的 queue 和独立的热加载开关，唯一共享的是同一个 JVM（堆、GC、CPU 配额）。多管道是 Logstash 从"一个 ETL 脚本"走向"一个 ETL 服务"的分水岭——它让 Nginx 流量爆炸不再能拖死支付日志的处理。

# 1. 进程内并行的多条流水线

## 1.1 单管道 vs 多管道

| 维度 | 单管道 | 多管道 |
|------|--------|--------|
| 定义方式 | 命令行 `-f` 指定配置文件 | pipelines.yml 声明多个管道 |
| Workers | 全流量共享一批线程 | 每条管道独立配额 |
| Queue | 一个队列 | 各自独立（可分别配 PQ） |
| 故障域 | 一处慢/堵，全线卡 | 隔离在单条管道内 |
| 配置 reload | 影响全部流量 | 只影响该管道 |

# 2. pipelines.yml：管道的注册表

```yaml
# /etc/logstash/pipelines.yml

# 管道 1: 接收 Filebeat 数据，按类型路由到不同管道
- pipeline.id: beats-input
  path.config: "/etc/logstash/conf.d/beats-input.conf"
  pipeline.workers: 2

# 管道 2: Nginx 日志处理
- pipeline.id: nginx-processing
  path.config: "/etc/logstash/conf.d/nginx.conf"
  pipeline.workers: 4
  queue.type: persisted

# 管道 3: 应用日志处理
- pipeline.id: app-processing
  path.config: "/etc/logstash/conf.d/app.conf"
  pipeline.workers: 6

# 管道 4: 错误日志 -> 告警
- pipeline.id: error-alerting
  path.config: "/etc/logstash/conf.d/error.conf"
  pipeline.workers: 2
```

关键规则有四条。**其一**，管道 id 必须唯一，且是日志、监控、API 里识别管道的 key，改名等于新增管道（PQ 数据还留在旧 id 目录下）。**其二**，pipelines.yml 里的任何设置**覆盖** logstash.yml 的全局值，粒度是"每管道"——`pipeline.workers`、`queue.type`、`dead_letter_queue.enable` 都能按管道单独设置；`path.data`、`path.logs` 这类进程级设置则不能在这里重复指定。**其三**，pipelines.yml 不存在或为空、且命令行没有 `-f` 时，Logstash 回退到 `path.config` 跑单条 `main` 管道。**其四**，`path.config` 可以指向**目录**，目录内所有 `*.conf` 按字母序合并为一条管道（下一节展开）。

```bash
# 启动时显式指定 pipelines 文件（默认读 $LS_HOME/config/pipelines.yml）
bin/logstash --pipelines.path /etc/logstash/pipelines.yml

# API 查看管道与单管道统计
curl -s localhost:9600/_node/pipelines?pretty
curl -s localhost:9600/_node/stats/pipelines/nginx-processing?pretty
```

# 3. 模块化配置拆分

```text
/etc/logstash/conf.d/
  ├── 01-input.conf        # Input 配置
  ├── 02-filter.conf       # Filter 配置
  └── 03-output.conf       # Output 配置
```

```ruby
# 01-input.conf
input {
  beats { port => 5044 }
}

# 02-filter.conf
filter {
  if [type] == "nginx" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  }
}

# 03-output.conf
output {
  elasticsearch { hosts => ["http://es:9200"] }
}
```

Logstash 按字母顺序加载 `conf.d/*.conf`，合并为一个管道配置。合并语义是**语法级拼接**而非覆盖：三个文件里的 input/filter/output 块会合并成同一条管道的三段，各文件里独立的 `filter { }` 块会按文件名顺序依次执行。数字前缀 `01-02-03` 的作用就是**确定这个执行顺序**——input 先注册、filter 按序串联、output 收尾。

实践上更常见的拆分是**按管道目录拆**，让 pipelines.yml 的 `path.config` 指向各自目录：

```text
/etc/logstash/pipelines/
  ├── beats-input/*.conf
  ├── nginx-processing/*.conf     # 01-input / 02-grok / 09-output
  └── app-processing/*.conf
```

这里有个隐藏坑：**合并后的配置在 `--config.test_and_exit` 校验时报错行号是"合并文件"的行号**，不是原始文件的行号，排查时要按前缀文件重新对位。

> **注意**：字母序排序用的是**字符串比较**，`10-x.conf` 会排在 `2-y.conf` 前面。前缀编号要留零（`01-`、`02-`），或者干脆用功能名 + 阶段前缀（`a-input.conf`、`b-filter.conf`、`z-output.conf`），别指望 1、2、…、10 自然排序。

# 4. 条件分支（单管道内）

多管道之外，最朴素的隔离手段是**一条管道内的 if-else**——它没有隔离性，但有极低的运维成本，中小规模完全够用：

```ruby
input {
  beats { port => 5044 }
}

filter {
  if [type] == "nginx" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
    geoip { source => "clientip" }
    mutate { remove_field => ["message", "timestamp"] }
  }
  else if [type] == "app" {
    json { source => "message" }
  }
}

output {
  if [type] == "nginx" {
    elasticsearch {
      hosts => ["http://es:9200"]
      index => "nginx-%{+YYYY.MM.dd}"
    }
  }
  else if [type] == "app" {
    elasticsearch {
      hosts => ["http://es:9200"]
      index => "app-%{+YYYY.MM.dd}"
    }
  }

  if "error" in [tags] {
    email {
      to => "alert@example.com"
      subject => "[ERROR] %{type}"
    }
  }
}
```

这段配置里有三个容易踩的点。第一，`[type]` 是 Filebeat 侧 `fields.type` 传上来的自定义字段，8.x ECS 模式下更推荐 `[log][file][path]` 或 `[service][name]` 这类 ECS 字段做条件——字段名要与 Filebeat 版本约定对齐，条件才不会永远为假。第二，**output 里的多个分支不互斥**：最后那个 `if "error" in [tags]` 会和前面的 elasticsearch 同时执行，事件被处理两次（一次写索引、一次发邮件），这是刻意设计（多播），但要清楚它不是 else-if 语义。第三，无 else 兜底时，未匹配的 type **静默丢弃**，规范做法是加 `else { mutate { add_tag => ["unrouted"] } }` 并输出到隔离索引。

# 5. 多管道 vs 条件分支

| 维度 | 多管道 (pipelines.yml) | 条件分支 (if-else) |
|------|----------------------|-------------------|
| 隔离性 | ★★★★★（独立 Workers/Queue） | ★★☆☆☆ |
| 运维复杂度 | 高 | 低 |
| 资源利用 | 可按需分配 Worker | 共享 Workers |
| 适用 | 大规模、多源异构数据 | 中小规模、同构数据 |

取舍的判断清单：**要不要隔离，看故障传导路径是否可接受**。下面几种情况下条件分支的共享队列/线程会变成实际问题，应当拆管道：

- 某类数据用了 geoip/ruby 这类慢 filter，周期性拖累同管道其他流量的延迟
- 某类数据（安全日志、支付日志）有独立的可靠性 SLA，要求独立 PQ 和 DLQ 开关
- 某类数据量大且入口不同（Kafka input 与 beats input 混用），生命周期与重启策略不同
- 需要**按业务分别灰度发布**配置：拆管道后 reload 影响面是单管道，条件分支的改动是全量生效

反过来，"数据源不到五个、吞吐不敏感、同一端口进来"的中小部署，条件分支就是更优解——多管道带来的 pipelines.yml、多套目录、监控碎片化，收益抵不过成本。

# 6. 程序化管道配置

**pipelines.yml 不只是"路径清单"，它是每管道的参数覆盖点**——把 logstash.yml 里几乎所有管道级设置搬进来按管道配，这就是"程序化配置"的含义：

```yaml
# pipelines.yml
- pipeline.id: prod
  pipeline.workers: 8
  pipeline.batch.size: 250
  path.config: "/etc/logstash/conf.d/prod.conf"

- pipeline.id: staging
  pipeline.workers: 2
  pipeline.batch.size: 50
  path.config: "/etc/logstash/conf.d/staging.conf"
```

不同环境的数据用不同管道处理，资源隔离。可覆盖的常用键：`pipeline.workers`、`pipeline.batch.size`、`pipeline.batch.delay`、`queue.type` / `queue.max_bytes`、`dead_letter_queue.enable`、`config.reload.automatic` / `config.reload.interval`、`pipeline.ecs_compatibility`。

**按流量特征分配 workers**：把 workers 理解为"该管道允许的并发加工度"。CPU 密集管道给"预留核数的 30%-50%"（大 grok 任务），IO 密集管道可给到核数甚至更高（等 ES 时线程在 park）；采集/转发型轻管道给 1-2 个就够。**总量不要按"加起来等于核数"设计**——真实流量会同时冲高，留出余量，靠压测调稳态值。

## 6.1 按业务隔离管道的划分依据

划分粒度没有标准答案，但有四种常用切分依据：

| 切分依据 | 典型管道 | 说明 |
|----------|----------|------|
| 按接入协议/端口 | beats-input / kafka-input / jdbc-sync | 生命周期与扩缩容策略天然不同 |
| 按业务域 | nginx-processing / app-processing / db-audit | 索引、模板、告警、负责人对齐 |
| 按处理开销 | light-relay / heavy-grok / geoip-enrich | 慢任务隔离，防止拖垮快任务 |
| 按 SLA/可靠级 | realtime-alert / batch-archive | 分别配 PQ、DLQ、重试策略 |

经验是**先按业务域切，再对其中"慢且量大"的那一支做二次拆分**。管道数不宜过多：每条管道有独立 workers 线程与队列内存，也各自向 ES 建连接池，拆到几十条时 JVM 线程和堆内存开销会明显起来；3-10 条是常见的舒适区间，再往上应考虑拆进程。

# 7. 管道自动重载

```yaml
# logstash.yml（也可按管道覆盖）
config.reload.automatic: true
config.reload.interval: 3s
```

修改配置文件后无需重启 Logstash，自动加载新配置。机制与边界要清楚：

**重载的粒度是管道**。Logstash 检测到某管道 `path.config` 目录下文件变化（按 `reload.interval` 轮询），会**重建该管道的整个拓扑**——先按新配置构造一套全新插件实例，成功后停掉旧拓扑、切换队列连接。所以：

- 热加载**只覆盖配置文件内容**（input/filter/output 定义）；`logstash.yml`、`jvm.options`、pipelines.yml 结构的变更**不能热加载**，必须重启。
- `queue.type`、`pipeline.ordered`、`api.http.*` 这类启动期设置即使在 pipelines.yml 里改了也不会被 reload 应用。
- 有状态插件（jdbc_static 的追踪列等）重载时重新初始化状态，可能一次重复或空窗，这类管道建议关自动 reload、走受控重启。
- reload **以"整条管道"为原子单位**：新配置里任何语法错误都会导致整次 reload 失败并**继续用旧配置跑**，日志报 `Pipeline configuration had validation errors`。这是好事，但意味着你要监控这条日志，否则改动"没生效"是无声的。

reload 失败修复后，文件 mtime 变化会自然触发重试；也可手动 `POST /reload`（前提 `config.reload.api.enabled: true`，8.9+）：

```bash
curl -s -XPOST localhost:9600/reload   # 触发全部管道重载
curl -s -XPOST localhost:9600/reload/nginx-processing
```

**发布纪律比开关更重要**：生产推荐 `config.reload.automatic: false` + **先 `--config.test_and_exit` 校验、再触发 reload/重启**。开启自动 reload 意味着"配置文件被误编辑会立刻影响生产"，Git 合并后 CI 校验通过再落盘是必要前提。

> **行业实践**：常见做法是**开发/预发开自动 reload（调配置省事），生产关自动 reload，改配置走 CI 校验 → 分发文件 → `POST /reload` 或滚动重启**，兼顾"变更受控"与"不必重启进程"。

# 8. 命名与索引路由规范

管道名和索引名是运维的第一语言，**命名混乱会直接放大故障排查成本**。建议规则：

| 对象 | 规范 | 示例 |
|------|------|------|
| pipeline.id | `<域>-<职能>[-<环境后缀>]`，全小写短横 | `nginx-ingest`、`app-processing` |
| 配置文件/目录 | 与 pipeline.id 一致 | `pipelines/nginx-ingest.conf` |
| conf.d 内文件 | `<两位阶段号>-<职责>.conf` | `01-input.conf`、`09-output.conf` |
| 输出索引 | `<域>-<类型>-%{+YYYY.MM.dd}` 或 ILM 别名 | `nginx-access-%{+YYYY.MM.dd}` |
| ILM 策略 | 与索引族同名 | `app-applog-30d` |
| ES 写入角色 | `logstash_writer_<域>` | 最小权限按业务隔离 |

索引路由的常用写法（同时演示按字段选择索引，以及把路由信息放进 `@metadata` 避免落盘）：

```ruby
filter {
  # 决定路由的目标"域"，@metadata 不会被写入 ES
  if [log][file][path] =~ "/var/log/nginx/" {
    mutate { add_field => { "[@metadata][domain]" => "nginx-access" } }
  } else if [service][name] {
    mutate { add_field => { "[@metadata][domain]" => "app-%{[service][name]}" } }
  } else {
    mutate { add_field => { "[@metadata][domain]" => "misc" } }
  }
}

output {
  elasticsearch {
    hosts => ["https://es:9200"]
    index => "%{[@metadata][domain]}-%{+YYYY.MM.dd}"   # 域-日期
  }
}
```

**@metadata 是路由专用位**：任何只用来做 if 判断、拼接索引名、选输出的中间字段都放这里，不污染 `_source` 也不增加 mapping 负担。日期粒度（日/小时）要与 shard 规划匹配——一般日志按日切，超高吞吐按小时；跨业务共享别名（如 `logs-app-*`）方便 Kibana 的 index pattern 与权限管理。

> > **注意**：索引名里的 `%{...}` 字段引用缺失时，索引名会渲染出 `%{xxx}` 字面量并写入 `%{xxx}-...` 垃圾索引——ELK 事故榜常客。修复三层：数据层查这类事件的共同特征（新数据源没带该字段、grok 失败没走兜底）；配置层给路由字段加 else 兜底（如上例的 `misc`）或改用条件路由；流程层上线前用样例事件确认所有动态索引名渲染正确。

# 9. 常见误区

**误区一：pipelines.yml 与命令行 `-f` 混用**。同时指定会让 `-f` 覆盖 pipelines.yml，只跑一条管道——"我明明定义了 4 条为什么只有 1 条在跑"先查启动命令。

**误区二：多管道共用同一端口**。两条管道各自 `beats { port => 5044 }` 会 BindException 启动失败。多入口要么不同端口，要么走一条采集管道 + Pipeline-to-Pipeline 分发（见第 10 篇）。

**误区三：workers 总和等于核数**。峰值叠加就会互相挤压，总量应留余量，靠压测校准。

**误区四：把 pipelines.yml 当环境开关**。`prod` / `staging` 写进同一份 pipelines.yml 一起跑是滥用——环境差异应在部署时渲染 pipelines.yml，或者拆进程。

**误区五：热加载能改一切**。改 `queue.type`、`http.host`、往 pipelines.yml 加管道，都不生效或需重启；改前先看"该键是否支持 reload"。

**误区六：一条大 if-else 堆几百个分支**。每个事件要顺序求值，且改一处全量生效、回滚风险高——这就是"到某个规模该拆管道"的真实阈值。

# 10. 小结

| 主题 | 关键结论 |
|------|----------|
| pipelines.yml | 每管道独立 workers/queue/reload 参数，覆盖 logstash.yml；id 唯一，改名等于新建 |
| 配置拆分 | 按管道分目录 + 阶段号前缀（`01-`、`02-`）保证加载顺序；`-f` 与 pipelines.yml 二选一 |
| 隔离切分 | 首选按业务域切，再拆"慢且量大"的那一支；单进程 3-10 条为舒适区 |
| 热加载 | 粒度是"整条管道"；启动期参数变更仍需重启；生产建议关自动 + `POST /reload` |
| 命名路由 | 管道 id / 配置目录 / 索引族三者对齐；动态字段放 `@metadata`；防 `%{xxx}` 字面量 |
