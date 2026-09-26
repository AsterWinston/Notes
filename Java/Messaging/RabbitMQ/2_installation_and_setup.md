
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，以 Docker 为主要安装方式，覆盖 Windows/macOS/Linux 环境。

---

# 1. Docker 安装（推荐）

```powershell
# 拉取镜像（含 Management 插件）
docker pull rabbitmq:3.13-management

# 启动容器
docker run -d `
  --name rabbitmq `
  -p 5672:5672 `
  -p 15672:15672 `
  -e RABBITMQ_DEFAULT_USER=admin `
  -e RABBITMQ_DEFAULT_PASS=admin123 `
  rabbitmq:3.13-management

# 查看状态
docker ps

# 查看日志
docker logs rabbitmq
```

| 端口 | 用途 |
|------|------|
| 5672 | AMQP 协议端口 |
| 15672 | Management UI（HTTP API + 管理界面） |
| 25672 | 集群通信端口（Erlang 分布式） |
| 1883 | MQTT 协议（需启用插件） |
| 61613 | STOMP 协议（需启用插件） |
| 15692 | Prometheus 指标暴露端口 |

## 1.1 完整 docker-compose.yml

```yaml
version: "3.8"
services:
  rabbitmq:
    image: rabbitmq:3.13-management
    container_name: rabbitmq
    hostname: rabbit-node-1
    ports:
      - "5672:5672"
      - "15672:15672"
      - "15692:15692"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
      - rabbitmq_log:/var/log/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  rabbitmq_data:
  rabbitmq_log:
```

---

# 2. Linux 包管理器安装

## 2.1 Ubuntu/Debian

```bash
# 添加 RabbitMQ 官方仓库
sudo apt update
sudo apt install -y curl gnupg apt-transport-https

# 导入签名密钥
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor > /usr/share/keyrings/com.rabbitmq.team.gpg

# 添加 apt 源
sudo tee /etc/apt/sources.list.d/rabbitmq.list <<EOF
deb [signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://ppa1.rabbitmq.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main
deb [signed-by=/usr/share/keyrings/com.rabbitmq.team.gpg] https://ppa1.rabbitmq.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main
EOF

sudo apt update
sudo apt install -y rabbitmq-server

# 启用管理插件
sudo rabbitmq-plugins enable rabbitmq_management

# 启动服务
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```

## 2.2 CentOS/RHEL

```bash
# 添加 RabbitMQ 仓库
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash

# 安装
sudo yum install -y rabbitmq-server

# 启用管理插件
sudo rabbitmq-plugins enable rabbitmq_management

# 启动服务
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```

---

# 3. Management UI 管理界面

访问 `http://localhost:15672`，使用创建的用户名密码登录。

## 3.1 主界面功能模块

| 模块 | 功能 |
|------|------|
| **Overview** | 集群概览：节点状态、消息速率、队列状态、连接数、内存/磁盘使用 |
| **Connections** | 连接管理：查看/关闭客户端连接，查看连接详情（IP、端口、用户、VHost） |
| **Channels** | 信道管理：查看/关闭信道，信道是轻量级逻辑连接 |
| **Exchanges** | 交换机管理：增删改查交换机，查看绑定关系、消息路由统计 |
| **Queues** | 队列管理：增删改查队列，查看消息堆积、消费者、绑定关系，支持 Purge/Delete |
| **Admin** | 用户/权限/策略管理：创建用户、设置 VHost 权限、定义 Policy |

## 3.2 常用操作

- **创建 Queue**：Queues → Add a new queue → 填写 Name/Durability/Auto delete/Arguments
- **创建 Exchange**：Exchanges → Add a new exchange → 选择 Type（direct/fanout/topic/headers）
- **创建 Binding**：Exchange 详情页 → Bindings → 填写 To queue 和 Routing key
- **发送测试消息**：Exchange 详情页 → Publish message → 填写 Routing key 和 Payload
- **查看队列消息**：Queue 详情页 → Get messages → 选择 Ack mode → Get Message(s)

---

# 4. CLI 命令行工具

## 4.1 rabbitmqctl（管理控制工具）

```bash
# 查看集群状态
rabbitmqctl cluster_status

# 查看节点状态
rabbitmqctl status

# 列出所有 VHost
rabbitmqctl list_vhosts

# 添加 VHost
rabbitmqctl add_vhost /myapp

# 删除 VHost
rabbitmqctl delete_vhost /myapp

# 列出发送用户
rabbitmqctl list_users

# 添加用户
rabbitmqctl add_user myuser mypassword

# 设置用户标签（management 表示可登录管理界面）
rabbitmqctl set_user_tags myuser management

# 设置用户权限（^ 开头的为正则匹配资源）
rabbitmqctl set_permissions -p /myapp myuser ".*" ".*" ".*"

# 列出权限
rabbitmqctl list_permissions -p /myapp

# 列出所有队列
rabbitmqctl list_queues name messages_ready messages_unacknowledged

# 删除队列
rabbitmqctl delete_queue my_queue

# 列出 Exchanges
rabbitmqctl list_exchanges

# 列出 Bindings
rabbitmqctl list_bindings

# 设置策略
rabbitmqctl set_policy ha-all "^ha\." '{"ha-mode":"all"}' --priority 1 --apply-to queues

# 列出策略
rabbitmqctl list_policies

# 清空队列
rabbitmqctl purge_queue my_queue

# 停止应用（保留 Erlang 运行时）
rabbitmqctl stop_app

# 启动应用
rabbitmqctl start_app

# 重置节点（清空所有数据）
rabbitmqctl reset
```

## 4.2 rabbitmq-plugins（插件管理）

```bash
# 列出所有插件
rabbitmq-plugins list

# 启用管理插件
rabbitmq-plugins enable rabbitmq_management

# 启用延迟消息插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 启用 Prometheus 监控
rabbitmq-plugins enable rabbitmq_prometheus

# 启用 MQTT 协议
rabbitmq-plugins enable rabbitmq_mqtt

# 启用 Federation
rabbitmq-plugins enable rabbitmq_federation
rabbitmq-plugins enable rabbitmq_federation_management

# 禁用插件
rabbitmq-plugins disable rabbitmq_mqtt
```

## 4.3 rabbitmq-diagnostics（诊断工具）

```bash
# 检查端口连通性
rabbitmq-diagnostics check_port_connectivity

# 检查虚拟主机可访问性
rabbitmq-diagnostics check_virtual_hosts

# 查看内存使用详情
rabbitmq-diagnostics memory_breakdown

# 查看环境变量
rabbitmq-diagnostics environment

# 查看 Erlang 进程列表
rabbitmq-diagnostics erlang_cookie_sources
```

## 4.4 rabbitmqadmin（HTTP API 管理工具）

```bash
# 下载 rabbitmqadmin（从 Management 插件获取）
wget http://localhost:15672/cli/rabbitmqadmin
chmod +x rabbitmqadmin

# 列出队列
rabbitmqadmin list queues

# 声明队列
rabbitmqadmin declare queue name=my_queue durable=true

# 声明 Exchange
rabbitmqadmin declare exchange name=my_exchange type=direct durable=true

# 声明绑定
rabbitmqadmin declare binding source=my_exchange destination=my_queue routing_key=my_key

# 发布消息
rabbitmqadmin publish exchange=my_exchange routing_key=my_key payload='{"msg":"hello"}'

# 获取消息
rabbitmqadmin get queue=my_queue requeue=false
```

---

# 5. 用户与权限管理

## 5.1 用户角色（Tags）

| Tag | 权限说明 |
|------|----------|
| （无） | 普通用户，只能通过 AMQP 客户端连接 |
| **management** | 可通过 Management UI / HTTP API 管理（等同于无 Tag + UI 访问） |
| **policymaker** | management 权限 + 管理 Policy 和 Parameter |
| **monitoring** | management 权限 + 可查看所有连接和信道信息 |
| **administrator** | 最高权限，可管理用户、VHost、权限，查看所有信息 |

```bash
# 创建 administrator 用户
rabbitmqctl add_user myadmin myadmin123
rabbitmqctl set_user_tags myadmin administrator
rabbitmqctl set_permissions -p / myadmin ".*" ".*" ".*"
```

## 5.2 权限模型

| 权限 | 对应操作 |
|------|----------|
| **configure** | 创建/删除 Exchange 和 Queue（声明和删除资源） |
| **write** | 向 Exchange 发布消息 |
| **read** | 从 Queue 消费消息、绑定 Queue 到 Exchange |

```bash
# 授予 myapp VHost 的完整权限
rabbitmqctl set_permissions -p /myapp myuser "^myapp\." "^(myapp|myexchange)\." "^myapp\."

# 格式：set_permissions -p <vhost> <user> <configure_regex> <write_regex> <read_regex>
```

---

# 6. 配置文件

RabbitMQ 支持多种配置方式：

## 6.1 经典配置（rabbitmq.conf，推荐）

```ini
# /etc/rabbitmq/rabbitmq.conf

## 监听配置
listeners.tcp.default = 5672
listeners.ssl.default = 5671

## 管理插件监听
management.tcp.port = 15672

## 磁盘空间告警
disk_free_limit.absolute = 1GB
disk_free_limit.relative = 1.0

## 内存告警
vm_memory_high_watermark.relative = 0.4

## 日志配置
log.dir = /var/log/rabbitmq
log.file = rabbitmq.log
log.file.level = info

## 心跳超时
heartbeat = 60

## 连接限制
channel_max = 2047
frame_max = 131072

## 用户限制
default_user = admin
default_pass = admin123
default_vhost = /
default_permissions.configure = .*
default_permissions.read = .*
default_permissions.write = .*

## 集群节点
# cluster_formation.peer_discovery_backend = rabbit_peer_discovery_classic_config
# cluster_formation.classic_config.nodes.1 = rabbit@node1
# cluster_formation.classic_config.nodes.2 = rabbit@node2
```

## 6.2 高级配置（advanced.config）

```erlang
% /etc/rabbitmq/advanced.config
[
  {rabbit, [
    {msg_store_file_size_limit, 16777216},  % 消息存储文件大小限制 (16MB)
    {queue_index_embed_msgs_below, 4096},    % 小消息嵌入队列索引 (4KB)
    {consumer_timeout, 900000}               % 消费者超时时间 (15 分钟，3.12+)
  ]}
].
```

---

# 7. 常用插件清单

| 插件名称 | 功能 | 是否需要手动启用 |
|----------|------|:---:|
| `rabbitmq_management` | Web 管理界面 + HTTP API | 是 |
| `rabbitmq_management_agent` | 管理统计信息收集（management 依赖） | 自动 |
| `rabbitmq_prometheus` | Prometheus 监控指标暴露 | 是 |
| `rabbitmq_delayed_message_exchange` | 延迟消息交换机 | 是（需手动下载安装） |
| `rabbitmq_shovel` | 跨 Broker / 跨集群消息搬运 | 是 |
| `rabbitmq_shovel_management` | Shovel 管理界面 | 是 |
| `rabbitmq_federation` | 跨集群/跨 Broker 联合 | 是 |
| `rabbitmq_federation_management` | Federation 管理界面 | 是 |
| `rabbitmq_mqtt` | MQTT 协议支持 | 是 |
| `rabbitmq_stomp` | STOMP 协议支持 | 是 |
| `rabbitmq_stream` | Stream 队列（流式消息） | 是 |

```bash
# 推荐生产环境启用的插件
rabbitmq-plugins enable rabbitmq_management
rabbitmq-plugins enable rabbitmq_prometheus
```

---

# 8. 环境验证

安装完成后，通过以下步骤验证环境是否正常：

```bash
# 1. 检查 RabbitMQ 状态
rabbitmqctl status

# 2. 检查端口监听
netstat -tlnp | grep -E "5672|15672"

# 3. 验证 HTTP API
curl -u admin:admin123 http://localhost:15672/api/overview

# 4. 浏览器访问
# http://localhost:15672
```

## 8.1 Spring Boot 连接测试

`pom.xml` 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

`application.yml`：

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: admin123
    virtual-host: /
    # 可选配置
    connection-timeout: 5000
    requested-heartbeat: 60
    publisher-confirm-type: correlated
    publisher-returns: true
```

测试类：

```java
import org.junit.jupiter.api.Test;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class RabbitMQConnectionTest {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    void testConnection() {
        rabbitTemplate.convertAndSend("test.exchange", "test.key", "Hello RabbitMQ!");
        System.out.println("消息发送成功，连接正常");
    }
}
```
