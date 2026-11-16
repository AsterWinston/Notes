# 1. 总论：Kibana 治理的目标

Kibana 用好与用坏的区别不在功能，而在**治理**：同一个 Lens，有人用它把值班效率提升一个量级，有人用它堆出 200 张没人看的看板、500 条没人认领的告警。治理目标可以压缩成四条——**信息找得到（命名与组织）、数据看得对（口径统一）、权限不越界（最小化）、系统不拖累（性能与告警疲劳控制）**。本篇把前面各篇的要点收敛成可执行的清单，作为团队落地 ELK/Kibana 时的对照表。先强调一个反直觉结论：**Kibana 的资产（看板/规则/Data View）是"配置"更是"产品"**，需要 owner、评审与退役机制，放任自流的平台三年后必然腐烂到无人敢动。

# 2. Spaces 规划

推荐按"组织角色 + 风险等级"划分 Space，而非无限细分：

```text
推荐 Spaces 结构：
  ├── Production (仅监控 Dashboard，权限 read-only)
  ├── Development (开发环境数据探索)
  ├── Security (SIEM 安全分析)
  ├── ML (机器学习作业)
  └── Admin (管理 Space，仅管理员)
```

划分的判断依据是**权限模型是否相同**：Production 的受众是"所有人看、少数人改"，Development 是"开发者全权"，Security 数据敏感须单独授权，Admin 只给平台团队。Space 数量失控（每小队一个、每项目一个）会导致看板模板复制爆炸、跨团队找数据变成碰运气——**同权限模型的人群合并进一个 Space，用 Tags 区分子域**，是更可持续的做法。Default Space 留空，作为迁移过渡的临时区，季度检查其存量对象是否清零。

# 3. Dashboard 组织与设计原则

Dashboard 是消费端资产，组织上采用**三层目录**：Overview（老板/值班视角，少而稳）、Domain（业务域/系统域，团队维护）、Ad-hoc（个人探索，允许脏但定期清）。设计原则的正反清单：

```text
✓ DO:
  - 控制面板数 ≤ 15 个（加载性能）
  - 使用 Controls 统一过滤
  - 设置合理的时间默认值（如 Last 15m）
  - Markdown 面板添加说明（数据口径、已知异常、owner）
  - 定期归档不再使用的 Dashboard

✗ DON'T:
  - 不要在一个 Dashboard 堆砌 30+ 面板
  - 不要使用过小的 refresh interval（生产 ≥ 60s）
  - 不要使用 * 开头的查询（全模糊，极慢）
  - 不要在 Dashboard 中暴露原始日志数据（用 Discover）
```

面板排序遵循值班动线：**左上角放"现在有没有事"（核心大盘/告警计数），往下放"哪里出事"（分维度），最底放"细节入口"（深链 Discover）**。每个面板标题带单位与聚合口径（"P95 延迟 / ms / 近 5 分钟"），Markdown 面板写清数据源与 owner——看板好不好用，一半在标题和说明文字。

> **行业实践**：看板评审制——新 Dashboard 由 owner 在站会演示"给谁看、回答什么问题、何时会失效"，通过后贴 Tag 归档；每季度对访问统计为 0 的看板执行退役（导出存档后删除）。没有退役机制的 Kibana 平台，Saved Objects 数量会单调增长直到启动与检索都变慢。

# 4. 命名规范

命名解决的是"三个月后还能不能找到并看懂"。统一约定如下（分隔符一律 `-`，域前缀开头）：

```text
推荐命名: {environment}-{datatype}-{detail}

示例:
  prod-logs-application
  prod-metrics-system
  staging-logs-nginx
  dev-logs-*

索引模式:
  - prod-logs-*  → 匹配所有生产日志索引
  - metricbeat-*  → 匹配指标（带时间字段）
```

各对象的命名扩展成一张表：

| 对象 | 规范 | 示例 |
|------|------|------|
| Data View | `{env}-{datatype}-{scope}` | `prod-logs-payment` |
| Dashboard | `{域}-{用途}` | `order-service-overview` |
| Saved Search | `{域}-{问题}` | `order-db-slow-queries` |
| Alert Rule | `{级别}-{域}-{现象}` | `P1-payment-latency-high` |
| Connector | `{渠道}-{受众}` | `slack-oncall-payments` |
| ILM Policy | `{数据类型}-{保留期}` | `logs-90d` |
| Agent Policy | `{env}-{角色}` | `prod-web-server` |

**注意**：命名里塞的 env 必须与 Space/Data View 指向的真实数据一致——名字叫 `prod-` 却查 staging 索引，比没有规范更危险；重命名工具化（API 批量改 title）而非手工，避免引用关系遗漏。

# 5. 权限最小化

```text
角色划分:
  ┌──────────┬─────────┬──────────┬───────┐
  │   Role   │ Discover│Dashboard │ Admin │
  ├──────────┼─────────┼──────────┼───────┐
  │ Viewer   │   ✓     │    ✓     │   ✗   │
  │ Analyst  │   ✓     │    ✓     │   ✗   │
  │ Admin    │   ✓     │    ✓     │   ✓   │
  └──────────┴─────────┴──────────┴───────┘

建议:
  - 生产环境：仅给 read 权限
  - 开发环境：给 all 权限（允许创建 Dashboard）
  - Dev Tools：仅 Admin 可访问
```

执行层面的三条铁律：**索引 pattern 显式列举**（杜绝 `*-*`）、**破坏性功能（Dev Tools、Index Management、Advanced Settings）默认全关**、**服务账号与人类账号分轨**（采集/脚本用 API Key 或 service account，禁止共享个人密码）。权限申请流程化：通过工单申请 role 变更，平台侧用 `PUT /_security/role` 脚本化下发，使"谁在什么时候拿到什么权限"可审计。

# 6. 性能最佳实践

性能问题的三大来源：查询写法、聚合选择、数据模型。按收益排序：

**① 高基数聚合慎用**——这是 Kibana 侧最重要的一条。terms 聚合的 `doc_count_error_upper_bound` 与协调节点内存都随基数恶化，对 `user_id`、`trace_id` 级别字段做 top terms 或脚本字段分组，轻则查询超时（`search:timeout` 后 ES 返回 40 秒前就放弃）、重则把协调节点打爆。**替代方案排序**：先收窄时间窗与过滤 → 用 runtime field/ingest 阶段预计算低基数派生字段 → 真要精确统计就 `composite` 分页 + 后台任务 → 或走 ESQL/聚合下推。

**② 查询写法的红线**：`*keyword*` 全模糊、wildcard/prefix 高基数、`scripted_metric`、超大 size 深分页——这些在 UI 里都能点出来，但每次都全表扫描。Discover 保持 `discover:sampleSize` 在 100–500，导出走报表 API 异步任务而不是同步 CSV。

**③ 平台与索引侧配合**：

```yaml
# kibana.yml
elasticsearch.requestTimeout: 30000
elasticsearch.shardTimeout: 30000
elasticsearch.maxSockets: 500

# Advanced Settings
state:storeInSessionStorage: false   # 防止 session storage 膨胀
csv:maxSizeBytes: 5242880           # 限制 CSV 导出
discover:sampleSize: 500             # 限制单次搜索返回数
```

索引侧的对应动作同样属于 Kibana 治理范围：dashboard 高频过滤字段设为 `keyword` 并开 `eager_global_ordinals`；时间字段必须正确（所有查询带 `@timestamp` 范围裁剪，否则全索引扫描）；ILM 把老索引降冷/force merge，否则 Discover 查一个月的日志会随时间线性变慢。

> **注意**：`elasticsearch.requestTimeout` 调大只是把问题掩埋——Kibana 超时说明底层查询本来就该重写或加缓存。**给"看起来慢"的看板做体检的标准工具是 Inspect → Request/Body + took 字段**，慢查询优化在 ES 侧（mapping、分片数、过滤下推）而不是给 Kibana 加超时。

# 7. 告警疲劳治理

告警治理的核心指标不是"告警数量下降"，而是**处理率与认领时效**（参考告警篇的分层设计）：

- **入口收敛**：P0 呼叫、P1 群、P2 邮件，低级别只进汇总；所有消息带深链与五要素上下文。
- **周度告警例会**：拉 Alert History 按规则统计触发量/认领率/误报原因，误报率 >70% 的规则当场改阈值或停用；"从未被认领"的规则直接进退役名单。
- **规则卫生**：每条规则有 owner、Investigation guide、关联看板；新规则先影子运行两周再进值班通道。
- **风暴预案**：维护窗口提前 Snooze、依赖抑制（网络类告警触发时业务超时类静音）、限流（同规则同分组 Throttle 5–10 分钟）。

> **行业实践**：把"值班打扰度"做成 SLO——**夜间可呼叫告警每月 ≤ N 次**，超了就立项降噪专项。这条指标能防止两类退化：一是"为了不漏报无限加规则"，二是"为了安静无限提阈值"，把平衡点变成可追踪的数字而不是个人偏好。

# 8. 版本升级 Checklist

Kibana 与 ES 必须同大版本、Kibana 不晚于 ES 的次序容许窗口有限（官方建议同批升级），执行清单：

```text
升级步骤:
  1. 备份 Saved Objects
     GET /api/saved_objects/_export?type=dashboard&type=visualization

  2. 备份 kibana.yml 配置文件

  3. 滚动升级（多实例时）
     - 停止 Kibana-1 → 升级 → 启动
     - 停止 Kibana-2 → 升级 → 启动

  4. 验证
     - 检查 Dashboard 是否显示正常
     - 检查告警规则是否正常

  5. 升级 Saved Objects（如有版本迁移提示）
```

在此基础上补齐容易漏掉的八项：

| # | 检查项 | 要点 |
|---|--------|------|
| 1 | Upgrade Assistant | 升级前一周清完黄红灯项 |
| 2 | 快照 | 含 `.kibana*` 与关键业务索引 |
| 3 | License 兼容性 | 试用功能过期前先换正式 key |
| 4 | Agent/集成版本 | Fleet artifact 镜像同步、policy 目标版本更新 |
| 5 | 插件/扩展 | 自研 UI 扩展与新 core 兼容性回归 |
| 6 | 废弃设置清理 | kibana.yml 中 deprecated key 逐项移除 |
| 7 | 金丝雀实例 | 多实例先升一台全功能 smoke（Discover/告警/看板/登录） |
| 8 | 回滚预案 | 降级不被支持——演练"快照回滚 + 配置回滚"路径 |

# 9. 监控 Kibana 自身

Kibana 是观测平台的观测对象，**监控盲区就是故障盲区**——它挂了意味着告警系统同时失明：

```text
必须监控的指标:
  □ Kibana 服务健康 (GET /api/status)
  □ ES 连接状态
  □ Kibana JVM 内存使用
  □ 请求延迟 P99
  □ 错误日志数量
  □ Saved Objects 数量（过多影响启动速度）

日志:
  tail -f /var/log/kibana/kibana.log | grep -E "ERROR|WARN"
```

`/api/status` 的 `core.status` 与 `plugins.*` 状态适合接入拨测；Kibana 进程日志建议用 Filebeat/Agent 采集回自家集群建专门的 `kibana-logs-*` 看板（自举观测的仪式感之外，真能提前发现 Task Manager 积压、告警执行失败）；`.kibana_task_manager` 积压与 `xpack.task_manager.*` 指标是告警延迟的先行指标。注意：**Kibana 自身监控不能只依赖 Kibana 告警**——至少保留一条独立通道（外部拨测 + 邮件）兜底"Kibana 挂了"这一种故障。

# 10. DON'T 清单汇总

```text
✗ 不要在生产环境用 Dev Tools 执行删除/修改操作
✗ 不要在 Dashboard 中引用过多 Data View（增加 ES 负载）
✗ 不要给所有用户 all 权限
✗ 不要在 Kibana 中存储超大 CSV 导出结果
✗ 不要混用不同版本的 Kibana + ES（版本必须匹配）
✗ 不要在 Dashboard 中使用 1s 刷新间隔
✗ 不要忘记定期清理旧的 Saved Objects
```

补充四条本篇新增的红线：**✗ 不要对高基数字段做交互式 terms 聚合**；**✗ 不要让无 owner 的告警规则进值班通道**；**✗ 不要把 Space 当数据隔离用（权限边界在 Role/索引层）**；**✗ 不要在未打快照前执行任何批量导入/覆盖**。

# 11. 小结

- **组织**：Space 按权限模型划分而非按组织架构穷举；看板三层目录 + 评审/退役机制；命名规范以 `{env}-{datatype}-{detail}` 为锚，表化到每种对象。
- **性能**：高基数聚合是第一杀手，写法红线 + 时间窗先行 + 索引侧 mapping/ILM 配合，慢查询用 Inspect 定位在 ES 侧解决。
- **告警**：以处理率和夜间打扰度为北极星，影子运行、周度降噪、风暴预案三板斧。
- **生命周期**：升级走 checklist（Assistant→快照→金丝雀→回滚演练），Kibana 自身必须被独立通道监控——观测平台的失明是全局性风险。
