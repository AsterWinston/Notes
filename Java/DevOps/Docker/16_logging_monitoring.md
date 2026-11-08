
# 1. 核心原则：应用输出到 stdout/stderr

容器时代最稳的日志模式：

1. 应用把日志写到 stdout/stderr。
2. Docker 负责收集与转发（logging driver）。
3. 需要集中化时，在宿主机侧或平台侧统一采集。

不要把日志写进容器文件系统当作主路径（容器可重建、文件可能丢失、权限复杂）。这条原则的深层逻辑是**十二要素应用的"日志是事件流"**：应用不管理日志的生命周期（轮转、压缩、清理、上传全部交给运行时环境），它只负责把结构化事件写到标准输出。对 Java 应用来说就是 Logback/Log4j2 配置一个 ConsoleAppender 关掉 FileAppender，让 logging driver 与平台采集器接管其余工作。写进容器内文件的代价：容器删除即丢失、多副本时文件分散在各节点、轮转策略无法统一管理、GC 日志/heapdump 这类高写入路径还会撑爆可写层。

# 2. Docker 日志基本命令

```bash
docker logs -f <container>
docker logs --since=10m <container>
docker logs --tail=200 <container>
docker logs --timestamps <container>      # 每条日志带时间戳, 对齐链路排障必备
```

提示：

1. `docker logs` 是否可用取决于 logging driver（例如 `none` driver 不支持；`fluentd`/`syslog` 远端存储后本地也就没了）。
2. 调试阶段很常用，但生产更需要集中采集与检索。

`docker logs` 的实现值得了解：读取的是 logging driver 的持久化输出（json-file 就是直接读 JSON 文件），多行日志（如 Java 堆栈）在 json-file 里是**每条 JSON 一行**，堆栈的多行被 `STREAM` 字段区分 stdout/stderr 保存完整；但如果应用逐行写且中间有换行异常（`printf` 跨调用），采集侧看到的可能是碎片——Java 端要保证一次 `logging.info` 输出一条完整多行记录。

# 3. Logging driver：常见选择与取舍

查看当前 driver：

```bash
docker info | grep "Logging Driver"
docker inspect <container> --format '{{.HostConfig.LogConfig.Type}}'
```

典型 driver：

| driver | 存储位置 | docker logs 支持 | 轮转 | 适用 |
|--------|----------|------------------|------|------|
| **json-file** | /var/lib/docker/containers/<id>/<id>-json.log | 支持 | 需手动配 max-size/max-file | 默认; 单机/小集群 |
| **local** | /var/lib/docker/containers/<id>/ 自定义格式 | 支持 | 默认开启 | 替代 json-file, 更省空间 |
| **journald** | systemd journal | 支持 | 随 journald 配置 | systemd 发行版统一管理 |
| **syslog** | 远端 syslog 服务器 | 不支持 | 服务器侧 | 对接传统日志平台 |
| **fluentd/fluent-bit** | 远端采集器 | 不支持 | 采集器侧 | EFK/自建日志管道 |
| **awslogs/gcplogs** | 云厂商服务 | 支持(经API) | 云侧 | 云上托管 |
| **none** | 无 | - | - | 安全敏感场景显式关闭 |

工程建议（偏 Docker 原生）：

1. 单机/小规模优先 `local` 或 `json-file + 轮转`。
2. 更复杂的集中化再考虑专门的日志采集链路，但不要把它写成"必须上 ELK/Prometheus"。

> **注意**：json-file 有两个隐性成本经常被低估——**逐行 JSON 转义的 CPU 与写放大**（一行日志包成 `{"log":"...","stream":"...","time":"..."}` 约多 30-50% 字节），以及**大行问题**：异常堆栈一行几百 KB 时 max-size 按文件计数、不按行计数，单条超大日志可以突破轮转粒度。高吞吐服务（网关、埋点入口）用 `local` driver 或 fluent-bit 直收能明显省 CPU。

# 4. 日志轮转（重点）：json-file vs journald

如果使用 `json-file`，必须配置轮转，否则宿主机磁盘可能被打爆。

daemon 级别配置（`/etc/docker/daemon.json`，Windows/WSL 路径不同）：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

也可以在容器级别覆盖：

```bash
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  my-app:1.0
```

Compose 等价写法：

```yaml
services:
  app:
    image: my-app:1.0
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

提示：

1. 配完记得重启 Docker daemon 才生效（daemon 配置），且**已存在的容器不继承新配置**，需要重建。
2. `max-size/max-file` 是最常用的安全阀，单容器日志占用上限 ≈ max-size × max-file。

**journald driver 的轮转是另一套机制**：日志写入 systemd journal，大小由 `/etc/systemd/journald.conf` 的 `SystemMaxUse`（默认磁盘的 10%，上限固定比例）控制，天然带结构化字段（`CONTAINER_NAME_TAG`、`UNIT`），查询用 `journalctl CONTAINER_NAME=my-app --since "10 min ago"`。两者取舍：

| 维度 | json-file + 轮转 | journald |
|------|------------------|----------|
| 轮转主体 | docker(按容器) | systemd(全局配额) |
| 单容器隔离 | 强(每容器独立上限) | 弱(全局共享, 一个容器刷屏挤掉别人) |
| 与 systemd 服务共存 | 各看各的 | 统一入口查询 |
| 采集器适配 | Filebeat 直接读文件 | journald input(读原始文件有格式耦合) |
| 适用 | 容器密度高、需要按容器限额 | 容器+systemd 混合的小型主机 |

# 5. 监控：Docker 自带的几个观察入口

## 5.1 `docker stats`：实时资源与其原理

```bash
docker stats --no-stream             # 一次性快照, 脚本采集用
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemPerc}}"
```

它能快速告诉你：CPU、内存、网络、块 IO 是否异常。**`docker stats` 的数据来源是 cgroup 计数器**（v1 读各子系统 `memory.usage_in_bytes`、`cpuacct.usage`；v2 读 `memory.current`、`cpu.stat`），网络与 blkio 统计由 libnetwork/存储侧补充，不是"容器里装了 agent"。两个易误读点：CPU% 可超过 100%（多核累计，宿主机 16 核跑满显示 1600%）；内存百分比 = usage / limit（**没设 limit 时按宿主机总内存算**，小值不代表健康，cache 也计入 usage，看到 90% 先分清 active_anon 与 file cache）。

## 5.2 `docker events`：事件流（排障非常有用）

```bash
docker events --since 30m
docker events --filter 'event=die' --filter 'event=oom'
```

你能看到：容器重启、OOM、拉取镜像、网络变更等事件。

## 5.3 `docker inspect`：配置与状态真相

重点字段：

1. `State`（退出码、是否 OOMKilled）
2. `HostConfig`（资源限制、重启策略、挂载、capabilities）

# 6. Prometheus + cAdvisor：生产指标链路

单机命令只能救火，容量与告警要靠指标系统。节点侧用 **cAdvisor**（container advisor）暴露容器级指标——它直接读 cgroup 与 /proc，是 Google 开源、K8s 时代节点监控的事实标准：

```yaml
# docker-compose.monitoring.yml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    privileged: true          # 需要读 /dev 与 cgroup, 或改用下面更安全的只读挂载组合
    devices: [/dev/kmsg]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    ports: ["8080:8080"]

  prometheus:
    image: prom/prometheus:v2.53.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports: ["9090:9090"]
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: cadvisor
    static_configs: [{ targets: ["cadvisor:8080"] }]
```

核心指标与告警思路：

```text
container_cpu_usage_seconds_total   → rate() 算实际用量, 对比 spec_cpu_limit 看 throttle 风险
container_memory_working_set_bytes  → K8s OOM 同源口径, 告警阈值 = limit 的 90%
container_cpu_cfs_throttled_seconds_total → 节流时长, 对应 12 篇的 cpu.stat
container_fs_reads/writes_bytes_total     → IO 热点定位
```

> **注意**：cAdvisor 需要访问宿主机 cgroup 与 docker socket，`privileged` 是常见但粗暴的给法，收紧方式是只读挂载 + 显式 `--docker_only`；K8s 环境里 kubelet 已内置 cAdvisor，无需再部署一份，`kubelet /metrics/cadvisor` 端点直接抓。Java 应用侧再补 Micrometer + `/actuator/prometheus`，业务指标（QPS、P99、线程池、GC）与容器指标在同一 Prometheus 里才能交叉定位（"P99 抖动 → 同时看 GC pause 与 cpu throttle"）。

# 7. 健康检查：从"进程活着"到"服务可用"

Docker 的 healthcheck 是基础能力，可作为：

1. Compose 的依赖条件（`depends_on` + `service_healthy`）。
2. 运维排障的信号：容器 running 但 unhealthy。

示例：

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --retries=5 --start-period=40s \
  CMD curl -fsS http://localhost:8080/actuator/health || exit 1
```

注意：

1. healthcheck 应轻量、快速。
2. 不要把健康检查写成"全链路深度探测"（连接 DB/调下游的 readiness 逻辑混进 liveness），容易把依赖抖动放大成整服务重启。
3. JVM 必须配 `start-period`，否则类加载期就被判死。
4. Docker 原生 healthcheck 失败默认**不会自动重启容器**（与 K8s liveness 语义不同），只有 Compose/K8s/Swarm 各自利用这个状态做治理。

# 8. Java 容器日志采集链路（Filebeat → ES）

生产常见链路：stdout 落 json-file → Filebeat 尾读 → （Logstash 可选加工）→ Elasticsearch → Kibana：

```yaml
# filebeat.yml(跑在每台宿主机的采集器, K8s 里则 DaemonSet)
filebeat.inputs:
  - type: container                # 原生支持 docker 日志格式, 自动附带容器元数据
    paths:
      - /var/lib/docker/containers/*/*-json.log

processors:
  - add_fields: { target: '', fields: { env: prod, cluster: pay-01 } }
  - drop_fields: { fields: ["agent.ephemeral_id", "host.os"], ignore_missing: true }

output.elasticsearch:
  hosts: ["es.corp.com:9200"]

setup.template.settings:
  index.number_of_shards: 3

# 按天滚动索引 + ILM 生命周期(冷热分层/过期删除)
setup.ilm.enabled: true
filebeat.registry.path: /var/lib/filebeat
```

```yaml
# logback-spring.xml 关键片段: JSON 一行一事件, 便于 ES mapping 与跨服务检索
appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <customFields>{"service":"order-service","env":"prod"}</customFields>
    <includeMdcKeyName>traceId</includeMdcKeyName>   # 链路 ID 必须进日志
  </encoder>
</appender>
```

链路设计要点：

```text
JVM stdout ──▶ docker json-file ──▶ Filebeat(每节点1个, 背压时丢弃旧行) ──▶ Kafka(可选削峰) ──▶ ES
```

1. **采集器与容器生命周期竞态**：容器删除时 Filebeat 要读完残留再 unregister（`clean_inactive` 设置不当会丢最后几条错误日志），type: container input 已处理这一时序。
2. **多行堆栈**：json-file 已按行封装，Java 侧用 JSON encoder 保证"一次异常 = 一条事件"，不要在采集侧拼多行。
3. **削峰**：大促/故障时日志洪峰会拖垮 ES，Kafka 缓冲 + Filebeat 限流（`rate_limit`）是大厂标配，小团队至少配 ES ILM 冷热分层 + Filebeat registry 背压容忍。

# 9. 排障清单（日志 + 监控联动）

当你遇到"容器一直重启/服务不可用"：

1. `docker ps -a` 看重启频率与退出码。
2. `docker logs --tail=200 <container>` 看最后一段日志。
3. `docker inspect <container>` 看 `OOMKilled`、资源限制、启动命令。
4. `docker stats` 看是否 CPU/内存顶满。
5. `docker events --since 30m` 看是否有 OOM、kill、pull 等事件。
6. Prometheus 面板对照时间线：重启时刻的 working_set 是否贴近 limit、throttled 是否突增、GC 暂停是否变长。

> **行业实践**：退出码是最快的分类器——**137** 找 OOM/kill（`12_resource_limits.md` 路径）、**143** 是 SIGTERM 优雅退出（重启策略或编排平台动作）、**1** 看应用异常栈、healthcheck 相关的反复重启用 `docker inspect` 的 `State.Health` 确认是不是探针误杀。

# 10. 面试要点

| 问题 | 答题关键词 |
|------|------------|
| 容器日志最佳实践 | stdout/stderr + logging driver, 应用不管生命周期 |
| 为什么要配置轮转 | 防止 json-file 把磁盘打爆 |
| json-file vs journald | 按容器限额 vs systemd 全局配额, 查询入口不同 |
| `docker stats` 原理 | cgroup 计数器(cpuacct/memory), 无 agent |
| cAdvisor 是什么 | 节点容器指标 exporter, K8s 内置于 kubelet |
| healthcheck 解决什么 | 区分 running 与可用，辅助编排依赖；Docker 原生不自动重启 |

# 11. 小结

可观测性这一篇按"**日志流**与**指标流**"两条线收束。日志线：应用只写 stdout（十二要素），Docker driver 负责落地与轮转——json-file 必须配 max-size/max-file（或换默认带轮转的 local），journald 走 systemd 全局配额，规模化后交给 Filebeat（type: container input 处理竞态与元数据）进 ES/Kafka；Java 侧用 JSON encoder 保证"一事件一行"并带 traceId。指标线：`docker stats` 本质是 cgroup 计数器的展示（注意 CPU% 多核累计与 cache 计入内存两个坑），生产用 cAdvisor（K8s 用 kubelet 内置端点）暴露 working_set、throttled 等关键序列，告警口径对齐"OOM 看 working_set 对 limit、延迟毛刺看 cpu throttle、重启先看 137/143"。healthcheck 是连接两者的信号：running ≠ 可用，start-period 与轻量探针是 JVM 场景的两条硬规则。编排平台如何消费这些信号做发布治理，是下一篇（`17_orchestration.md`）的主题。
