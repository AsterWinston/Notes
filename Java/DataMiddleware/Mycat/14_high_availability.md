# 1. Mycat2 自身 HA

**Mycat 是无状态代理**：它不存储数据，会话状态基本只有"这条前端连接绑定了哪些后端连接"与事务标志，配置来自文件/配置中心。无状态是高可用友好属性——**实例可以随时增删，挂一台不影响数据正确性**，HA 的全部工作因此收敛为两件事：入口流量如何转移到存活实例（VIP/LVS），以及配置如何保持一致（配置中心）。

Mycat2 本身是无状态的（配置存储在文件/ZK/Nacos），可以通过多实例部署实现自身高可用。

```text
                    ┌─────────────┐
                    │   VIP/LVS   │
                    │  (浮动 IP)   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Mycat2-1 │ │ Mycat2-2 │ │ Mycat2-3 │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
             └────────────┼────────────┘
                          │
                   ┌──────▼──────┐
                   │ Nacos (配置)│
                   └─────────────┘
```

**至少两台的底线之外还有容量账**：常见误区是"N 台扛 N% 流量，部署 N/2+1 台就够"。正确口径是**故障域冗余**：单机房部署时，一台故障后剩余实例要能扛住全量峰值（含重连风暴），即 N 台实例按 (N-1) 台满负荷容量规划；跨机房部署时，要以"整个机房失联"为故障单元做容量核算。Mycat 前端是长连接模型（应用连接池通常持有连接不释放），**故障实例上的存量连接不会自动迁移**——客户端连接池里的死连接要等超时/校验失败才重建，这决定了 HA 切换的 RTO 大头在客户端而不是服务端。

# 2. 配置中心模式

多实例的第一杀手是**配置漂移**：3 台 Mycat 各自维护本地配置文件，改一台忘两台，结果同一 SQL 在不同实例路由结果不一致——这比宕机更可怕，因为它是静默的数据正确性问题。

```yaml
# mycat.yml — ZK 配置中心
server:
  configCenter:
    type: zookeeper
    address: zk1:2181,zk2:2181,zk3:2181
    sessionTimeout: 30000
    rootPath: /mycat2
    clusterId: mycat-cluster-1
```

```yaml
# Nacos 配置中心
server:
  configCenter:
    type: nacos
    address: nacos1:8848,nacos2:8848,nacos3:8848
    namespace: mycat2
    group: MYCAT_GROUP
```

**配置中心模式优势**：
1. 所有 Mycat2 实例共享同一份配置
2. 修改配置 → 推送到配置中心 → 所有实例自动加载
3. 新实例加入自动从配置中心拉取配置

> **注意**：接入配置中心后，**配置文件从"部署产物"变成"运行时数据"**，本地 conf 目录退化为缓存。必须同步改变操作纪律：任何人不再直接编辑某台机器的本地配置（改了也会被中心覆盖），一切变更走配置中心的发布流程 + 版本历史 + 回滚。MyCat 1.6.x 的内置 ZK 模式（`cluster.xml`）在此之外还同步"在线修改"（`reload` 命令写入 ZK 再广播），2.x 思路相同但数据格式换成 JSON；不用配置中心时，reload 只作用于单实例，集群一致性靠发布系统保证。

# 3. 实例故障转移

入口层方案对比：

| 方案 | 原理 | 切换速度 | 复杂度 | 适用 |
|------|------|---------|--------|------|
| Keepalived (VRRP) | 主备抢占 VIP，健康检查失败则 VIP 漂移 | 秒级（1-3s） | 低 | 2-3 台同机房，最常见 |
| LVS (DR/NAT) | 四层负载均衡 + 健康检查摘除 | 秒级 | 中 | 实例多、需按连接数调度 |
| F5 / 云 SLB | 商用/云负载均衡 + 探测 | 秒级 | 低（钱换） | 云上标准做法 |
| 客户端多地址 | JDBC URL 列多台，驱动重试 | 首次建连即感知 | 低 | 应用池配好重试参数 |

```text
Client JDBC Connection URL:

jdbc:mysql://mycat2-vip:8066/db?
  loadBalanceHosts=true
  &retriesAllDown=3
  &connectTimeout=3000
```

```text
Client 连接策略：
  1. 连接到 VIP/LVS → 路由到可用 Mycat2 实例
  2. Mycat2-1 宕机 → LVS 健康检查失败 → 摘除
  3. 新连接路由到 Mycat2-2 或 Mycat2-3
  4. 已有连接需要在客户端重试
```

**Keepalived 部署要点**：`vrrp_script` 的健康检查不要只 `pgrep mycat`——进程在但端口不通（FullGC 卡死、后端全挂导致的拒绝服务）是常态故障形态，检查脚本应实测 8066 端口 TCP 可连（`nc -z`）甚至执行一条探测 SQL。`state MASTER/BACKUP` + `priority` 决定正常时 VIP 归属，**务必设置 `preempt false`（非抢占）**，否则 BACKUP 机器重启后 VIP 抢回、连接再漂移一次，故障窗口被人为拉长一倍。

> **注意（脑裂）**：VRRP 组播报文跨不了网段/机房，Keepalived 只在**同一二层域**内工作；跨机房做 VIP 需要单播模式或干脆放弃 VIP 改用 DNS/SLB。经典的 VRRP 脑裂是网络分区后两台都持有 VIP，出现"两个 Mycat 同时接流量"——对 Mycat 本身后果有限（无状态，重复接请求而已），真正的风险在下面第 4 节的**后端 MySQL 主从脑裂**：两个实例都认为自己可写，binlog 分叉。中间件 HA 与数据库 HA 必须作为一个整体设计。

# 4. 后端 MySQL HA

Mycat 高可用做得再好，数据层的 HA 才是重头。Mycat 侧要配置的是"如何感知并跟随主从角色变化"。

## 4.1 主从切换

```json
// clusters.json
{
  "clusters": {
    "order_cluster": {
      "clusterType": "MASTER_SLAVE",
      "readBalanceType": "BALANCE_ALL",
      "master": {
        "name": "m1",
        "url": "jdbc:mysql://192.168.1.10:3306/order_db"
      },
      "replicas": [
        {
          "name": "s1",
          "url": "jdbc:mysql://192.168.1.11:3306/order_db"
        },
        {
          "name": "s2",
          "url": "jdbc:mysql://192.168.1.12:3306/order_db"
        }
      ],
      "switchType": "AUTO",
      "slaveThreshold": 60
    }
  }
}
```

**切换流程**：

```text
1. Master 心跳失败（连续 3 次）
2. Mycat2 从 Slave 中选择数据最新的
3. 将该 Slave 提升为 Master（change master to ...）
4. 更新路由表
5. 告警通知
```

> **设计争议**：上面这条链路的每一步都有"简化假设"——"数据最新"依据 Seconds_Behind_Master/位点判断，异步复制下**选不出绝对最新**；提升新主只改了 Mycat 视角，旧主带着原 VIP/域名可能仍在服务其他直连系统。因此生产环境更常见的分工是：**MHA/Orchestrator/RDS 高可用体系负责"选主 + 补 binlog + 隔离旧主"，Mycat 配置为被动跟随**（switchType 关闭自动提升，仅靠心跳发现新主地址）。Mycat 自带的切换能力定位是"没有专职 HA 组件时的兜底"，不要当成企业级切换方案使用。

## 4.2 Galera Cluster

```json
{
  "clusters": {
    "galera_cluster": {
      "clusterType": "GARELA_CLUSTER",
      "members": [
        { "name": "node1", "url": "jdbc:mysql://192.168.1.10:3306/db" },
        { "name": "node2", "url": "jdbc:mysql://192.168.1.11:3306/db" },
        { "name": "node3", "url": "jdbc:mysql://192.168.1.12:3306/db" }
      ]
    }
  }
}
```

Galera 集群中所有节点均可读写，Mycat2 按权重轮询。**多主形态下防脑裂的职责从中间件转移到了 Galera 自身**（quorum 仲裁）：少数派分区自动失联禁写，Mycat 心跳发现后摘除——这是"用带仲裁的复制协议换切换复杂度"的路线。代价是写冲突回滚、单行全局锁开销、以及节点重启可能触发 SST 全量同步。

# 5. K8s 部署

容器化把"Mycat 无状态"的红利兑现成运维能力：探活、自愈、滚动升级都变成声明式配置。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mycat2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mycat2
  template:
    spec:
      containers:
        - name: mycat2
          image: mycat2/mycat2:1.22
          ports:
            - containerPort: 8066
            - containerPort: 9066
          env:
            - name: MYCAT_CONFIG_CENTER
              value: "nacos"
            - name: NACOS_ADDR
              value: "nacos-headless:8848"
          resources:
            requests:
              cpu: 2
              memory: 2Gi
            limits:
              cpu: 4
              memory: 4Gi
          readinessProbe:
            tcpSocket:
              port: 8066
            initialDelaySeconds: 15
          livenessProbe:
            tcpSocket:
              port: 8066
            initialDelaySeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: mycat2
spec:
  type: ClusterIP
  ports:
    - name: data
      port: 8066
    - name: manager
      port: 9066
  selector:
    app: mycat2
```

> **注意**：K8s 上跑长连接代理有两个必须处理的点。其一，**livenessProbe 用 TCP 端口连通过于乐观**——进程假死时端口往往仍 accept，建议就绪探针测 8066、存活探针结合管理端口探测或 exec 检查，避免"探针通过但请求全超时"的僵尸 Pod。其二，**Pod 销毁 = 存量长连接被杀**，需要 `preStop` 钩子 + `terminationGracePeriodSeconds` 给连接排空时间，配合 Service 摘除 endpoints 的先后顺序实现优雅下线；不做这些，每次发版都等价于一轮小型故障切换。

# 6. 平滑升级

```text
零停机升级步骤：
  1. 部署新版本 Mycat2 实例（新端口）
  2. 验证新实例功能正常
  3. 从 LVS/SLB 摘除一个旧实例
  4. 已有连接逐步排空（等待自然断开或超时）
  5. 重复 3-4 直到所有旧实例替换
  6. 关闭旧实例
```

第 4 步的"排空"（drain）在长连接场景需要耐心：摘除后 LB 不再分配**新**连接，但存量连接不受影响，应用连接池的 `minIdle` 会维持一批空闲连接长期挂在旧实例上。实操办法：摘除后观察 `showFrontends` 连接数衰减曲线，等待超过连接池驱逐周期（或主动重启应用池）再关进程。**"摘了 LB 就立刻 kill"是平滑升级最常见的翻车点**，那一瞬间应用池里的活跃连接全部报错。

# 7. 故障切换演练

**没演练过的 HA 等于没有 HA**。切换演练要回答的不是"能不能切"，而是三个量化问题：发现多久（心跳窗口）、恢复多久（客户端连接池完成重建）、丢不丢（切换瞬间在途请求与未同步事务）。

```text
季度演练清单：
  1. kill -9 一台 Mycat 实例
     → 预期：VIP/探针 10s 内摘除；应用报错尖刺 < 30s；无脏数据
  2. 断后端 MySQL Master 网络 (iptables DROP)
     → 预期：心跳 3 周期后 OFFLINE；读流量转移；写流量按配置的切换策略处理
  3. Slave 制造复制延迟 (从库 STOP SLAVE 60s)
     → 预期：延迟超 slaveThreshold 自动踢出读池，追平自动回池
  4. 全量重启配置中心
     → 预期：Mycat 按本地缓存配置继续服务，重连后恢复同步
  5. （有专职 HA 组件时）由 HA 组件触发主从切换
     → 预期：Mycat 心跳跟随新主，写恢复时间 = HA 切换时间 + Mycat 探测窗口
```

演练必须**在预发环境走全流程、在生产走低风险项**（第 2、3 项可生产做，第 1 项视流量窗口），并记录每一步的时间戳，形成"切换耗时基线"。真正故障时没人来得及现场推理，基线表就是判断"现在正常吗"的依据。

# 8. 数据备份策略

```text
分层备份：
  ├── Mycat2 配置 → Git / Nacos 历史版本
  ├── MySQL 全量备份 → mysqldump + 定时任务
  ├── MySQL 增量备份 → binlog 备份
  └── 跨机房容灾 → MySQL 异步复制到异地
```

分库分表让备份恢复复杂度上升一个维度：一次 PITR（point-in-time recovery）要**同时恢复 N 个分片到同一逻辑时刻**，各分片 binlog 位点必须对齐——用 Mycat 全局序列时可用序列值当逻辑时间锚点，否则依赖 LVM/云盘快照的原子性打点。备份策略设计时就要把"N 分片一致恢复"的演练纳入，恢复不了等于没备份。

# 9. 小结

| 层次 | HA 手段 | 责任边界 |
|------|--------|---------|
| Mycat 实例 | ≥2 台 + VIP/LVS/SLB + 实测端口健康检查 | 无状态随便摘，瓶颈在客户端长连接重建 |
| 配置一致性 | ZK/Nacos 配置中心 + 禁止本地手改 | 防路由不一致的静默错误 |
| 后端主从 | 心跳探测 + 跟随专职 HA 组件切换 | Mycat 不做脑裂仲裁，只跟随 |
| 编排环境 | 探针分级 + preStop 排空 + 非抢占 VIP | 发版不当故障用 |
| 兜底 | 季度演练 + 切换耗时基线 + N 分片一致恢复 | 量化 RTO，拒绝"应该可以" |

Mycat 高可用的心智模型一句话：**实例层随便死，配置层不许漂，数据层交给专职仲裁，客户端才是切换耗时的真正大头**。下一篇（15 动态配置）深入配置变更的生效机制与边界。
