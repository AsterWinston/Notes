# 1. 文档模型

文档是 ES 里的最小数据单元：一段 JSON，挂在一个 ID 上，落在一个分片里。ES 7.x 移除 mapping type 之后，8.x 的文档模型已经收敛成一套非常简单的元数据语义：

```text
Index: orders
  └── _doc (type, 7.x 后固定)
        └── Document (JSON)
              ├── _id: 文档唯一标识
              ├── _source: 原始 JSON
              ├── _version: 乐观锁版本
              ├── _seq_no: 序列号（8.x 推荐替代 version）
              └── _primary_term: 主分片任期
```

几个元字段的行为差异决定了大量 API 设计：**_id** 全局（索引内）唯一，相同 _id 的再次写入不是"插入第二条"而是**整篇替换**（见第 6 节的版本控制）；**_source** 是原文档的完整副本，与倒排索引分开存放——倒排结构负责"搜得到"，_source 负责"取回原样"，这也是为什么高亮、partial update 都必须以 `_source` 开启为前提。**_version** 是一个单调递增计数器，任何写操作（包括最终内容相同的重复写入）都 +1，它只回答"写过几次"；**_seq_no** 是分片级的操作序列号，**_primary_term** 在主分片发生故障切换（任期变更）时递增，两者组合才能唯一定位"某个分片某任期内的第 N 次变更"——这才是并发控制需要的坐标。

> **注意**：`PUT /orders/_doc/1` 写入与已存在 _id=1 完全相同的文档，_version 与 _seq_no 依然会增长，且底层照常执行"标删旧文档 + 索引新文档"，**照样消耗 IO 并产生合并压力**。对"重复写入幂等"有执念的系统应先用业务层去重，或接受 `_create` 的 409 冲突语义（见第 2 节），而不是指望 ES 识别内容相同。

# 2. 创建文档

```json
// 指定 ID
PUT /orders/_doc/1
{ "order_no": "ORD001", "amount": 100.0 }

// 自动生成 ID
POST /orders/_doc
{ "order_no": "ORD002", "amount": 200.0 }

// op_type=create: 仅当不存在时写入，否则 409（天然的幂等防重）
PUT /orders/_doc/1?op_type=create
{ "order_no": "ORD001", "amount": 100.0 }

// 等价写法
POST /orders/_create/1
{ "order_no": "ORD001", "amount": 100.0 }
```

指定 ID 与自动生成 ID 的选择，本质是在选**数据语义**：自动 ID（POST）适合日志、事件这类"只追加、永不按业务键更新"的数据，每条独立、由 ES 生成 UUID；指定 ID（PUT）适合文档有业务主键（订单号、用户 ID）的场景——**同 ID 覆盖写**让它天然成为 MySQL→ES 同步链路的幂等原语：binlog 重放同一变更一百次，结果与写一次相同，同步链路因此敢重试。但代价是写入路径上每次都要先做存在性检查并走版本递增逻辑，比纯追加略贵，且"覆盖"是整篇替换而非合并。

`op_type=create` 值得单独记住：它是"只有第一次写入才算成功"的声明，配合外部系统生成的唯一 ID，可以在 ES 层面挡住重复投递（消息队列 at-least-once 场景的标配）。

# 3. 查询文档

```json
// 按 ID 查询（实时可读，不依赖 refresh，可跨副本负载均衡）
GET /orders/_doc/1

// 按 ID 批量查询
GET /orders/_mget
{ "ids": ["1", "2", "3"] }

// 仅返回部分字段
GET /orders/_doc/1?_source=order_no,amount

// 排除敏感字段
GET /orders/_doc/1?_source=false            // 只要元数据
GET /orders/_doc/1?_source_excludes=password // 8.x 语法
```

**GET by ID 与 search 是两条完全不同的路径**。GET 走"主分片（或副本）查 ID 定位 → 取 _source"，命中即返回，**不需要文档已 refresh**，写入成功后立刻 GET 一定读得到最新值；search 走倒排索引，只能看到已 refresh 的段。由此推出两条实践：其一，写后读校验、详情页按 ID 取文档，用 GET 而不是 `{"term": {"_id": 1}}`——后者既受 refresh 窗口影响，又无法利用"ID 路由直达分片"的优化；其二，mget/GET 可按需裁剪 `_source`，列表页只要三五个字段时不要整篇回传，这是比任何缓存都便宜的性能优化。

> **设计考量**：如果查询时显式带上 `?routing=user_123`，GET 只需访问单个分片；不带 routing 时 ES 会用 _id 算路由——**写入用什么 routing，读取就必须用什么 routing**，否则"文档明明存在却 404"。

# 4. 更新文档

```json
// 全量更新：整篇替换，未提及的字段直接消失
PUT /orders/_doc/1
{ "order_no": "ORD001", "amount": 150.0, "status": "paid" }

// 部分更新：只改给出的字段，其余保留
POST /orders/_update/1
{
  "doc": { "status": "paid" }
}

// 部分更新 + 不存在则创建（UPSERT）
POST /orders/_update/1
{
  "doc": { "status": "paid", "amount": 150.0 },
  "doc_as_upsert": true
}

// Script 更新：原子地在服务端读改（如累加、数组去重追加）
POST /orders/_update/1
{
  "script": {
    "source": "ctx._source.amount += params.increment",
    "params": { "increment": 50 }
  }
}

// 脚本 upsert：不存在时用脚本分支创建
POST /orders/_update/1
{
  "script": {
    "source": "if (ctx.op == 'index') { ctx._source.cnt = 1 } else { ctx._source.cnt += 1 }"
  },
  "upsert": { "order_no": "ORD001", "cnt": 1 }
}
```

**内部原理**：ES 不支持原地更新。`_update` 的实现是 **GET → 在内存里合并 doc/执行脚本 → 重新索引整篇文档（标记旧版本删除，段合并时清理）**。理解这一点，两类"坑"就有了统一的解释：

- **并发丢失更新**：GET 与重新索引之间没有锁。两个进程同时 `_update` 同一文档，后提交的会基于各自读到的旧 _source 覆盖对方——**partial update 不是原子操作**。需要读-改-写的原子性，用 script（脚本在分片上对最新版本执行，配合重试），或者用 `if_seq_no` 乐观并发（第 6 节）。
- **PUT 全量更新的隐性数据丢失**：PUT 只写你给的字段，没给的字段会**从新文档里消失**。业务方常常把 PUT 当 PATCH 用，导致下游同步链路里 ES 文档字段莫名缺失——"更新一个状态字段"永远应该用 `_update`。

UPSERT 的价值是把"有则更新、无则插入"压缩成一次请求，免去应用层"先 GET 判断再 PUT/POST"的两步竞态；`doc_as_upsert` 最常用（不存在时整个 doc 作为新文档插入），脚本版 upsert 适合"计数器"场景（不存在时 cnt=1，存在时 cnt+=1）。

> **注意**：`_update` 的 script 有执行成本且受 `script.max_compilations_rate` 限流，高并发累加（如点赞数）更推荐 ES 自带的 `ctx.op` 模式或改用外部计数存储；另外 update 对 `_source` 有强依赖，若索引显式关闭了 source（`_source: {enabled: false}`），partial update 会直接报错。

# 5. 删除文档

```json
// 删除
DELETE /orders/_doc/1

// 也可带版本条件删除：只有当前版本匹配才执行
DELETE /orders/_doc/1?if_seq_no=5&if_primary_term=1

// 条件批删（本质 scroll+bulk delete，慢操作，带查询词必须精确）
POST /orders/_delete_by_query
{ "query": { "term": { "status": "cancelled" } } }
```

删除后，文档被标记为 deleted，段合并时才物理删除。这条机制链的下游影响不少：`docs.count` 里含未清理的已删文档；删除比例高的索引查询变慢（每个 posting list 遍历都要过滤删除项）；最直接的后果是**删除不可逆**——误删只能靠 snapshot 或从真相源重放。

`_delete_by_query` 要按"批处理"对待：它内部是 scroll 查询 + bulk 删除的循环，不加 `conflicts=proceed` 时一旦版本冲突（并发写入导致 seq_no 变化）整个任务 409 中止；不加 `requests_per_second` 限速时它会把线程池占满，挤垮在线查询。生产用法永远是：精确的 query、限速、可重入（查询条件命中幂等）。

# 6. 乐观锁（并发控制）

ES 没有事务和行锁，防"两个写者互相覆盖"的机制是**乐观并发控制（OCC）**：写入时携带"我读到的版本坐标"，服务端比对不匹配就拒绝（409），让应用重读重试。

```json
// 使用 _version（旧式，已被 if_seq_no 取代）
PUT /orders/_doc/1?version=5
{ "amount": 200.0 }
// → 409 Conflict if version mismatch

// 使用 _seq_no + _primary_term（8.x 推荐）
PUT /orders/_doc/1?if_seq_no=5&if_primary_term=1
{ "amount": 200.0 }

// 查看当前坐标
GET /orders/_doc/1
// { "_seq_no": 5, "_primary_term": 1 }
```

**为什么 seq_no + primary_term 两个数缺一不可**：_seq_no 只在单个分片的当前任期内递增。当主分片故障切换、新主分片上任（primary_term +1）后，seq_no 的语义空间变了——只比对 seq_no 可能在任期变更后误放行旧写入。两者组合才能表达"我对这个分片这一次任期内第 N 个版本做出修改"。旧式的 `?version=` 是文档级计数，无法表达任期概念，且不支持并发副本读，8.x 中只作兼容保留。

> **行业实践**：典型的 OCC 闭环在应用侧完成——读取时把 `_seq_no`/`_primary_term` 随业务对象一起取出（可缓存在页面/会话状态里），提交时带上它们；收到 409 说明文档已被他人改动，按业务选择"重新合并后重试（有限次数）"或"提示用户刷新"。库存扣减这类高频冲突场景，更推荐用脚本在服务端完成"读改写"（`ctx._source.stock -= params.n` 配合 `if (ctx.op == "index") ctx.op = "noop"` 防负库存），把重试压回分片内部。

`op_type=create` 其实是 OCC 的退化形式（断言"此 ID 不存在"），而 `if_seq_no` 是完整形式（断言"此 ID 是我读到的那个版本"）。两套机制共同构成 ES 唯一的并发正确性保证。

# 7. Bulk API

_bulk 把多个动作压进一个 NDJSON 请求，是 ES 写入吞吐的基础设施：

```json
POST /_bulk
{ "index": { "_index": "orders", "_id": "1" } }
{ "order_no": "ORD001", "amount": 100.0 }
{ "create": { "_index": "orders", "_id": "2" } }
{ "order_no": "ORD002", "amount": 200.0 }
{ "update": { "_index": "orders", "_id": "1" } }
{ "doc": { "amount": 150.0 } }
{ "delete": { "_index": "orders", "_id": "3" } }
```

格式规则必须背熟（NDJSON 是最常见的报错来源）：**动作行与数据行严格成对交替**（delete 只有动作行）；**每行一个紧凑 JSON，行尾必须换行，整个 payload 最后一个空行不能省**；`Content-Type: application/x-ndjson`（或 `application/json` 亦可，但别用 form）。索引名与 ID 写在动作行而非文档体里。

Bulk 的语义是**各条动作独立成败，整体 HTTP 200**：响应里 `errors: true` 时逐 item 看 status 字段——单条 mapping 冲突、版本冲突不会让整批失败，也不触发任何回滚。所以 Logstash/同步程序必须检查 errors 数组并按条目重放，只判 HTTP 状态码等于丢数据。

## 7.1 Bulk 最佳实践

```bash
# 1. 每批 5-15MB（按字节而非条数衡量，条数随文档大小变化太大）
# 2. 单批文档数 500-1000（普通业务文档的经验区间）
# 3. 多线程并行提交，观察线程池拒绝数再逐步加压
# 4. 检查 bulk response 中的 errors，对 429/503 条目做退避重试

curl -X POST "localhost:9200/_bulk" \
  -H "Content-Type: application/x-ndjson" \
  --data-binary @bulk.json
```

批量导入的组合拳是固定套餐：`refresh_interval=-1` + `number_of_replicas=0` + bulk 分批 + 队列化的多线程提交，导完恢复刷新与副本、forcemerge。批大小的科学衡量方式是**盯两个指标找拐点**：写入线程池 `rejected` 不再增长、`GET /_nodes/stats/breaker` 无熔断，此时再增大只会把队列推长。

> **注意**：429（`EsRejectedExecutionException: queue full`）不是集群坏了，是客户端压太快——**必须带退避重试**（指数退避 + 抖动），盲目立即重放只会让队列雪崩。批量窗口期 `refresh_interval=-1` 记得导入后恢复并手动 `POST /index/_refresh`，否则"导入成功但搜不到"。

# 8. Refresh 控制

```json
// 插入后立即刷新（单文档级，代价是全分片级）
PUT /orders/_doc/1?refresh=true

// 只刷新涉及的分片，代价更低（8.x 推荐 wait_for 或 false 默认）
POST /orders/_doc/2?refresh=wait_for

// 批量导入时关闭刷新
PUT /orders/_settings
{ "refresh_interval": "-1" }

// 导入完成恢复
PUT /orders/_settings
{ "refresh_interval": "30s" }
```

`?refresh` 的三档语义对应三种承诺强度：默认 `false`（不触发额外 refresh，1s 窗口内可见）、`wait_for`（**不阻塞写入，只是把本次写请求挂到下一个自然 refresh 点返回**——写多时请求自动搭最近一班"车"，成本被均摊）、`true`（立即强制 refresh，写请求串行等一个昂贵的同步动作）。单条测试用 `true` 图方便可以理解，但**高并发写入路径上永远不要用 `refresh=true` 解决"写完查不到"**——它会让 refresh 频率随 QPS 线性增长，段爆炸 + 搜索变慢，正确解法是 GET by ID（实时）或 `wait_for`。

`refresh_interval=-1` 配合恢复与手动 `_refresh` 是批量导入的固定流程；写后立即可见的需求若属于搜索框输入提示这类交互，优先考虑"应用侧缓存 + 异步落 ES"而不是调低 refresh。

# 9. Routing 手动路由

```json
// 指定 routing（相同 routing 值路由到同一分片）
PUT /orders/_doc/1?routing=user_123
{ "order_no": "ORD001" }

// 查询时也必须指定 routing
GET /orders/_search?routing=user_123
{ "query": { "match_all": {} } }
```

默认路由公式 `hash(_id) % num_primary_shards` 让文档均匀撒满全部分片；`?routing=` 把"按什么算哈希"换成业务键，于是**同一个键（如租户 ID、用户 ID）的数据落进同一个分片**。收益是查询只扇出到 1/N 分片——单用户/单租户检索从"N 分片各查一遍再归并"变成"单分片本地 Top-N"，深分页与聚合误差也同步受益；代价是**数据倾斜**（大客户压垮单分片）、reindex/split 后路由关系要重新规划、**读写两侧 routing 必须一致**（漏带 routing 的查询靠 _id 去别的分片找不到，文档"凭空 404"）。

> **行业实践**：routing 是"查询必带租户维度"场景（SaaS 多租户搜索、个人订单列表）的性能利器，但默认情况下**不要带**——普通搜索场景的扇出成本本来就不高，过早绑死 routing 会让分片扩容与 reindex 复杂一个数量级。

# 10. 小结

| API/机制 | 关键结论 |
|----------|---------|
| GET by ID vs search | GET 实时（走 translog/副本，不经 refresh），search 近实时（默认 1s 窗口） |
| PUT 同 ID | 整篇替换且必然版本递增；PUT 不带 _source 会丢字段，局部修改用 _update |
| _update | 内部是 GET+合并+重索引，无原子性；读改写并发用 script 或 if_seq_no |
| UPSERT | doc_as_upsert / 脚本 upsert，一次请求解决"有则改无则加" |
| 乐观并发 | if_seq_no + if_primary_term（409 后重试）取代旧式 ?version；create 是存在性特例 |
| _bulk | 各条独立成败、整体 200，必须查 errors；5–15MB/批，429 退避重试 |
| 批量导入套餐 | refresh=-1 + replicas=0 + bulk 限速 + 完成后恢复/forcemerge |
| routing | 查询扇出降到单分片，代价是倾斜与读写必须同 routing |

文档层打通后，第 6 篇进入 schema 的世界——mapping：字段类型家族、text 与 keyword 的分野、dynamic mapping 的失控风险，以及 index/doc_values/norms 三个开关背后的存储取舍。
