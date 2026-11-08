# 1. 环境要求

部署 ES 前的检查清单比安装步骤本身更能决定线上稳定性，因为 ES 对操作系统资源有几处硬性依赖，默认参数的 Linux 发行版几乎全都不满足：

| 组件 | 要求 |
|------|------|
| JDK | ES 8.x 自带捆绑 JDK（无需单独安装，优先使用） |
| OS | Linux/macOS/Windows (WSL2) |
| 内存 | 最小 2GB，生产 16GB+ |
| 文件描述符 | >= 65536 |
| vm.max_map_count | >= 262144 |

这些要求背后的原因各不相同：**捆绑 JDK** 是 Elastic 官方保证调优过的版本，生产环境不要指定外部 `JAVA_HOME`，除非清楚兼容性矩阵；**nofile 65536** 是因为每个 Lucene segment 的每个文件都占一个句柄，一个中等索引轻松打开数千文件，默认 1024/4096 会导致"too many open files"直接拒绝写入；**vm.max_map_count 262144** 是因为 Lucene 用 mmap 读取段文件，默认值不够时节点直接启动失败——这也是 Docker 部署必须先改宿主机内核参数（或给容器加 `--sysctl`）的原因；**内存**的大头要留给 OS page cache 而不是 JVM 堆，这一点在第 6 节解释。

# 2. Docker 部署（推荐）

```bash
# 单节点（宿主机先执行: sysctl -w vm.max_map_count=262144）
docker run -d \
  --name es01 \
  -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
  docker.elastic.co/elasticsearch/elasticsearch:8.11.0

# 验证
curl http://localhost:9200
# {
#   "name": "es01",
#   "cluster_name": "docker-cluster",
#   "version": { "number": "8.11.0" }
# }
```

镜像内预置了 `elasticsearch` 用户（uid 1000）与路径，最容易被坑的是**数据卷权限**：把宿主机目录挂载到 `/usr/share/elasticsearch/data` 时，如果目录属主不是 uid 1000，容器会以 `AccessDeniedException` 启动失败，标准解法是 `chown -R 1000:0 ./data`。生产 Compose 部署还应显式设置 `bootstrap.memory_lock=true` 并给容器配置 memlock ulimit，配合下面 `swappiness=1` 防止堆被换出。

`discovery.type=single-node` 与 `xpack.security.enabled=false` 这两个开关要明白各自关掉了什么：前者跳过集群引导（不配 seed hosts、不做选举），后者把 8.x 默认开启的 TLS+认证整个关闭。**它们只适用于本地开发**。

zip/tar 包部署则解压即用（自带 JDK），核心是三件事：以非 root 普通用户运行、把 `config/jvm.options.d/` 作为 JVM 参数覆盖目录、用 systemd 托管。tar 包适合"固定机器、长期运行的正式集群"，Docker 适合"快速起环境或编排平台交付"，两者参数语义完全一致。

# 3. 系统参数调优

```bash
# /etc/sysctl.conf
vm.max_map_count = 262144    # Lucene mmap 段文件所需，默认 65536 太小
vm.swappiness = 1             # 禁止 JVM 堆被换出，swap 是延迟毛刺的头号来源

sysctl -p

# /etc/security/limits.conf
elasticsearch  -  nofile  65536    # 文件描述符，对应每个 segment 的文件句柄
elasticsearch  -  nproc   4096     # 线程数上限
elasticsearch  -  memlock unlimited # 配合 bootstrap.memory_lock 锁堆
```

systemd 托管时 limits.conf 可能不生效，须在 unit 文件里写 `LimitNOFILE=65536`、`LimitMEMLOCK=infinity`；Docker 则用 `--ulimit nofile=65536:65536 --ulimit memlock=-1:-1`。`vm.swappiness=1`（而非 0）是经验值：0 会在内存吃紧时过度回收 page cache，反而伤搜索性能，1 既保护堆又保留回收能力。

# 4. Kibana Dev Tools

```bash
# 安装 Kibana（与 ES 同 docker 网络时可直接用服务名互访）
docker run -d \
  --name kibana \
  --link es01:es01 \
  -p 5601:5601 \
  -e ELASTICSEARCH_HOSTS=http://es01:9200 \
  docker.elastic.co/kibana/kibana:8.11.0

# 访问 http://localhost:5601 → Dev Tools
```

**版本必须与 ES 完全一致**（8.11.x 配 8.11.x），跨小版本混搭轻则功能缺失、重则拒绝连接。Kibana 自身是无状态的，所有对象（Dashboard、数据视图）都存在 ES 的 `.kibana` 索引里。

Dev Tools 是全系列笔记里所有 REST 示例的运行场所：左侧写 `GET /_cat/indices?v` 这类伪 REST 请求，右侧看格式化结果，并自带 DSL 补全。相比 curl，它省掉转义与 header 的噪音，是调试 Query DSL 的事实标准。

# 5. 第一个索引与文档

```json
// 创建索引
PUT /products
{
  "settings": { "number_of_shards": 1, "number_of_replicas": 0 },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "standard" },
      "price": { "type": "double" },
      "category": { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}

// 插入文档
POST /products/_doc/1
{
  "title": "iPhone 15 Pro",
  "price": 8999.00,
  "category": "电子产品",
  "created_at": "2024-01-15T10:00:00"
}

POST /products/_doc/2
{
  "title": "MacBook Pro 16",
  "price": 19999.00,
  "category": "电子产品",
  "created_at": "2024-01-16T10:00:00"
}

// 全文搜索
GET /products/_search
{
  "query": {
    "match": { "title": "iPhone" }
  }
}

// 聚合统计
GET /products/_search
{
  "size": 0,
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "by_category": {
      "terms": { "field": "category" },
      "aggs": { "avg_price": { "avg": { "field": "price" } } }
    }
  }
}
```

这段最小闭环值得逐行读一遍，因为它同时展示了 ES 的三个身份：`PUT /products` 建索引时**settings 决定分布式形态**（1 主分片 0 副本——本地测试的标配，生产单节点也应是 0 副本，否则永远 yellow）、**mappings 决定字段语义**（title 用 text 才能全文检索，category 用 keyword 才能精确过滤和聚合，详见第 6 篇）；`POST /products/_doc/1` 写入是**文档模型**（无 schema 迁移，但字段类型由 mapping 锁定）；最后的 `match` 查询与 `aggs` 聚合分别是**搜索引擎**和**分析引擎**的入口——`size: 0` 表示只看聚合不要明细，是报表类查询的惯用写法。

> **注意**：刚插入文档立刻 `GET /products/_search` 有可能查不到（refresh 默认 1s 窗口），但 `GET /products/_doc/1` 一定查得到。这个差异是第 2 篇 NRT 机制的直接体现，也是新手最常见的"数据丢了"误报。

# 6. jvm.options 关键参数

```text
# config/jvm.options（或以 jvm.options.d/*.options 分文件覆盖，推荐后者）
-Xms16g                       # 堆初始值
-Xmx16g                       # 堆最大值，必须与 Xms 相等，避免运行时伸缩引发长 GC

## GC: 8.x 默认 G1GC，一般无需改动
-XX:+UseG1GC

## 堆外直接内存上限（netty 网络缓冲用，默认与堆同量级）
-XX:MaxDirectMemorySize=8g
```

JVM 堆设置的三条铁律：

1. **Xms = Xmx**：堆伸缩会在流量高峰触发一次代价高昂的 heap resize + GC，ES 应从启动就锁定堆。
2. **不超过 31GB（压缩指针界限）**：堆 ≤ 31GB 时 JVM 开启 compressed oops，对象引用只占 4 字节；超过后切到 8 字节，有效堆反而缩水。大内存机器要拆成多个节点，而不是一台机器上开 60G 堆——**Lucene 段文件读取靠 OS page cache，堆越大留给 page cache 的越少**，经验法则是"一半内存给堆、一半给文件系统缓存"，堆不超过物理内存 50%。
3. **堆不是缓存**：堆里放的是分片的元数据结构、请求缓冲与聚合中间态；文档数据的"缓存"在 page cache。盲目调大 Xmx 常常让查询更慢。

> **注意**：`ES_JAVA_OPTS` 环境变量在 Docker/K8s 里是覆盖堆参数的便捷入口，但如果同时存在 `jvm.options.d/heap.options` 文件，后者优先。多处配置并存时，用 `GET /_nodes/jvm` API 查实际生效值最可靠。

# 7. elasticsearch.yml 核心配置

```yaml
# 集群配置
cluster.name: my-cluster           # 同网段多集群靠它隔离，发现只认同名集群
node.name: node-1                  # 节点身份，选举与 _cat/nodes 里可见，勿复用

# 节点角色（8.x 推荐写法）
node.roles: [master]               # 或 [data, ingest] / [] 纯协调

# 数据路径
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# 网络
network.host: 0.0.0.0              # 默认 127.0.0.1 仅本机可访，组集群必须改
http.port: 9200                    # REST 端口（客户端流量）
transport.port: 9300               # 节点间内部通信端口（集群流量，不对客户端暴露）

# 集群发现（8.x）
discovery.seed_hosts: ["node1:9300", "node2:9300", "node3:9300"]
cluster.initial_master_nodes: ["node1", "node2", "node3"]   # 仅首次引导需要

# 锁定内存，防止堆被 swap（配合 memlock unlimited）
bootstrap.memory_lock: true

# 慢日志阈值（排障利器）
index.search.slowlog.threshold.query.warn: 10s
index.indexing.slowlog.threshold.index.warn: 10s
```

**http 与 transport 双端口的意义**：9200 走 REST/JSON，供应用、curl、Kibana 访问；9300 走 ES 私有的二进制协议，负责节点间分片复制、集群状态发布。防火墙/安全组只对集群成员开 9300，是基本的安全姿势——**transport 端口一旦暴露到公网等于交出集群**。

从单节点到三节点集群的演进路径：单机开发用 `discovery.type=single-node`；正式集群至少三台 master-eligible 节点（多数派 2/3，任一节点挂掉仍可选举），data 节点按容量独立扩展。K8s 上交给 ECK Operator（StatefulSet + 证书自动签发）是最省心的生产方案，手工 YAML 部署则要自行解决持久卷权限与 seed 发现。

> **注意**：`cluster.initial_master_nodes` 与 `discovery.seed_hosts` 的分工极容易写反——seed_hosts 是"打招呼用的电话本"（每台节点长期配置），initial_master_nodes 是"建国时的开国名单"（只在集群第一次 bootstrap 时使用，全部节点起来后应从配置中移除）。老集群重建某节点时若把旧 initial 名单原样保留，节点会试图另立门户。

# 8. 安全配置（8.x 默认开启）

8.x 的 zip/docker 首次启动会自动做三件事：生成 CA 与节点证书、给 HTTP 层开 TLS、为 `elastic` 超级用户设置随机密码并打印到日志。开发环境嫌麻烦可以 `xpack.security.enabled: false`（如第 2 节示例），但生产必须走完整流程：

```bash
# 1) 一键初始化内置用户密码
bin/elasticsearch-setup-passwords interactive

# 8.x 更推荐: 用 enrollment token 把 Kibana 接入
bin/elasticsearch-create-enrollment-token -s kibana

# 2) 重置 elastic 密码
bin/elasticsearch-reset-password -u elastic

# 3) 验证：跳过证书校验 + 带认证访问
curl -k -u elastic:密码 "https://localhost:9200/_cluster/health?pretty"
```

安全体系分三层，对应不同配置面：**传输加密**（`xpack.security.http.ssl.*` 与 `transport.ssl.*`，8.x 默认全开）、**认证**（内置用户 + API Key + 域/LDAP realms）、**授权**（RBAC，见第 16 篇）。最小可用配置：

```yaml
xpack.security.enabled: true
xpack.security.enrollment.enabled: true
xpack.security.http.ssl.enabled: true
xpack.security.http.ssl.keystore.path: certs/http.p12
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: certs/transport.p12
xpack.security.transport.ssl.truststore.path: certs/transport.p12
```

> **行业实践**：内网集群也**不要裸奔关闭 security**——历史上大量"删库跑路"事故都源于公网暴露的无认证 ES（扫描引擎常年能搜到上万台）。低成本底线是：至少开启认证 + 只监听内网地址 + 9200 不对公网开放；应用侧用 API Key 或专用最小权限账号，而不是 elastic 超管直连。

# 9. 常用系统 API

```bash
# 集群健康（green/yellow/red）
GET /_cluster/health

# 节点信息（ram/heap 使用率、load、角色）
GET /_cat/nodes?v

# 索引列表（docs.count、store.size 一屏纵览）
GET /_cat/indices?v&h=index,health,pri,rep,docs.count,store.size&s=store.size:desc

# 分片分配（看分片落在哪个节点、是否未分配）
GET /_cat/shards?v

# 集群状态
GET /_cluster/stats?human

# 任务管理（长查询/重建索引进度）
GET /_cat/tasks?v

# 排障三件套：线程热点、节点统计、未分配原因
GET /_nodes/hot_threads
GET /_nodes/stats/jvm,os,fs
GET /_cluster/allocation/explain
```

`_cat` 系列 API 是给人看的（返回文本表），`?v` 出表头、`?h=` 选列、`?s=` 排序；脚本化集成时应改用对应的 JSON API。`_cluster/health` 的三种状态语义必须刻进脑子里：**green** 所有主副分片就位；**yellow** 主分片齐全但副本未分配（单节点起集群必现，多节点出现要查磁盘或节点数）；**red** 有主分片丢失，这部分数据当前不可读写——**red 是唯一需要立刻打电话的告警级别**。

**启动排障速查**：

| 症状 | 根因 | 处理 |
|------|------|------|
| max virtual memory areas too low | vm.max_map_count 未调 | `sysctl -w vm.max_map_count=262144` |
| max file descriptors too low | nofile 限制 | limits.conf / systemd LimitNOFILE |
| can not run elasticsearch as root | 以 root 运行 | 切普通用户 |
| failed to obtain node locks | path.data 被占用/权限 | 检查属主与是否双实例 |
| master 反复选举 | GC 停顿/网络分割 | 看 GC 日志与防火墙 |

# 10. 小结

| 要点 | 一句话 |
|------|--------|
| 系统参数 | nofile 65536、vm.max_map_count 262144、swappiness=1、memlock 锁堆，缺一个就等着随机崩溃 |
| 堆设置 | Xms=Xmx、≤31GB、不超过物理内存 50%，剩下的内存留给 OS page cache |
| 双端口 | 9200 REST 面向客户端，9300 transport 面向集群，只对内开放 |
| 发现配置 | seed_hosts 长期保留，initial_master_nodes 只在首次 bootstrap 用 |
| 安全 | 8.x 默认 TLS+认证；开发可关，生产绝不允许裸奔 |
| 运维入口 | _cat 人读 / JSON 机读；health 三态中只有 red 需要立即干预 |

环境跑起来之后，第 4 篇开始处理第一个真正的对象——索引：如何创建、别名与模板怎么组织生命周期、分片数如何规划、以及如何靠读写别名做零停机重建。
