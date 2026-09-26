# 1.版本控制（Versioning）

## 1.1版本控制的概念

MinIO 的版本控制是 Bucket 级别的功能。开启后，每个对象的每次 PUT 操作都会创建一个新版本，DELETE 操作会创建一个删除标记而非物理删除。

### 版本控制的状态

| 状态 | 说明 |
|------|------|
| `UNVERSIONED` | 默认状态，无版本控制。同名 PUT 直接覆盖，DELETE 物理删除 |
| `ENABLED` | 版本控制已开启且不可还原。所有操作保留历史版本 |
| `SUSPENDED` | 版本控制已开启但暂时挂起。新 PUT 覆盖 null 版本的对象，不再生成新版本 |

> **重要**：版本控制一旦开启（ENABLED），永远不能关闭。可以挂起（SUSPEND），但无法回到 UNVERSIONED。

## 1.2开启版本控制

```java
// Java SDK 设置版本控制
minioClient.setBucketVersioning(
    SetBucketVersioningArgs.builder()
        .bucket("my-bucket")
        .config(new VersioningConfiguration(
            VersioningConfiguration.Status.ENABLED,  // 开启
            false                                     // 不使用 MFA 删除
        ))
        .build()
);

// 挂起版本控制	
minioClient.setBucketVersioning(
    SetBucketVersioningArgs.builder()
        .bucket("my-bucket")
        .config(new VersioningConfiguration(
            VersioningConfiguration.Status.SUSPENDED,
            false
        ))
        .build()
);

// 获取版本控制状态
VersioningConfiguration config = minioClient.getBucketVersioning(
    GetBucketVersioningArgs.builder()
        .bucket("my-bucket")
        .build()
);
System.out.println("状态: " + config.status());
```

## 1.3版本控制下的对象操作

```java
// 上传对象（每次都生成新版本）
minioClient.putObject(PutObjectArgs.builder()
    .bucket("my-bucket")
    .object("data.txt")
    .stream(inputStream, size, -1)
    .build());
// 返回的响应中包含 versionId

// 列举对象的所有版本
Iterable<Result<Item>> versions = minioClient.listObjects(
    ListObjectsArgs.builder()
        .bucket("my-bucket")
        .prefix("data.txt")
        .includeVersions(true)    // 包含所有版本
        .build()
);

for (Result<Item> result : versions) {
    Item item = result.get();
    System.out.println(
        "Key: " + item.objectName() +
        " | Version: " + item.versionId() +
        " | IsLatest: " + item.isLatest() +
        " | IsDeleteMarker: " + item.isDeleteMarker() +
        " | Size: " + item.size()
    );
}

// 下载特定版本
try (InputStream stream = minioClient.getObject(
        GetObjectArgs.builder()
            .bucket("my-bucket")
            .object("data.txt")
            .versionId("specific-version-id")
            .build()
    )) {
    // 处理特定版本的数据
}

// 删除特定版本（物理删除）
minioClient.removeObject(
    RemoveObjectArgs.builder()
        .bucket("my-bucket")
        .object("data.txt")
        .versionId("old-version-id")
        .build()
);

// 删除对象（不指定版本ID = 创建删除标记）
minioClient.removeObject(
    RemoveObjectArgs.builder()
        .bucket("my-bucket")
        .object("data.txt")
        .build()
);
// 删除标记创建后，对象的最新版本变为不可见
// 可以删除这个删除标记来恢复对象
```

## 1.4版本控制下的存储开销

```text
对象 data.txt 的版本历史:

v1 (null version):  10MB   ← 开启版本控制前已存在
v2 (UUID-abc123):   12MB   ← 第一次修改
v3 (UUID-def456):    8MB   ← 第二次修改
v4 (UUID-ghi789):   DeleteMarker ← 被删除

总存储: 10MB + 12MB + 8MB + 标记元数据 = 约 30MB

实际消耗: 每次修改都保留完整副本（无增量/差异存储）
```

> **存储优化建议**：如果对象频繁修改，开启版本控制可能导致存储迅速膨胀。需要配合**生命周期规则**自动清理旧版本。

---

# 2.生命周期管理（Lifecycle / ILM）

## 2.1生命周期规则的概念

生命周期规则用于自动管理对象的生命周期，包括：
- **过期删除**：超过指定天数后自动删除对象
- **版本清理**：自动清理非当前版本的对象
- **删除标记清理**：自动清理无关联对象的历史删除标记
- **分层存储**：自动将对象转移到更低成本的存储层（如 Glacier）

## 2.2生命周期规则 XML 格式

MinIO 的生命周期规则通过 XML 定义（与 S3 兼容）：

```xml
<LifecycleConfiguration>
    <Rule>
        <ID>expire-old-logs</ID>
        <Status>Enabled</Status>
        <Filter>
            <Prefix>logs/</Prefix>
        </Filter>
        <!-- 当前版本对象过期 -->
        <Expiration>
            <Days>90</Days>
        </Expiration>
        <!-- 非当前版本对象过期 -->
        <NoncurrentVersionExpiration>
            <NoncurrentDays>30</NoncurrentDays>
        </NoncurrentVersionExpiration>
        <!-- 清理已过期的删除标记 -->
        <AbortIncompleteMultipartUpload>
            <DaysAfterInitiation>7</DaysAfterInitiation>
        </AbortIncompleteMultipartUpload>
    </Rule>
</LifecycleConfiguration>
```

## 2.3Java SDK 设置生命周期

```java
import io.minio.messages.*;
import io.minio.messages.lifecycle.*;

public class LifecycleConfigDemo {

    private final MinioClient minioClient;

    public LifecycleConfigDemo(MinioClient minioClient) {
        this.minioClient = minioClient;
    }

    public void configureLifecycle(String bucket) throws Exception {
        LifecycleRule rule1 = new LifecycleRule(
            Status.ENABLED,
            new AndOperator(
                "logs/",
                Map.of(
                    "environment", "production",
                    "retention", "short"
                )
            ),
            new Expiration((ZonedDateTime) null, 90, null, null),
            new NoncurrentVersionExpiration(30, null),
            null,  // abort incomplete multipart upload
            null,
            "expire-logs-rule"
        );

        // 清理未完成的分片上传（防止存储泄漏）
        LifecycleRule rule2 = new LifecycleRule(
            Status.ENABLED,
            null,
            null,
            null,
            new AbortIncompleteMultipartUpload(7),
            null,
            "abort-incomplete-uploads"
        );

        LifecycleConfiguration config = new LifecycleConfiguration(
            List.of(rule1, rule2)
        );

        minioClient.setBucketLifecycle(
            SetBucketLifecycleArgs.builder()
                .bucket(bucket)
                .config(config)
                .build()
        );

        System.out.println("生命周期规则已设置");
    }

    // 获取当前生命周期配置
    public void getLifecycleConfig(String bucket) throws Exception {
        LifecycleConfiguration config = minioClient.getBucketLifecycle(
            GetBucketLifecycleArgs.builder().bucket(bucket).build()
        );

        for (LifecycleRule rule : config.rules()) {
            System.out.println("规则: " + rule.id() + " - " + rule.status());
        }
    }

    // 删除所有生命周期规则
    public void deleteLifecycleConfig(String bucket) throws Exception {
        minioClient.deleteBucketLifecycle(
            DeleteBucketLifecycleArgs.builder().bucket(bucket).build()
        );
    }
}
```

## 2.4生命周期规则类型速查

| 规则类型 | 适用对象 | 说明 |
|----------|----------|------|
| `Expiration` | 当前版本 | 当前版本对象在指定天数后过期删除 |
| `NoncurrentVersionExpiration` | 非当前版本 | 非当前版本在指定天数后过期删除 |
| `AbortIncompleteMultipartUpload` | 分片上传 | 指定天数后清理未完成的分片上传（防止存储泄漏） |
| `ExpiredObjectDeleteMarker` | 删除标记 | 清理已无关联对象的删除标记 |
| `Transition` | 当前版本 | 将对象转移到其他存储类（MinIO 当前支持有限） |
| `NoncurrentVersionTransition` | 非当前版本 | 将非当前版本转移到其他存储类 |

## 2.5生命周期规则设计建议

```text
Bucket: my-app-bucket
版本控制: ENABLED

规则1: 清理临时文件
  前缀: tmp/
  当前版本过期: 7 天
  非当前版本过期: 3 天

规则2: 清理旧版本日志
  前缀: logs/
  当前版本过期: 90 天
  非当前版本过期: 30 天
  清理过期删除标记: 开启

规则3: 清理未完成分片
  前缀: （所有）
  取消未完成分片上传: 7 天
```

---

# 3.对象锁（Object Lock / WORM）

## 3.1对象锁概述

对象锁提供 WORM（Write Once Read Many）保护，防止对象被删除或覆盖。主要用于：
- 合规性要求（如 SEC 17a-4、HIPAA、GDPR）
- 防勒索软件保护（不可篡改的备份）
- 法律证据保存

### 两种保留模式

| 模式 | 说明 | 谁能解除 |
|------|------|----------|
| **Governance** (治理模式) | 大多数用户无法修改/删除 | 具有 `bypassGovernanceRetention` 权限的用户可解除 |
| **Compliance** (合规模式) | 任何人都无法修改/删除（包括 root 用户） | 无人能解除（保留到设置的日期） |

### 依法保留（Legal Hold）

类似于 Governance 模式但无到期日，完全冻结对象，直到明确移除 `Legal Hold`。

## 3.2启用对象锁

对象锁需要在**创建 Bucket 时**启用，且**不可逆**：

```java
// 创建启用对象锁的 Bucket
minioClient.makeBucket(
    MakeBucketArgs.builder()
        .bucket("locked-bucket")
        .objectLock(true)  // 关键参数
        .build()
);
```

## 3.3设置默认保留策略（Bucket 级别）

```java
import io.minio.messages.ObjectLockConfiguration;
import io.minio.messages.RetentionDurationDays;
import io.minio.messages.RetentionMode;

// 设置 Bucket 的默认保留策略
// 所有未显式指定保留期限的对象将使用此默认值
minioClient.setObjectLockConfiguration(
    SetObjectLockConfigurationArgs.builder()
        .bucket("locked-bucket")
        .config(new ObjectLockConfiguration(
            RetentionMode.COMPLIANCE,     // 默认合规模式
            new RetentionDurationDays(2555) // 默认 7 年 (2555天)
        ))
        .build()
);
```

## 3.4对象级保留设置

```java
import io.minio.messages.Retention;

// 上传对象时设置保留期限
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("locked-bucket")
        .object("contracts/agreement-2024.pdf")
        .stream(inputStream, fileSize, -1)
        
        // 设置保留到指定日期
        .retention(Retention.create(
            RetentionMode.COMPLIANCE,
            ZonedDateTime.now().plusYears(7)  // 保留 7 年
        ))
        
        // 设置依法保留
        .legalHold(true)
        
        .build()
);

// 单独设置依法保留
minioClient.setObjectLegalHold(
    SetObjectLegalHoldArgs.builder()
        .bucket("locked-bucket")
        .object("evidence/file.txt")
        .legalHold(true)   // 锁定
        .build()
);

// 移除依法保留（仅 Governance 模式或有权限的用户）
minioClient.setObjectLegalHold(
    SetObjectLegalHoldArgs.builder()
        .bucket("locked-bucket")
        .object("evidence/file.txt")
        .legalHold(false)  // 解除
        .build()
);

// 延长保留期限（只能延长，不能缩短）
minioClient.setObjectRetention(
    SetObjectRetentionArgs.builder()
        .bucket("locked-bucket")
        .object("contracts/agreement-2024.pdf")
        .retention(Retention.create(
            RetentionMode.COMPLIANCE,
            ZonedDateTime.now().plusYears(10)  // 延长到 10 年
        ))
        .build()
);

// 获取对象的保留信息
io.minio.messages.Retention retention = minioClient.getObjectRetention(
    GetObjectRetentionArgs.builder()
        .bucket("locked-bucket")
        .object("contracts/agreement-2024.pdf")
        .build()
);
System.out.println("保留模式: " + retention.mode());
System.out.println("保留至: " + retention.retainUntilDate());
```

---

# 4.Bucket 复制（Replication）

## 4.1复制概述

Bucket 复制用于将对象从一个桶自动同步到另一个桶（同集群或跨集群）。MinIO 支持两种复制：

| 复制类型 | 说明 | 版本控制要求 |
|----------|------|-------------|
| **Bucket Replication** | 桶级别的对象实时同步 | 源和目标桶都必须开启版本控制 |
| **Site Replication** | 站点级别的全量同步（包括 IAM、Bucket、对象） | 需要先配置 Site Replication |

## 4.2同集群 Bucket 复制配置

```java
import io.minio.messages.*;
import io.minio.messages.lifecycle.*;

public class ReplicationDemo {

    private final MinioClient minioClient;

    public ReplicationDemo(MinioClient minioClient) {
        this.minioClient = minioClient;
    }

    public void setupReplication(String sourceBucket, String targetBucket) 
            throws Exception {
        // 前提：源和目标桶都开启了版本控制
        minioClient.setBucketVersioning(SetBucketVersioningArgs.builder()
            .bucket(sourceBucket)
            .config(new VersioningConfiguration(
                VersioningConfiguration.Status.ENABLED, false))
            .build());

        minioClient.setBucketVersioning(SetBucketVersioningArgs.builder()
            .bucket(targetBucket)
            .config(new VersioningConfiguration(
                VersioningConfiguration.Status.ENABLED, false))
            .build());

        // 创建复制规则
        ReplicationRule rule = new ReplicationRule(
            new DeleteMarkerReplication(Status.ENABLED),
            new ReplicationDestination(
                null,           // destination bucket ARN (minio 可为空)
                targetBucket,   // 目标桶
                null, null, null, null, null, null, null,
                null, null     // encryption 和 storage class
            ),
            null,              // existing object replication
            new RuleFilter(new AndOperator("important/", null)),
            "replication-rule-1",   // rule id
            null,              // prefix (已使用 filter)
            1,                 // priority
            Status.ENABLED,
            new DeleteReplication(Status.ENABLED),  // 同步删除操作
            null,
            null, // existing object replication
            null, null
        );

        // 创建复制配置
        ReplicationConfiguration config = new ReplicationConfiguration(
            "ReplicationRole",  // IAM role ARN (minio 可为空)
            List.of(rule)
        );

        minioClient.setBucketReplication(
            SetBucketReplicationArgs.builder()
                .bucket(sourceBucket)
                .config(config)
                .build()
        );
    }
}
```

## 4.3复制最佳实践

```text
场景1: 同集群备份
  源桶: my-data
  目标桶: my-data-backup
  目的: 防止误删、逻辑错误
  配置: 全量复制 + 删除同步(开启)

场景2: 跨区域灾备
  源集群: dc-primary (北京)
  目标集群: dc-dr (上海)
  目的: 灾难恢复
  配置: 全量复制 + 不开启删除同步(源端删除时目标保留)

场景3: 数据分层
  源桶: hot-data (高性能存储)
  目标桶: archive-bucket (低成本存储)
  目的: 30天前的数据自动迁移到低成本存储
  配置: 只复制特定前缀 + 生命周期自动清理源端
```

---

# 5.事件通知（Bucket Notifications）

## 5.1事件通知概述

MinIO 支持在对象发生特定事件时，将通知推送到各种目标系统。

### 支持的事件类型

```text
对象事件:
  s3:ObjectCreated:Put         新对象创建 (PUT)
  s3:ObjectCreated:Post        新对象创建 (POST表单上传)
  s3:ObjectCreated:Copy        新对象创建 (COPY)
  s3:ObjectCreated:CompleteMultipartUpload  分片上传完成
  s3:ObjectRemoved:Delete      对象被删除
  s3:ObjectRemoved:DeleteMarkerCreated  删除标记被创建
  s3:ObjectAccessed:Get        对象被访问（慎用，性能开销大）

复制事件:
  s3:Replication:OperationCompletedReplication  复制完成
  s3:Replication:OperationFailedReplication     复制失败

ILM 事件:
  s3:ObjectRestore:Post        对象从 Glacier 恢复
  s3:ObjectRestore:Completed   恢复完成
```

### 支持的通知目标

| 目标类型 | 说明 | 典型场景 |
|----------|------|----------|
| Webhook | HTTP POST 到指定 URL | 触发业务处理（缩略图生成、病毒扫描） |
| Kafka | 推送到 Kafka Topic | 数据管道、流处理、RocketMQ 替代 |
| AMQP | RabbitMQ 等消息队列 | 异步任务处理 |
| MQTT | IoT 消息协议 | 边缘设备通知 |
| NATS | 轻量级消息系统 | 云原生微服务通知 |
| Redis | Redis Pub/Sub | 实时通知到应用层 |
| MySQL/PostgreSQL | 关系型数据库 | 同步对象元数据到业务数据库 |
| Elasticsearch | 搜索引擎 | 实时索引对象元数据 |

## 5.2配置事件通知（Java SDK + mc CLI）

事件通知的配置目前主要通过 `mc` 命令行工具完成。SDK 虽然也支持但较为繁琐。

### 通过 mc 配置 Webhook 通知

```bash
# 设置 Webhook 通知
mc event add myminio/mybucket \
  arn:minio:sqs::PRIMARY:webhook \
  --event "put,delete" \
  --prefix "uploads/" \
  --suffix ".jpg"

# 查看通知配置
mc event list myminio/mybucket

# 删除通知配置
mc event remove myminio/mybucket arn:minio:sqs::PRIMARY:webhook

# 通过环境变量配置 Webhook 目标
# MINIO_NOTIFY_WEBHOOK_ENABLE_PRIMARY=on
# MINIO_NOTIFY_WEBHOOK_ENDPOINT_PRIMARY=http://my-app:8080/minio/events
```

### 配置 Kafka 通知

```bash
# 通过环境变量配置（在 minio server 启动时设置）
export MINIO_NOTIFY_KAFKA_ENABLE_PRIMARY=on
export MINIO_NOTIFY_KAFKA_BROKERS_PRIMARY=kafka1:9092,kafka2:9092
export MINIO_NOTIFY_KAFKA_TOPIC_PRIMARY=minio-events

# 然后通过 mc 配置事件
mc event add myminio/mybucket arn:minio:sqs::PRIMARY:kafka \
  --event "put,delete,get"
```

## 5.3接收 Webhook 通知的 Spring Boot 端点

```java
@RestController
@RequestMapping("/minio")
@Slf4j
public class MinioEventController {

    /**
     * 接收 MinIO 事件通知
     */
    @PostMapping("/events")
    public ResponseEntity<String> handleEvent(@RequestBody String eventJson) {
        // MinIO 发送的是 JSON 数组（可能一次发送多个事件）
        // 格式: {"Records": [{...}, {...}]}
        
        try {
            JsonNode root = objectMapper.readTree(eventJson);
            JsonNode records = root.get("Records");

            if (records != null && records.isArray()) {
                for (JsonNode record : records) {
                    String eventName = record.get("eventName").asText();
                    String bucketName = record.get("s3").get("bucket").get("name").asText();
                    String objectKey = record.get("s3").get("object").get("key").asText();
                    long objectSize = record.get("s3").get("object").get("size").asLong();
                    String etag = record.get("s3").get("object").get("eTag").asText();

                    log.info("事件: {} | Bucket: {} | Key: {} | Size: {}",
                        eventName, bucketName, objectKey, objectSize);

                    // 根据事件类型执行不同操作
                    switch (eventName) {
                        case "s3:ObjectCreated:Put" ->
                            handleObjectCreated(bucketName, objectKey, objectSize);
                        case "s3:ObjectRemoved:Delete" ->
                            handleObjectDeleted(bucketName, objectKey);
                    }
                }
            }
        } catch (Exception e) {
            log.error("处理 MinIO 事件失败", e);
            return ResponseEntity.internalServerError().body("处理失败");
        }

        // 必须返回 200，否则 MinIO 会重试
        return ResponseEntity.ok("OK");
    }

    @Async
    public void handleObjectCreated(String bucket, String key, long size) {
        log.info("新对象: {}/{} ({} bytes)", bucket, key, size);
        // 触发后续处理：病毒扫描、缩略图生成、元数据索引等
    }

    @Async
    public void handleObjectDeleted(String bucket, String key) {
        log.info("对象已删除: {}/{}", bucket, key);
        // 清理关联资源：数据库记录、缓存等
    }
}
```

## 5.4事件通知的 JSON 格式

```json
{
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "minio:s3",
      "awsRegion": "",
      "eventTime": "2024-01-15T10:30:00Z",
      "eventName": "s3:ObjectCreated:Put",
      "userIdentity": {
        "principalId": "minioadmin"
      },
      "requestParameters": {
        "sourceIPAddress": "192.168.1.100"
      },
      "responseElements": {
        "x-amz-request-id": "17A3F4B5E6C8D9E0",
        "x-minio-origin-endpoint": "http://minio1:9000"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "Config",
        "bucket": {
          "name": "my-bucket",
          "ownerIdentity": { "principalId": "minioadmin" },
          "arn": "arn:aws:s3:::my-bucket"
        },
        "object": {
          "key": "photos/avatar.jpg",
          "size": 1048576,
          "eTag": "abc123def456",
          "contentType": "image/jpeg",
          "userMetadata": {
            "content-type": "image/jpeg",
            "X-Amz-Meta-Author": "john"
          },
          "sequencer": "17A3F4B5E6C8D9E0"
        }
      }
    }
  ]
}
```

---

# 6.服务端加密（SSE）

## 6.1三种加密方式对比

| 加密方式 | 密钥管理 | 客户端透明 | 适用场景 |
|----------|----------|------------|----------|
| **SSE-S3** | MinIO 管理密钥 | 是 | 一般加密需求，最简单 |
| **SSE-C** | 客户端提供密钥 | 否（每次请求需传密钥） | 客户端完全控制密钥 |
| **SSE-KMS** | KES + Vault 管理密钥 | 是 | 企业级密钥管理、合规审计 |

### SSE-S3 加密原理

```text
上传流程：
1. 客户端上传明文数据 → MinIO
2. MinIO 生成一个随机的 Data Key
3. MinIO 用 Data Key 加密对象数据（AES-256-GCM）
4. MinIO 用 Master Key 加密 Data Key，附加到对象元数据中
5. 加密后的对象数据 + 加密后的 Data Key 一起存储
6. 元数据中标记: x-amz-server-side-encryption: AES256

下载流程：
1. 客户端请求对象 → MinIO
2. MinIO 从元数据中提取加密的 Data Key
3. MinIO 用 Master Key 解密 Data Key
4. MinIO 用 Data Key 解密对象数据
5. 返回明文给客户端
```

## 6.2使用 SSE-S3 加密

```java
import io.minio.ServerSideEncryption;

// 上传时指定 SSE-S3
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("encrypted-file.bin")
        .stream(inputStream, fileSize, -1)
        .sse(ServerSideEncryption.atMinio())  // SSE-S3
        .build()
);

// 下载时无需任何额外参数，MinIO 自动解密
try (InputStream stream = minioClient.getObject(
        GetObjectArgs.builder()
            .bucket("my-bucket")
            .object("encrypted-file.bin")
            .build()
    )) {
    // stream 内容已被自动解密
}
```

## 6.3使用 SSE-C 加密

```java
import io.minio.ServerSideEncryptionCustomerKey;

// 生成 256 位（32 字节）密钥
String customerKey = "my-secret-key-must-be-32-bytes!";
byte[] keyBytes = customerKey.getBytes(StandardCharsets.UTF_8);

ServerSideEncryptionCustomerKey sseC = 
    new ServerSideEncryptionCustomerKey(keyBytes);

// 上传时提供密钥
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("sse-c-file.bin")
        .stream(inputStream, fileSize, -1)
        .sse(sseC)
        .build()
);

// 下载时必须提供相同的密钥
try (InputStream stream = minioClient.getObject(
        GetObjectArgs.builder()
            .bucket("my-bucket")
            .object("sse-c-file.bin")
            .ssec(sseC)   // 必须提供相同的密钥
            .build()
    )) {
    // 如果密钥不匹配，会返回 403 错误
}
```

## 6.4设置 Bucket 默认加密

```java
import io.minio.messages.SseConfiguration;

// 设置 Bucket 级别的默认加密（SSE-S3）
SseConfiguration sseConfig = new SseConfiguration(
    new SseConfigurationRule(
        new SseConfigurationRule.ApplySseByDefault(ServerSideEncryption.atMinio())
    )
);

minioClient.setBucketEncryption(
    SetBucketEncryptionArgs.builder()
        .bucket("my-bucket")
        .config(sseConfig)
        .build()
);

// 获取加密配置
SseConfiguration config = minioClient.getBucketEncryption(
    GetBucketEncryptionArgs.builder()
        .bucket("my-bucket")
        .build()
);

// 删除默认加密配置
minioClient.deleteBucketEncryption(
    DeleteBucketEncryptionArgs.builder()
        .bucket("my-bucket")
        .build()
);
```

---

# 7.多站点复制（Site Replication）

## 7.1Site Replication 与 Bucket Replication 的区别

| 维度 | Site Replication | Bucket Replication |
|------|-----------------|-------------------|
| 同步范围 | IAM 策略 + Bucket + 对象 | 仅指定 Bucket 的对象 |
| 配置方式 | mc admin + K8s Operator | mc / SDK |
| 复制模式 | Active-Active 或 Active-Passive | Active-Passive |
| 节点要求 | 至少 3 个独立 MinIO 集群 | 源和目标集群均可 |
| 使用场景 | 多数据中心全局统一管理 | 特定桶的数据同步备份 |

## 7.2配置 Site Replication

```bash
# 在集群 A 上创建站点复制别名
mc alias set cluster-a http://cluster-a.example.com:9000 \
  ${ACCESS_KEY} ${SECRET_KEY}
mc alias set cluster-b http://cluster-b.example.com:9000 \
  ${ACCESS_KEY} ${SECRET_KEY}

# 配置双向站点复制
mc admin replicate add cluster-a cluster-b --mode "full"

# 查看站点复制状态
mc admin replicate status cluster-a

# 移除站点复制
mc admin replicate remove cluster-a cluster-b
```

---

# 8.分层存储（Tiering）

## 8.1Tiering 概述

MinIO 支持将低频访问的数据自动迁移到远程存储层，如 AWS S3、Azure Blob、GCS 等。这通过生命周期 Transition 规则实现。

```bash
# 添加远程存储层
mc ilm tier add minio myminio S3TIER \
  --endpoint https://s3.amazonaws.com \
  --access-key ${AWS_ACCESS_KEY} \
  --secret-key ${AWS_SECRET_KEY} \
  --bucket my-remote-bucket \
  --prefix minio-transition/ \
  --storage-class STANDARD_IA
```

```java
// 过渡规则：30天后转移到远程层
Transition transition = new Transition(
    ZonedDateTime.now().plusDays(30),   // 或直接指定天数
    null,                                // date
    30,                                  // days
    "S3TIER"                            // 目标存储层
);

// 应用到生命周期规则中
LifecycleRule rule = new LifecycleRule(
    Status.ENABLED,
    new AndOperator("archive/", null),
    null,    // expiration
    null,    // noncurrent version expiration
    null,    // abort incomplete
    transition,
    "tier-to-s3"
);
```

> **注意**：分层存储是 MinIO 商业版全功能的一部分，开源版中此功能有受限。本地开发验证建议使用 MinIO 开源版的基本功能。
