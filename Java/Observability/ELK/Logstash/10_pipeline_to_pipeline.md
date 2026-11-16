
# 1. 概述

多管道（multi-pipeline）解决的是"一个进程里跑多条独立流水线"，但管道之间天生是隔离的——各跑各的 Input/Filter/Output，**没有官方通道互相传数据**。Pipeline-to-Pipeline（管道间通信）补的就是这块：它提供一组 `pipeline` 伪插件（一个 output 插件 + 一个 input 插件），让事件在**同一 Logstash 进程内的两条管道之间**直接传递，不走网络、不做序列化，开销近似一次内存引用交接。

```text
传统: 部署多个 Logstash 实例
  Logstash-1 → (网络/Kafka) → Logstash-2 → ES

Pipeline-to-Pipeline: 同一进程
  Pipeline A → (Virtual Output→Input) → Pipeline B → ES
```

没有它之前，想把"统一入口 + 分类重加工"拆成多个管道，要么把所有逻辑塞回一条大管道用 if-else（失去独立 workers/queue 的隔离收益），要么拆进程再加一层 Kafka/Redis 中转（运维两套东西）。Pipeline-to-Pipeline 是第三条路：**用配置层级的拆分换来与拆进程接近的隔离性，但成本只是进程内的一个队列交接**。这个特性从 7.4 起正式可用，8.x 成熟稳定。

## 1.1 解决的问题与设计定位

| 需求 | 不用 P2P 的做法 | 用 P2P 的做法 |
|------|----------------|---------------|
| 按类型隔离资源 | 单管道 + if-else 分支 | 上游汇聚管道 + 下游独立管道 |
| 多下游分发 | 多个 output 复制事件 | `send_to` 多地址分发 |
| 进程拆分开销大 | 多实例 + Kafka 中转 | 一进程多管道内存直传 |
| 灰度替换处理逻辑 | 改配置全量生效 | 新管道并行接流对比 |

> **设计考量**：P2P 传递的是 **Event 对象引用的复制**（浅拷贝语义，事件字段在下游独立修改不影响上游副本），因此零序列化；但"零开销"是相对的——事件会**同时占用两条管道的内存足迹**，链路变长、堆积更难定位，这是后面限制一节要展开的代价。

# 2. 配置

三个角色：源管道（output 端写 `pipeline`）、目标管道（input 端写 `pipeline`）、连接声明（在源管道的 input 段或 config 里声明 `pipeline.send_to` 目标）。注意目标管道的 input 必须**先声明、再被引用**，Logstash 按 pipelines.yml 顺序加载。

```yaml
# pipelines.yml
- pipeline.id: input-pipeline
  path.config: /etc/logstash/conf.d/input.conf

- pipeline.id: processing-pipeline
  path.config: /etc/logstash/conf.d/processing.conf

- pipeline.id: output-pipeline
  path.config: /etc/logstash/conf.d/output.conf
```

```ruby
# input.conf — 采集管道
input {
  beats { port => 5044 }
}

output {
  pipeline {
    send_to => ["processing-pipeline"]   # 发送到处理管道
  }
}
```

```ruby
# processing.conf — 处理管道
input {
  pipeline {
    address => "processing-pipeline"     # 接收管道（地址名全局唯一）
  }
}

filter {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  geoip { source => "clientip" }
}

output {
  pipeline {
    send_to => ["output-pipeline"]
  }
}
```

```ruby
# output.conf — 输出管道
input {
  pipeline {
    address => "output-pipeline"
  }
}

output {
  elasticsearch { hosts => ["http://es:9200"] }
}
```

三个易错点：`address` 名字必须在**整个进程范围内唯一**（不是管道内唯一），重名会在启动时报 `Duplicate connection names detected`；`send_to` 引用未声明的地址同样是启动期失败，热加载时会让该次 reload 整体不生效；下游管道的 filter workers 数量、queue 类型都由自己的 pipelines.yml 条目决定，**上游不会替你继承任何参数**。

# 3. 分发器模式

一个管道分发到多个下游管道：

```ruby
# distributor.conf
output {
  pipeline {
    send_to => ["nginx-pipeline", "app-pipeline", "error-pipeline"]
  }
}

# 下游管道自动负载均衡
```

`send_to` 列表里给**多个地址 = 负载均衡**：每条事件只会被发送到其中一个下游（round-robin 语义，不是广播），适合把同类流量摊给多个同构处理管道，等价于"管道级的 worker 扩容"。想要**一份数据发给所有下游**（广播/复制），则写多个独立的 `pipeline` output 块：

```ruby
output {
  pipeline { send_to => ["search-pipeline"] }   # 每块各发一份
  pipeline { send_to => ["audit-pipeline"] }    # 事件会被复制成两份
}
```

条件分发：

```ruby
# 条件分发
output {
  if [type] == "nginx" {
    pipeline { send_to => ["nginx-pipeline"] }
  } else if [type] == "app" {
    pipeline { send_to => ["app-pipeline"] }
  }
}
```

```text
              ┌→ nginx-pipeline  (grok+geoip, workers=4)
汇聚管道 ──┬──┼→ app-pipeline    (json+date,  workers=4)
 (workers=2) │  └→ error-pipeline (告警+文件,  workers=1)
              └─ 负载均衡时: 同名管道开 2 个实例 round-robin
```

> **注意**：条件分发里**没有兜底分支的事件就消失了**——`if/else if` 漏掉的 type 不会有任何 output 接住它，事件被静默丢弃。规范写法是最后加一个 `else { pipeline { send_to => ["fallback-pipeline"] } }`，或者至少 `else { mutate { add_tag => ["unrouted"] } }` + file 输出留痕，否则新业务接入忘配分支时会"丢得无声无息"。

# 4. 使用场景

```text
场景 1: 预处理 → 分类 → 输出
  Input Pipeline (采集) → Processing Pipeline (解析+分类)
    ├── Nginx Pipeline (Grok + GeoIP → ES)
    ├── App Pipeline (JSON + Mutate → ES)
    └── Error Pipeline (告警 + 文件)

场景 2: 隔离资源
  采集管道 (轻量) + 处理管道 (重量)
  不同优先级的数据用不同 Worker 数

场景 3: 解耦测试
  上游不变，替换下游管道进行测试
```

场景 1 是最典型的"**一个端口对外、多条管道对内**"：beats input 只监听一处（5044 端口不能被两个管道重复占用），汇聚管道做最轻的解析和打标，重型加工（geoip、大量 grok、ruby）下沉到各自下游，各自配独立的 `pipeline.workers`。这样 nginx 流量突增打满下游 workers 时，**队列堆积只发生在 nginx-pipeline 自己的 queue 里**，app 管道的处理延迟不受牵连——这正是单管道 if-else 模式做不到的（一条大管道里所有分支共享同一批 workers，一个慢 filter 拖累全部流量）。

场景 2 的资源账要算清：多管道共享的是**同一个 JVM 堆**，workers 只是线程配额隔离，不是内存隔离。下游重管道大量堆积照样把堆吃光引发 Full GC，反过来拖慢所有管道。真要硬隔离内存，得拆进程。

场景 3 常用于灰度：把 `send_to` 从旧处理管道切到新处理管道（新管道消费同一 address 名或改一行配置），出问题切回去即可，上游采集端完全不动。

# 5. 性能优势

```text
Pipeline-to-Pipeline vs External Queue:
  - 零序列化：内存中直接传递 Event 对象
  - 零网络开销：进程内通信
  - 低延迟：亚毫秒级传递

适用条件:
  - 同一台机器上运行的多个管道
  - 需要隔离不同类型数据处理资源
  - 需要分发数据到多个下游
```

| 对比项 | Pipeline-to-Pipeline | Kafka/Redis 中转 | 单管道 if-else |
|--------|---------------------|------------------|----------------|
| 传输开销 | 进程内引用传递，亚毫秒 | 序列化+网络+反序列化，毫秒级 | 无 |
| 故障隔离 | 中（共享 JVM/磁盘） | 强（独立进程） | 无（共享一切） |
| 持久化缓冲 | ✗（依赖各管道自己的 PQ） | ✓ | 管道自己的 PQ |
| 跨机器 | ✗ | ✓ | ✗ |
| 运维对象 | 一份部署 | 额外中间件集群 | 一份部署 |
| 配置复杂度 | 中 | 高 | 低 |

需要澄清一个常见误解：**P2P 链路本身不持久化，但链路上每条管道仍然各自享有完整的 Queue（可配 PQ）**。事件从 A 交到 B，是放进 B 管道自己队列的 Input 侧，B 若开了 `queue.type: persisted`，落盘照常发生。所以"B 管道崩溃会丢 A→B 在途事件"是错的——B 重启后从自己的 checkpoint 恢复；真正丢数据的场景是 **A 被停掉/崩溃时它自己队列里还没交出去的部分**，那由 A 的 PQ 兜底。

# 6. 限制

```text
- 只能在线程间传递（不支持跨进程）
- 不支持持久化（不同于 PQ）
- 需要提前定义 pipelines.yml
- 循环管道需要避免（A→B→C→A）
```

逐条展开最要紧的几条限制：

**地址注册与重载顺序**。connection 地址在管道加载时注册、卸载时注销。热加载（`config.reload.automatic`）一个下游管道时存在短暂的"地址已注销、上游还在 send"窗口，此时上游会报 `Connection name not registered` 并抛事件写失败日志——低峰期 reload 影响不大，高峰期 reload 会看到成片的发送错误。规范动作是先 reload 上游（把它切到备用地址或容忍丢弃），再动下游，或者干脆关闭自动 reload、用受控滚动重启。

**无跨进程能力**。`pipeline` input/output 只在同进程内寻址，两台机器之间想复制这套拓扑只能上 Kafka/Beats，别指望"分布式的 pipeline 地址"。

**循环引用**。A→B→A 会在注册期或运行期造成事件死循环，配置评审时要人工看拓扑，Logstash 不做环路检测。

**可观测性碎片化**。一条逻辑流水线的堆积被拆到 N 个管道各自的 `queue.events_count` 里，`_node/stats/pipelines` 要逐个管道看，慢下游的表现是"上游管道 output 发送计数不涨、下游 queue 涨"，排查路径比单管道长。

**吞吐上限仍是 CPU**。P2P 只是省了中转开销，不增加任何算力；下游 grok 慢，拆管道后照样慢，只是慢得更"局部"。

> **注意**：管道数不是越多越好。每条管道有自己的 workers 线程、队列和插件实例（geoip 数据库这类插件级资源**每个实例各自加载一份**），拆 10 条管道用 geoip，堆里就是 10 份.mmdb 映射。经验值是单进程 3-8 条管道，再往上拆进程。

# 7. 小结

| 主题 | 关键结论 |
|------|----------|
| 定位 | 同进程管道间零序列化传数据：源管道 `output { pipeline { send_to } }`，目标管道 `input { pipeline { address } }` |
| 多下游 | `send_to` 多地址 = round-robin 负载均衡（每条一份）；多个 pipeline output 块 = 广播复制 |
| 核心价值 | 一条慢流量只堆积在自己的管道里，workers/queue 按管道独立配额 |
| 持久化 | P2P 链路本身不持久化，但每条管道自己的 Queue（含 PQ）照常生效 |
| 主要坑 | address 全局唯一、reload 窗口期发送失败、无环路检测、geoip 等插件资源按管道复制 |
| 边界 | 只在同进程内工作；内存仍是共享的，硬隔离要拆进程；单进程建议 3-8 条管道 |
