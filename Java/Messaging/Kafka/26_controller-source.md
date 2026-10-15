# 1. Controller 源码总览

## 1.1 KafkaController 核心构建（ZK 模式）

```scala
// KafkaController.scala

class KafkaController(val config: KafkaConfig,
                       zkClient: KafkaZkClient,
                       time: Time,
                       metrics: Metrics,
                       initialBrokerInfo: BrokerInfo,
                       ...) extends Logging {

  // 核心组件
  val controllerContext = new ControllerContext    // 控制器上下文（缓存）
  var partitionStateMachine: PartitionStateMachine
  var replicaStateMachine: ReplicaStateMachine
  val controllerEventManager = new ControllerEventManager(...)

  // 控制器上下文
  class ControllerContext {
    var epoch: Int = KafkaController.InitialControllerEpoch
    var epochZkVersion: Int = KafkaController.InitialControllerEpochZkVersion
    var allTopics: Set[String] = Set.empty
    var partitionReplicaAssignment: Map[TopicPartition, Seq[Int]] = Map.empty
    var partitionLeadershipInfo: Map[TopicPartition, LeaderIsrAndControllerEpoch] = Map.empty
    val stats: ControllerStats = new ControllerStats
    // ... 其他控制器级别的缓存
  }

  // 事件类型
  sealed trait ControllerEvent
  case class BrokerChange(added: Seq[Int], removed: Seq[Int]) extends ControllerEvent
  case class TopicChange(topics: Set[String]) extends ControllerEvent
  case class TopicDeletion(topics: Set[String]) extends ControllerEvent
  case class PartitionModifications(topic: String) extends ControllerEvent
  case class PartitionReassignment(reassignment: Map[TopicPartition, Seq[Int]]) extends ControllerEvent
  case class IsrChangeNotification() extends ControllerEvent
  case class LeaderAndIsrResponseReceived(...) extends ControllerEvent
  case class AutoPreferredLeaderElection() extends ControllerEvent
  case class ControlledShutdown(id: Int, controlledShutdownCallback: ...) extends ControllerEvent
}
```

> 文件位置: `core/src/main/scala/kafka/controller/KafkaController.scala:82`

# 2. Controller 选举

## 2.1 ZK 模式选举

```scala
// KafkaController.scala

def startup(): Unit = {
  // ① 注册 Broker 变更监听
  zkClient.subscribeBrokerChangeListener(brokerChangeHandler)

  // ② 注册 Topic 变更监听
  zkClient.subscribeTopicChangeListener(topicChangeHandler)
  zkClient.subscribeTopicDeletionListener(topicDeletionHandler)

  // ③ 注册 Partition 重组监听
  zkClient.subscribePartitionReassignment(partitionReassignmentHandler)

  // ④ 注册 ISR 变更监听
  zkClient.subscribeIsrChangeNotification(isrChangeNotificationHandler)

  // ⑤ 注册 Preferred Leader Election 监听
  zkClient.subscribePreferredReplicaElection(preferredReplicaElectionHandler)

  // ⑥ 启动事件处理线程
  controllerEventManager.start()

  // ⑦ 尝试选举为 Controller
  elect()
}

private def elect(): Unit = {
  val timestamp = time.milliseconds()
  // ① 尝试在 ZK 创建 /controller 临时节点
  val result = zkClient.registerControllerAndIncrementControllerEpoch(
    config.brokerId,
    controllerContext.epoch,
    timestamp)

  result match {
    case Right(zkVersion) =>
      // ② 成功 → 成为 Active Controller
      info(s"${config.brokerId} successfully elected as the controller")
      onControllerFailover()
      // ③ 注册 Controller 变更监听（监听其他 Broker 抢占）
      zkClient.subscribeControllerChangeListener(controllerChangeHandler)
    case Left(exception) =>
      // ④ 失败 → 已有 Controller → 注册监听等待
      zkClient.subscribeControllerChangeListener(controllerChangeHandler)
  }
}

// ===== onControllerFailover: Controller 切换后的恢复流程 =====
private def onControllerFailover(): Unit = {
  info("Reading controller epoch from ZooKeeper")

  // ① 从 ZK 读取最新的 controller_epoch + 1
  controllerContext.epoch = zkClient.getControllerEpoch + 1
  controllerContext.epochZkVersion = // ...

  // ② 从 ZK 读取实时 Broker 列表
  val (liveBrokerIds, allBrokerIds) = getAllBrokers
  controllerContext.liveBrokers = liveBrokerIds.toSet

  // ③ 从 ZK 读取所有 Topic 和 Partition
  val allTopics = zkClient.getAllTopicsInCluster()
  controllerContext.allTopics = allTopics

  // ④ 注册所有 Partition 的 ISR 监听
  registerPartitionModificationsHandlers(allTopics)

  // ⑤ 处理所有 Partition 的 Leader 选举
  replicaStateMachine.startup()
  partitionStateMachine.startup()

  // ⑥ 执行离线 Partition 迁移到 Online
  partitionStateMachine.triggerOnlinePartitionStateChange()

  // ⑦ 发送 LeaderAndIsr / UpdateMetadata
  sendInitialLeaderAndIsrRequest()
  sendInitialUpdateMetadataRequest()

  info("Controller has been elected and is ready")
}
```

> 文件位置: `core/src/main/scala/kafka/controller/KafkaController.scala:300`

## 2.2 ControllerChangeHandler（ZK Watcher）

```scala
// KafkaController.scala

class ControllerChangeHandler extends IZkDataListener {
  override def handleDataChange(dataPath: String, data: Array[Byte]): Unit = {
    // Controller 数据变化（节点内容变化）→ 不处理
  }

  override def handleDataDeleted(dataPath: String): Unit = {
    // /controller 临时节点被删除（Controller 宕机）
    info("Controller ephemeral node deleted, re-electing")
    elect() // 重新选举
  }
}
```

# 3. PartitionStateMachine 分区状态机

## 3.1 状态转换

```scala
// PartitionStateMachine.scala

class PartitionStateMachine(controller: KafkaController) extends Logging {

  private val controllerContext = controller.controllerContext

  // 状态定义
  sealed trait PartitionState
  case object NonExistentPartition extends PartitionState
  case object NewPartition extends PartitionState
  case object OnlinePartition extends PartitionState
  case object OfflinePartition extends PartitionState

  private val partitionState: mutable.Map[TopicPartition, PartitionState] =
    mutable.Map.empty

  // ===== 将分区转为 Online =====
  def handleStateChanges(
      partitions: Set[TopicPartition],
      targetState: PartitionState,
      ...): Map[TopicPartition, Either[Throwable, LeaderIsrAndControllerEpoch]] = {

    val result = mutable.Map[TopicPartition, Either[Throwable, LeaderIsrAndControllerEpoch]]()

    when(targetState) {
      // ① NonExistent → New
      //     创建分区（分配副本到 Broker）
      case NewPartition =>
        partitions.foreach { partition =>
          val assignment = controllerContext.partitionReplicaAssignment(partition)
          partitionState.put(partition, NewPartition)
        }

      // ② New / Offline → Online
      //     选举 Leader, 发送 LeaderAndIsr
      case OnlinePartition =>
        val partitionsToElectLeader = partitions.filter { tp =>
          partitionState.get(tp).contains(OfflinePartition) ||
          partitionState.get(tp).contains(NewPartition)
        }

        val (successful, failed) = doElectLeaderForPartitions(
          partitionsToElectLeader.toSeq, ...)

        successful.foreach { case (tp, leaderIsr) =>
          partitionState.put(tp, OnlinePartition)
          // 记录到 ZK
          if (zkVersion.isDefined) {
            // 将 ISR 持久化到 ZK
          }
        }
        result ++= successful.mapValues(Right(_))

      // ③ Online → Offline
      //     分区 Leader 不可用
      case OfflinePartition =>
        partitions.foreach { tp =>
          partitionState.put(tp, OfflinePartition)
        }

      // ④ 任意 → NonExistent
      //     分区删除
      case NonExistentPartition =>
        partitions.foreach { tp =>
          partitionState.remove(tp)
        }
    }
    result.toMap
  }
}
```

> 文件位置: `core/src/main/scala/kafka/controller/PartitionStateMachine.scala:47`

## 3.2 Leader 选举算法实现

```scala
// PartitionStateMachine.scala

private def doElectLeaderForPartitions(
    partitions: Seq[TopicPartition],
    ...): (Map[TopicPartition, LeaderIsrAndControllerEpoch], Seq[TopicPartition]) = {

  val successful = mutable.Map[TopicPartition, LeaderIsrAndControllerEpoch]()
  val failed = mutable.Buffer[TopicPartition]()

  for (tp <- partitions) {
    val replicas = controllerContext.partitionReplicaAssignment(tp)
    val isr = controllerContext.partitionLeadershipInfo.get(tp)
      .map(_.isr)
      .getOrElse(replicas)

    // ① 选举 Leader
    val (newLeader, newIsr) = electLeaderForPartition(tp, isr, replicas)

    if (newLeader != -1) {
      // ② 创建 LeaderIsrAndControllerEpoch
      val leaderAndIsr = LeaderIsrAndControllerEpoch(
        LeaderAndIsr(
          leader = newLeader,
          leaderEpoch = controllerEpochIncremented,
          isr = newIsr,
          ...),
        controllerContext.epoch)

      // ③ 发往 Leader Broker + Follower Brokers
      sendLeaderAndIsrRequest(leaderAndIsr)

      successful.put(tp, leaderAndIsr)
    } else {
      // 没有可用的 Leader → 失败
      failed += tp
    }
  }

  (successful.toMap, failed)
}

// 选举策略
private def electLeaderForPartition(
    tp: TopicPartition,
    isr: Seq[Int],
    replicas: Seq[Int]): (Int, Seq[Int]) = {

  val liveBrokers = controllerContext.liveBrokers

  // 优先 1: 从 ISR 中选存在且存活的 Broker
  val liveIsr = isr.filter(liveBrokers.contains)
  if (liveIsr.nonEmpty) {
    return (liveIsr.head, liveIsr)
  }

  // 优先 2 (unclean.leader.election.enable=true):
  //      从所有副本中选
  if (config.uncleanLeaderElectionEnable) {
    val liveReplicas = replicas.filter(liveBrokers.contains)
    if (liveReplicas.nonEmpty) {
      return (liveReplicas.head, liveReplicas)
    }
  }

  // 无可用 Leader
  (-1, Seq.empty)
}
```

> 文件位置: `core/src/main/scala/kafka/controller/PartitionStateMachine.scala:170`

# 4. ReplicaStateMachine 副本状态机

## 4.1 状态转换

```scala
// ReplicaStateMachine.scala

class ReplicaStateMachine(controller: KafkaController) extends Logging {

  sealed trait ReplicaState
  case object NewReplica extends ReplicaState
  case object OnlineReplica extends ReplicaState
  case object OfflineReplica extends ReplicaState
  case object ReplicaDeletionStarted extends ReplicaState
  case object ReplicaDeletionSuccessful extends ReplicaState
  case object NonExistentReplica extends ReplicaState

  private val replicaState: mutable.Map[PartitionAndReplica, ReplicaState] = ...

  def handleStateChanges(
      replicas: Set[PartitionAndReplica],
      targetState: ReplicaState,
      ...): Unit = {

    when(targetState) {
      case NewReplica =>
        // 通知 Broker 创建新副本（LeaderAndIsr 请求）
        replicas.foreach { replica =>
          replicaState.put(replica, NewReplica)
        }

      case OnlineReplica =>
        // Broker 上的副本已上线
        replicas.foreach { replica =>
          replicaState.put(replica, OnlineReplica)
        }

      case OfflineReplica =>
        // Broker 故障/下线 → 标记副本离线
        replicas.foreach { replica =>
          replicaState.put(replica, OfflineReplica)
        }

      case ReplicaDeletionStarted =>
        // 副本删除已发起（发送 StopReplica 请求）
      case ReplicaDeletionSuccessful =>
        // 副本已删除成功
      case NonExistentReplica =>
        // 副本删除完成，从缓存中移除
        replicas.foreach { replica =>
          replicaState.remove(replica)
        }
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/controller/ReplicaStateMachine.scala:61`

# 5. ControllerEventManager 事件处理

## 5.1 单线程事件模型

```scala
// ControllerEventManager.scala

class ControllerEventManager(controllerId: Int,
                               processor: ControllerEventProcessor,
                               ...) extends Logging {

  private val queue = new LinkedBlockingQueue[QueuedEvent]
  private val eventThread = new ControllerEventThread(name)

  def start(): Unit = eventThread.start()

  def put(event: ControllerEvent): Unit = {
    val queuedEvent = QueuedEvent(event, ...)
    queue.put(queuedEvent)
  }

  class ControllerEventThread(name: String) extends Thread(name) {
    override def run(): Unit = {
      while (isRunning) {
        // ① 从队列取事件
        val event = queue.take()

        try {
          // ② 事件分发（按类型）
          event.event match {
            // Broker 变更
            case BrokerChange(added, removed) =>
              processBrokerChange(added, removed)

            // Topic 变更
            case TopicChange(topics) =>
              processTopicChange(topics)

            // Topic 删除
            case TopicDeletion(topics) =>
              processTopicDeletion(topics)

            // 分区修改
            case PartitionModifications(topic) =>
              processPartitionModifications(topic)

            // 分区副本迁移
            case PartitionReassignment(reassignment) =>
              processPartitionReassignment(reassignment)

            // ISR 变更
            case IsrChangeNotification() =>
              processIsrChangeNotification()

            // LeaderAndIsr 响应
            case LeaderAndIsrResponseReceived(...) =>
              processLeaderAndIsrResponseReceived(...)

            // ...
          }
        } catch {
          case e: Throwable =>
            error("Error processing event", e)
        }
      }
    }
  }
}
```

> 文件位置: `core/src/main/scala/kafka/controller/ControllerEventManager.scala:40`

# 6. Broker 变更事件处理

## 6.1 Broker 下线的完整流程

```scala
// KafkaController.scala

private def processBrokerChange(added: Seq[Int], removed: Seq[Int]): Unit = {
  info(s"Broker change: added=$added, removed=$removed")

  val addedBrokers = added.filter(b => !controllerContext.liveBrokers.contains(b))
  val removedBrokers = removed.filter(b => controllerContext.liveBrokers.contains(b))

  // ① 更新控制器上下文
  if (addedBrokers.nonEmpty) {
    controllerContext.liveBrokers ++= addedBrokers
  }
  if (removedBrokers.nonEmpty) {
    controllerContext.liveBrokers --= removedBrokers
  }

  // ② 处理下线的 Broker
  if (removedBrokers.nonEmpty) {
    onBrokerFailure(removedBrokers)
  }

  // ③ 处理上线的 Broker
  if (addedBrokers.nonEmpty) {
    onBrokerStartup(addedBrokers)
  }
}

// Broker 故障处理
private def onBrokerFailure(deadBrokers: Seq[Int]): Unit = {
  // ① 获取这些 Broker 上的所有 Partition
  val affectedPartitions = controllerContext.partitionLeadershipInfo
    .filter { case (tp, leaderIsr) =>
      // 该 Broker 是 Leader 或位于 ISR 中的 Partition
      deadBrokers.contains(leaderIsr.leaderAndIsr.leader) ||
      leaderIsr.leaderAndIsr.isr.exists(deadBrokers.contains)
    }

  // ② 这些 Partition 转为 Offline
  partitionStateMachine.handleStateChanges(
    affectedPartitions.keys.toSet, OfflinePartition)

  // ③ 触发新的 Leader 选举 → 转为 Online
  partitionStateMachine.handleStateChanges(
    affectedPartitions.keys.toSet, OnlinePartition)

  // ④ ISR 收缩（故障 Broker 从 ISR 中移除）
  affectedPartitions.foreach { case (tp, _) =>
    val newIsr = ... // 从 ISR 中移除 dead brokers
    // 持久化新 ISR
  }
}
```

> 文件位置: `core/src/main/scala/kafka/controller/KafkaController.scala:549`

## 6.2 Broker 上线的处理

```scala
private def onBrokerStartup(newBrokers: Seq[Int]): Unit = {
  // ① 检查是否有 Partition 可以迁移到新 Broker
  // ② 触发 Preferred Leader Election
  // ③ 检查 Partition Reassignment 是否需要在新 Broker 上创建副本
}
```

# 7. LeaderAndIsr 请求处理

## 7.1 发送 LeaderAndIsr

```scala
// KafkaController.scala

private def sendLeaderAndIsrRequest(
    leaderAndIsr: Map[TopicPartition, LeaderIsrAndControllerEpoch],
    brokers: Seq[Int]): Unit = {

  // ① 按 Broker 分组 Partition
  val perBroker = leaderAndIsr
    .flatMap { case (tp, leaderIsr) =>
      leaderIsr.leaderAndIsr.isr.map(brokerId => (brokerId, tp, leaderIsr))
    }
    .groupBy(_._1)

  // ② 向每个 Broker 发送 LeaderAndIsr 请求
  perBroker.foreach { case (brokerId, partitions) =>
    val iterator = partitions.iterator
    val requestBuilder = new LeaderAndIsrRequest.Builder(
      controllerContext.epoch,
      controllerContext.brokerEpoch(brokerId),
      partitions.map { case (_, tp, leaderIsr) =>
        (tp, leaderIsr.leaderAndIsr)
      }.toMap,
      liveBrokers
    )

    // ③ 发送异步请求 + 注册响应回调
    controllerChannelManager.sendRequest(
      brokerId, requestBuilder,
      (response: LeaderAndIsrResponse) => {
        // ④ 处理响应 → 向事件队列发送 LeaderAndIsrResponseReceived
        controllerEventManager.put(
          LeaderAndIsrResponseReceived(...))
      }
    )
  }
}
```

> 文件位置: `core/src/main/scala/kafka/controller/KafkaController.scala:1024`

# 8. KRaft 模式下的 Controller

## 8.1 QuorumController（Kafka 3.3+）

```java
// QuorumController.java (KRaft 模式)

public class QuorumController implements Controller {
    private final RaftClient<ApiMessageAndVersion> raftClient;
    private final ClusterControlManager clusterControl;
    private final ReplicationControlManager replicationControl;
    private final FeatureControlManager featureControl;

    @Override
    public CompletableFuture<Void> activate() {
        // ① 从 Raft 日志加载当前元数据快照
        raftClient.loadSnapshot();
        // ② 处理自上次 checkpoint 以来的所有 metadata records
        // ③ 变为 Active Controller → 接受请求
    }

    // KRaft 中的 Leader 选举
    // Raft 协议自动处理 → 不需要 ZK
}
```

> 文件位置: `metadata/src/main/java/org/apache/kafka/controller/QuorumController.java:156`

# 9. Controller 源码中的关键路径

```
Controller 某个Broker下线 → 完整处理流程:

① ZK Watcher 触发
  zkClient.subscribeBrokerChangeListener.brokerChangeHandler.handleChildChange()
    → 检测到 /brokers/ids/{brokerId} 节点删除

② 放入事件队列
  controllerEventManager.put(BrokerChange(added = [], removed = [deadBroker]))

③ ControllerEventThread 处理事件
  processBrokerChange(removed = [deadBroker])
    → controllerContext.liveBrokers -= deadBroker
    → onBrokerFailure([deadBroker])

④ 获取受影响的 Partitions
  找出 Leader 是 deadBroker 的所有 Partition

⑤ 状态机转换
  partitionStateMachine.handleStateChanges(partitions, OfflinePartition)
    → 标记为 Offline
  partitionStateMachine.handleStateChanges(partitions, OnlinePartition)
    → 选举新 Leader

⑥ 选举新 Leader
  electLeaderForPartition(isr, replicas, liveBrokers)
    从 ISR 中选择存活的 Broker 为 Leader

⑦ 发送 LeaderAndIsr
  sendLeaderAndIsrRequest(...)
    → 各 Broker 收到后:
      Leader Broker: 初始化 Leader Partition
      Follower Broker: 初始化 Follower Partition → 从新 Leader 拉取

⑧ 发送 UpdateMetadata
  通知所有 Broker 更新元数据（Leader 变更）

⑨ Producer/Consumer 收到 NOT_LEADER_OR_FOLLOWER 错误
  → 刷新元数据 (Metadata Request)
  → 发现新 Leader → 恢复写入/消费
```
