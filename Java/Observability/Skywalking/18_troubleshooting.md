
# 1. 排错的方法论

SkyWalking 的故障可以分成四象限：**没有数据**（Agent→OAP→存储→UI 任一环断）、**数据不对**（断链、指标失真、重复服务）、**系统变慢**（agent 开销或 OAP 积压）、**运维异常**（启动失败、存储膨胀）。排错总原则是一条链路上**二分定位**：先用 OAP 侧证据（GraphQL 查询、日志、存储计数）判断断点在链路上游还是下游，再往下钻——漫无目的地翻日志效率极低。

```text
断链/无数据定位流程（自底向上）：

 Agent 进程 ──▶ OAP 接收 ──▶ 存储写入 ──▶ UI 查询
     │             │            │            │
     ▼             ▼            ▼            ▼
 日志有启动记录？  端口通？      索引有今天     getAllServices
 skywalking.log    收到上报？    的数据？       里有该服务？
 -javaagent 生效   OAP 日志      TTL 是否       时间范围/
 版本号对？        overflow？    误删？         namespace 对？
     │             │            │            │
     ▼             ▼            ▼            ▼
   否→查Agent     否→查网络    否→查OAP写入   否→查UI配置
   (1/2节)       /OAP日志      /存储健康      (查询条件)
```

# 2. Agent 不生效 / 启动失败

```bash
# 1. 确认 Agent JAR 路径正确、文件完整
ls -la /path/to/skywalking/agent/skywalking-agent.jar
unzip -l skywalking-agent.jar | head -5     # 损坏的包在此暴露

# 2. 检查 JVM 参数是否真的带上了 agent
ps aux | grep java | grep skywalking-agent
# 或确认 JAVA_TOOL_OPTIONS 环境变量注入成功
cat /proc/$(pgrep -f app.jar)/environ | tr '\0' '\n' | grep -i java_tool

# 3. 查看 Agent 日志（默认 logs/skywalking-api.log，K8s 里注意容器内路径）
tail -100 logs/skywalking-api.log
# 正常: "SkyWalking agent start successfully" / 版本 banner
# 异常: "Failed to connect to collector" / "Agent startup failed"
```

Agent 启动失败的高频原因按概率排序：**路径错误**（K8s initContainer 拷贝出双层目录、挂载点与 `-javaagent` 路径不一致）、**agent.config 语法错误**（非法字符导致整个配置加载失败，Agent 静默退出）、**JDK 版本不满足**（Java Agent 9.x 要求 JDK 8+，但对 JDK 17/21 需要较新的 agent 版本，老 agent 在新 JDK 上会报 InaccessibleObjectException）、**字节码冲突**（其他 APM/安全 Agent 同场增强同一批类，日志里表现为 `ClassCastException`/`VerifyError`）。

> **注意**：Agent 启动失败**默认不影响业务进程**（premain 异常被吞并记日志），这是特性不是 bug——但也因此容易长期无人发现。上线检查清单里应加一条"看 Agent 日志有成功启动 banner + UI 里服务出现"，双证齐全才算接入完成。

# 3. OAP 启动失败

```bash
# 1. 检查端口占用
ss -tlnp | grep -E "11800|12800"

# 2. 检查存储连接
# ES:
curl http://es:9200/_cluster/health
# MySQL:
mysql -h localhost -u skywalking -p skywalking -e "SELECT 1"
# BanyanDB:
grpcurl -plaintext banyandb:17912 list

# 3. 查看 OAP 日志
tail -f logs/skywalking-oap-server.log
# "Could not connect to Elasticsearch"  → ES 不可达/鉴权失败
# "Table 'xxx' doesn't exist"           → SQL 存储未初始化（首次启动需执行建表脚本或 init-mode）
# "cluster registration failed"         → SW_CLUSTER 配置与实际环境不符（如 k8s 模式缺 RBAC）
```

启动顺序依赖要记住：**OAP 强依赖存储可达**（连不上直接退出，不会降级运行），但**不强依赖 Agent/上游流量**；UI 强依赖 OAP 的 GraphQL。集群模式启动失败的另一大类原因是**多个 OAP 连了不同存储/不同 namespace**，注册上了也无法协同，表现为"节点互相看不见"的日志。

# 4. Trace 断链定位

断链是最高频也最容易被误判的问题——**多数断链不是 SkyWalking 的 bug，而是传播上下文在某个环节被人为切断**。标准排查顺序：

```text
① 断点在哪一跳？ ──打开上下游两条 Segment 的 refs 字段对比：
                    refs 为空 → 上游没传下去（发送侧问题）
                    refs 有但指向不存在的服务 → sw8 被改写（中间件问题）
② 链路中有无未插桩组件？ 网关(Nginx/APISIX)、LB、消息队列、自建 RPC
③ 有无异步/线程切换？     线程池提交任务、CompletableFuture、MQ 生产消费
④ 采样/版本因素？         两端 agent 版本跨度大、sw8 被降为 sw8-correlation 混用
```

各类断因的处置：

| 断因 | 现象 | 处置 |
|------|------|------|
| 网关不透传 header | HTTP 一跳即断 | Nginx `proxy_set_header sw8 $http_sw8;`；APISIX/SCG 检查插件是否重写 header |
| 异步线程池丢上下文 | 同进程内 Segment 断、日志 TID 丢失 | 引入 apm-toolkit-trace，用 `@TraceCrossThread`/`RunnableWrapper` 包装；或升级 agent 开启对 common pool 的增强 |
| MQ 消费断链 | 生产到消费之间断开 | 确认使用已支持的 MQ 插件；自研封装需手动把 sw8 写入消息 header（ContextManager.inject/extract） |
| HTTP 客户端裸用 HttpURLConnection 之外的未支持库 | Exit Span 缺失 | 换受支持客户端或加 toolkit `@Trace` 手工补 Span |
| 下游未装 agent | 拓扑正常但该服务无内部结构 | 安装 agent 或接受"虚拟服务"展示 |

**sw8 header 本体问题**有个快速验证手段：在断点两侧打印 header（网关访问日志加 `$http_sw8`），对比"出 A 时有、进 B 时无"即可锁定切断者。跨语言场景注意各探针版本对 sw8 字段扩展（如 sw8-correlation、直接传播所需的 profile 字段）支持度不同，混用老版本探针时最容易在 refs 关联上丢上下文。

# 5. UI 无数据四步查

服务存在、Agent 正常，但页面空白/图表断续——按固定四步走，每步都有客观证据：

```bash
# Step 1: 实体在不在？——GraphQL 直查 OAP，绕开 UI（查询名以所用版本文档 schema 为准）
curl -s -X POST http://oap:12800/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ getAllServices { name key } }"}' | head -c 500

# Step 2: 数据进没进存储？——看当天索引/表有没有新记录
curl "es:9200/_cat/indices/sw_*?v" | grep $(date +%Y%m%d)   # ES
# MySQL: SELECT COUNT(*) FROM segment_instance_20261108;

# Step 3: OAP 处理没处理？——日志与 self-o11y 队列
grep -E "buffer is full|Consumer failure" logs/skywalking-oap-server.log | tail
curl -s http://oap:1234/metrics | grep -E "queue|buffer"     # telemetry 端口

# Step 4: 查询条件对不对？——时间窗口时区、namespace、layer、分钟粒度边界
```

经验分布：**Step 4 的时区/时间范围问题占"偶发无数据"的一半以上**（浏览器时区与 OAP 服务器时区不一致、UI 默认窗口没覆盖数据时刻）；Step 2 无数据而 Step 1 有实体，多半是存储写入失败被 OAP 日志坐实；Step 1 查不到实体则是上游 Agent/OAP 断链，回到第 2、4 节流程。

# 6. 采样导致"丢数据"的排查

"Trace 少了"与"Trace 断了"要分开：前者是**整个 Segment 都没落存储**，后者是**落了一半**。疑似采样丢失时按顺序核对三个开关：Agent 侧 `sample_n_per_3_secs`（每 3 秒限流，高 QPS 服务最容易在这里丢）、OAP 侧 `trace-sampling-policy-settings.yml` 的 `rate`（1/10000 精度，10000=全采）、以及错误/慢 Trace 的强制保留开关（`forceSampleErrorSegment`、`duration`）是否被关闭。验证方法很直接：同一分钟内对比"入口服务 Trace 数（CPM 指标）与 Trace 列表检索数"，比值异常低即坐实降采；再按服务逐个查 rate 覆盖配置。

> **注意**：采样丢弃发生在 **Agent/OAP 内存层，不写日志**，不要指望日志里找到"哪条被丢"。用指标（trace_in_latency、采样率配置）推断，别用日志找证据。

# 7. ES 索引膨胀 / 存储暴涨

```bash
# 1. 按大小排出索引，定位膨胀来源（segment 类 or metrics 类 or log 类）
curl "http://es:9200/_cat/indices/sw_*?v&s=store.size:desc" | head -20

# 2. 检查 TTL 配置是否生效
grep -E "recordDataTTL|metricsDataTTL" config/application.yml
# dataKeeper 是否被关闭: enableDataKeeperExecutor: false

# 3. 手动清理过期索引（TTL 生效则不需要）
curl -X DELETE "http://es:9200/sw_segment_instance-20261025"

# 4. 找异常膨胀的根因：单个索引远超日均？→ 端点爆炸/大 payload
curl "es:9200/sw_segment-20261108/_count"
curl "es:9200/_cat/shards/sw_*?v&s:desc" | head    # 分片是否倾斜
```

膨胀的三大根因各有指纹：**TTL 配置丢失或被注释**（所有天的索引同宽增长）、**端点爆炸**（segment 索引里同一 URI 成千变体，查 UI Endpoint 列表基数即可确认）、**Span 塞大 payload**（单条 segment 文档体积从 KB 级涨到 MB 级，`_cat/segments` 或抽样 `_source` 可见）。处置优先级：先止血（调 TTL、清索引），再治理源头（归一化规则、ignore_path、参数采集开关）。

# 8. 内存与 CPU 过高

**OAP 侧**：`jstat -gcutil $(pgrep -f oap-server) 1000 5` 看老年代是否持续增长、FGC 是否频繁；堆 dump 前先看 self-o11y 队列指标——队列深通常是**存储写不动的果**而不是内存的因，按第 7 节先查存储。**Agent 侧**：正常增量 50~150MB；明显偏高时依次检查采样配置是否全量、`span_limit_per_segment` 是否失守（循环打 Span）、插件是否加载过多、以及"agent 开销误判"——应用 RSS 增长里混着业务对象增长，用**开关对比法**（同负载带/不带 agent 各压一轮）拿差值说话，避免把业务内存泄漏算到探针头上。

# 9. 版本兼容矩阵的坑

| 组合 | 兼容性 | 已知坑 |
|------|--------|--------|
| 新 Agent → 老 OAP | **不保证** | 新数据字段/协议被静默丢弃或解析失败，表现为断链/无数据 |
| 老 Agent → 新 OAP | 通常 ✓ | OAP 协议向后兼容大版本内，但过老的 6.x agent 建议升级 |
| 9.x/10.x OAP 混布 | 仅滚动期 | 集群内长期混跑不同大版本不保证行为一致，升级应尽快收敛 |
| Agent 大版本跳跃（8→9） | 需评估 | 默认插件集变化、optional-plugins 路径变化、JDK 要求变化 |
| sw8 v3 与更老 header | 部分 | 混合语言栈跨版本传播时 refs 丢失 |

纪律重申：**升级顺序先 OAP 后 Agent**（第 17 节），且一次只跳一个 minor 版本的大步方案要读 release notes 的 breaking change 条目；K8s 上 Agent 版本随应用滚动天然分批，注意灰度期间新老并存时 UI 指标口径（如新字段缺失）的短暂不一致是预期行为。

# 10. 常见错误速查表

| 错误日志/现象 | 原因 | 解决 |
|---------------|------|------|
| `Failed to connect to collector` | OAP 不可达/端口错 | 检查网络、11800 端口、SW_AGENT_COLLECTOR_BACKEND_SERVICES |
| UI 服务列表出现两个同名服务 | namespace 不一致 / 服务名拼写漂移 | 统一 agent.namespace 与服务命名登记 |
| `TID: N/A` 出现在日志 pattern | 该输出行本无 Trace 上下文（启动/定时任务） | 正常现象，不是断链 |
| `Could not find class` / `VerifyError` | 插件与框架版本不兼容 | 升级 Agent；禁用对应 optional-plugin |
| ES `index read-only` (403 flood-stage) | 磁盘水位触发只锁 | 清磁盘/扩容，解除 `read_only_allow_delete` |
| `DEADLINE_EXCEEDED` / gRPC keepalive 断 | 网络抖动、LB idle 超时短于 keepalive | 链路 LB 空闲超时调到大于 gRPC keepalive |
| 拓扑图服务间箭头抖动 | 实例重启+sw8 重建，正常 | 若伴随断链，回第 4 节 |
| Profiling 任务永远"运行中" | Agent 版本过老不支持 / 实例已销毁 | 升级 Agent，检查实例存活 |

# 11. 诊断命令合集

```bash
# Agent 侧：确认 agent 注册与上报队列
grep -E "start|register|send" logs/skywalking-api.log | tail

# OAP 侧：GraphQL 直查实体与 Trace（绕开 UI）
curl -s -X POST http://oap:12800/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ getAllServices { name } }"}'
# 查询某服务近一小时 Trace（10.x queryBasicTraces 形参，具体以所用版本文档的 schema 为准）
curl -s -X POST http://oap:12800/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ queryBasicTraces(condition:{serviceId:\"1\", traceState:ALL, queryOrder:BY_START_TIME, queryDuration:{start:\"2026-11-08 0900\",end:\"2026-11-08 1000\",step:MINUTE}}) { traces { segmentId } } }"}'

# OAP 自身：GC / 队列 / 线程
jstat -gcutil $(pgrep -f oap-server) 1000 5
jstack $(pgrep -f oap-server) > oap.jstack
curl -s http://oap:1234/metrics | grep skywalking_oap
```

# 12. 小结

排错入口先分象限（没数据/数据不对/变慢/运维异常），再用二分定位断点。本篇的主线流程：

- **断链**：refs 字段对比定断点 → 网关透传/异步包装/MQ header/版本兼容逐类处置
- **无数据四步查**：GraphQL 实体 → 存储索引 → OAP 队列日志 → UI 查询条件（时区/namespace 占大头）
- **采样丢失**：Agent rate + OAP trace-sampling 两级核对，用指标比值推断，日志里找不到被丢的 Trace
- **膨胀与开销**：索引大小指纹定根因；agent 开销用带/不带对比法测量，不凭感觉定罪
- **版本**：先 OAP 后 Agent、混布注意灰度期不一致是预期

下一篇做横向方案对比（SkyWalking vs Zipkin/Jaeger/Pinpoint/OpenTelemetry）。
