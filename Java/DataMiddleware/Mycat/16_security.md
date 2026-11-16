# 1. 用户认证

**Mycat 的安全模型是"代理层账号体系"**：应用不直接持有 MySQL 账号，而是登录 Mycat（8066），由 Mycat 用它自己的后端账号连 MySQL。这带来两个安全效果——数据库账号与密码被收敛到中间件一处管理，应用侧泄露面收窄；同时 Mycat 获得了"统一管控点"，权限、白名单、防火墙都在这一层做。代价是 Mycat 自身的 users 配置成了新的要害：**拿到 8066 的合法账号，等于拿到所有分片数据的入口**，其保护级别应当等同甚至高于数据库账号。

## 1.1 conf/users.json

```json
{
  "users": {
    "root": {
      "password": "123456",
      "schemas": ["order_db", "user_db"],
      "ipWhiteList": "192.168.1.0/24,10.0.0.100"
    },
    "readonly": {
      "password": "readonly123",
      "schemas": ["order_db"],
      "readOnly": true
    },
    "app_user": {
      "password": "app123",
      "schemas": ["order_db"]
    }
  }
}
```

| 配置项 | 说明 |
|--------|------|
| `password` | 明文或加密密码 |
| `schemas` | 可访问的逻辑 Schema 列表 |
| `ipWhiteList` | IP 白名单（CIDR 格式） |
| `readOnly` | 只读用户（所有写操作被拒绝） |
| `maxCon` | 该用户最大连接数 |

**账号划分的实操原则是"按应用 × 按用途"**：不要所有服务共用一个 app_user——一旦某应用被入侵或写出烂 SQL，既封不掉（会影响其他应用），也无法在审计日志里区分来源。至少拆成：在线写应用账号、报表/BI 只读账号、运维排障账号（仅管理 IP 可登）、以及 9066 管理账号（只给发布平台/堡垒机）。Mycat 1.6.x 对应配置在 server.xml 的 `<user>` 节点（含 `schemas`、`readOnly` 属性），语义一致。

> **注意**：Mycat 的权限粒度只到 **schema 级 + 读写级**，没有表级/列级授权，也没有角色体系。需要"这个服务只能读 orders 表""手机号列对 BI 不可见"这类细粒度控制时，必须在**上层解决**：为不同用途建不同逻辑 schema（表集合不同的 schema），或列级脱敏交给应用层/独立数据网关。把"MySQL 的 user 权限模型"套到 Mycat 上是常见的安全设计误判。

# 2. IP 白名单

```json
{
  "users": {
    "app_user": {
      "password": "app123",
      "schemas": ["order_db"],
      "ipWhiteList": "192.168.1.0/24,10.0.0.0/8"
    }
  }
}
```

支持格式：
- 单个 IP：`192.168.1.100`
- CIDR 网段：`192.168.1.0/24`
- 多个条目：逗号分隔
- 通配：不设置或 `*` 表示所有 IP

白名单校验发生在**连接建立时**，与密码校验同属认证链。两个容易失守的形态：其一是容器/K8s 环境里 Pod IP 是动态的，写 `10.0.0.0/8` 图省事等于没写——正确做法是配合网络策略把 Mycat 的 8066 限制为仅集群 NodePort/内网 SLB 可达，白名单填那些固定入口地址；其二是**经过代理的地址伪装**——若客户端经另一层代理（如自研 DB 网关）连入，Mycat 看到的源 IP 是代理机而非真实调用方，白名单此时只能约束"哪台代理能进来"，真实来源审计要靠上层日志。

# 3. 密码加密

```bash
# Mycat2 提供密码加密工具
cd mycat/lib
java -cp mycat2-1.22-release-jar-with-dependencies.jar \
     io.mycat.util.SecurityUtil \
     my_password

# 输出：
# 加密后：AES:xxxxxxxxxxxxxxxxxxxx
```

```json
{
  "users": {
    "root": {
      "password": "AES:xxxxxxxxxxxxxxxxxxxx",
      "encryptType": "AES"
    }
  }
}
```

**先想清楚这层加密防的是谁**：users.json 里的密文防的是"配置文件被读走/截图/进错代码仓库"这类**静态泄露**，不防运行时入侵——进程内存里总要还原明文语义完成认证，且 Mycat 解密用的私钥（1.6.x 是 `encrypt.key`）如果同机可见，加密强度就退化为混淆。同理，**后端 MySQL 账号密码写在 datasources 配置里**，配置中心与机器权限的安全等级就是整个数据层的底线。

> **行业实践**：密码管理的优先级排序应该是：KMS/密管系统（配置里只放引用）> 配置中心加密项 + 严格 ACL > 本地 AES 密文 > 明文。无论哪一档，**默认密码（123456）与示例账号必须在上线前清理**，这是安全检查清单的第一项不是没有原因的。Mycat 与 MySQL 之间的认证仍走 MySQL 原生协议握手（mysql_native_password/caching_sha2），users.json 的加密只作用于"应用→Mycat"这一段的存储形态，不改变线上握手机制。

# 4. 权限控制

## 4.1 Schema 级别

```json
{
  "users": {
    "order_admin": {
      "password": "admin123",
      "schemas": ["order_db"]       // 只能访问 order_db
    }
  }
}
```

## 4.2 读写权限

```json
{
  "users": {
    "report_user": {
      "password": "report123",
      "schemas": ["order_db"],
      "readOnly": true              // 只读
    }
  }
}
```

`readOnly` 是**语句类型级**判定：INSERT/UPDATE/DELETE/DDL 一律拒绝，与 SQL 是否路由到 Slave 无关。要利用读写分离做"报表压力与在线隔离"，正确姿势是**给报表单独账号 + 单独 schema 视图 + readOnly**，而不是在业务代码里"记得"报表只写 SELECT。

## 4.3 连接限制

```json
{
  "users": {
    "app_user": {
      "password": "app123",
      "schemas": ["order_db"],
      "maxCon": 100,                // 最大连接数
      "maxConnectionsPerIP": 10     // 单 IP 最大连接数
    }
  }
}
```

用户级 maxCon 是**防单点滥用烧穿全局资源**的闸门：MyCAT 前端连接会放大占用后端连接（一条前端跨分片查询可能借出 N 个后端连接），没有用户配额时，一个失控应用（循环建新连接不归还）就能耗尽整台 Mycat 与前端的文件句柄。`maxConnectionsPerIP` 对容器场景基本失效（IP 即 Node IP），主要给固定网段的办公/运维入口用。

# 5. SQL 防火墙

Mycat 内置两道语句级防线：**黑名单/白名单拦截（sqlReject）**与**语句模板统计**，1.6.x 时代还有著名的 sqlfilter 插件体系（正则改写/拒绝）。

```xml
<!-- Mycat 1.6.x server.xml：SQL 黑名单 -->
<system>
    <!-- 命中即拒绝，支持多条目分号分隔 -->
    <property name="sqlReject">
        drop\s+table|truncate\s+table|sleep\(
    </property>
    <!-- 语句表统计白名单：未登记的表不纳入统计，防全表扫描类误报漏报 -->
    <property name="sqlAllowed">select|insert|update|delete</property>
</system>
```

```yaml
# Mycat2 风格的防火墙开关（示意）
server:
  firewall:
    enabled: true
    rejectPatterns:            # 命中即拒绝并记录审计
      - "drop\\s+table"
      - "truncate\\s+table"
      - ";\\s*delete\\s+from"  # 拦截多语句拼接
    slowSqlThresholdMs: 1000   # 超过即进慢 SQL 统计与告警
```

**防火墙的正确定位是"最后一道兜底网，不是主防线"**：

> **注意**：正则黑名单存在天花板——绕过写法（注释变形、等价语句、编码技巧）难以穷尽，误杀合法 SQL 又会造成生产事故。它的价值在于拦截**高频恶意模式与手滑灾难**（`DELETE` 不带 WHERE、直连跑 `DROP`），把这类操作从"能执行"降级为"要审批走别的通道"。真正的权限收敛仍然依赖：应用账号不给 DROP/DDL 权限（后端 MySQL 层面同样只授权 DML）、高危变更走 DBA 工单直连备库路径、`sqlMaxLimit` 强制 LIMIT 兜底大结果集。

慢 SQL 防线（`slowSqlThresholdMs`，对应管理命令 `showSlowSQL` 与统计日志）与防火墙是两套机制：慢 SQL 不拒绝、只观测，输出语句模板、次数、平均耗时与**路由分片数**——后者是 Mycat 特有的观测维度，一条 SQL 慢是因为索引还是因为扫了 64 个分片，在这里一眼分开（见第 17 篇）。

# 6. SSL/TLS 加密

```yaml
# mycat.yml
server:
  ssl:
    enabled: true
    clientAuth: false
    keyStorePath: conf/ssl/mycat2.keystore
    keyStorePassword: mycat123
    trustStorePath: conf/ssl/mycat2.truststore
    trustStorePassword: mycat123
```

```bash
# 生成自签名证书
keytool -genkey -alias mycat2 \
  -keyalg RSA -keysize 2048 \
  -keystore mycat2.keystore \
  -storepass mycat123 \
  -dname "CN=mycat2.example.com"
```

**传输加密要分三段看，任何一段裸奔都算没加密**：

```text
应用 ──(1) JDBC→Mycat 8066──> Mycat ──(2)→ 后端 MySQL 3306 ──(3 备份/管理流量)
```

- 段 (1)：上面的 server.ssl 配置，JDBC URL 需配 `useSSL=true&requireSSL=true`，仅 useSSL 不 requireSSL 时中间人可降级明文。
- 段 (2)：datasources 的 ssl 配置 + MySQL 侧 `require_secure_transport=ON`，跨机房部署必开。
- 全内网 + 网络隔离到位时不启 TLS 也是可辩护的决策（性能与证书运维成本），但"合规要求加密"场景下必须把两段都覆盖。

> **设计争议**：MySQL 协议 TLS 在中间件生态的落地一直不理想——老驱动对 caching_sha2_password + TLS 组合的支持参差不齐，连接池重连风暴在 TLS 握手放大 CPU 开销时更难收敛。因此不少团队的选择是**协议层不加密、网络层加密**：8066/3306 只暴露在内网/VPC + 安全组白名单，跨网传输走 mTLS 网关或专线。两种路线都成立，成立的前提都是"我明确知道自己保护的是哪一段"，最怕的是"配了 keystore 就以为全局安全"。

# 7. 审计日志

```yaml
server:
  audit:
    enabled: true
    logFile: /var/log/mycat2/audit.log
    logFormat: "json"
    includeSQL: true          # 是否记录 SQL
    includeParams: false      # 是否记录参数值（敏感）
    maxSQLLength: 1024
```

```json
// audit.log 示例
{
  "timestamp": "2024-01-15T10:30:00.123",
  "user": "app_user",
  "ip": "192.168.1.100",
  "schema": "order_db",
  "sqlType": "SELECT",
  "sql": "SELECT * FROM orders WHERE id = ?",
  "shards": ["dn0"],
  "rt": 25,
  "rows": 1
}
```

审计与配置变更日志（第 15 篇 audit）分开存放：**数据访问审计**回答"谁在什么时候读了什么数据"，是等保/合规检查的硬材料，日志本身包含业务信息，采集链路要做脱敏与访问控制；`includeParams=false` 的默认取舍是"记模板不记值"，需要排查具体参数错误时靠 traceId 关联应用日志，而不是把参数值长期落盘。

# 8. 后端连接安全

```json
{
  "datasources": {
    "ds1": {
      "user": "mycat",
      "password": "mycat123",
      "security": {
        "ssl": true,
        "sslMode": "PREFERRED"
      }
    }
  }
}
```

后端 MySQL 安全建议：
1. Mycat2 使用专用数据库账号（最小权限）
2. Mycat2 账号仅允许从 Mycat2 所在 IP 连接
3. 后端 MySQL 开启 `require_secure_transport`
4. 定期轮换数据库密码

最小权限的边界常有人问："Mycat 要不要给 DROP？"——不要。**Mycat 后端账号授到 `SELECT,INSERT,UPDATE,DELETE + REPLICATION SLAVE/CLIENT`（心跳与主从判定需要）即可**，DDL 由 DBA 直连执行（变更后再 reload 元数据，见第 15 篇边界表）。这一刀同时消掉了"应用注入出 DDL"和"中间件账号被用作 DDL 通道"两条路径。注意 MySQL 8.0 下 Mycat 心跳用到的一些 `SHOW` 命令需要额外权限（SHOW SLAVE STATUS 需 REPLICATION CLIENT），按报错补最小项而不是图省事给 SUPER/ALL。

# 9. 与上层网关的分工

**Mycat 的安全能力止步于"连接与语句"，身份与业务授权必须在它之上**。典型分层：

```text
┌──────────────────────────────────────────────┐
│ 应用/BI/运维      ← 人的身份：SSO/LDAP、工单审批  │
│ 服务网关/DB 网关  ← 服务身份：mTLS/SPIFFE、令牌    │
│ Mycat             ← 账号+白名单+schema 权限+防火墙│
│ MySQL             ← 最后防线：账号最小授权         │
└──────────────────────────────────────────────┘
```

分工要点：**认证**（你是谁）在网关/平台层解决，Mycat 的 users 是"服务账号"而非"人的账号"，共享账号的追责靠上层审计映射（谁借用了 app_user）；**授权**（你能碰哪些数据）两处分：粗粒度（schema、读写）在 Mycat，细粒度（行级、列级、脱敏）在网关或应用；**审计**两边都要有且能用时间轴对齐。"要不要在 Mycat 前面再放一层 DB 网关"的判据是合规细粒度需求：普通互联网业务 Mycat + 账号划分够用；涉及个人信息/金融数据强合规，列级脱敏、动态授权、统一凭证轮换就需要专门网关，此时 Mycat 退居纯路由层。

# 10. 安全检查清单

```text
□ users.json 中密码使用加密存储
□ 生产环境密码不要使用默认值
□ root 用户仅限管理 IP 访问
□ 应用用户设置 readOnly、ipWhiteList
□ 启用审计日志
□ 管理端口 (9066) 不对外暴露
□ 限制单用户最大连接数
□ 监控异常连接尝试
□ 定期检查未授权 Schema 访问
□ SSL/TLS 加密敏感数据传输
□ 后端账号最小授权（无 DDL/DROP）
□ SQL 黑名单覆盖 drop/truncate/多语句拼接
□ 9066 变更操作纳入工单流程，禁止直连敲命令
□ 配置文件与配置中心纳入密管/KMS 访问控制
```

# 11. 小结

| 层面 | 机制 | 边界 |
|------|------|------|
| 认证 | users 账号 + AES 存储 + 握手校验 | 粒度到 schema 为止，人无账号概念 |
| 访问控制 | ipWhiteList、readOnly、maxCon | 容器动态 IP 下白名单退化，需网络层补位 |
| 语句防线 | sqlReject 黑名单、慢 SQL 观测、sqlMaxLimit | 正则可绕过，只做兜底不做主防 |
| 传输 | 三段 TLS（前端/后端/旁路） | 只配一段等于没配；内网隔离可作替代辩护 |
| 纵深 | 网关管身份细授权，Mycat 管账号粗授权，MySQL 管底线 | 任何单层都不满足合规，日志要能对齐 |

一句话：**Mycat 把"数据库直连时代的人人都能碰库"收敛成"少数账号、可审计、可限流"的入口，但它是入口管控，不是数据权限系统**——细粒度的事情，留给它上面那层。下一篇（17 性能调优）转向另一个主题：这台守门机器本身怎么调。
