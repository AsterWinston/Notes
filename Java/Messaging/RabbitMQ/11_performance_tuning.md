
> 版本说明：本文档基于 **RabbitMQ 3.13.x**，从 Producer、Broker、Consumer、OS 四个层面讲解性能调优。

---

# 1. 性能基准参考

| 场景 | 吞吐量（QPS） | 延迟 |
|------|:---:|------|
| 简单发送（无确认、无持久化） | 100K~200K | <1ms |
| 发送 + Publisher Confirm | 50K~100K | 1~5ms |
| 发送 + Confirm + 持久化 | 10K~30K | 1~10ms |
| 发送 + Quorum Queue | 5K~20K | 5~20ms |
| 消费（autoAck） | 100K+ | <1ms |
| 消费（manualAck + 持久化） | 10K~30K | 1~5ms |

> **测试环境**：单节点、8 核 CPU、16GB 内存、SSD，消息大小 100 字节。

---

# 2. Producer 优化

## 2.1 优化策略

```java
public class HighThroughputProducer {

    public static void main(String[] args) throws Exception {
        try (Connection connection = RabbitMQUtil.getConnection();
             Channel channel = connection.createChannel()) {

            // 1. 启用 Publisher Confirm（异步）
            channel.confirmSelect();

            // 2. 声明非持久化队列（如果可容忍丢失）
            channel.queueDeclare("perf.queue", false, false, true, null);

            // 3. 批量发送
            List<byte[]> batch = new ArrayList<>();
            for (int i = 0; i < 10000; i++) {
                batch.add(("msg-" + i).getBytes(StandardCharsets.UTF_8));
            }

            long start = System.currentTimeMillis();
            for (byte[] msg : batch) {
                channel.basicPublish("", "perf.queue", null, msg);
            }
            channel.waitForConfirmsOrDie(5000); // 等待批量确认

            long elapsed = System.currentTimeMillis() - start;
            System.out.printf("发送 %d 条消息，耗时 %dms，吞吐 %.0f msg/s%n",
                    batch.size(), elapsed, batch.size() * 1000.0 / elapsed);
        }
    }
}
```

## 2.2 Producer 优化要点

| 优化项 | 说明 | 效果 |
|--------|------|------|
| **重用 Connection** | 一个应用创建一个 Connection，多线程共享 | 减少连接开销 |
| **ThreadLocal Channel** | 每个线程独立 Channel | 避免 Channel 竞争 |
| **异步 Confirm** | 使用 `addConfirmListener` 代替同步等待 | 提升 10 倍吞吐 |
| **压缩消息体** | Gzip/Snappy 压缩大消息 | 减少带宽和存储 |
| **使用批量发布** | 合并多个 basicPublish 到一批 | 减少网络往返 |
| **设置 mandatory=false** | 不关心路由失败时关闭 mandatory | 减少 Broker 侧开销 |

---

# 3. Broker 优化

## 3.1 rabbitmq.conf 核心调优参数

```ini
# ========== 内存 ==========
vm_memory_high_watermark.relative = 0.4       # 内存告警阈值（生产建议 0.4~0.6）

# ========== 磁盘 ==========
disk_free_limit.absolute = 2GB                # 磁盘剩余空间告警阈值

# ========== 连接 ==========
heartbeat = 60                                # 心跳间隔（秒）
channel_max = 0                               # 最大 Channel 数（0=无限制）
frame_max = 131072                            # 最大帧大小（128KB）
tcp_listen_options.backlog = 128              # TCP backlog
tcp_listen_options.nodelay = true             # 禁用 Nagle

# ========== 日志 ==========
log.file.level = info                         # 日志级别（高并发设为 warn）

# ========== 限流 ==========
credit_flow_default_credit = {200, 50}        # 默认流控信用值
```

## 3.2 高级配置（advanced.config）

```erlang
[
  {rabbit, [
    %% 消息存储文件大小限制
    {msg_store_file_size_limit, 16777216},     % 16MB
    %% 小消息嵌入队列索引（减少磁盘 I/O）
    {queue_index_embed_msgs_below, 4096},      % 4KB 以下的消��嵌入索引
    %% 消息存储 GC 间隔
    {msg_store_credit_disc_bound, {4000, 800}},
    %% 队列主进程 hibernation（内存优化）
    {background_gc_enabled, true},
    {background_gc_target_interval, 60000}     % 60 秒
  ]},
  {rabbitmq_management, [
    {rates_mode, basic},                        % 避免管理界面性能开销
    {sample_retention_policies, [
      {global, [{605, 1}]},
      {basic, [{605, 1}]},
      {detailed, [{10, 2}]}
    ]}
  ]}
].
```

## 3.3 队列设计优化

| 优化项 | 说明 |
|--------|------|
| **避免过多队列** | Broker 性能与队列数量成正比，大量队列（>10万）会显著拖慢性能 |
| **使用合适的队列类型** | Quorum Queue 可靠但比 Classic 慢约 2 倍 |
| **限制队列长度** | 设置 `x-max-length` 防止消息无限堆积 |
| **惰性队列** | 大消息堆积时使用 Lazy Queue 降低内存压力 |
| **定期清理** | 监控并清理不再使用的 Queue 和 Binding |

---

# 4. Consumer 优化

## 4.1 关键参数调优

```java
public class HighThroughputConsumer {

    public static void main(String[] args) throws Exception {
        Connection connection = RabbitMQUtil.getConnection();
        Channel channel = connection.createChannel();

        // 1. 增加 Prefetch（提升吞吐）
        channel.basicQos(200);  // 默认 0=无限制

        // 2. 批量确认
        // 3. 多消费者并发

        channel.basicConsume("perf.queue", false,
            new DeliverCallback() {
                int count = 0;

                @Override
                public void handle(String consumerTag, Delivery delivery) throws IOException {
                    // 快速处理
                    count++;

                    // 批量确认（每 100 条确认一次）
                    if (count % 100 == 0) {
                        channel.basicAck(delivery.getEnvelope().getDeliveryTag(), true);
                    }
                }
            },
            consumerTag -> {}
        );
    }
}
```

## 4.2 Consumer 优化要点

| 优化项 | 建议值 | 说明 |
|--------|--------|------|
| **Prefetch** | 100~500 | 批量获取消息，减少网络往返 |
| **Concurrency** | CPU 核数 × 2 | 初始并发消费者数 |
| **Max Concurrency** | 按积压调整 | 队列堆积时自动扩容 |
| **autoAck** | false（手动 ACK） | 生产环境必须手动确认 |
| **批量 ACK** | 每 100~500 条 | `basicAck(tag, true)` 批量确认 |

## 4.3 Spring Boot 消费者配置

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 250
        concurrency: 5
        max-concurrency: 20
        acknowledge-mode: manual
        batch-size: 100        # 批量消费
        # 启动超时
        startup-timeout: 30000
```

---

# 5. OS 层面优化

## 5.1 文件描述符

```bash
# 查看当前限制
ulimit -n

# 设置为 65536（推荐）
ulimit -n 65536

# 永久生效 (/etc/security/limits.conf)
rabbitmq soft nofile 65536
rabbitmq hard nofile 65536

# RabbitMQ 的用户是 rabbitmq，确保 /etc/systemd/system/rabbitmq-server.service.d/limits.conf 也配置
[Service]
LimitNOFILE=65536
```

## 5.2 内核参数

```ini
# /etc/sysctl.conf
# TCP 优化
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 20
net.ipv4.tcp_keepalive_probes = 3

# 网络缓冲区
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
```

---

# 6. 压测工具

## 6.1 PerfTest（官方压测工具）

```bash
# 安装 PerfTest
wget https://github.com/rabbitmq/rabbitmq-perf-test/releases/download/v2.19.0/rabbitmq-perf-test-2.19.0.jar

# 基础压测：1 个 Producer，1 个 Consumer，无 ACK
java -jar rabbitmq-perf-test-2.19.0.jar \
  --uri amqp://admin:admin123@localhost:5672

# 混合负载（1 个 Producer + 1 个 Consumer）
java -jar rabbitmq-perf-test-2.19.0.jar \
  --uri amqp://admin:admin123@localhost:5672 \
  --producers 1 --consumers 1 \
  --flag persistent \
  --queue-pattern 'perf-test-%d' \
  --queue-pattern-from 1 --queue-pattern-to 10

# 多生产者多消费者
java -jar rabbitmq-perf-test-2.19.0.jar \
  --uri amqp://admin:admin123@localhost:5672 \
  --producers 5 --consumers 3 \
  --pmessages 100000 \
  --size 1024 \
  --confirm 1

# 常用参数
# --producers N          生产者数量
# --consumers N          消费者数量
# --pmessages N          每条信道发送的消息数
# --size N               消息体大���（bytes）
# --confirm N            每 N 条确认一次
# --autoack              自动确认
# --flag persistent      持久化消息
# --queue-pattern        队列名模式
# --queue-args           队列参数（如 x-queue-type=quorum）
# --rate N               限速（每秒消息数）
```

## 6.2 压测结果解读

```
id: test-123, time: 30.0s
    sent:   50000 msg/s     → Producer 发送速率
    recv:   48000 msg/s     → Consumer 消费速率
    min/median/75th/95th/99th latency: 120/340/450/780/1200 μs
```

---

# 7. 常见性能问题与排查

## 7.1 问题一：消息堆积

```bash
# 检查堆积
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers

# 原因与方案
# 1. 消费慢 → 增加并发消费者 / 优化消费者代码
# 2. Prefetch 太小 → 调大 basicQos
# 3. 消息大 → 压缩消息 / 限制消息体大小
# 4. 队列设计问题 → 拆分队列
```

## 7.2 问题二：内存高

```bash
rabbitmq-diagnostics memory_breakdown

# 常见原因
# 1. 大量未确认消息 → 增加消费速度或降低 Prefetch
# 2. 大量队列或绑定 → 清理无用队列
# 3. 管理插件采样 → 降低采样频率
# 4. 流控制触发 → 检查 vm_memory_high_watermark 设置
```

## 7.3 问题三：吞吐量突然下降

```
诊断步骤：
1. rabbitmq-diagnostics status → 查看报警状态
2. rabbitmqctl environment → 查看配置
3. 检查磁盘 I/O（iostat / iotop）
4. 检查网络 (netstat / ss)
5. 检查 Erlang 进程调度（rabbitmq-diagnostics runtime_thread_stats）
```

---

# 8. 性能优化速查表

| 层面 | 优化项 | 效果 |
|------|--------|:---:|
| **Producer** | 异步 Confirm | ⬆⬆⬆ |
| **Producer** | 批量发送 | ⬆⬆ |
| **Producer** | 压缩消息体 | ⬆ |
| **Broker** | 减少不必要持久化 | ⬆⬆⬆ |
| **Broker** | 使用 Classic Queue（而非 Quorum） | ⬆⬆ |
| **Broker** | 调整 vm_memory_high_watermark | ⬆ |
| **Consumer** | 增加 Prefetch | ⬆⬆ |
| **Consumer** | 批量 ACK | ⬆⬆ |
| **Consumer** | 增加并发消费者 | ⬆⬆ |
| **OS** | 调整 ulimit（文件描述符） | ⬆ |
| **OS** | TCP 缓冲区和连接参数调优 | ⬆ |
| **OS** | 使用 SSD | ⬆⬆ |

---

# 9. docker-compose 全功能编排

```yaml
version: "3.8"
services:
  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    container_name: rabbitmq
    hostname: rabbit-node
    ports:
      - "5672:5672"
      - "15672:15672"
      - "15692:15692"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_VM_MEMORY_HIGH_WATERMARK: "0.4"
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - ./advanced.config:/etc/rabbitmq/advanced.config
      - rabbitmq_data:/var/lib/rabbitmq
    ulimits:
      nofile:
        soft: 65536
        hard: 65536

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
    volumes:
      - grafana_data:/var/lib/grafana
      - ./datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml

volumes:
  rabbitmq_data:
  prometheus_data:
  grafana_data:
```
