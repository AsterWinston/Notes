# 1. 环境要求

| 组件 | 要求 |
|------|------|
| JDK | 11+（8.x）/ 8+（7.x） |
| 内存 | 最小 1GB，推荐 2GB+ |
| OS | Linux/macOS/Windows |

Logstash 8.x 安装包**自带 JDK（OpenJDK  bundled）**，"装 Logstash 先装 Java"在 8.x 已经是误解——除非你想让它用系统 JDK（通过 `LS_JAVA_HOME` 指定，常见动机是统一升级节奏或合规要求）。真正的容量规划在内存：默认 `jvm.options` 给出 `-Xms1g -Xmx1g`，这只是"能跑起来"的下限；经验法则是**堆至少 2GB、不超过物理内存的一半**，且同一台机器不要塞多个 Logstash 实例共享一个小堆。生产环境常见的翻车是"2GB 的机器跑 Logstash 又跑别的"，堆外内存（JRuby 运行时、Netty 直接内存、metaspace）被挤掉触发 OS OOM Killer——JVM 被 `kill -9` 且日志里没有任何 OutOfMemoryError。

> **注意**：`-Xmx` 与物理内存之间要给 JVM 堆外留余量，官方建议堆不超过物理内存的 50%~75%。beats input、tcp input 用 Netty 堆外缓冲，高连接数场景下 RSS 明显大于 Xmx 是正常现象，不要用"Xmx=机器内存"的直觉配生产。

# 2. Docker 部署

```bash
# Logstash 8.x
docker run -d \
  --name logstash \
  -v /path/to/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
  -v /path/to/logs:/var/log/app \
  docker.elastic.co/logstash/logstash:8.11.0
```

镜像把 pipeline 目录挂载点约定在 `/usr/share/logstash/pipeline/`：该目录下**所有 `.conf` 文件按文件名顺序合并成一条管道**（等价于 `path.config` 指向 `pipeline/*.conf`），换多条管道则挂载 `pipelines.yml` 覆盖默认。常用环境变量的完整清单见第 8 节；`-e LS_JAVA_OPTS=-Xms2g -Xmx2g` 调堆是 Docker 场景的标配，因为镜像默认堆对小容器规格并不合适。

**设计考量**：容器化 Logstash 的存储语义要想清楚——sincedb、PQ、DLQ 都在容器文件系统里，容器重建即归零。**至少把 `/usr/share/logstash/data`（默认数据目录，含上述三者）挂出来**，否则每次发布都等于"从头重读所有日志 + 丢队列"。K8s 里 Logstash 更适合 StatefulSet（有盘、有稳定标识），无状态 Deployment 形态只适合"无持久化需求的直通管道"。

# 3. Linux 安装

```bash
# CentOS/RHEL
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat > /etc/yum.repos.d/logstash.repo << EOF
[logstash-8.x]
name=Logstash repository for 8.x
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
enabled=1
EOF

yum install -y logstash
systemctl start logstash
```

RPM/DEB 安装的默认布局值得背下来，排障时全靠它：

| 路径 | 内容 |
|------|------|
| `/usr/share/logstash` | 程序本体（含 bundled JDK） |
| `/etc/logstash/logstash.yml` | 主配置 |
| `/etc/logstash/conf.d/` | 管道配置目录（`path.config` 默认指向） |
| `/etc/logstash/jvm.options` | JVM 参数 |
| `/var/lib/logstash` | 数据目录（sincedb、PQ、DLQ） |
| `/var/log/logstash` | Logstash 自身的日志 |

服务以 `logstash` 用户运行，`systemctl status logstash` 看状态，`journalctl -u logstash -f` 看启动输出。tar 包解压安装则一切路径自己定，适合"多实例、自定义挂载盘"的场景；`LS_HOME` 环境变量控制实例根目录。Debian/Ubuntu 用 `apt-get install logstash` 同理，仓库配 `https://artifacts.elastic.co/packages/8.x/apt`。

# 4. Hello World — stdin → stdout

```text
# logstash.conf
input {
  stdin { }
}

output {
  stdout { codec => rubydebug }
}
```

```bash
bin/logstash -f logstash.conf

# 输入 "hello world"
# 输出:
# {
#   "message" => "hello world",
#   "@timestamp" => 2024-01-15T10:30:00.000Z,
#   "host" => "my-host",
#   "@version" => "1"
# }
```

这个最小例子透露了三个默认行为：没有 filter 时原始输入**原样进 `message` 字段**；`@timestamp` 取"事件进入管道的时间"（不是日志发生的时间——这就是为什么后面凡是日志类管道都要配 date filter）；`host` 字段由 input 自动添加。stdin input 按行读取，输入 EOF（Ctrl-D/Ctrl-Z）会结束管道从而退出进程，常被用来"跑完即停"地测试一段配置。

## 4.1 -e 单行调试

`-e` 允许把整条管道写进命令行，不用建文件，是验证插件参数和 grok pattern 的最快手段：

```bash
bin/logstash -e 'input { stdin{} } filter { mutate { add_field => { "hello" => "world" } } } output { stdout { codec => rubydebug } }'
```

配合 `echo '{...}' | bin/logstash -e '...'` 还能把构造样本喂给管道做一次性回归。

## 4.2 -t 配置测试与 --config.test_and_exit

`-t`（全写 `--config.test_and_exit`）**只解析并校验配置就退出**，不连接任何真实数据源、不启动 worker：

```bash
bin/logstash -f pipeline.conf --config.test_and_exit
# Configuration OK
```

它检查 DSL 语法、插件参数合法性、条件表达式合法性，但**不校验**：grok pattern 对不对得上真实日志、ES mapping 会不会冲突、账号密码是否可用——这些是运行期错误。CI 里放一步 `-t` 能拦掉绝大部分"低级配置错误"，性价比极高；进阶做法是 `--config.debug`（打印解析后的完整配置对象，排查变量插值问题时尤其有用）。

# 5. 完整管道 — File → Grok → ES

```text
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/sincedb"
  }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
  geoip {
    source => "clientip"
  }
  mutate {
    remove_field => ["@version", "message", "timestamp"]
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "nginx-access-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ES_PASSWORD}"
  }
  stdout { codec => rubydebug }
}
```

逐段读这份"模板级"配置：file input 用 `start_position => "beginning"` 从文件头读（**仅对 sincedb 中无记录的文件生效**，重复测试要删 sincedb 或改路径）；grok 复用内置组合模式拆出 `clientip/timestamp/response` 等；date 把 nginx 的 HTTPDATE 字符串换算成真正的 `@timestamp`；geoip 基于 `clientip` 挂上地理块；mutate 在写 ES 前删掉已无用的原始字段瘦身。output 同时写 ES 和 stdout——`rubydebug` 打印用来肉眼验证最终文档长相，验证完删掉，别带到生产。

> **注意**：8.x 默认对 ES 开启安全认证，`user/password` 与 TLS 是必配项而非可选项，明文 `http://` + 无密码的组合只在明确关闭安全的实验环境成立。生产密码永远不写死在配置文件里——用 `${ES_PASSWORD}` 环境变量插值或 keystore（见第 8 节）。

# 6. logstash.yml 核心配置

```yaml
# 管道配置路径
path.config: /etc/logstash/conf.d/*.conf

# 数据路径
path.data: /var/lib/logstash
path.logs: /var/log/logstash

# 队列
queue.type: persisted
queue.max_bytes: 2gb
path.queue: /var/lib/logstash/queue

# 死信队列
dead_letter_queue.enable: true

# 管道性能
pipeline.workers: 4
pipeline.batch.size: 125
pipeline.batch.delay: 50

# JVM 配置 (jvm.options)
# -Xms1g -Xmx2g

# API 端点
api.enabled: true
api.http.host: "0.0.0.0"
api.http.port: 9600

# 自动重载
config.reload.automatic: true
config.reload.interval: 3s
```

按用途分组记忆：**路径组**（`path.config/data/logs/queue`）决定"程序和状态各在哪"，`path.data` 是 7.x 之后多数状态文件的统一根（PQ、DLQ、sincedb 默认都在它下面派生）；**队列组**决定可靠性形态；**管道组**决定吞吐；**API 组**暴露监控端点；**重载组**决定改配置要不要重启。JVM 参数不在此文件，而在 `jvm.options`（8.x 默认 1GB 堆，务必按容量规划调）。`config.reload.automatic` 的"热重载"实际是**整条管道 teardown 再重建**：不丢队列里的事件，但插件内部状态（如 translate 缓存、ruby filter 累加值）会重置，且重建瞬间有吞吐凹陷——把重载间隔设太短（如 1s）等于持续抖动。

# 7. 配置校验

```bash
# 校验语法（不实际运行）
bin/logstash -f pipeline.conf --config.test_and_exit

# 自动重载（修改配置后无需重启）
# logstash.yml: config.reload.automatic: true

# 查看 API
curl http://localhost:9600/_node/stats
curl http://localhost:9600/_node/pipelines
```

9600 端口的 Monitoring API 是排障第一现场，常用四个端点：`/_node/pipelines`（管道状态）、`/_node/stats/events`（吞吐与 in_flight/queued）、`/_node/stats/queue`（PQ 容量与数据量）、`/_node/stats/plugins`（每个插件的耗时与失败数，定位"慢在哪个 filter"）。

```bash
# 只取最关心的指标
curl -s http://localhost:9600/_node/stats | \
  jq '{events: .events, queue: .persistent_queue, pipeline: .pipelines}'
```

> **行业实践**：把 `api.http.host` 绑到内网地址并加防火墙/反向代理认证——`_node/stats` 会暴露路径、插件版本等指纹信息，公网裸奔等于送侦察情报。接入 Kibana Stack Monitoring（`xpack.monitoring.enabled` 相关配置）或 Prometheus 的 logstash exporter，把 `events.in_flight`、`events.queued`、PQ 水位三项做成告警。

# 8. 环境变量与 keystore

```text
# pipeline.conf 中使用环境变量
input {
  kafka {
    bootstrap_servers => "${KAFKA_BROKERS:localhost:9092}"
    topics => ["${TOPIC:default-topic}"]
  }
}

output {
  elasticsearch {
    hosts => ["${ES_HOST:localhost:9200}"]
    user => "${ES_USER:elastic}"
    password => "${ES_PASSWORD}"
  }
}
```

`${VAR}` 在**配置加载时**做一次文本替换，`${VAR:default}` 提供缺省值——两个细节容易踩：其一，插值发生在字符串层面，`"${PORT}"` 替换后仍是字符串 `"9600"`，某些要求整数的参数会报错，正确写法是 `port => "${PORT:5044}"` 且插件侧声明了类型强转（多数核心插件已处理）；其二，**默认值只在变量未定义时生效，定义为空串也会命中空值导致启动失败**。

**keystore** 是官方推荐的敏感信息管理方式，密文存盘、值不落配置文件：

```bash
# 创建 keystore 并写入密钥
bin/keystore create
bin/keystore add ES_PASSWORD
# 交互式输入密码，密文保存在 <path.data>/keystore

# 使用与配置文件里的 ${ES_PASSWORD} 完全相同，无需改动管道
bin/logstash -f pipeline.conf

# 常用子命令
bin/keystore list              # 列出已存密钥名
bin/keystore remove ES_PASSWORD
```

keystore 的密码保护（`bin/keystore --no-keystore-file-creation-check`、`LS_KEYSTORE_PASSWD`）可防止"文件复制即密钥泄露"；CI/CD 场景则常用 `bin/keystore import` 从加密文件批量恢复。

# 9. 运行方式与守护

三种启动形态各有归宿：**前台 `bin/logstash -f`** 用于调试；**systemd 服务**（rpm/deb 安装自带 unit）用于虚机生产；**容器**用于 K8s/ECS。多管道统一用 `pipelines.yml` 声明（每行一个 `pipeline.id + path.config`），`bin/logstash` 不带 `-f` 时默认读取它。平滑停止用 `Ctrl-C` 或 `systemctl stop`（SIGTERM）：Logstash 会等当前 batch 处理完、尽量 flush 输出后退出——这也是"PQ 里的事件重启不丢、内存队列里的事件重启即丢"在运维层面的直接体现。

# 10. 小结

安装运行的要点可以归为五句话：**用自带 JDK 但按容量规划调堆**（默认 1GB 只是下限，堆外要留余量）；**路径布局背熟**（conf.d 放管道、data 放状态、logs 放自身日志）；**`-t` 进 CI、`-e` 做单行实验、`rubydebug` 做肉眼验证**；**生产必开 PQ + DLQ + keystore**，密码不进配置文件；**9600 API 是排障第一现场**——`in_flight` 看下游、`queued` 看 worker、PQ 水位看磁盘，三项做告警。Docker/K8s 部署时把 `data` 目录持久化出来，是所有"重启丢进度"事故的统一解药。
