
前面几篇分别讲了 PQ、性能、安全与多管道，本篇把它们收拢成一套"上线前照着走、出事先照着查"的实践规范：配置怎么管、字段怎么定、失败怎么分类处理、变更怎么灰度回滚、容量怎么扩、指标看哪些。这些条目没有一条是"新技术"，但生产事故复盘里八成能找到它们被跳过的身影。

# 1. 管道设计原则

```text
✓ DO:
  - 一个源一条管道：Nginx / App / Syslog 分开
  - 先过滤再处理: if [type] == "nginx" 减少不必要的 Grok
  - 尽可能用 Dissect 代替 Grok（5-10x 性能）
  - 输出到不同索引：nginx-*, app-*, syslog-*
  - 给字段起有意义的名字

✗ DON'T:
  - 不要所有规则堆在一个管道
  - 不要在 Filter 中使用多个 Grok 匹配同一 message
  - 不要让 Grok 无锚定匹配（^ 开头）
  - 不要在生产用 stdout { codec => rubydebug }
```

几条展开说。**"一个源一条管道"的本质是控制故障传导范围**：grok 模式改错、geoip 数据库损坏、某业务 mapping 爆炸，影响面都被圈在自己管道内，重启/reload 也只动一条流。**"先过滤再处理"决定 CPU 账单**——filter 是按顺序对每个事件执行的，把最便宜的条件判断放最外层，让绝大多数事件早退出，比任何 JVM 调参都有效。**`stdout { codec => rubydebug }` 在生产是隐形炸弹**：它不只是打印一行文本，rubydebug codec 会为每个事件构造调试字符串，高吞吐下 stdout 管道阻塞还会反压整条流水线；调试用 file 输出到 `/tmp` 抽样，或用 `-v` 级别日志替代。

> **注意**："给字段起有意义的名字"在 8.x 有具体含义——**默认对齐 ECS**（`@timestamp`、`message`、`host.name`、`client.ip`、`http.response.status_code`）。自造字段名（`ip`、`status`、`time`）短期省事，长期会在 Kibana 可视化、告警规则、跨源关联三处同时付利息：每个新看板都要重新理解一遍你的私有词表。

# 2. Grok vs Dissect 选择

```text
分隔符明确 + 结构固定?
  ├── 是 → Dissect（快 10 倍！）
  └── 否 → Grok（用 ^ 锚定）

示例:
  "2024-01-15 10:30:00 INFO com.example.Service - user created"
  → Dissect: "%{ts} %{level} %{class} - %{msg}"

  "192.168.1.1 - - [15/Jan/2024:10:30:00 +0800] \"GET /api/user HTTP/1.1\" 200 45"
  → Grok: COMBINEDAPACHELOG
```

判断标准可以再细一层：**dissect 要求字段位置和分隔符都固定**，不认"可选字段"，遇到长度变化的空格分隔会错位；grok 用正则容忍格式漂移，代价是回溯。一条实用组合策略是 **dissect 打底 + grok 兜底**：

```ruby
dissect {
  mapping => { "message" => "%{ts} %{+ts} %{level} %{class} - %{msg}" }
  tag_on_failure => ["_dissectfailure"]        # 错位打 tag，不丢数据
}
if "_dissectfailure" in [tags] {
  grok { match => { "message" => "^%{TIMESTAMP:ts} %{LOGLEVEL:level} ..." } }
}
```

**设计考量**：

- **格式契约优先**：能让上游应用改日志格式的，就不要在管道里堆正则——结构化输出（JSON lines）能让 Grok/Dissect 双双下岗
- **老日志、第三方产品日志（改不了）→ grok**；自家新服务 → 直接打 JSON
- **同一 message 上串行跑多个 grok 是反模式**，改为单块多候选数组，命中即停

# 3. 错误处理

失败要**分类处理，而不是一律打 tag 了事**。三类失败的归属完全不同：

```ruby
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    tag_on_failure => ["_grokparsefailure"]
  }

  # 仅处理成功的事件
  if "_grokparsefailure" not in [tags] {
    date { match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ] }
  }

  # 失败事件单独处理
  if "_grokparsefailure" in [tags] {
    mutate {
      add_tag => ["unparsed"]
      add_field => { "original_message" => "%{message}" }
    }
  }
}

output {
  if "unparsed" in [tags] {
    file {
      path => "/var/log/logstash/unparsed.log"
    }
  }
}
```

| 失败类型 | 检测手段 | 处置 | 去向 |
|----------|----------|------|------|
| 解析失败 | `_grokparsefailure` / `_dissectfailure` / `_jsonparsefailure` tag | 保留原文打 tag，降级写入 | 隔离索引（`logs-parsefailure-*`），保留可查性 |
| 字段异常（类型冲突） | ES 400 + DLQ | `dead_letter_queue.enable: true` 自动落盘 | DLQ 目录，定期重放 |
| 下游不可用（503/超时） | output 重试计数 | 指数退避重试，靠 PQ/Kafka 缓冲 | 不丢弃，等待恢复 |

**tag 体系要当作接口来管理**：内置 tag（下划线开头）与自定义 tag（如 `unparsed`、`sampled`、`from_archive`）分开命名，下游所有条件判断只依赖"稳定的 tag 契约"，不直接猜别人会打什么 tag；给每个 tag 在配置仓库 README 里登记含义与产生位置。file 兜底解析失败的做法适合小流量，量大时隔离索引更好——`unparsed.log` 本身会成为无人处理的垃圾山。

> **行业实践**：把"解析失败率"做成一级指标：`_grokparsefailure` 占比超过 1% 说明上游格式在漂移，应当主动排查而不是继续加兜底规则；每加一条 grok 兜底，都在为"日志格式失控"这种债续期。

# 4. 配置管理

```bash
logstash/
  ├── config/
  │     ├── pipelines.yml           # 管道定义
  │     ├── logstash.yml            # 全局配置
  │     └── jvm.options             # JVM 参数
  ├── conf.d/
  │     ├── beats-input.conf        # Input
  │     ├── nginx-filter.conf       # Nginx 处理
  │     ├── app-filter.conf         # 应用处理
  │     └── es-output.conf          # Output
  └── patterns/
        └── custom.grok             # 自定义 Grok
```

```bash
# 配置校验（CI/CD）
bin/logstash -f /etc/logstash/conf.d --config.test_and_exit

# 版本管理
git init && git add . && git commit -m "Logstash config v1.0"
```

配置即代码的三件套：**Git 版本化 + CI 校验 + 部署下发**。CI 里除 `--config.test_and_exit`（只校验语法与插件参数，不启动管道）外，建议再加两级：**样例数据回归**——用一个 fixture 管道跑历史样本日志，对比 grok 前后的字段集合 diff，防"改了 A 模式打碎了 B 索引"；**mapping 兼容性检查**——新增字段先过 ECS/命名规范，同名异型字段（昨天 `bytes` 是 long 今天变 keyword）是 ES 写入拒绝的头号来源，CI 里拿样例对照索引模板校验一遍。自定义 grok pattern 放 `patterns/` 目录、`grok { patterns_dir => [...] }` 显式引用，不要内联散落在配置里。

> **注意**：**keystore 不进 Git，配置文件里的 `${VAR}` 占位符才进 Git**；`logstash.keystore`、TLS 私钥、DLQ 内容都属于"带密码学价值"的产物，误提交一次就等于泄露一次。`.gitignore` 里从第一天就写好。

# 5. 生产配置

```yaml
# logstash.yml
pipeline.workers: 8
pipeline.batch.size: 250
pipeline.batch.delay: 25
queue.type: persisted
queue.max_bytes: 4gb
queue.checkpoint.writes: 1024
dead_letter_queue.enable: true
dead_letter_queue.max_bytes: 2gb
config.reload.automatic: false       # 生产建议手动重启
log.level: warn
```

逐条的理由在第 9、11、13 篇都展开过，这里只强调容易被"调优冲动"破坏的三条：**PQ + DLQ 双开是数据不丢的下限配置**，性能问题应该动 workers/batch，不该动这两个开关；**`config.reload.automatic: false`** 配合 CI 下发 + 受控重启/reload，把"生产配置变更"纳入发布流程；**`log.level: warn`** 之后要确保 `Pipeline configuration had validation errors`、DLQ 写入、429 拒绝这些关键告警仍会输出（它们都是 warn/error 级），debug/info 留在预发。

# 6. 灰度与回滚

配置变更的风险排序：**grok 模式 > 索引路由/模板 > workers/batch > 其他**。按风险选灰度手段：

| 变更类型 | 灰度方式 | 回滚方式 |
|----------|----------|----------|
| 解析逻辑（grok/dissect） | 影子管道：新旧模式各跑一份，写对比索引验证字段覆盖率 | 切回旧配置 reload |
| 索引/模板变更 | 新模板指向新别名，先双写再切读 | 别名指回旧索引 |
| 参数（batch/workers/PQ） | 单机金丝雀：一台实例改参数压测对比 eps/GC/延迟 | 重启恢复 |
| 管道拆分/合并 | Pipeline-to-Pipeline 改 `send_to` 指向 | 改回一行 send_to |

影子管道是最常用也最被低估的手法——**解析类变更永远无法靠语法校验保证正确**（grok 匹配成功但字段抓错位置是最常见的事故形态），用 `if [tags]` 或双 output 把同一批流量复制一份到新旧两套逻辑，各写隔离索引，Kibana 里对比一小时再切正式。回滚能力的底线是：**旧索引数据还在、旧别名没删、旧配置在 Git 里可直接 revert**；三者缺一，回滚就变成二次事故。

> **设计考量**：Logstash 没有内建流量百分比灰度（不像 Istio 那种按比例分流），**"按管道灰度"是它的天然粒度**——新配置只接管一条管道（一个新业务/一个低价值数据源），稳定一周再推广到高价值管道。把"管道"当成灰度单元设计，比在单管道里做条件分流简单得多也安全得多。

# 7. 容量与水平扩展

Logstash 无共享状态（PQ 目录属于单实例私有），**水平扩展就是"复制部署 + 上游分摊"**，扩展方式取决于 Input 类型：

| Input 类型 | 扩展方法 | 注意 |
|-----------|----------|------|
| beats | LB（四层）挂多实例 | Filebeat 自动重试；会话粘性不需要 |
| kafka | 同 consumer group 加实例 | **实例数 ≤ 分区数**，多出的实例空转 |
| http/tcp | LB 多后端 | 注意各实例独立 PQ 容量 |
| file（本地文件） | 不可横向扩 | 属于单机采集形态，应换 Beats |
| jdbc | 加实例要分片查询 | 需自管分片键，避免重复拉取 |

容量规划按"**峰值 eps ÷ 单实例压测吞吐**"算实例数，再乘 1.5 冗余（保留一台故障余量 + 高峰缓冲）；Kafka 场景下同步核对分区数与消费实例数的匹配。**纵向扩（加核加内存）收益很快见顶**——堆超 31g 有压缩指针悬崖，grok 的 CPU 效率与核数近似线性但受 GC 抖动影响，所以优先加实例。ES 端别忘了同步扩容：Logstash 加机器只是把压力更快地推到 ES，真正接不住时看第 11 篇的 ES 侧清单（refresh_interval、副本、分片数）。

> **行业实践**：给每类数据源定"**降级预案**"比堆容量更有效——高峰期可丢的 debug 日志在 Filebeat 侧按文件排除、次要业务管道降 workers 让路给核心管道、Kafka 侧拉长 retention 让 Logstash 慢慢消化。分级处理永远优于无差别扩容。

# 8. 监控指标基线

| 指标 | 来源 | 健康基线 | 告警阈值参考 |
|------|------|----------|--------------|
| 吞吐 `events.out` rate | `_node/stats` | 与业务峰值曲线吻合 | 环比跌 30% 告警 |
| 队列堆积 `queue.events_count` | `_node/stats` | 稳态接近 0 | > 10000 或持续上升 |
| PQ 磁盘水位 | node_exporter 目录监控 | < 70% `max_bytes` | 75% warn / 90% crit |
| 解析失败率 `_grokparsefailure` | ES 聚合 / pipeline 计数器 | < 1% | > 5% |
| DLQ 大小 | 目录 + stats API | 恒为 0（稳态） | 非空即告警 |
| JVM heap | `_node/stats/jvm` | 锯齿平稳 < 75% | > 85% 持续 / Old GC 频繁 |
| GC 停顿时间 | `_node/stats/jvm` | Young GC 平次 < 200ms | 单次 Old GC > 2s |
| ES bulk 延迟 / 重试 | outputs 插件统计 | 延迟稳定、retries=0 | retries 增量 > 0 |
| 进程存活 | `_node` 探活 | 全部 UP | 失联 1 分钟 |

基线的含义是"**先观测两周拿到你自己的正常曲线，再设阈值**"，抄别人的数字没有意义；但 DLQ 非空、`events.in` 与 `events.out` 速率长期背离这两条例外，它们是**无条件异常**。落地组合通常是：Prometheus + logstash-exporter 抓 `_node/stats` 做曲线与告警，Kibana Stack Monitoring 看趋势，脚本 cron 兜底检查 DLQ 目录。

# 9. DON'T

```text
✗ 不要在 Filter 中重复解析 (先 Grok 再 JSON 再 Grok)
✗ 不要用多个 GREEDYDATA 捕获
✗ 不要在循环中使用 ruby filter (性能杀手)
✗ 不要忘记设置 ES ILM (索引会无限增长)
✗ 不要用 Memory Queue 跑生产
✗ 不要让 pipeline.batch.size > 500 (内存风险)
✗ 不要忽略 DLQ 中的死信数据
```

再补几条本篇范围内的：**不要在没开 `--config.test_and_exit` 的情况下直接 reload**；**不要让多条管道共用一个 ES 写入超管账号**（最小权限见第 12 篇）；**不要把管道命名成 `main`/`pipeline2`/`test` 这类无语义名字**——事故之夜你是靠管道名找到流量归属的；**不要用 git 仓库存 keystore/证书私钥**；**不要让 grok 自定义 pattern 与内置 pattern 同名**（加载顺序会静默覆盖，行为随版本漂移）。

# 10. 汇总清单

| 类别 | 上线前检查项 | 对应篇章 |
|------|--------------|----------|
| 可靠性 | PQ persisted + DLQ 开启；checkpoint 默认；上游有 registry/Kafka | 9 |
| 性能 | workers=核数起步；batch 125-250；grok 锚定/dissect 优先；压测过 | 11 |
| 安全 | keystore 管密码；TLS 全链路；ES 账号最小权限；API 端点收口 | 12 |
| 结构 | 按业务拆管道；配置按阶段编号；命名与索引族对齐 ECS | 13 |
| 变更 | Git 管理 + CI test_and_exit + 样例回归；影子管道灰度；可回滚 | 本篇 4/6 |
| 容量 | 峰值÷压测吞吐×1.5；Kafka 分区≥实例；ES 同步扩容；降级预案 | 本篇 7 |
| 监控 | 吞吐/堆积/失败率/DLQ/heap 五件套基线化；DLQ 非空即告警 | 本篇 8 |
| 文档 | tag 契约表、管道-业务映射表、值班 runbook | 本篇 3 |

# 11. 小结

| 原则 | 一句话 |
|------|--------|
| 管道即故障域 | 按源拆管道，reload 与故障都只圈在一条流里 |
| 解析从简 | JSON > Dissect > Grok，能改上游就别堆正则 |
| 失败分级 | 解析失败进隔离索引，写失败进 DLQ，下游抖动靠缓冲重试 |
| 配置即代码 | Git + CI 校验 + 影子灰度，生产关自动 reload |
| 水平扩展优先 | 无状态复制部署，Kafka 分区数是扩展天花板 |
| 先基线后告警 | 抄的阈值不可信，DLQ 非空与 in/out 背离除外 |
