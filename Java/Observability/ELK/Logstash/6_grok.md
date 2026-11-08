# 1. Grok 原理

Grok 是 Logstash 的**正则表达式引擎**，将非结构化日志解析为结构化字段。

```text
日志行: 192.168.1.1 - - [15/Jan/2024:10:30:00 +0800] "GET /api/user HTTP/1.1" 200 45

Grok Pattern: %{IP:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:request} HTTP/%{NUMBER:httpversion}" %{NUMBER:response} %{NUMBER:bytes}
```

**Grok 不是新语言，而是"正则 + 命名捕获 + 模式库"的宏预处理器**：写下的每个 `%{PATTERN:field}` 会在配置加载时被替换成对应的正则片段，最终交给 **Oniguruma 正则引擎**（Ruby 生态的高性能 C 正则库）执行。理解这一点就理解了 grok 的一切特性——它的表达力上限是正则（能回溯、能贪婪、能灾难性回溯），它的复用能力来自宏（内置 200+ patterns，可层层嵌套），它的失败行为是正则的失败（整串匹配不上返回 nil，插件给事件打 `_grokparsefailure`）。

语法三形态要分清：

```text
%{PATTERN}                # 只用正则，不捕获（匹配但丢弃）
%{PATTERN:field}          # 捕获到字段 field
%{PATTERN:field:type}     # 捕获并转型（integer/float/boolean），减少后面 mutate convert
```

一个常见误解：grok 成功后 **`message` 原文并不删除**，解析出的字段是"追加"的；想省空间要自己 `remove_field`。另外 grok 默认在**整行**上做匹配（隐式非锚定），pattern 里没有 `^`/`$` 时前后缀内容被忽略——这既是宽容也是隐患（见第 7 节）。

# 2. 内置 Pattern

| Pattern | 匹配 | 示例输入 |
|---------|------|---------|
| `IP` | IPv4/IPv6 | 192.168.1.1 |
| `HOSTNAME` | 主机名 | web-server-01 |
| `WORD` | 字母数字 | GET |
| `INT` | 整数 | 200 |
| `NUMBER` | 数字 | 45, 3.14 |
| `DATA` | 非贪婪任意 | /api/user |
| `GREEDYDATA` | 贪婪任意 | 整行剩余内容 |
| `HTTPDATE` | HTTP 日期 | 15/Jan/2024:10:30:00 +0800 |
| `QUOTEDSTRING` | 带引号字符串 | "hello world" |
| `TIMESTAMP_ISO8601` | ISO8601 时间 | 2024-01-15T10:30:00.000Z |
| `NOTSPACE` | 无空格字符串 | active |
| `LOGLEVEL` | 日志级别 | INFO、ERROR、DEBUG |

这张表的正确读法是"**按精确度排序选词**"：能用 `IPV4` 就别用 `IP`，能用 `NOTSPACE` 就别用 `DATA`，`GREEDYDATA` 只配出现在 pattern 末尾。原因有二：**精确 pattern 展开出的正则短且确定，匹配快**；`DATA`（`.*?`）与 `GREEDYDATA`（`.*`）是回溯重灾区——它们几乎总能匹配上，会让"本该失败的行"错误地解析出一堆垃圾字段，比失败更难排查。内置 patterns 的源码在 `vendor/bundle/jruby/*/gems/logstash-patterns-core-*/patterns/`（grok-patterns、java、firewalls、aws 等分文件存放），本地直接翻文件是最权威的参考。

> **注意**：pattern 库会随版本演进（部分老 patterns 被标记废弃、ECS 模式下字段名会变形，如 `COMMONAPACHELOG` 在 `ecs_compatibility => v8` 下捕获名变成 `source.ip`、`http.response.status_code` 而不是 `clientip`、`response`）。升级 Logstash 大版本时，**grok 输出字段名是最容易悄悄变化的部分**，索引 mapping 会因此分叉——升级前用 `-e` 跑一遍样本对比输出。

# 3. 复合 Pattern

```text
COMBINEDAPACHELOG:
  %{IP:clientip} %{USER:ident} %{USER:auth}
  \[%{HTTPDATE:timestamp}\] "%{WORD:verb} %{DATA:request} HTTP/%{NUMBER:httpversion}"
  %{NUMBER:response} %{NUMBER:bytes} "%{DATA:referrer}" "%{DATA:agent}"

COMMONAPACHELOG:
  %{IP:clientip} %{USER:ident} %{USER:auth}
  \[%{HTTPDATE:timestamp}\] "%{WORD:verb} %{DATA:request} HTTP/%{NUMBER:httpversion}"
  %{NUMBER:response} %{NUMBER:bytes}

JAVASTACKTRACEPART:
  at %{NOTSPACE:class}\.%{NOTSPACE:method}\(%{DATA:file}(?::%{INT:line})?\)
```

复合 pattern 揭示了 grok 宏系统的本质：**patterns 文件里定义的每个名字本身也是"含 %{XXX} 的字符串"，加载时递归展开成最终正则**。这带来两个工程含义：其一，自定义 pattern 里可以继续嵌套内置 pattern（`ORDER_ID [A-Z]{3}\d{6}` 是纯正则，`MYLOG %{TIMESTAMP_ISO8601:ts} %{LOGLEVEL:level}` 则复用内置件）；其二，**嵌套深度与展开后长度决定编译成本**，一个引用了 `COMBINEDAPACHELOG` 的 pattern 展开后是几百字符的巨型正则——这也是 grok 慢的根源之一。

`match` 参数还支持两种数组形态应对变体日志：

```text
grok {
  # 同一字段依次尝试多个 pattern，命中即停
  match => { "message" => [ "%{COMBINEDAPACHELOG}", "%{COMMONAPACHELOG}" ] }
  # 多个字段分别尝试（message 不行再试其他字段）
  match => { "message" => "%{SYSLOGBASE}", "syslog_message" => "%{BASH_ACC}" }
  break_on_match => false        # 关掉"命中即停"，全部 pattern 都执行
}
```

# 4. 常见场景

## 4.1 Spring Boot 日志

```text
grok {
  match => {
    "message" => "%{TIMESTAMP_ISO8601:log_time} %{LOGLEVEL:level} %{NUMBER:pid} --- \[%{DATA:thread}\] %{DATA:class} : %{GREEDYDATA:msg}"
  }
}
# 输入: 2024-01-15 10:30:00.123 INFO 12345 --- [main] com.example.App : Started in 3.5s
```

这是 Java 体系最常用的一条 pattern：Spring Boot 默认日志格式（`%date %-5level %pid --- [%thread] %logger{36} - %msg`）结构极稳定，`GREEDYDATA:msg` 收尾吃掉消息体。两个细节：**`%{NUMBER:pid}` 建议写成 `%{INT:pid:int}`** 顺手转型；WARN 级别在 Spring 默认格式里被补空格成 `WARN `，`LOGLEVEL` 能吃掉但要确认尾部空格由 `\s` 表达。更好的消息是：**日志框架输出 JSON（logstash-logback-encoder）后，这一切 grok 都不需要**——"能不 grok 就不 grok"的第一优先级方案是改造日志格式本身。

## 4.2 Nginx Error Log

```text
grok {
  match => {
    "message" => "%{DATA:timestamp} \[%{WORD:level}\] %{DATA:pid}#%{NUMBER:tid}: \*%{INT:cid} %{GREEDYDATA:msg}"
  }
}
```

nginx error log 的形态是 `2024/01/15 10:30:00 [error] 1234#5678: *90 client request body...`，`pid#tid` 与 `*连接ID` 是其特征段。注意 timestamp 里日期和时间之间有空格，`%{DATA:timestamp}` 会贪婪不足、可能只吃到日期——更稳的写法是 `%{DATA:ts_date} %{DATA:ts_time}` 两段再拼接，或直接改用 **access log 配 JSON log_format**（`log_format json escape=json '{"time":"$time_iso8601",...}'`）从源头消灭 error log 之外的解析。

## 4.3 自定义业务日志

```text
grok {
  match => {
    "message" => "%{TIMESTAMP_ISO8601:ts} \[%{WORD:service}\] %{WORD:action} userId=%{WORD:userId} amount=%{NUMBER:amount} status=%{WORD:status}"
  }
}
# 输入: 2024-01-15T10:30:00 [order] CREATE userId=U123 amount=99.9 status=SUCCESS
```

对这类"等号 KV 风格"日志，grok 其实不是最优解——**kv filter 才是为 `key=value` 而生的插件**（`kv { field => "message" }` 一行自动拆出全部键值对），或者 dissect（`userId=%{user_id} amount=%{amount}`）。grok 适合的是"分隔符不规则、需要正则能力（可选段、前瞻）"的场景；格式有规律的 KV/CSV 日志用 grok 是杀鸡用牛刀且更慢。

## 4.4 Java 异常堆栈

```text
grok {
  match => { "message" => "(?:%{JAVASTACKTRACEPART:stack_part}|%{JAVALEXCEPTION:exception})" }
  # JAVALEXCEPTION: %{CLASS:exception_class}:? %{GREEDYDATA:exception_message}
  # JAVACLASS:      (?:[a-zA-Z$_][a-zA-Z0-9$_]+)(?:\.(?:[a-zA-Z$_][a-zA-Z0-9$_]+))+
}
```

`JAVASTACKTRACEPART/JAVALEXCEPTION/JAVACLASS` 是 java patterns 文件里的三件套，用来把 `at com.example.Service.method(Service.java:42)` 拆成 class/method/file/line。**前提是堆栈已经先被 multiline codec 合并成单个事件**（见 `8_codec.md`）——逐行 grok 堆栈没有意义。实际项目中更少见的路线是：整段堆栈保留为 `exception.stacktrace` 字段（ECS 约定），只对首行异常类名做 grok。

# 5. 自定义 Pattern

```bash
# patterns/custom.grok
ORDER_ID [A-Z]{3}\d{6}
CUSTOM_DATE \d{4}-\d{2}-\d{2}
CUSTOMER [a-z_]+
```

```text
filter {
  grok {
    patterns_dir => ["/etc/logstash/patterns"]
    match => { "message" => "%{CUSTOM_DATE:date} \[%{ORDER_ID:order_id}\] %{CUSTOMER:user} %{GREEDYDATA:msg}" }
  }
}
```

文件格式规则：**每行一条 `NAME regex`，以第一个空格切分名字与表达式**；`#` 开头是注释；表达式里可以出现空格（从名字之后整段保留）。同名 pattern 的覆盖顺序是"后定义覆盖先定义、patterns_dir 覆盖内置"。**命名纪律**：全大写下划线、语义明确（`ORDER_ID` 而非 `T3`），团队级 patterns 目录进 Git 管理。

# 6. Grok Debugger

```text
Kibana → Dev Tools → Grok Debugger
  输入 Sample Data + Grok Pattern → 查看解析结果

或在线工具: https://grokdebugger.com
```

Grok Debugger（Kibana 内置，`_grok/debugger` API）是 pattern 开发的事实标准工作流：**左边贴一行真实日志，右边写 pattern，实时显示捕获字段树**；点 "Patterns" 链接可以浏览全部内置 patterns 的定义，点具体 pattern 还能一键插入。它的价值在于把"改配置→重启管道→等数据→看日志"的分钟级循环压缩成秒级。

> **行业实践**：pattern 在 Debugger 里调好后，不要直接上生产——先落地成"离线回归"：取 1000 行生产真实日志存成文件，用 `bin/logstash -e 'input{file{path=>"/tmp/sample.log" start_position=>"beginning" sincedb_path=>"/dev/null"}} filter{grok{...}} output{stdout{codec=>rubydebug}}'` 全量跑一遍，肉眼抽查 + 统计 `_grokparsefailure` 比例，接近 0 才合并进配置。**Debugger 用的是样本，生产面对的是分布**——日期、空值、多语言消息、超长 UA 这些"分布尾巴"才是失败的来源。

# 7. 性能优化

```text
# ✗ 差: 没有锚定，引擎回溯
grok { match => { "message" => "%{DATA:field1} %{DATA:field2}" } }

# ✓ 好: 使用 ^ 锚定开头
grok { match => { "message" => "^%{DATA:field1} %{DATA:field2}" } }

# ✓ 好: 用 Dissect 替代 Grok (分隔符明确时)
dissect {
  mapping => { "message" => "%{field1} %{field2}" }
}

# ✓ 好: 先过滤再 Grok
if [type] == "nginx" {
  grok { ... }
}

# ✗ 差: 贪婪匹配过多
# %{GREEDYDATA} 只在最后使用一个
```

**grok 是 Logstash 管道里最烧 CPU 的插件**，没有之一，原因在 Oniguruma 的回溯机制：`.*?`/`.*` 这类量词匹配失败时，引擎要逐个移动起始位置、逐字符尝试所有组合。**灾难性回溯**（catastrophic backtracking）发生在"嵌套量词 + 可多选"的 pattern 上，如 `(a+)+b` 形态——一条精心构造的 200 字符日志能让单条 grok 耗掉整核 CPU 几十秒，表现为"管道吞吐断崖 + 某条数据反复超时"。规避手法按收益排序：

- **`^` 锚定开头**：把"在每个起点尝试"降为"只在一个起点尝试"，实测常见 3~10 倍提速；`$` 锚定对结尾同理但收益小。
- **精确 pattern 替代通配**：`%{IP:client} %{INT:port}` 优于 `%{NOTSPACE:client} %{NUMBER:port}` 优于 DATA/GREEDYDATA。
- **GREEDYDATA 只出现一次且在末尾**：多个 `.*?` 的组合空间是乘性的。
- **条件前置**：不需要 grok 的事件（结构化 JSON 日志）先用 if 分流，别让它们空转正则。
- **match 数组用"最可能的 pattern 在前"**：命中即停，排序即性能。
- **能 dissect 就 dissect**：分隔符固定的日志用 dissect 快 5~10 倍且无回溯风险。

> **注意**：`timeout_millis`（grok 单次匹配超时，默认不设）与 `regex_timeout` 是防"一条毒日志卡死 worker"的保险丝，设置后超时的条目打 `_grokparsefailure` 继续流转。生产建议显式设一个宽松值（如 30000ms）——它救不了性能，但能把"灾难性回溯"从"全管道停摆"降级成"单条失败"。

# 8. 失败处理

grok 失败不抛异常、不丢事件，只留下三个痕迹：`tags` 里的 `_grokparsefailure`（名字可用 `tag_on_failure` 改）、**已捕获的部分字段**（失败前匹配到的段仍会写入，事件呈"半解析"状态）、`message` 原文完好。据此有三条标准动作：

```text
grok {
  match => { "message" => [ "%{COMBINEDAPACHELOG}", "%{COMMONAPACHELOG}" ] }
  tag_on_failure => ["_grokparsefailure", "grok_failed_%{[log][file][path]}"]
  # tag 值支持字段插值：失败 tag 直接带上来源路径，排查时无需再翻事件
}

if "_grokparsefailure" in [tags] {
  # 失败事件单独索引留底，不阻塞主链路
  mutate { add_tag => ["unparsed"] }
}
```

**部分捕获是比"失败"更危险的状态**：字段存在但值错位（`%{DATA:request}` 吃进了后面的 method），下游 date/geoip 基于脏字段继续加工，最终产出一条"看起来解析成功了"的错误数据。所以失败判定的黄金指标是 `_grokparsefailure` 的占比（Kibana 里一条 count 比率监控），并配合对关键字段的抽样校验。半解析状态无法从 tag 上区分，唯一防线是 pattern 写得严格（锚定 + 精确 pattern），让"该失败的失败、不该错的不错"。

# 9. Grok vs Dissect

| 维度 | Grok | Dissect |
|------|------|---------|
| 原理 | 正则表达式 | 分隔符定位 |
| 灵活性 | ★★★★★ | ★★★☆☆ |
| 性能 | 中 | 高（5-10x faster） |
| 适用 | 复杂/不规则日志 | 分隔符明确的日志 |
| 学习曲线 | 中 | 低 |

判断线一句话：**格式"会变"的用 grok，格式"会坏"的用 dissect**。grok 能容忍字段宽度变化、可选段、模式内嵌分支（正则表达力），代价是慢与回溯风险；dissect 假设"分隔符永远在那"，日志格式一旦有可选空格、不定长段就直接错位——但它线性扫描、失败即 `_dissectfailure` 大声报错，不会像 grok 那样悄悄吞下错误结构。

**设计考量**：三者（grok / dissect / 源头 JSON）构成一条演进路径——**能改日志生产端就输出 JSON（logstash-logback-encoder、nginx json log_format），什么都不用解析；改不了生产端但格式规整用 dissect；真正不规则的历史遗产才上 grok**。管道解析成本是转嫁给生产端的：一条 grok pattern 的 CPU 开销乘以亿级日日志量，往往比"说服业务方改日志格式"的成本高得多。

# 10. 小结

Grok 的本质是**宏展开的正则**：`%{PATTERN:field:type}` 三层语法（引用、捕获、转型）+ 递归嵌套的 patterns 库 + Oniguruma 回溯引擎，决定了它表达力强（200+ 内置件覆盖 Apache/Nginx/Java/syslog/aws）而成本高。性能六板斧按收益排序：锚定、精确 pattern、GREEDYDATA 限量、if 前置分流、match 数组排序、能 dissect 就 dissect；`timeout_millis` 是防毒日志的保险丝。失败哲学是"打标记不崩溃"：`_grokparsefailure` + 部分捕获的"半解析"状态要求 pattern 严格、监控占比、失败留底。工作流上，Kibana Grok Debugger 负责秒级迭代，`-e` + 千行样本离线回归负责上线验证。最终极的建议不在 grok 内部：**最好的 grok 是不需要 grok**——日志源头直接输出 JSON 才是终局。
