
# 1. Mycat2 自身 HA

Mycat2 本身是无状态的（配置存储在文件/ZK/Nacos），可以通过多实例部署实现自身高可用。

```
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

# 2. 配置中心模式

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

# 3. 实例故障转移

```
Client JDBC Connection URL:

jdbc:mysql://mycat2-vip:8066/db?
  loadBalanceHosts=true
  &retriesAllDown=3
  &connectTimeout=3000
```

```
Client 连接策略：
  1. 连接到 VIP/LVS → 路由到可用 Mycat2 实例
  2. Mycat2-1 宕机 → LVS 健康检查失败 → 摘除
  3. 新连接路由到 Mycat2-2 或 Mycat2-3
  4. 已有连接需要在客户端重试
```

# 4. 后端 MySQL HA

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

```
1. Master 心跳失败（连续 3 次）
2. Mycat2 从 Slave 中选择数据最新的
3. 将该 Slave 提升为 Master（change master to ...）
4. 更新路由表
5. 告警通知
```

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

Galera 集群中所有节点均可读写，Mycat2 按权重轮询。

# 5. K8s 部署

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

# 6. 平滑升级

```
零停机升级步骤：
  1. 部署新版本 Mycat2 实例（新端口）
  2. 验证新实例功能正常
  3. 从 LVS/SLB 摘除一个旧实例
  4. 已有连接逐步排空（等待自然断开或超时）
  5. 重复 3-4 直到所有旧实例替换
  6. 关闭旧实例
```

# 7. 数据备份策略

```
分层备份：
  ├── Mycat2 配置 → Git / Nacos 历史版本
  ├── MySQL 全量备份 → mysqldump + 定时任务
  ├── MySQL 增量备份 → binlog 备份
  └── 跨机房容灾 → MySQL 异步复制到异地
```
