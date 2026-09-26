
# 1. Canal Admin 概述

Canal Admin 是 Canal 的 Web 管理平台，提供了对 Canal Server 和 Instance 的集中化管理能力，包括实例的创建、配置修改、运行状态监控、日志查看、HA 管理等。

```
┌───────────────────────────────────────────────────────────────┐
│                     Canal Admin (Web UI)                       │
│                        Port: 8089                             │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │
│  │   Server    │  │  Instance   │  │   Cluster/HA        │   │
│  │   Management│  │  Management │  │   Management        │   │
│  └─────────────┘  └─────────────┘  └─────────────────────┘   │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                     MySQL (元数据库)                      │ │
│  │                   canal_manager                          │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                  ZooKeeper (可选, HA 模式)                │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌────────────┐    ┌────────────┐    ┌────────────┐
       │ Canal      │    │ Canal      │    │ Canal      │
       │ Server 1   │    │ Server 2   │    │ Server 3   │
       └────────────┘    └────────────┘    └────────────┘
```

# 2. 安装与配置

## 2.1 数据库初始化

```sql
-- 创建管理数据库
CREATE DATABASE IF NOT EXISTS canal_manager DEFAULT CHARSET utf8mb4;

-- 导入初始化 SQL（在 canal.admin 发布包的 conf 目录下）
source /opt/canal-admin/conf/canal_manager.sql;

-- 初始化后的表结构
-- canal_manager 数据库包含以下表：
-- canal_user       : 用户表
-- canal_config     : 系统配置表
-- canal_instance_config : 实例配置表
-- canal_node_server    : Canal Server 节点表
-- canal_cluster        : 集群信息表
```

## 2.2 Admin 配置

```yaml
# conf/application.yml
server:
  port: 8089
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8

  datasource:
    address: 127.0.0.1:3306
    database: canal_manager
    username: root
    password: root_password
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://${spring.datasource.address}/${spring.datasource.database}?useUnicode=true&characterEncoding=UTF-8&useSSL=false
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2

canal:
  adminUser: admin           # 管理员用户名
  adminPasswd: admin123      # 管理员密码
```

## 2.3 登录界面

启动后访问 `http://localhost:8089`，默认账号：`admin / 123456`

# 3. 核心功能

## 3.1 Server 管理

管理所有注册到 Admin 的 Canal Server 节点：

```bash
# 查看所有 Server
curl http://localhost:8089/api/v1/canal/nodeServers

# 返回
{
  "code": 20000,
  "data": [
    {
      "id": 1,
      "name": "canal-server-01",
      "ip": "192.168.1.10",
      "port": 11111,
      "status": "RUNNING",    # RUNNING / STOPPED / UNKNOWN
      "clusterId": 1,
      "modifiedTime": "2024-01-01 10:00:00"
    }
  ]
}
```

## 3.2 Instance 管理

创建、修改、启动/停止 Instance：

```bash
# 创建新的 Instance
curl -X POST http://localhost:8089/api/v1/canal/instance \
  -H "Content-Type: application/json" \
  -d '{
    "clusterServerId": 1,
    "name": "example2",
    "content": "canal.instance.master.address=192.168.1.100:3306\ncanal.instance.dbUsername=canal\ncanal.instance.dbPassword=canal\ncanal.instance.filter.regex=.*\\\\..*"
  }'

# 启动 Instance
curl -X PUT http://localhost:8089/api/v1/canal/instance/start/192.168.1.10/example2

# 停止 Instance
curl -X PUT http://localhost:8089/api/v1/canal/instance/stop/192.168.1.10/example2

# 删除 Instance
curl -X DELETE http://localhost:8089/api/v1/canal/instance/example2
```

## 3.3 配置模板管理

Admin 支持配置模板功能，快速创建相似配置的 Instance：

```properties
# 模板示例（canal-template.properties）
#################################################
#########    mysql serverId     #################
canal.instance.mysql.slaveId = ${slaveId}

#################################################
#########    mysql instance     #################
canal.instance.master.address = ${masterAddress}
canal.instance.master.journal.name =
canal.instance.master.position =

#################################################
#########    username/password   #################
canal.instance.dbUsername = ${dbUsername}
canal.instance.dbPassword = ${dbPassword}
canal.instance.connectionCharset = UTF-8

#################################################
#########    table filter regex  #################
canal.instance.filter.regex = ${filterRegex}
```

```bash
# 通过 API 使用模板创建 Instance
curl -X POST http://localhost:8089/api/v1/canal/instance/template \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my_instance",
    "clusterServerId": 1,
    "templateId": 1,
    "params": {
      "slaveId": "1234",
      "masterAddress": "192.168.1.100:3306",
      "dbUsername": "canal",
      "dbPassword": "canal_pwd",
      "filterRegex": "test\\\\..*"
    }
  }'
```

## 3.4 位点管理

通过 Admin 界面或 API 管理消费位点：

```bash
# 查看位点
curl http://localhost:8089/api/v1/canal/instance/log/192.168.1.10/example

# 返回信息包含：
# - journalName: 当前 binlog 文件名
# - position: 当前偏移量
# - timestamp: 位点时间戳
# - gtid: GTID 信息（如果启用）
```

## 3.5 日志查看

Admin 提供了远程查看 Canal Server 日志的能力：

```
Web UI: Instance 详情页 → "查看日志"
API:    GET /api/v1/canal/instance/log/{serverIp}/{instance}?size=100
```

# 4. Canal Server 注册到 Admin

## 4.1 Server 端配置

```properties
# canal.properties — 让 Server 注册到 Admin

# 1. 指定 Admin 地址
canal.admin.manager = 127.0.0.1:8089

# 2. Admin 端口（用于 Admin 向 Server 发起管理请求）
canal.admin.port = 11110

# 3. Admin 用户名密码
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441

# 4. 是否自动注册到 Admin
canal.admin.register.auto = true

# 5. Server 信息
canal.register.ip = 192.168.1.10
canal.register.cluster = default_cluster
```

## 4.2 注册流程

```java
public class CanalAdminConnector {
    
    /**
     * Canal Server 启动时自动注册到 Admin
     */
    public void registerToAdmin() {
        
        // 1. 构建注册请求
        AdminRequest request = new AdminRequest();
        request.setName(canalServerName);
        request.setIp(canalRegisterIp);
        request.setPort(canalPort);
        request.setAdminPort(adminPort);
        request.setClusterName(clusterName);
        
        // 2. 发送注册请求
        String adminUrl = "http://" + adminManagerAddress + "/api/v1/canal/nodeServers";
        
        try {
            String response = HttpClient.post(adminUrl, request);
            
            if (isSuccess(response)) {
                logger.info("registered to admin successfully");
                
                // 3. 定期心跳
                startHeartbeatToAdmin();
                
                // 4. 监听 Admin 下发的配置变更
                startConfigWatcher();
            }
        } catch (Exception e) {
            logger.warn("failed to register to admin, will retry", e);
            scheduleRetry(30);  // 30秒后重试
        }
    }
    
    /**
     * 定期向 Admin 发送心跳
     */
    private void startHeartbeatToAdmin() {
        scheduler.scheduleAtFixedRate(() -> {
            try {
                String url = adminManagerAddress + "/api/v1/canal/nodeServers/heartbeat";
                HeartbeatRequest heartbeat = new HeartbeatRequest();
                heartbeat.setIp(canalRegisterIp);
                heartbeat.setPort(canalPort);
                HttpClient.post(url, heartbeat);
            } catch (Exception e) {
                logger.warn("heartbeat to admin failed", e);
            }
        }, 0, 30, TimeUnit.SECONDS);
    }
}
```

# 5. 集群管理

## 5.1 集群概念

Admin 可以管理多个 Canal 集群，每个集群包含多个 Canal Server 节点：

```
Canal Admin
    │
    ├── Cluster: "production" (生产集群)
    │   ├── Server: 192.168.1.10:11111
    │   ├── Server: 192.168.1.11:11111
    │   └── Server: 192.168.1.12:11111
    │
    └── Cluster: "development" (开发集群)
        ├── Server: 192.168.2.10:11111
        └── Server: 192.168.2.11:11111
```

## 5.2 集群中的 Instance 分布

```bash
# 为集群中的某个 Server 启动 Instance
# Admin 会根据各 Server 的负载和 Instance 分布进行调度

# 查看集群中的 Instance 分布
curl http://localhost:8089/api/v1/canal/cluster/1/instances
```

# 6. 主备切换管理

## 6.1 ZK HA 模式的 Admin 视图

当 Canal 集群基于 ZooKeeper 做 HA 时，Admin 可以展示主备状态：

```
Instance: example
├── Running Server: 192.168.1.10:11111  (Master)
└── Standby Servers:
    ├── 192.168.1.11:11111  (Standby)
    └── 192.168.1.12:11111  (Standby)
```

## 6.2 手动主备切换

```bash
# 强制切换（将 Instance 从当前 Running Server 切换到指定 Server）
curl -X PUT http://localhost:8089/api/v1/canal/instance/move \
  -H "Content-Type: application/json" \
  -d '{
    "serverId": 2,
    "instanceId": "example"
  }'
```

# 7. 系统监控

## 7.1 监控指标

Admin 集成了 Canal Server 的监控数据展示：

| 指标 | 说明 |
|------|------|
| **Server 运行状态** | RUNNING / STOPPED |
| **Instance 运行状态** | START / STOP |
| **Binlog 订阅延迟** | 当前位点与 MySQL 最新位点的差距 |
| **每秒处理事件数** | TPS（Transactions Per Second） |
| **客户端连接数** | 当前连接的 Client 数量 |
| **内存使用量** | JVM 堆内存使用情况 |
| **网络流量** | 入站/出站流量 |

## 7.2 监控数据 API

```bash
# 获取 Server 运行状态
curl http://localhost:8089/api/v1/canal/nodeServers/1/runningStatus

# 获取 Instance 的消费延迟
curl http://localhost:8089/api/v1/canal/instance/log/192.168.1.10/example
```

# 8. 告警机制

## 8.1 内置告警

Canal Admin 支持通过邮件发送告警：

```properties
# canal.properties
canal.alarm.mode = email   # 告警方式: email / sms / dingtalk

# 邮件告警配置
canal.alarm.email.host = smtp.example.com
canal.alarm.email.port = 25
canal.alarm.email.from = canal@example.com
canal.alarm.email.to = ops@example.com
canal.alarm.email.username = canal
canal.alarm.email.password = password
```

## 8.2 告警触发条件

```java
public class AlarmConditionChecker {
    
    /**
     * 检查是否需要告警
     */
    public List<Alarm> check(CanalInstance instance) {
        List<Alarm> alarms = new ArrayList<>();
        
        // 1. 消费延迟告警
        long delay = instance.getDelaySeconds();
        if (delay > 60) {  // 超过 1 分钟
            alarms.add(new Alarm(AlarmType.DELAY_HIGH, 
                "消费延迟 " + delay + " 秒"));
        }
        
        // 2. 连接断开告警
        if (!instance.isConnected()) {
            alarms.add(new Alarm(AlarmType.CONNECTION_LOST, 
                "MySQL 连接断开"));
        }
        
        // 3. 内存使用告警
        long memoryUsage = instance.getMemoryUsagePercent();
        if (memoryUsage > 80) {
            alarms.add(new Alarm(AlarmType.MEMORY_HIGH,
                "内存使用率 " + memoryUsage + "%"));
        }
        
        // 4. DDL 变更告警（可选的，表结构变更通知）
        if (instance.isDdlDetected()) {
            alarms.add(new Alarm(AlarmType.DDL_DETECTED,
                "检测到 DDL 变更"));
        }
        
        return alarms;
    }
}
```

# 9. Admin API 参考

## 9.1 Server 相关 API

| Method | Path | 说明 |
|--------|------|------|
| GET | `/api/v1/canal/nodeServers` | 获取所有 Server 列表 |
| GET | `/api/v1/canal/nodeServers/{id}` | 获取 Server 详情 |
| GET | `/api/v1/canal/nodeServers/{id}/runningStatus` | 获取 Server 运行状态 |
| POST | `/api/v1/canal/nodeServers` | 注册新 Server |

## 9.2 Instance 相关 API

| Method | Path | 说明 |
|--------|------|------|
| GET | `/api/v1/canal/instances` | 获取所有 Instance 列表 |
| GET | `/api/v1/canal/instance/{instanceId}` | 获取 Instance 详情 |
| POST | `/api/v1/canal/instance` | 创建新 Instance |
| PUT | `/api/v1/canal/instance` | 修改 Instance 配置 |
| DELETE | `/api/v1/canal/instance/{instanceId}` | 删除 Instance |
| PUT | `/api/v1/canal/instance/start/{serverIp}/{instance}` | 启动 Instance |
| PUT | `/api/v1/canal/instance/stop/{serverIp}/{instance}` | 停止 Instance |
| GET | `/api/v1/canal/instance/log/{serverIp}/{instance}` | 查看 Instance 日志/位点 |

## 9.3 集群相关 API

| Method | Path | 说明 |
|--------|------|------|
| GET | `/api/v1/canal/clusters` | 获取集群列表 |
| GET | `/api/v1/canal/cluster/{id}` | 获取集群详情 |
| POST | `/api/v1/canal/cluster` | 创建集群 |
| PUT | `/api/v1/canal/instance/move` | 迁移 Instance 到其他 Server |

# 10. 权限管理

Canal Admin 支持多用户权限管理：

```sql
-- canal_user 表结构
CREATE TABLE canal_user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(32) NOT NULL,
    password VARCHAR(128) NOT NULL,
    name VARCHAR(32),
    roles VARCHAR(512),         -- 角色: admin/operator/user
    `desc` VARCHAR(256),
    creation_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 默认管理员账号
-- INSERT INTO canal_user (username, password, name, roles) 
-- VALUES ('admin', 'E10ADC3949BA59ABBE56E057F20F883E', 'Administrator', 'admin');
```

## 10.1 权限角色说明

| 角色 | 权限 |
|------|------|
| `admin` | 全部权限：Server 管理、Instance 管理、集群管理、用户管理 |
| `operator` | 运维权限：查看状态、启停 Instance、查看日志 |
| `user` | 只读权限：查看状态和日志 |

# 11. Admin 的部署建议

## 11.1 高可用部署

```
┌───────────────────┐    ┌───────────────────┐
│  Canal Admin 1    │    │  Canal Admin 2    │
│  (Active)         │    │  (Active)         │
└────────┬──────────┘    └────────┬──────────┘
         │                        │
         │     ┌──────────┐       │
         └─────│  MySQL   │───────┘
               │ (主库)    │
               │canal_mgr  │
               └──────────┘
```

- Admin 本身是无状态的，通过 MySQL 共享数据
- 可以部署多个 Admin 实例，前端通过 Nginx 做负载均衡
- MySQL 需要配置主从或集群保证高可用

## 11.2 安全建议

```nginx
# Nginx 配置 Admin 的访问控制
server {
    listen 80;
    server_name canal-admin.internal.com;
    
    # IP 白名单
    allow 192.168.0.0/16;
    deny all;
    
    location / {
        proxy_pass http://localhost:8089;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # 基础认证
        auth_basic "Canal Admin";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```
