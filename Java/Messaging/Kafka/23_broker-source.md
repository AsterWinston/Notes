# 1. 源码分析总览

## 1.1 Broker 核心类图

```
KafkaServer
│
├── SocketServer (网络层)
│   ├── Acceptor (接收连接)
│   ├── Processor (NIO 读写)
│   └── RequestChannel (请求/响应队列)
│
├── KafkaApis (请求处理层)
│   ├── KafkaRequestHandlerPool (I/O 线程池)
│   │   └── KafkaRequestHandler (处理线程)
│   └── handle*() (各类请求处理方法)
│
├── KafkaScheduler (后台定时任务)
│
├── MetadataCache (元数据缓存)
│
├── KafkaController (控制器, 仅活跃 Controller)
│   ├── PartitionStateMachine (分区状态机)
│   └── ReplicaStateMachine (副本状态机)
│
├── ReplicaManager (副本管理)
│   ├── ReplicaFetcherManager (Follower 拉取管理)
│   └── Partition (分区对象)
│
├── GroupCoordinator (消费者组协调)
│   └── GroupMetadataManager
│
├── TransactionCoordinator (事务协调)
│
└── KafkaZkClient / QuorumController (元数据存储)
```

# 2. KafkaServer 启动流程

## 2.1 主启动方法

```scala
// KafkaServer.scala

class KafkaServer(val config: KafkaConfig, ...) {

  def startup(): Unit = {
    try {
      info("starting")

      // === 阶段 1: 初始化基础组件 ===
      if (isShuttingDown.get) throw new IllegalStateException("...")
      isStartingUp = true
      this.logContext = new LogContext(s"[KafkaServer id=${config.brokerId}] ")

      // === 阶段 2: 初始化 Metrics ===
      metrics = new Metrics(metricConfig, reporters, time, ...)

      // === 阶段 3: 初始化 ZK / KRaft 客户端 ===
      if (config.requiresZookeeper) {
        // ZK 模式
        zkClient = KafkaZkClient(...)
        // 注册 Broker
        zkClient.registerBroker(brokerInfo)
      } else {
        // KRaft 模式
        this.metadataPublishers = ...
      }

      // === 阶段 4: 启动日志管理器 ===
      logManager = LogManager(config, initialOfflineDirs, ...)
      logManager.startup(metadataPublisher)
      //    ├── 扫描 log.dirs 下所有日志目录
      //    ├── 加载每个分区目录下的所有 Segment 文件
      //    ├── 校验并恢复 Segment（清理脏数据）
      //    ├── 启动 LogCleaner 线程（如果使用 compact 策略）
      //    └── 启动定时任务（flush / checkpoint / retention）

      // === 阶段 5: 启动网络层 ===
      socketServer = new SocketServer(config, metrics, time, ...)
      socketServer.startup()
      //    ├── 创建 Acceptor + Processor 线程
      //    ├── 绑定监听端口
      //    └── 开始接收连接

      // === 阶段 6: 启动副本管理器 ===
      replicaManager = createReplicaManager(isShuttingDown)
      replicaManager.startup()
      //    ├── 初始化 ReplicaFetcherManager（Follower 拉取管理）
      //    ├── 启动 Fetcher 线程（从 Leader 同步数据）
      //    └── 启动 ISR 过期检查

      // === 阶段 7: 启动请求处理层 ===
      dataPlaneRequestHandlerPool = new KafkaRequestHandlerPool(
        config.brokerId, socketServer.dataPlaneRequestChannel, apis, time,
        config.numIoThreads, ...)
      //    ├── 创建 num.io.threads 个 I/O 线程
      //    └── 每个线程从 RequestChannel 取请求并处理

      // === 阶段 8: 创建请求处理器 (KafkaApis) ===
      apis = new KafkaApis(
        socketServer.dataPlaneRequestChannel,
        replicaManager,
        groupCoordinator,
        txnCoordinator,
        ...)

      // === 阶段 9: 启动 Controller ===
      if (config.requiresZookeeper) {
        lifecycleManager = new ZooKeeperLifecycleManager(zkClient, config, ...)
      } else {
        lifecycleManager = new KRaftLifecycleManager(metadataPublishers, ...)
      }
      lifecycleManager.startup()
      //    → 在 ZK 模式: 注册 Controller 选举回调
      //    → 在 KRaft 模式: 检查 Raft Leader/Active Controller

      // === 阶段 10: 后续启动 ===
      // 启动 GroupCoordinator
      groupCoordinator.startup()
      // 启动 TransactionCoordinator
      transactionCoordinator.startup()
      // 启动其他组件...

      // 标记启动完成
      isStartingUp = false
      lifecycleManager.setReady()
      info("started")
    } catch {
      case e: Throwable =>
        fatal("Fatal error during KafkaServer startup. Prepare to shutdown", e)
        shutdown()
        throw e
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/server/KafkaServer.scala:184`

# 3. SocketServer 网络架构

## 3.1 初始化

```scala
// SocketServer.scala

class SocketServer(val config: KafkaConfig,
                   val metrics: Metrics,
                   val time: Time,
                   val credentialProvider: CredentialProvider)
  extends Logging with KafkaMetricsGroup {

  // 多监听器
  private val endpoints = config.listeners.map(listener => {
    val listenerName = listener.listenerName
    val securityProtocol = listener.securityProtocol
    EndPoint.createEndPoint(listenerName.value, securityProtocol, ...)
  })

  // 每个监听器对应一组 Acceptors + Processors
  private val dataPlaneAcceptors = new ConcurrentHashMap[EndPoint, Acceptor]()
  dataPlaneRequestChannel = new RequestChannel(maxQueuedRequests, ...)

  def startup(): Unit = {
    // 为每个 EndPoint 创建 Acceptor
    for (endPoint <- endpoints) {
      val acceptor = new Acceptor(
        endPoint,
        sendBufferSize, recvBufferSize,
        config.numNetworkThreads, // Processor 数量
        dataPlaneRequestChannel, ...)
      dataPlaneAcceptors.put(endPoint, acceptor)
      // Acceptor 内部会创建 numNetworkThreads 个 Processor
      Utils.newThread(s"kafka-socket-acceptor-${endPoint.listenerName}", acceptor, false).start()
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/network/SocketServer.scala:130`

## 3.2 Acceptor 线程

```scala
// SocketServer.scala → Acceptor

private[kafka] class Acceptor(val endPoint: EndPoint,
                               val sendBufferSize: Int,
                               val recvBufferSize: Int,
                               numProcessors: Int,
                               requestChannel: RequestChannel, ...)
  extends AbstractServerThread(connectionQuotas) {

  // 创建 N 个 Processor
  private val processors = new ArrayBuffer[Processor]()
  for (i <- 0 until numProcessors) {
    processors += new Processor(
      i, time, requestChannel, connectionQuotas, ...)
  }

  def run(): Unit = {
    // 打开 ServerSocketChannel 并绑定
    val serverChannel = openServerSocket(endPoint.host, endPoint.port)
    serverChannel.register(selector, SelectionKey.OP_ACCEPT)

    while (isRunning) {
      try {
        // 等待新连接
        acceptNewConnections()
      } catch {
        case e: Throwable => error("Error accepting connection", e)
      }
    }
  }

  private def acceptNewConnections(): Unit = {
    val ready = selector.select(500)
    if (ready > 0) {
      val keys = selector.selectedKeys()
      val iter = keys.iterator()
      while (iter.hasNext && isRunning) {
        val key = iter.next()
        iter.remove()
        if (key.isAcceptable) {
          val socketChannel = key.channel().asInstanceOf[ServerSocketChannel].accept()
          // 轮询分配一个 Processor 处理此连接
          val processor = processors(currentProcessorIndex)
          currentProcessorIndex = (currentProcessorIndex + 1) % processors.size
          processor.accept(socketChannel)
        }
      }
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/network/SocketServer.scala:586`

## 3.3 Processor 线程

```scala
// SocketServer.scala → Processor

private[kafka] class Processor(val id: Int,
                                time: Time,
                                requestChannel: RequestChannel, ...)
  extends AbstractServerThread(connectionQuotas) {

  // 新连接队列（从 Acceptor 移交过来的）
  private val newConnections = new ArrayBlockingQueue[SocketChannel](connectionQueueSize)
  private val selector = NSelector.open()

  def accept(socketChannel: SocketChannel): Unit = {
    newConnections.add(socketChannel)
    selector.wakeup()
  }

  override def run(): Unit = {
    while (isRunning) {
      try {
        // ① 处理新连接（从 Acceptor 移交）
        configureNewConnections()

        // ② 注册新的 NIO OP_READ 事件
        processNewResponses()

        // ③ 执行 NIO Selector poll
        selector.poll(300)

        // ④ 处理已就绪的事件（读请求）
        processCompletedReceives()

        // ⑤ 处理待发送的响应（写回）
        processCompletedSends()

        // ⑥ 关闭断开/空闲的连接
        processDisconnected()
      } catch {
        case e: Throwable => ...
      }
    }
  }

  private def processCompletedReceives(): Unit = {
    // 遍历所有 SelectionKey，检查 OP_READ 就绪
    selector.completedReceives.forEach { receive =>
      val req = RequestChannel.Request(processor = id,
        context = receive.context,
        startTimeNanos = receive.startTimeNanos,
        ...)

      // 将请求放入 RequestChannel
      requestChannel.sendRequest(req)
    }
  }

  private def processCompletedSends(): Unit = {
    // 从 RequestChannel 响应队列取响应
    selector.completedSends.forEach { send =>
      // 记录指标
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/network/SocketServer.scala:756`

# 4. KafkaApis 请求处理

## 4.1 请求分发器

```scala
// KafkaApis.scala

class KafkaApis(val requestChannel: RequestChannel,
                val replicaManager: ReplicaManager,
                val groupCoordinator: GroupCoordinator,
                val txnCoordinator: TransactionCoordinator,
                ...) extends ApiRequestHandler {

  def handle(request: RequestChannel.Request): Unit = {
    try {
      trace(s"Handling request: ${request.header}")

      request.header.apiKey match {
        case ApiKeys.PRODUCE          => handleProduceRequest(request)
        case ApiKeys.FETCH            => handleFetchRequest(request)
        case ApiKeys.LIST_OFFSETS     => handleListOffsetRequest(request)
        case ApiKeys.METADATA         => handleTopicMetadataRequest(request)
        case ApiKeys.LEADER_AND_ISR   => handleLeaderAndIsrRequest(request)
        case ApiKeys.STOP_REPLICA     => handleStopReplicaRequest(request)
        case ApiKeys.UPDATE_METADATA  => handleUpdateMetadataRequest(request)
        case ApiKeys.OFFSET_COMMIT    => handleOffsetCommitRequest(request)
        case ApiKeys.OFFSET_FETCH     => handleOffsetFetchRequest(request)
        case ApiKeys.FIND_COORDINATOR => handleFindCoordinatorRequest(request)
        case ApiKeys.JOIN_GROUP       => handleJoinGroupRequest(request)
        case ApiKeys.HEARTBEAT        => handleHeartbeatRequest(request)
        case ApiKeys.SYNC_GROUP       => handleSyncGroupRequest(request)
        case ApiKeys.INIT_PRODUCER_ID => handleInitProducerIdRequest(request)
        case ApiKeys.ADD_PARTITIONS_TO_TXN  => handleAddPartitionsToTxnRequest(request)
        case ApiKeys.ADD_OFFSETS_TO_TXN     => handleAddOffsetsToTxnRequest(request)
        case ApiKeys.END_TXN                => handleEndTxnRequest(request)
        case ApiKeys.WRITE_TXN_MARKERS      => handleWriteTxnMarkersRequest(request)
        case ApiKeys.TXN_OFFSET_COMMIT      => handleTxnOffsetCommitRequest(request)
        // ... 50+ API 类型
        case _ => throw new IllegalStateException(
          s"No handler for request api key ${request.header.apiKey}")
      }
    } catch {
      case e: FatalExitError => throw e
      case e: Throwable => handleError(request, e)
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/server/KafkaApis.scala:150`

## 4.2 I/O 线程池 (KafkaRequestHandlerPool)

```scala
// KafkaRequestHandlerPool.scala

class KafkaRequestHandlerPool(val brokerId: Int,
                               val requestChannel: RequestChannel,
                               val apis: ApiRequestHandler,
                               time: Time,
                               numThreads: Int,
                               ...) extends Logging {

  val runnables = new Array[KafkaRequestHandler](numThreads)
  for (i <- 0 until numThreads) {
    runnables(i) = new KafkaRequestHandler(i, brokerId, aggregateIdleMeter, numThreads, requestChannel, apis, time)
    Utils.newThread(s"kafka-request-handler-$i", runnables(i), false).start()
  }

  def shutdown(): Unit = {
    runnables.foreach(_.shutdown())
  }
}

class KafkaRequestHandler(id: Int,
                           brokerId: Int,
                           val requestChannel: RequestChannel,
                           apis: ApiRequestHandler,
                           time: Time) extends Runnable {

  override def run(): Unit = {
    while (!stopped) {
      // ① 从 RequestChannel 取一个请求（阻塞）
      val req = requestChannel.receiveRequest(300)

      // ② 调用 KafkaApis.handle 处理
      apis.handle(req)
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/server/KafkaRequestHandler.scala:34`

# 5. ReplicaManager 副本管理

```scala
// ReplicaManager.scala

class ReplicaManager(val config: KafkaConfig,
                     metrics: Metrics,
                     time: Time,
                     scheduler: Scheduler,
                     val logManager: LogManager,
                     ...) extends Logging {

  // Broker 所有本地 Partition 副本
  private val allPartitions = new Pool[TopicPartition, Partition]

  // === 核心方法: appendRecords (处理 Produce 请求) ===
  def appendRecords(timeout: Long,
                    requiredAcks: Short,
                    internalTopicsAllowed: Boolean,
                    isFromClient: Boolean,
                    entriesPerPartition: Map[TopicPartition, MemoryRecords],
                    responseCallback: Map[TopicPartition, PartitionResponse] => Unit,
                    ...): Unit = {

    // ① 遍历每个分区的消息
    val partitionResults = entriesPerPartition.map { case (tp, records) =>
      val partition = getPartitionOrException(tp)
      // ② 追加到日志
      val info = partition.appendRecordsToFollowerOrLeader(records, isFromClient, requiredAcks)
      // ③ 收集结果
      (tp, LogAppendResult(info))
    }

    // ④ 根据 acks 策略处理响应
    if (requiredAcks == 0) {
      // acks=0 → 立刻响应（不等待）
      responseCallback(partitionResults.mapValues(_.error))
    } else if (requiredAcks == 1) {
      // acks=1 → Leader 写入完成后响应
      responseCallback(partitionResults.mapValues(...))
    } else if (requiredAcks == -1) {
      // acks=all → 进入 Purgatory 等待 ISR 全部完成
      val delayedProduce = new DelayedProduce(timeout, produceMetadata, this, responseCallback)
      delayedProducePurgatory.tryCompleteElseWatch(delayedProduce, produceMetadata.keys)
    }
  }

  // === 核心方法: fetchMessages (处理 Fetch 请求) ===
  def fetchMessages(timeout: Long,
                    replicaId: Int,
                    fetchMinBytes: Int,
                    fetchMaxBytes: Int,
                    fetchInfos: Seq[(TopicPartition, PartitionData)],
                    responseCallback: Seq[(TopicPartition, FetchPartitionData)] => Unit,
                    ...): Unit = {

    // ① 从日志读取数据
    val logReadResults = readFromLog(fetchInfos)

    // ② 检查是否满足 fetchMinBytes
    val bytesReadable = logReadResults.map(_.info.records.sizeInBytes).sum
    if (bytesReadable >= fetchMinBytes) {
      responseCallback(logReadResults)
    } else {
      // 数据不足 → 进入 Purgatory 等待
      val delayedFetch = new DelayedFetch(timeout, ...)
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/server/ReplicaManager.scala:373`

# 6. GroupCoordinator 消费者组协调

```scala
// GroupCoordinator.scala

class GroupCoordinator(val brokerId: Int,
                        val interBrokerProtocolVersion: ApiVersion,
                        val config: GroupCoordinatorConfig,
                        ...) extends Logging {

  // 所有 Consumer Group 的状态
  private val groupMetadataCache = new ConcurrentHashMap[String, GroupMetadata]

  def handleJoinGroup(groupId: String,
                      memberId: String,
                      groupInstanceId: Option[String],
                      protocolType: String,
                      protocols: List[(String, Array[Byte])],
                      responseCallback: JoinGroupCallback): Unit = {

    // ① 获取或创建 GroupMetadata
    val group = groupMetadataCache.synchronized {
      groupMetadataCache.getOrElseUpdate(groupId,
        new GroupMetadata(groupId, Initial, time))
    }

    // ② 检查 Static Membership（group.instance.id）
    //     如果已有相同 instanceId 的成员 → 替换旧成员

    // ③ 添加新成员
    group.add(member)

    // ④ 触发 Rebalance（如果需要）
    group.currentState match {
      case Dead | Empty => ...
      case Stable | CompletingRebalance =>
        // 如果非首次成员 → 触发 Rebalance
        group.prepareRebalance()
    }

    // ⑤ 返回 JoinGroup 响应（DelayedJoin）
    val delayedJoin = new DelayedJoin(this, group, ...)
    delayedJoinPurgatory.tryCompleteElseWatch(delayedJoin, Seq(GroupKey(group.groupId)))
  }
}
```

> 文件位置: `core/src/main/scala/kafka/coordinator/group/GroupCoordinator.scala:228`

# 7. 延迟操作与 Purgatory

```scala
// DelayedProduce.scala

class DelayedProduce(delayMs: Long,
                     produceMetadata: ProduceMetadata,
                     replicaManager: ReplicaManager,
                     responseCallback: Map[TopicPartition, PartitionResponse] => Unit)
  extends DelayedOperation(delayMs) {

  override def tryComplete(): Boolean = {
    // 逐个检查每个 Partition
    produceMetadata.partitionStatus.foreach { case (tp, status) =>
      if (status.acksPending) {
        // 检查 ISR 内所有副本是否都已完成复制
        val (hasEnough, error) = replicaManager.getPartitionOrException(tp)
          .checkEnoughReplicasReachOffset(status.requiredOffset)
        if (error != Errors.NONE) {
          status.error = error
          status.acksPending = false
        } else if (hasEnough) {
          status.acksPending = false
        }
      }
    }

    // 如果所有分区的 acksPending = false → 完成
    if (!produceMetadata.partitionStatus.exists { case (_, status) => status.acksPending }) {
      responseCallback(produceMetadata.partitionStatus)
      true
    } else {
      false
    }
  }

  override def onExpiration(): Unit = {
    // 超时处理 → 设置错误
    produceMetadata.partitionStatus.foreach { case (_, status) =>
      status.error = Errors.REQUEST_TIMED_OUT
      status.acksPending = false
    }
    responseCallback(produceMetadata.partitionStatus)
  }
}
```

> 文件位置: `core/src/main/scala/kafka/server/DelayedProduce.scala:31`

# 8. 请求处理完整时序

```
Client Produce Request 的完整路径:

① 网络数据到达
  NIC → OS Kernel Buffer

② Processor 读取
  Processor.processCompletedReceives()
    → 反序列化请求头 (ApiKey, Version, CorrelationId)
    → 放入 RequestChannel.requestQueue

③ RequestHandler 处理
  KafkaRequestHandler.run()
    → requestChannel.receiveRequest()  // 取请求
    → apis.handle(request)             // KafkaApis.handle
      → handleProduceRequest()
        → replicaManager.appendRecords()
          → partition.appendRecordsToFollowerOrLeader()
            → log.appendAsLeader()     // 追加到日志
            → 注册到 Purgatory (如果 acks=all)
          → responseCallback()         // 生成响应
        → requestChannel.sendResponse() // 放入响应队列

④ Processor 写回
  Processor.processCompletedSends()
    → 序列化响应
    → 写回 SocketChannel
    → TCP 传输到 Client
```
