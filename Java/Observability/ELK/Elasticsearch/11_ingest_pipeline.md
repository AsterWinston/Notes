# 1. 概述

Ingest Pipeline 是 Elasticsearch 内置的**文档预处理框架**：文档在写入索引之前，先经过一串 processor 的流水线处理——补字段、解析非结构化文本、转换类型、地理富化、脱敏删除——然后才被索引。它的出现解决了一个长期痛点：过去这类"入库前加工"必须依赖 Logstash 这个额外进程，而 Ingest 把它做成了集群内功能，写入链路少一跳、少一套系统要运维。

```text
Document → Ingest Node (默认所有节点都是) → Ingest Pipeline → Index
                                  │
                                  ├── Set Processor(设置字段)
                                  ├── Grok Processor(正则解析)
                                  ├── Date Processor(日期转换)
                                  ├── Convert Processor(类型转换)
                                  ├── Script Processor(Painless 脚本)
                                  ├── GeoIP Processor(IP 地理富化)
                                  └── Remove Processor(删除字段)
```

**执行位置值得强调**：pipeline 运行在协调节点或 ingest 节点的线程池（`thread_pool.write` 同族资源）上，是**同步串行**的——文档按 processor 数组顺序逐个加工，全部跑完才进入索引流程。这意味着 pipeline 里的重活（Grok 回溯、脚本循环）会直接压进写入延迟；它是"索引前的一道函数"，不是异步消息系统。

Pipeline 与索引的绑定方式有三种，按"侵入性"从高到低：

```json
# 1. 请求级: 单次写入显式指定
PUT /orders/_doc/1?pipeline=order_pipeline
{ "amount": "100.0" }

# 2. 索引级: 写进 settings, 该索引所有写入默认走这条管道
PUT /orders
{
  "settings": { "default_pipeline": "order_pipeline" },
  "mappings": { ... }
}

# 3. 全局级: 集群设置兜底(常配"命中条件"避免误伤所有索引)
PUT /_cluster/settings
{
  "persistent": {
    "default_pipeline": "common_enrich_pipeline",
    "index.default_pipeline": "..."
  }
}
```

> **注意**：`index.final_pipeline` 是另一个常被忽略的钩子——它在**普通管道之后、索引之前**再执行一道"收尾"处理（典型用途：统一脱敏、加租户标记）。排错时如果发现"字段在 default_pipeline 里 set 了却被某处删掉"，八成是 final_pipeline 干的。两者顺序是 default → final。

# 2. 创建 Pipeline

Pipeline 本身是存在集群状态里的一等对象，CRUD 都走 `_ingest/pipeline` API：

```json
PUT /_ingest/pipeline/order_pipeline
{
  "description": "订单数据预处理",
  "processors": [
    {
      "set": {
        "field": "processed_at",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "convert": {
        "field": "amount",
        "type": "double"
      }
    },
    {
      "script": {
        "source": """
          ctx.total = ctx.amount * ctx.quantity;
          ctx.year = ctx.created_at.substring(0, 4);
        """
      }
    },
    {
      "remove": {
        "field": ["temp_field", "internal_notes"],
        "ignore_missing": true
      }
    }
  ]
}

# 使用 Pipeline
PUT /orders/_doc/1?pipeline=order_pipeline
{ "order_no": "ORD001", "amount": "100.0", "quantity": 3, "created_at": "2024-01-15" }
```

这套语法里有两个贯穿所有 processor 的通用约定，值得单独拎出来：**模板语法 `{{field}}`**——凡是能接受模板的字段（几乎所有 processor 的 value/if/target_field 等），都用 `{{ }}` 引用文档上下文，`{{_ingest.timestamp}}`、`{{_id}}`、`{{_index}}` 是元数据，嵌套用 `{{user.name}}`，还支持 mustache 逻辑片段；**上下文变量 `ctx`**——processor 里（脚本、条件）访问文档一律用 `ctx`，它是当前文档的轻量视图，`ctx.amount += 1` 这类写法直接改写待索引文档。注意 `ctx` 只在 pipeline 上下文里存在，查询期的 Painless 用的是 `doc`/`params`，两套 API 不要混。

> **行业实践**：processor 的顺序不是随便排的——惯例是"先清洗（convert/date/gsub）再计算（script）再富化（geoip/enrich）最后裁剪（remove/redact）"。比如例子里 `convert` 必须排在 `script` 之前：`amount` 还是字符串 `"100.0"` 时，`ctx.amount * ctx.quantity` 的结果在 Painless 里会类型异常或字符串拼接，先转 double 才是数值乘法。

# 3. Processors

processor 家族几十个，按用途分五组记忆最省力：

| 分组 | 代表 processor | 一句话职责 |
|------|---------------|-----------|
| 结构类 | append、remove、rename、copy、set、flatten | 改字段形状 |
| 解析类 | grok、json、kv、csv、xml、dissect | 把字符串拆成字段 |
| 转换类 | convert、date、uppercase/lowercase、gsub、trim | 改值类型/形态 |
| 富化类 | geoip、user_agent、enrich、script、foreach | 补外部信息 |
| 控制类 | if/on_failure/ignore_failure（所有 processor 通用）、pipeline、fail、drop | 流程编排 |

## 3.1 Set — 设置字段

```json
{
  "set": {
    "field": "env",
    "value": "production",
    "override": false          // 字段已存在时不覆盖(默认 true)
  }
}
```

set 是最简单的 processor，但有两个易错参数：`override: false` 用于"仅当字段不存在才补默认值"的场景；`ignore_empty_branch`/模板解析失败时（引用了不存在的字段）`{{missing}}` 会原样留在值里，所以补默认值建议配 `if: "ctx.env == null"` 判断而不是裸 set。

## 3.2 Grok — 正则解析

```json
{
  "grok": {
    "field": "message",
    "patterns": ["%{IP:client_ip} %{WORD:method} %{URIPATHPARAM:url} %{NUMBER:status:int} %{NUMBER:response_time:float}"]
  }
}

# 输入: "192.168.1.1 GET /api/user 200 25.5"
# 输出: { client_ip: "192.168.1.1", method: "GET", url: "/api/user", status: 200, response_time: 25.5 }
```

**Grok 是"命名正则 + 模式库"**：官方提供 300 多个预定义 pattern（`%{IP}`、`%{TIMESTAMP_ISO8601}`、`%{COMMONAPACHELOG}`），`%{NUMBER:status:int}` 里冒号后是命名捕获、再冒号是类型转换。它的价值在于解析 Nginx/Apache/Java 堆栈这类"没有 schema 的文本"，是 Beats/ES 采集日志的标准入口。代价也要讲清楚：**pattern 按数组顺序逐个尝试，第一个匹配即停；正则回溯在坏数据上可能极慢**，所以生产上要把最常见格式放第一个、用 `trace` 级别日志观察耗时、能用 dissect 的场景不要用 grok。

```json
# dissect — 无正则的分隔符解析, 快于 grok 数倍
{
  "dissect": {
    "field": "message",
    "pattern": "%{client_ip} %{method} %{url} %{status} %{response_time}"
  }
}
```

## 3.3 Date — 日期转换

```json
{
  "date": {
    "field": "created_at",
    "formats": ["yyyy-MM-dd HH:mm:ss", "ISO8601"],
    "target_field": "@timestamp",
    "timezone": "Asia/Shanghai"
  }
}
```

date 把字符串/数值时间解析成 UTC 的 `date` 值写入目标字段。**`timezone` 参数是中文场景的最大坑**：日志里 `2024-01-15 08:00:00` 若不声明时区，会被当作 UTC，数据在 Kibana 里就差了 8 小时；`formats` 按顺序尝试，注意 `HH` 与 `hh`、`MM` 与 `mm` 的大小写陷阱。**多格式解析失败默认整个文档报错进 on_failure**，所以给旧数据兜底时把 `if` 判断和 `ignore_failure` 配好，或者按日志版本分流到不同 pipeline。

## 3.4 Script — Painless 脚本

```json
{
  "script": {
    "source": """
      if (ctx.amount >= 10000) {
        ctx.risk = "high";
      } else {
        ctx.risk = "normal";
      }
    """
  }
}
```

script processor 是"其他 processor 覆盖不了时"的逃生舱，Painless 语法、可配 `lang: painless`、可用 `params` 传参、可引用 `ctx.`、`_ingest.` 上下文。使用纪律：**只做轻量变换，避免大循环与字符串正则重活**；复杂逻辑（需要查表、调外部 HTTP）不要塞在索引路径上，挪到 enrich/离线任务。**foreach** 是 script 的兄弟场景：对数组字段逐元素应用子 processor，常用于 tags 清洗、log.lines 展开。

## 3.5 Pipeline — 嵌套管道

```json
{
  "pipeline": {
    "name": "another_pipeline"
  }
}
```

pipeline processor 让管道组合成树：公共管道（打标、时间戳、租户路由）在外层，业务专属管道在内层，复用与分治一次性解决。嵌套深度没有硬限制，但**每一层都会复制一次 ctx**，超深嵌套 + 大文档的内存放大要压测确认；循环引用（A 调 B 调 A）执行时报错，创建时不检查。

## 3.6 GeoIP / User-Agent / Enrich — 富化三件套

```json
{
  "geoip": {
    "field": "client_ip",
    "target_field": "geo",
    "properties": ["city_name", "country_name", "location"]
  }
},
{
  "user_agent": {
    "field": "agent",
    "target_field": "user_agent",
    "regex_file": "ua-regexes.yml"
  }
},
{
  "enrich": {
    "policy_name": "user_lookup",
    "field": "user_id",
    "target_field": "user"
  }
}
```

geoip 内置 MaxMind 库，8.x 首次使用需 `PUT /_ingest/geoip/download_pipelines` 类配置下载数据库（或离线放 mmdb 到 config）；user_agent 解析浏览器/OS/设备。**enrich processor 是"用索引当维表"**：先建一个 `enrich` 策略（源索引做 lookup/match/geo 三种模式），主写入时按某个键去查另一张索引，把命中的字段贴进文档——相当于流式 join，常见于"按 user_id 补部门/等级、按 IP 补自建机房归属"。注意 enrich 走的是**集群状态里的专用索引副本**，更新维表后策略要重建（`PUT /_enrich/policy/x/_execute`）才生效，不是实时 join。

# 4. 条件执行

每个 processor 内建三段控制： `if` 条件、`on_failure` 失败分支、`ignore_failure` 兜底。

```json
{
  "processors": [
    {
      "set": {
        "if": "ctx.amount != null && ctx.amount >= 10000",
        "field": "review_required",
        "value": true
      }
    },
    {
      "set": {
        "if": "ctx.status == 'rejected'",
        "field": "reject_reason",
        "value": "manual",
        "on_failure": [
          {
            "set": {
              "field": "error",
              "value": "failed to process rejected order"
            }
          }
        ]
      }
    }
  ]
}
```

`if` 里写 Painless 布尔表达式，可用 `ctx`、`_ingest` 和元数据；别忘了 Painless 访问不存在字段返回 null，条件里裸 `ctx.foo.bar` 会 NPE，要 `ctx?.foo?.bar` 或先判空。**on_failure 的上下文里 `_ingest.on_failure_message` 携带原始报错**，这是排错关键——把 `{"set": {"field": "ingest_error", "value": "{{{_ingest.on_failure_message}}}"}}` 放进每个易挂 processor 的 on_failure，坏文档带着错误原因落进索引，而不是整条管道静默失败。

```json
# drop: 条件丢弃(不进索引, 不报错)
{ "drop": { "if": "ctx.level == 'DEBUG'" } }

# fail: 主动把文档变成"失败"并带自定义原因, 常用于校验
{ "fail": { "if": "ctx.amount == null", "message": "amount is required" } }
```

> **注意**：drop 的文档在 bulk 响应里表现为 `result: noop`（200），不计错误——日志管道靠它做采样/降噪没问题，业务数据千万别"默认 drop"，否则丢得无声无息。校验不过应当 `fail`（客户端拿到明确错误码）而不是 drop。

# 5. Pipeline 调试

```json
# 模拟处理(不实际索引): docs 可传多条验证批量行为
POST /_ingest/pipeline/order_pipeline/_simulate
{
  "docs": [
    {
      "_source": { "order_no": "ORD001", "amount": "100.0", "quantity": 3 }
    }
  ]
}

# 带 verbose: 返回每个 processor 执行后的中间态与耗时
POST /_ingest/pipeline/order_pipeline/_simulate?verbose
{ "docs": [ { "_source": { "message": "192.168.1.1 GET / 200" } } ] }

# 未落库的管道也能模拟: 直接在请求体里给 pipeline 定义
POST /_ingest/pipeline/_simulate
{
  "pipeline": { "processors": [ { "json": { "field": "payload" } } ] },
  "docs": [ { "_source": { "payload": "{\"a\":1}" } } ]
}
```

**`_simulate` 是 pipeline 开发的单元测试**：不写入任何数据，完整走一遍处理器链，返回最终 `_source`（verbose 模式给逐 processor 轨迹）。**上线纪律：任何 pipeline 改动必须先在 _simulate 里用"真实样本 + 已知脏样本"各过一遍**——脏样本尤其重要，因为 on_failure 分支只有异常数据才能触发。另两个实用 API：`GET /_ingest/processor/grok`（内置 pattern 查表）与 `GET /_nodes/stats/ingest`（看各节点 ingest 耗时与失败数）。

```bash
# 查看管道列表与定义
GET /_ingest/pipeline
GET /_ingest/pipeline/order_pipeline

# 删除管道(索引 default_pipeline 引用了它要先清理引用, 否则写入报错)
DELETE /_ingest/pipeline/order_pipeline
```

> **行业实践**：pipeline 配置纳入 Git 管理，用 CI 把"PUT 创建 + _simulate 断言"做成发布前脚本。管道定义存在集群状态里，改一次全集群生效，没有版本历史，误删引用它的索引配置会引发批量写入失败——把它当"代码"而不是"一次性手工配置"是血泪教训。

# 6. vs Logstash

| 维度 | Ingest Pipeline | Logstash |
|------|----------------|----------|
| 部署 | ES 内置，零额外组件 | 独立 JVM 进程/集群 |
| 执行位置 | 索引路径上，同步 | 索引前异步，有内存队列缓冲 |
| 性能 | 中（占用写线程池与堆） | 高（独立资源可横向扩） |
| 输入生态 | 仅 HTTP 写入时触发 | JDBC/Kafka/S3/Filebeat 上百种 input |
| 有状态处理 | ✗（无聚合窗口） | ✓（aggregate 插件按 trace_id 拼日志） |
| 重试/背压 | 写入失败由客户端负责 | pipeline 级 retry + 持久队列（PQ） |
| 外部输出 | ✗ 只进 ES | ✓ 可多路 fan-out（S3/Kafka/ES） |
| 功能广度 | Grok/date/enrich 等基础处理 | ruby/条件路由/复杂 ETL |

选型思路按问题拆：

- **数据源是 Filebeat/Beats，只做基础解析？** 优先 Ingest：Beats 模板 + `default_pipeline` 已经覆盖 90% 的 Nginx/系统日志场景，少运维一套 Logstash；
- **需要缓冲削峰、多目标输出、跨行聚合、或从非 HTTP 源拉数?** 上 Logstash：它本质是数据管道，Ingest 只是索引前函数，没有队列语义；
- **两者混用也正常**：Logstash 负责采集-路由-兜底，Ingest 负责写入瞬间的统一富化（如 geoip），避免两套维表。

> **注意**：Ingest 跑在写入路径上意味着**它的失败与延迟会直接暴露给客户端**，压测要把"带 pipeline 的 bulk 吞吐"单独测——同一份 Grok 规则在 Logstash 独立节点上慢一点无所谓（队列顶着），在 Ingest 里就是写入 TPS 下降。

# 7. 小结

| 主题 | 核心结论 |
|------|---------|
| 定位 | 索引前同步流水线，跑在 ingest 节点；绑定三级：请求 ?pipeline / 索引 default（含 final_pipeline）/ 集群全局 |
| 通用约定 | `{{模板}}` + `ctx` 上下文；processor 顺序：清洗→计算→富化→裁剪 |
| 家族 | set/convert/grok/dissect/date/script/foreach/geoip/enrich/drop/fail + 全 processor 通用的 if/on_failure |
| 解析选型 | 格式规整用 dissect（比 grok 快），乱格式用 grok,pattern 数组顺序即匹配顺序 |
| 容错 | on_failure 里记 `{{{_ingest.on_failure_message}}}`，脏数据带原因入库而不是整链静默失败 |
| 调试 | `_simulate`（带 verbose/不落库）是 pipeline 的单元测试，上线必过 |
| vs Logstash | Ingest=索引路径上的函数（轻量、零运维、无队列）；Logstash=独立数据管道（缓冲/多源/多目标/有状态） |

管道解决了"进来之前把数据洗干净"；下一篇（12_search_performance）转向另一头——数据进去之后，怎么把它高效、正确地查出来。
