
# 1. 高可用架构设计

Nacos 集群高可用通过以下几个层面保证：

```
              Nacos 高可用体系

    ┌──────────────────────────────────────┐
    │          客户端侧高可用              │
    │  ┌──────────┐  ┌──────────────────┐ │
    │  │ 本地缓存  │  │ 多服务端地址切换  │ │
    │  │ 快照容灾  │  │ ServerListManager│ │
    │  └──────────┘  └──────────────────┘ │
    └──────────────────────────────────────┘
                      │
    ┌──────────────────────────────────────┐
    │          服务端侧高可用              │
    │  ┌──────────┐  ┌──────────────────┐ │
    │  │ 集群部署  │  │ 数据一致性协议    │ │
    │  │ 无单点    │  │ Distro / Raft    │ │
    │  └──────────┘  └──────────────────┘ │
    └──────────────────────────────────────┘
                      │
    ┌──────────────────────────────────────┐
    │          存储层高可用                │
    │  ┌──────────┐  ┌──────────────────┐ │
    │  │ MySQL    │  │ 内置数据快照       │ │
    │  │ 主从架构  │  │ dump 文件自动恢复 │ │
    │  └──────────┘  └──────────────────┘ │
    └──────────────────────────────────────┘
```

# 2. 集群部署拓扑

## 2.1 标准三节点集群

```
        ┌──────────────┐
        │  Load Balancer│
        │  (Nginx/HAProxy/ │
        │   SLB/VIP)    │
        └──────┬───────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌──────┐  ┌──────┐  ┌──────┐
│Node-1│  │Node-2│  │Node-3│
│:8848 │  │:8848 │  │:8848 │  HTTP + Console
│:7848 │  │:7848 │  │:7848 │  gRPC Client
│:9848 │  │:9848 │  │:9848 │  gRPC Server
│:9849 │  │:9849 │  │:9849 │  Raft
└──┬───┘  └──┬───┘  └──┬───┘
   └─────────┼─────────┘
             ▼
     ┌──────────────┐
     │  MySQL       │
     │  (共享存储)   │
     └──────────────┘
```

## 2.2 多机房部署

```
            跨机房集群部署

   ┌───────── IDC-1 ─────────┐   ┌───────── IDC-2 ─────────┐
   │                          │   │                          │
   │  ┌──────┐  ┌──────┐     │   │  ┌──────┐               │
   │  │Node-1│  │Node-2│     │   │  │Node-3│               │
   │  └──────┘  └──────┘     │   │  └──────┘               │
   │                          │   │                          │
   │  ┌──────────────┐       │   │  ┌──────────────┐       │
   │  │ MySQL Master │<──────┼───┼──│ MySQL Slave  │       │
   │  └──────────────┘       │   │  └──────────────┘       │
   └──────────────────────────┘   └──────────────────────────┘

   建议：MySQL 跨机房主从复制延迟需 < 50ms
   注意：跨机房网络延迟对 Raft 选举影响较大
```

# 3. 节点故障处理

## 3.1 单节点故障

| 故障场景 | 影响 | 恢复时间 |
|---------|------|---------|
| 单个 Follower 宕机 | 无影响，集群正常服务 | 节点重启后自动恢复 |
| Leader 宕机 | 10s 内完成选举切换，期间 CP 写入短暂不可用 | 自动选举新 Leader |
| 单节点宕机超过保护时间 | 集群降级运行（2 节点） | 节点恢复后自动同步 |

## 3.2 Leader 选举流程（Raft）

```
正常状态:
  ┌────────┐  ┌──────────┐  ┌──────────┐
  │Leader  │  │Follower-1│  │Follower-2│
  │(Node-1)│  │(Node-2)  │  │(Node-3)  │
  └────────┘  └──────────┘  └──────────┘

Leader 宕机:
  ┌────────┐  ┌──────────┐  ┌──────────┐
  │DOWN    │  │Follower-1│  │Follower-2│
  │(Node-1)│  │(Node-2)  │  │(Node-3)  │
  └────────┘  └──────────┘  └──────────┘
                   │              │
                   │ 心跳超时      │ 心跳超时
                   ▼              ▼
              ┌──────────┐  ┌──────────┐
              │Candidate │  │Candidate │
              │发起选举    │  │发起选举    │
              └──────────┘  └──────────┘
                   │              │
                   ├── 投票请求 ──→│
                   │←── 投票回复 ──┤
                   │              │
                   ▼              │
              ┌──────────┐        │
              │当选 Leader│        │
              │(Node-2)  │  降级   │
              └──────────┘        ▼
                            ┌──────────┐
                            │Follower  │
                            │(Node-3)  │
                            └──────────┘

   容错数: 3 节点容忍 1 个故障
   选举超时: 150ms ~ 300ms (随机)
   Leader 心跳: 100ms
```

# 4. 客户端容灾策略

## 4.1 多服务端地址切换

```java
// ServerListManager.java
public class ServerListManager {
    
    private List<String> serverList;
    private volatile String currentServer;
    
    public String getCurrentServer() throws NacosException {
        if (currentServer != null) {
            return currentServer;
        }
        
        // 轮询尝试各个服务端
        for (int i = 0; i < serverList.size(); i++) {
            currentServer = serverList.get(i);
            if (healthCheck(currentServer)) {
                return currentServer;
            }
        }
        throw new NacosException("All Nacos servers are unavailable");
    }
    
    public void switchServer() {
        int idx = serverList.indexOf(currentServer);
        // 切换到下一个服务端
        currentServer = serverList.get((idx + 1) % serverList.size());
    }
}
```

## 4.2 本地缓存容灾

服务端全部不可用时，客户端仍可使用本地缓存继续服务：

```java
// 配置中心容灾
String config = configService.getConfig(dataId, group, 5000);
// Nacos Server 不可用时，返回本地快照内容

// 注册中心容灾
ServiceInfo serviceInfo = hostReactor.getServiceInfo(serviceName, clusters);
// Nacos Server 不可用时，返回本地磁盘缓存的实例列表
```

## 4.3 Spring Cloud 配置容灾

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 10.0.0.1:8848,10.0.0.2:8848,10.0.0.3:8848
        timeout: 3000
        # 启动时不强制要求 Nacos 可用
        fail-fast: false
```

# 5. 数据同步机制

## 5.1 新节点加入集群

```
新节点 (Node-4) 加入 3 节点集群:

1. 修改 Node-4 的 cluster.conf，添加所有 4 个节点

2. 启动 Node-4
   ├── 加入集群网络（向其他节点注册自己）
   ├── Distro 数据自动分配
   │     └── hash 重新分配 → 从旧 Owner 同步数据
   ├── Raft 节点加入
   │     └── 作为 Learner 从 Leader 同步日志和快照
   └── 就绪后可接收请求

3. 更新负载均衡器配置（添加 Node-4）
```

## 5.2 节点间数据同步时机

| 同步场景 | Distro (AP) | Raft (CP) |
|---------|-------------|-----------|
| 正常写入 | 异步复制到所有节点 | 同步复制到多数派 |
| 新节点加入 | 全量校验 + 增量同步 | 快照 + 日志追平 |
| 节点重启 | 反熵对账修复差异 | 日志回放 + 快照恢复 |
| 定时对账 | 每 5s 全部数据 checksum 对比 | Raft 心跳自带一致性保证 |

# 6. 负载均衡策略

## 6.1 Nginx 负载均衡

```nginx
upstream nacos-cluster {
    # 轮询策略（默认）
    server 10.0.0.1:8848 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8848 weight=5 max_fails=3 fail_timeout=30s;
    server 10.0.0.3:8848 weight=5 max_fails=3 fail_timeout=30s;
    
    # 长连接优化
    keepalive 32;
    keepalive_requests 1000;
    keepalive_timeout 60s;
}

# HTTP 服务负载均衡
server {
    listen 80;
    server_name nacos.internal.com;
    
    location /nacos/ {
        proxy_pass http://nacos-cluster;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 3s;
        proxy_read_timeout 30s;
    }
}

# gRPC 四层负载均衡
stream {
    upstream nacos-grpc {
        server 10.0.0.1:9848;
        server 10.0.0.2:9848;
        server 10.0.0.3:9848;
    }
    
    server {
        listen 9848;
        proxy_pass nacos-grpc;
        proxy_connect_timeout 3s;
    }
}
```

## 6.2 虚拟 IP（VIP）+ Keepalived

```bash
# /etc/keepalived/keepalived.conf
vrrp_instance VI_NACOS {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass nacos123
    }
    virtual_ipaddress {
        10.0.0.100/24   # VIP
    }
    track_script {
        chk_nacos
    }
}

vrrp_script chk_nacos {
    script "/usr/bin/curl -s -o /dev/null -w '%{http_code}' http://localhost:8848/nacos/v1/console/health/readiness"
    interval 3
    weight -2
}
```

# 7. 监控与告警

## 7.1 关键健康指标

| 指标 | 告警阈值 | 严重程度 |
|------|---------|---------|
| 集群节点数 | < 3 | 严重 |
| 节点健康状态 | DOWN | 严重 |
| 数据库连接状态 | 失败 | 严重 |
| Leader 切换次数 | > 3次/小时 | 警告 |
| 内存使用率 | > 85% | 警告 |
| GC 频率 | Full GC > 5次/小时 | 警告 |
| 请求延迟 P99 | > 500ms | 警告 |

## 7.2 健康检查 API

```bash
# 节点就绪检查
curl http://localhost:8848/nacos/v1/console/health/readiness

# 集群状态
curl http://localhost:8848/nacos/v1/core/cluster/nodes

# Leader 信息
curl http://localhost:8848/nacos/v1/core/cluster/self
```

# 8. 容量规划

## 8.1 推荐配置

| 集群规模 | 服务实例数 | 推荐节点数 | 服务器规格 | MySQL |
|---------|----------|-----------|-----------|-------|
| 小型 | < 1000 | 3 | 4C8G | 4C8G |
| 中型 | 1000-10000 | 3 | 8C16G | 8C16G |
| 大型 | 10000-100000 | 5 | 16C32G | 16C64G |
| 超大型 | > 100000 | 7 | 32C64G | 32C128G + 读写分离 |

## 8.2 集群扩展

```bash
# 1. 节点集群 → 7 节点集群

# 1. 启动新节点（修改 cluster.conf 添加 7 个节点）
# 2. 新节点自动从集群同步数据
# 3. 更新负载均衡器配置
# 4. 验证集群状态
curl http://new-node:8848/nacos/v1/core/cluster/nodes

# Distro 数据会自动重新分片（一致性 Hash）
# Raft 配置变更需要多数派确认
```
