# 1. 部署方式选型

Kibana 的部署方式没有绝对优劣，核心决策点是**运维归属**（交给平台/容器编排还是本机 systemd）与**规模**（单实例体验还是多实例生产）。四条主流路径：Docker/Compose 适合本地开发与单机自托管；RPM/DEB/ tar.gz 适合传统虚拟机 systemd 管理；Kubernetes 官方 ECK Operator（`kind: Kibana` CRD）适合已容器化的团队；Elastic Cloud 托管则完全免运维。选型对照：

| 方式 | 适用场景 | 升级方式 | 高可用 |
|------|---------|---------|--------|
| Docker/Compose | 本地开发、Demo、单机 | 改镜像 tag | 需自己加 LB 多副本 |
| RPM/DEB | 虚拟机、systemd 管控 | `yum/apt upgrade` | 多实例 + Nginx |
| tar.gz 解压 | 无 root、自定义路径 | 换目录软链 | 多实例 + Nginx |
| ECK (K8s) | 容器化团队 | 改 CR 版本 | Operator 自动多副本 |
| Elastic Cloud | 免运维 | 控制台点选 | 平台内置 |

> **注意**：无论哪种方式，Kibana 与 ES 必须**版本一致**（至少主版本一致且 ES 补丁版本不低于 Kibana），并推荐同批次一起升级。8.x 默认开启安全与 TLS，"装完就能连裸 ES"的 7.x 时代体验已不存在，首次连接必然涉及凭据或 enrollment token（见 3.2）。

# 2. 前提：ES 侧准备

Kibana 对 ES 没有任何额外要求，只要求**能连上、有凭据、集群健康**。部署前先验证 ES 可达：

```bash
# 1) 确认 ES 健康（黄色可接受，绿色最佳）
curl -k -u elastic:密码 https://es1:9200/_cluster/health?pretty

# 2) 若 ES 启用了 xpack.security.enabled=true（8.x 默认），
#    必须为内置的 kibana_system 账号设置密码：
bin/elasticsearch-reset-password -u kibana_system
#    输出生成的密码，稍后填进 kibana.yml
```

`kibana_system` 是 ES 内置的、专为 Kibana 服务端连接设计的系统账号，权限只覆盖 `.kibana*` 索引读写与集群状态查看——**它不能读业务索引**，Kibana 界面里普通用户能否看某索引数据，取决于登录用户自己的角色授权，而不是这个账号。

# 3. Docker 部署

Docker 是最快的起步方式。单机联调可直接用环境变量覆盖 kibana.yml：

```bash
# Kibana 8.x
docker run -d \
  --name kibana \
  -p 5601:5601 \
  -e ELASTICSEARCH_HOSTS=http://es:9200 \
  -e ELASTICSEARCH_USERNAME=kibana_system \
  -e ELASTICSEARCH_PASSWORD=${KIBANA_PASSWORD} \
  docker.elastic.co/kibana/kibana:8.11.0

# 验证
curl http://localhost:5601/api/status
# {"status": {"overall": {"level": "available"}}}
```

若 ES 也开安全（8.x 默认），最简单的联调方式是用 **elastic 超管账号**临时连接，或走 enrollment 流程；正式环境务必换回 `kibana_system`。`/api/status` 是 Kibana 最重要的健康端点：`overall.level` 取 `available`（正常）、`degraded`（部分插件红，如某 ES 节点不可达）、`critical`（核心不可用），K8s 的 readiness/liveness 探针就用它。

> **行业实践**：`elastic` 超管账号只用于初始配置与救火，Kibana 服务连接永远使用专用的 `kibana_system` 账号——既是权限最小化的要求，也方便审计"哪些请求来自 Kibana 本身"。

生产镜像默认以 **uid 1000（kibana 用户）**运行，挂载本地目录做日志/数据持久化时注意目录属主，否则报 `EACCES: permission denied`。K8s 下推荐用 ECK：

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-prod
spec:
  version: 8.11.0
  count: 2                       # 两副本 + LB 即高可用
  elasticsearchRef:
    name: es-prod                # 同 namespace 的 ES CR，自动注入凭据
  config:
    server.publicBaseUrl: "https://kibana.example.com"
```

# 4. Linux 安装（RPM/DEB/tar.gz）

传统虚拟机上用发行版包管理器安装，systemd 接管生命周期。CentOS/RHEL：

```bash
# CentOS/RHEL
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch

cat > /etc/yum.repos.d/kibana.repo << EOF
[kibana-8.x]
name=Kibana repository for 8.x packages
baseurl=https://artifacts.elastic.co/packages/8.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
EOF

yum install -y kibana
systemctl start kibana
systemctl enable kibana
```

Debian/Ubuntu 对应 apt 源（`deb https://artifacts.elastic.co/packages/8.x/apt stable main`），装完同样 `systemctl enable --now kibana`。无 root 或自定义路径场景用 tar.gz：解压后改 `config/kibana.yml`，用 `bin/kibana` 前台起或自行写 systemd unit；`KIBANA_HOME` 由 unit 里的 `Environment` 决定。包安装的标准路径约定值得记住：配置在 `/etc/kibana/kibana.yml`，数据在 `/var/lib/kibana`，日志在 `/var/log/kibana`，运行用户为 `kibana`。

# 5. 首次连接与安全开关

安装完成后的第一次启动是 Kibana 部署里最容易卡住的环节，取决于 ES 的安全模式：

```text
情形 A：ES 开启安全（8.x 默认）
  1) bin/elasticsearch-reset-password -u kibana_system   # 设系统账号密码
  2) kibana.yml 填 elasticsearch.username/password + CA 证书
  3) 启动 kibana，浏览器打开 http://localhost:5601
  4) 用 elastic（或自建管理员）登录 Web UI

情形 B：本地实验，关掉 ES 安全（仅实验，勿上生产）
  elasticsearch.yml:  xpack.security.enabled: false
  kibana.yml:         elasticsearch.hosts: ["http://localhost:9200"]
```

与"安全"相关的 Kibana 侧 xpack 开关按需了解即可，默认值多数不用动：

```yaml
xpack.security.enabled: true                 # 登录鉴权（8.x 默认开）
xpack.security.encryptionKey: "32+字符随机串" # 会话加密，必设
xpack.encryptedSavedObjects.encryptionKey: "..." # 连接器凭据加密，告警必设
xpack.security.authc.api_key.enabled: true   # 供脚本/API Key 访问
```

> **注意**：8.x 起若 ES 启用了 TLS，Kibana 的 `elasticsearch.ssl.certificateAuthorities` 必须配信任 CA，否则启动即报 `unable to verify the first certificate`；自签证书场景只配 CA 路径即可，不要用 `certificateAuthorities: []` 之类的"绕过校验"写法。同时 `xpack.security.encryptionKey` 未设置时每次重启都会随机生成，导致所有人会话失效——生产环境务必写死并纳入密钥管理。

# 6. 导入示例数据

装好后的第一件事是造数据验证全链路。Kibana 内置样例数据集，Home 页或 `Stack Management → Trial and sample data` 一键导入：

```text
Kibana Home → Try sample data
  ├── Sample eCommerce orders   → 电商订单数据
  ├── Sample flight data        → 航班数据
  └── Sample web logs           → Web 访问日志

点击 "Add data" → 自动创建索引 + Dashboard + Visualizations
```

导入动作会创建 `kibana_sample_data_ecommerce` 等索引、灌入约 1 万条文档，并**自动建好对应的 Data View 和示例 Dashboard**，几分钟内就能看到完整效果，非常适合验证"写入-检索-可视化"是否通畅。用命令行造数据时用 bulk API 更高效（详见 ELK 其他篇），不要为了测试而依赖样例数据长期运行。

# 7. 第一个 Dashboard

样例数据就绪后，走一遍最小闭环，串起后续各篇的核心概念（Data View → Discover → Lens → Dashboard）：

```text
1. 创建 Data View
   Stack Management → Data Views → Create
   Name: orders
   Index pattern: kibana_sample_data_ecommerce
   Time field: order_date

2. 进入 Discover
   选择 orders Data View
   搜索: customer_gender:FEMALE

3. 创建 Lens 图表
   拖拽 category.keyword → 柱状图
   拖拽 taxful_total_price → 求平均值作为 Y 轴

4. 保存到 Dashboard
   新建 Dashboard → 添加 Lens 面板 → 调整布局
```

这一步跑通说明：ES 连接、认证、索引读取、字段解析、聚合渲染全链路正常，之后排障就不必怀疑"环境问题"。

# 8. Spaces 多空间

多团队共用一套 Kibana 时，用 Spaces 做资源与权限隔离，而不是建多套 Kibana：

```text
Stack Management → Spaces → Create Space
  Name: Team A
  Features: Discover, Dashboard, Visualize
  Permissions: 仅 Team A 成员可访问

创建 Space 后：
  - Discover 仅显示该 Space 的 Data View
  - Dashboard 仅显示该 Space 的仪表盘
  - 切换 Space 顶栏下拉菜单
```

隔离的粒度是 **Space + 角色**两层：Space 决定"哪些资源可见"，角色（在 Security 里按 Space 分配）决定"能不能看/能不能改"。典型用法是给每个业务团队一个 Space、绑定 `kibana_space_<team>` 角色，成员默认落在自己空间，运维团队用 `default` 空间统管。

# 9. 核心 UI 导航

Kibana 的左侧菜单按"分析场景"分组，8.x 的标准结构：

```text
┌─────────────────────────────────────┐
│  Main Menu                          │
│  ├── Discover          (数据探索)    │
│  ├── Dashboards        (仪表盘)     │
│  ├── Visualize Library (可视化库)   │
│  ├── Maps              (地图)       │
│  └── Stack Management  (管理)       │
│                                     │
│  Observability                       │
│  ├── Overview                        │
│  ├── APM                             │
│  ├── Infrastructure                  │
│  └── Uptime                          │
│                                     │
│  Security                            │
│  ├── Alerts                          │
│  └── Cases                           │
│                                     │
│  Analytics                           │
│  ├── Machine Learning               │
│  ├── Graph                          │
│  └── Canvas                         │
└─────────────────────────────────────┘
```

Observability/Security/Analytics 三组是否出现取决于对应 xpack 模块是否启用与 License 层级——Basic 下 ML、SIEM 部分功能会提示升级。**Java 后端日常 90% 时间在 Main Menu 的 Discover/Dashboards/Stack Management**，其余按需进入。

# 10. Dev Tools 控制台

Dev Tools 的 Console 是操作 ES 的瑞士军刀，它把请求透明代理到当前连接的 ES，免去了手拼 curl 与认证：

```json
// Dev Tools → Console（类似 ES REST 客户端）
GET /_cat/indices?v
GET /orders/_search
PUT /test/_doc/1
{ "name": "test" }
```

Console 的 Tab 补全、多语句选中执行（选中部分点 Run）、自动格式化都极提升效率；它后面还有几个常被忽略的标签页：**Search Profiler** 可视化查询执行过程定位慢聚合，**Grok Debugger** 在线调试 Logstash/Ingest 的 grok 模式，**Console 右侧命令面板**（Ctrl/Cmd+K）可一键插入常用 API。

# 11. 常用配置速查

最小可用配置（本地 HTTP 无安全场景）：

```yaml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://es:9200"]
kibana.defaultAppId: "discover"
logging.root.level: info
```

所有配置项都有等价环境变量（点号与横线换成下划线并大写，如 `server.port` → `SERVER_PORT`），K8s/Docker 场景优先用环境变量注入，敏感值走 Secret，避免把密码写进镜像里的 yml。

# 12. 小结

部署路径按规模选：本地 Docker 起步、虚拟机用包管理器 + systemd、容器化交给 ECK、想省心用 Elastic Cloud；无论哪种，先备好 ES 侧条件（健康 + `kibana_system` 密码 + 版本对齐），再写 kibana.yml 的三组核心配置（server、elasticsearch、xpack/security 加密键）。首次启动最容易卡在 TLS 证书与 enrollment 流程，用 `curl /api/status` 与 `overall.level` 快速判定健康。落地验证走"导入 sample data → 建 Data View → Discover → Lens → Dashboard"最小闭环即可确认全链路通畅，多团队用 Spaces + 角色隔离，日常操作与调试离不开 Dev Tools Console。
