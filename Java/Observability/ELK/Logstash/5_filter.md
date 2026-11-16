# 1. Filter 层的定位

**filter 是 Logstash 的价值核心：逐条改写 Event 的加工车间**。input 把原始数据变成带着 `message` 的毛坯事件，output 只负责搬运成品，中间"拆字段、纠正时间、补外部信息、清洗瘦身"全部发生在这一层。三条结构性规则先立住：filter **按配置书写顺序串行执行**；**每个 worker 线程持有 filter 插件的独立实例**；filter 失败**打标记不崩溃**（错误处理见第 12 节）。

一个典型的加工顺序体现设计意图：先 `grok/dissect/json` 拆结构，再 `date` 纠正时间，再 `geoip/translate/useragent` 富化，最后 `mutate` 清理——**昂贵的放后面**（前面的分支与丢弃能减少进入昂贵插件的事件量），**清理放最后**（清理太早会让后面的插件取不到依赖字段）。

```text
filter {
  if [log][file][path] =~ /nginx/ {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }   # 1.拆结构
    date { match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ] } # 2.纠正时间
    geoip { source => "clientip" }                              # 3.富化
    mutate { remove_field => ["message","timestamp"] }          # 4.清理
  }
}
```

# 2. Grok — 正则解析

```text
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
}
```

grok 是"正则 + 命名捕获 + 模式库"的组合拳，把非结构化文本变成字段——Logstash 最强大也最昂贵的 filter，详见 `6_grok.md`。要点预告：能用 dissect 就别用 grok；grok 失败打 `_grokparsefailure` 标记而不是抛异常。

常用插件速查（按"该用谁"归类）：

| 需求 | 首选插件 | 失败标记 |
|------|----------|----------|
| 拆非结构化文本 | grok / dissect | `_grokparsefailure` / `_dissectfailure` |
| 拆 JSON 字符串 | json | `_jsonparsefailure` |
| 纠正事件时间 | date | `_dateparsefailure` |
| 字段增删改名/类型 | mutate | `_mutate_failed` |
| IP 转地理 | geoip | `_geoipparsefailure` |
| UA 拆分 | useragent | 返回 "Other" |
| 字典翻译 | translate | `fallback` 值 |
| 任意定制逻辑 | ruby | 异常静默 |

# 3. Mutate — 字段操作

```text
filter {
  mutate {
    # 重命名
    rename => { "old_field" => "new_field" }

    # 删除
    remove_field => ["@version", "message", "beat", "host", "input", "tags"]

    # 替换
    replace => { "env" => "production" }

    # 类型转换
    convert => { "response" => "integer" }
    convert => { "bytes" => "integer" }
    convert => { "duration" => "float" }

    # 值合并
    coerce => { "bytes" => 0 }        # 空值 → 0

    # 复制
    copy => { "clientip" => "src_ip" }

    # 字符串操作
    gsub => [ "message", "\\r\\n", " " ]
    strip => ["message", "user_agent"]
    lowercase => ["method"]
    uppercase => ["country"]
  }
}
```

mutate 是"字段瑞士军刀"：上表的操作全部在一个插件实例里链式完成（同一段配置内按固定顺序执行）。三个高频用法单独说。**`convert`** 解决 ES 动态映射的头号事故——grok 拆出来的 `response` 永远是字符串 `"200"`，第一批数据进 ES 被映射成 keyword 后，想按数值做范围查询就永远做不了；`convert => { "response" => "integer" }` 必须在入库前做。**`add_field` / `add_tag`** 是打标签的正规通道（配合 `%{field}` 插值）。**`gsub`** 的正则参数要写双反斜杠（DSL 字符串多一层转义）。

> **注意**：`convert` 失败时（字符串 `"N/A"` 转 integer）不中断管道，打标记并保留原值；`coerce` 与 `convert` 的区别正在于此——coerce 允许"转换失败时落到默认值"，脏数据大户要用它兜底。另外删 `@timestamp` 是合法但危险操作——删完文档就没有时间轴，Kibana 直接查不到；删字段前先问"下游靠什么排序"。

# 4. Date — 时间字段

```text
filter {
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z", "ISO8601" ]
    target => "@timestamp"            # 替换默认时间字段
    timezone => "Asia/Shanghai"
  }
}
```

**重要性**：`@timestamp` 是 ES 的默认时间字段，不正确设置将导致 Kibana 时间显示错误。

**date filter 的本质是"时间校正器"**：没有它，`@timestamp` 是"事件进入管道的时间"；有了它，日志里记录的事件时间被解析并**覆盖** `@timestamp`——两者在补历史数据、Kafka 积压、管道故障重放时会差几小时到几天。`match` 数组是"依次尝试多种格式"，命中即停，全部落空打 `_dateparsefailure`。

两个必踩过的坑。**其一是 timezone 的语义**：`timezone => "Asia/Shanghai"` 告诉解析器"原始字符串是东八区时间"（字符串不带时区信息时的解释方式）；解析结果始终以 UTC 存储，Kibana 展示时再按浏览器时区换算回去——**"存 UTC、展示本地"是全链路唯一自洽的约定**，日志本身已带时区偏移（`Z` 或 `+0800`）时不要再配 timezone。**其二是 locale**：`MMM` 解析英文月份缩写建议显式 `locale => "en"`，中文"1月"用默认 locale 必败。

# 5. JSON — 解析 JSON

```text
filter {
  json {
    source => "message"               # 从哪个字段解析
    target => "parsed"                # 输出到哪个字段（默认根级别）
    skip_on_invalid_json => true
  }
}

# 输入: '{"user":"alice","action":"login","status":200}'
# 输出: { "user" => "alice", "action" => "login", "status" => 200 }
```

**json filter 处理"套娃 JSON"**：日志经 Filebeat/Kafka 到达时，真实结构常藏在 `message` 字符串里，需要二次解析。`target` 决定解析结果放哪——**默认合并到根级别，会覆盖同名既有字段**，这是"字段莫名其妙丢了"的常见根因；稳妥做法是指定 `target => "parsed"` 先隔离。`skip_on_invalid_json => true` 让非 JSON 消息静默跳过（打 `_jsonparsefailure` 标记而非抛错）。

**设计考量**：codec `json`（input 阶段）与 filter `json`（filter 阶段）的分工——input codec 解析**整条消息**，非法 JSON 原始字节进 `message` 并打失败标记；filter json 只解析**指定字段**，能嵌在条件分支里按来源选择性执行。混合内容流选后者，规整 JSON 流选前者。

# 6. Dissect — 分隔符解析

Dissect 比 Grok 更快（无需正则），适合分隔符明确的结构化日志：

```text
filter {
  dissect {
    mapping => {
      "message" => "%{ts} [%{level}] %{class} - %{msg}"
    }
  }
}

# 输入: "2024-01-15 10:30:00.123 [INFO] com.example.Service - user created"
# 输出:
# { "ts" => "2024-01-15 10:30:00.123",
#   "level" => "INFO",
#   "class" => "com.example.Service",
#   "msg" => "user created" }
```

**dissect 用"定界符切块"替代正则回溯**：`%{field}` 声明捕获段，段与段之间的普通字符就是分隔符，解析是线性的字符串查找，性能通常比等价 grok 高 5~10 倍且没有回溯风险。代价是灵活性：只能处理"分隔符固定"的行。

## 6.1 Dissect 高级

```text
# 跳过字段: %{}
# 重复分隔符: %{+order}
dissect {
  mapping => {
    "message" => "%{ts} %{+ts} [%{level}] %{logger} - %{msg}"
  }
}
# 输入: "2024-01-15 10:30:00.123 [INFO] com.example.Service - hello world"
# ts => "2024-01-15 10:30:00.123"
# level => "INFO"
```

三个修饰符覆盖 90% 的进阶需求：**`%{}` 丢弃一段**（占位不取名，对应日志里无关紧要的字段）；**`%{+name}` 把多段拼回同一个字段**（日期和时间被空格分开，`%{ts} %{+ts}` 用默认分隔符合成完整时间戳）；**`%{+name/2}` 与 `%{+name/1->2}`** 控制拼接顺序、对齐与追加分隔符。dissect 失败给 `_dissectfailure`，**它比 grok 失败得更"响"**：格式稍微一变就整段错位，所以 dissect 只适合格式极稳定的日志。

# 7. GeoIP — IP 地理位置

```text
filter {
  geoip {
    source => "clientip"              # IP 字段
    target => "geoip"                 # 输出目标
    database => "/usr/share/logstash/GeoLite2-City.mmdb"
    fields => ["city_name", "country_name", "location", "region_name"]
  }
}
```

geoip 加载 MaxMind MMDB 库把 IP 解析成国家/城市/坐标，是 Kibana 地图可视化的数据源。实践要点：**`fields` 白名单务必配置**（默认输出十几个字段，白白膨胀索引）；内网 IP（192.168.x.x）解析不出结果，统计前先排除；GeoLite2 需注册 Key 下载，8.x 支持 `database_reload_interval` 热加载。

> **注意**：8.x 默认 ECS 模式下 geoip 输出到 `client.geo.*` 而不是示例中的扁平 `geoip` 对象——字段名以实际输出为准，Kibana 可视化层配置要跟着 ECS 走。

# 8. UserAgent — 解析 UA

```text
filter {
  useragent {
    source => "agent"                 # nginx 日志里的 user_agent 字段
    target => "ua"
  }
}

# 输入: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0 Safari/537.36"
# 输出: { "ua" => { "name" => "Chrome", "os" => "Windows 10",
#                   "major" => "120", "device" => "Other" } }
```

useragent 基于浏览器指纹库（正则树）把 UA 字符串拆成 name/os/device，用于流量构成分析。三个坑：**它是重型 filter**，先 `if [agent]` 判空再进；解析失败返回 `"Other"` 而不是报错，质量差不代表成功；指纹库有版权与更新策略（旧版内置库停更）。

# 9. Translate — 字典映射

```text
filter {
  translate {
    field => "status_code"
    destination => "status_desc"
    dictionary => {
      "200" => "OK"
      "301" => "Moved Permanently"
      "404" => "Not Found"
      "500" => "Internal Server Error"
    }
    fallback => "Unknown"
  }
}
```

translate 做"值→值"的字典翻译：状态码转文案、错误码转业务含义、机房代号转全称。**字典规模决定实现形态**：小字典直接写配置；大字典用 `dictionaries_files` 指向 yaml/json 文件；十万级查表建议换成外部富化服务，别把字典内存压力全给 Logstash。`fallback` 不配时未命中则目标字段不写——建议永远显式给 fallback。

# 10. Ruby — 自定义脚本

```text
filter {
  ruby {
    code => '
      # 自定义复杂逻辑
      if event.get("amount")
        event.set("tax", event.get("amount").to_f * 0.13)
        event.set("total", event.get("amount").to_f + event.get("tax"))
      end
    '
    # init => '@sum = 0'          # 每个 worker 实例执行一次
    # code => '... @sum ...'      # 实例变量是 per-worker 的，不是全局
  }
}
```

ruby filter 是"逃生舱口"：日期加减、字段组合算桶号、条件打 tag……内置插件没覆盖的逻辑用 `event.get/event.set` 手写。**三条使用纪律**：**其一，它跑在 worker 线程上且每 worker 一个实例**，`init` 里的变量不跨 worker，累计计数会散落在每个副本上；**其二，Ruby 异常不会炸管道**，单条事件抛错打标记继续跑，容易静默吞 bug，上线前用 `-e` 喂样本验证；**其三，热路径上的 ruby 代码性能敏感**（JRuby 解释执行），能用内置插件下沉的坚决不用 ruby 写。

# 11. 条件执行

```text
filter {
  # 仅在匹配时执行
  if [type] == "nginx" {
    grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  }

  if "error" in [message] or "ERROR" in [message] {
    mutate { add_tag => ["error"] }
  }

  if [response] =~ /^5\d\d$/ {
    mutate { add_field => { "alert" => "true" } }
  }
}
```

if/else if/else 是 filter 层的调度器，运算符覆盖比较（`==` `!=` `<` `>`）、正则（`=~`）、包含（`in`）、存在性（`!([field])`）与逻辑组合。两个高频错误：**字段引用用 `[field][sub]` 方括号形式**（裸写 `[type]` 是历史字段，按文件路径路由用 `[log][file][path] =~ /nginx/` 更可靠）；**正则匹配要求字段是字符串**，对 integer 字段用 `=~` 会静默不命中，先 `convert` 或改用数值比较。

**设计考量**：条件的组织方式直接影响性能——**把"排除大多数事件"的判断放最外层、把昂贵插件包在最里层**；多级来源路由（if-elsif 十几路）建议改写成"按 `@metadata` 路由 + 多管道"。

# 12. 错误处理

```text
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    tag_on_failure => ["_grokparsefailure"]
  }

  # 只处理成功解析的事件
  if "_grokparsefailure" not in [tags] {
    date { match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ] }
  }
}
```

**filter 层的容错哲学是"打标记，不崩溃"**：grok/json/date/dissect 失败只给 `tags` 追加失败标记，事件继续往下走。标准应对模式是"成功走精加工，失败走留底"（解析失败的原始文本单独进可检索索引，修复 pattern 后回放）：

```text
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  if "_grokparsefailure" in [tags] {
    mutate { add_tag => ["needs_review"] }
  }
}

output {
  if "needs_review" in [tags] {
    elasticsearch { index => "logs-unparsed-%{+YYYY.MM.dd}" }   # 单独索引，可检索可回放
  } else {
    elasticsearch { index => "nginx-access-%{+YYYY.MM.dd}" }
  }
}
```

> **行业实践**：给 `_grokparsefailure` 等标记配独立索引或单独告警，**解析失败率是全链路最灵敏的"格式变更报警器"**——业务方改了日志格式没通知你，十分钟内就能在监控上看见。永远不要把失败事件直接丢掉：进了 ES 的脏数据能捞，被丢弃的数据就是丢了。

# 13. 字段命名规范与 ECS

Elasticsearch 生态的字段命名已收敛到 **ECS（Elastic Common Schema）**：一套"公共字段名 + 层级语义"的标准字典（`@timestamp`、`message`、`host.name`、`client.ip`、`http.response.status_code`……）。Logstash 8.x 核心插件默认 `ecs_compatibility => v8`，输出直接贴 ECS；自产字段遵守三条纪律：**其一，业务字段放独立顶层命名空间**（`biz.order_id`），不与 ECS 保留字段撞名；**其二，数值就是数值**（配合 mutate convert），别让字符串 `"200"` 混进被映射成 long 的列；**其三，统一方括号引用语法**——`"client.ip"` 写法可能指向"键名带点的扁平键"，留下极难清理的假嵌套。

> **注意**：新建索引一律对齐 ECS，Kibana 的 Lens、告警规则、ML 作业全部依赖统一字段名；一个团队里"半 ECS 半自造"是最贵的命名状态。

# 14. 小结

filter 层是 Logstash 的"加工车间"：**grok/dissect/json** 拆结构（正则强但贵，分隔符快但脆），**date** 纠正时间（存 UTC、展示本地），**mutate** 做字段增删改转（convert 必须在入库前，防 mapping 冲突），**geoip/useragent/translate** 做外部富化，**ruby** 兜底定制但守三条纪律（per-worker 实例、异常静默、热路径慎用）。所有插件共享"打标记不崩溃"的容错哲学，`_grokparsefailure` 进独立索引 + 监控是标准动作。组织方式上：昂贵插件包在条件最里层、失败率当格式变更报警器、字段命名对齐 ECS——filter 写得好不好，最终体现在 ES 的 mapping 干不干净和 Kibana 的查询顺不顺手。
