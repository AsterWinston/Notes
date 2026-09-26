# 1.认识MinIO

## 1.1存储系统基础概念

在理解 MinIO 之前，需要先理解三种基本存储形态：

### 块存储（Block Storage）
块存储将存储设备上的原始存储空间划分为大小固定的块（Block），每块有唯一地址，操作系统直接读写块设备。

- 代表产品：SAN (Storage Area Network)、iSCSI、AWS EBS、Ceph RBD
- 特点：低延迟（微秒~毫秒级）、随机读写能力强、适合数据库和虚拟机磁盘
- 局限：不支持多机共享（一个块设备通常只能挂载到一台主机）、扩展复杂、无文件语义

```text
应用
 ↓ (SCSI/iSCSI/FC协议)
块设备 (/dev/sdb)
 ↓
磁盘阵列 (RAID)
```

### 文件存储（File Storage）
在块存储之上叠加文件系统层，以目录树和文件的形式组织数据，通过 NFS/CIFS 等网络协议共享。

- 代表产品：NFS、NAS、CephFS、AWS EFS、GlusterFS
- 特点：支持文件级共享、目录结构、POSIX文件权限（rwx）
- 局限：随着文件数量增长，元数据管理成为瓶颈（如目录含有百万文件时 ls 极慢）；扩展性受限于目录树结构

```text
应用
 ↓ (NFS/CIFS协议)
文件系统 (ext4/xfs/zfs)
 ↓
块设备
 ↓
磁盘阵列
```

### 对象存储（Object Storage）
对象存储摒弃了传统的目录树结构，将所有数据作为"对象"管理在扁平的命名空间（Bucket）中。每个对象包含：
- **数据本身**（任意二进制数据）
- **元数据**（自定义属性，如 Content-Type、标签等）
- **全局唯一标识符**（对象名/Key）

- 代表产品：AWS S3、MinIO、Ceph RGW (RADOS Gateway)、OpenStack Swift、Google Cloud Storage、Azure Blob Storage
- 特点：海量扩展（EB级）、平面命名空间（无目录层级限制）、RESTful API、强最终一致性、内置冗余与加密
- 局限：不适合需要 POSIX 语义（如文件锁、Append写）或极低延迟随机读写的场景

```text
应用
 ↓ (HTTP REST API - S3协议)
对象存储网关 (MinIO Server)
 ↓
Erasure Coding / 分布式块存储
 ↓
物理磁盘 (JBOD，无需RAID)
```

### 三种存储形态对比

| 维度 | 块存储 | 文件存储 | 对象存储 |
|------|--------|----------|----------|
| 数据组织 | 固定大小块 | 目录+文件 | Bucket + Object (Key-Value) |
| 访问协议 | SCSI / iSCSI / FC | NFS / SMB / CIFS | HTTP REST (S3 API) |
| 一致性 | 强一致 | 强一致 | 最终一致 / 强一致 |
| 扩展能力 | TB~PB（垂直为主） | PB级（受元数据限制） | EB级（水平扩展） |
| 典型延迟 | 微秒~毫秒 | 毫秒 | 毫秒~秒 |
| 元数据规模 | N/A | 百万~千万文件 | 千亿级对象 |
| 共享访问 | 单机（除非集群文件系统） | 多客户端 | 多客户端（HTTP） |
| 代表场景 | 数据库、虚拟机磁盘 | 企业共享目录、日志 | 影像存储、备份归档、数据湖、静态网站托管 |

---

## 1.2什么是MinIO

MinIO 是一个高性能、Kubernetes 原生、S3 兼容的对象存储系统，使用 Go 语言编写，采用 Apache License 2.0 开源协议（AGPLv3 用于商业再分发）。

### 设计哲学
MinIO 的设计理念是 **"简单即高效"**：
- 单个静态二进制文件，无外部依赖，部署只需一条命令
- 不依赖外部数据库存储元数据（元数据与对象数据一同存储在内置的 xl.meta 中）
- 通过 Erasure Coding（纠删码）替代 RAID，在软件层面实现数据保护
- 原生支持 S3 API，与 AWS S3 SDK 完全兼容，降低迁移成本

### 创始人背景
MinIO 由 Garima Kapoor、Anand Babu Periasamy 和 Harshavardhana（社区通称"harsha"）于 2014 年创立。核心团队曾参与 GlusterFS 项目开发，在分布式存储领域拥有深厚积累。创始团队总结了 GlusterFS 的复杂性问题，决定打造一个极致简单的对象存储系统。

### 核心特性一览

| 特性 | 说明 |
|------|------|
| S3 API 兼容 | 完全兼容 AWS S3 v2/v4 签名，现有 S3 工具/SDK 可直接接入 |
| Erasure Coding | 以对象级纠删码替代 RAID，支持 n/2 块磁盘故障容忍（默认配置） |
| Bit Rot Protection | 以每条 64MB 为单位进行数据块哈希校验，自动检测静默数据损坏 |
| 高性能 | 单机 GET 可达 10GB/s+（NVMe 存储），PUT 可达 5GB/s+ |
| 无单点故障 | 分布式模式下任一节点/磁盘故障不影响整体服务（N/2 容量容忍） |
| 加密 | 支持 SSE-S3、SSE-C、SSE-KMS（与 HashiCorp Vault 集成） |
| 身份管理 | 内置 IAM，支持 OpenID Connect (OIDC) 和 LDAP/AD |
| 不可变存储 | 支持 WORM (Write Once Read Many) 对象锁定（合规/监管模式） |
| 版本控制 | Bucket 级对象版本管理，支持删除标记 |
| 生命周期 | ILM (Information Lifecycle Management) 自动过期/转换存储层 |
| 事件通知 | 支持 Webhook、Kafka、NATS、MySQL、PostgreSQL、Redis、Elasticsearch、AMQP 等 |
| 站点复制 | 多站点同步复制，支持 active-active/active-passive |
| 批量复制 | Bucket 级实时复制到远程 MinIO 集群 |
| 轻量级 | 单个 ~100MB 静态二进制，极低内存占用 |
| Kubernetes 原生 | 提供 Operator、Console 管理 UI、Tenant CRD |

---

## 1.3MinIO 与同类产品的对比

### MinIO vs AWS S3

| 维度 | MinIO | AWS S3 |
|------|-------|--------|
| 部署方式 | 私有化部署（DC / 边缘 / 混合云） | 公有云托管服务 |
| 延时 | 极低（局域网/同机房，毫秒级） | 相对较高（公网传输，数十~数百毫秒） |
| 数据主权 | 完全自主可控 | 数据存储在 AWS 数据中心 |
| 带宽成本 | 内网免费 | 公网上传/下载按流量收费 |
| 容量上限 | 硬件限制（EB 级扩展） | 理论上无限 |
| 生态集成 | 原生 Kubernetes Operator | AWS 全生态（Lambda、Athena、Glacier 等） |
| 成本模型 | 一次性硬件 + 运维成本 | 按使用量付费（存储 + 请求 + 流量） |
| S3 API 兼容度 | 高（核心 API 100% 兼容） | 100%（官方实现） |
| 高级功能 | 持续追赶（对象锁、复制、生命周期已成熟） | 最全面（Intelligent-Tiering、Transfer Acceleration 等） |

**选型建议**：MinIO 适合对数据主权、低延迟、高吞吐有要求的私有云/混合云/边缘场景，或需要控制成本的场景。AWS S3 适合完全不需要管理基础设施的全托管场景。

### MinIO vs Ceph (RADOS Gateway)

| 维度 | MinIO | Ceph RGW |
|------|-------|----------|
| 定位 | 纯对象存储 | 统一存储（块/文件/对象） |
| 架构复杂度 | 极简（单二进制） | 极其复杂（MON/OSD/MGR/MDS/RGW） |
| 部署难度 | 低（分钟级） | 高（需要专业运维团队） |
| 性能（对象存储） | 高（Go 语言优化，无中间层） | 中（经 Ceph 中间层转译） |
| 扩展性 | 纯水平扩展（Server Pool） | 水平扩展（但受 CRUSH 算法复杂约束） |
| 元数据管理 | 内置（xl.meta，无外部依赖） | MON 集群管理 |
| 小文件性能 | 极优（设计目标之一） | 一般（Ceph 更适合大文件） |
| 社区活跃度 | 高（GitHub 40K+ Stars） | 高（OpenStack 基金会托管） |

**选型建议**：如果只需要对象存储，MinIO 的部署和维护成本远低于 Ceph。如果需要块+文件+对象统一存储，Ceph 是更好的选择。

### MinIO vs OpenStack Swift

| 维度 | MinIO | Swift |
|------|-------|------|
| S3 兼容性 | 原生 S3 API | 需通过 Swift3 中间件 |
| 性能 | 更高（Go 语言，直接 I/O） | 较低（Python 代理层） |
| 部署复杂度 | 极低 | 中等（需 Keystone 认证等） |
| 生态 | 独立生态，K8s 原生 | OpenStack 生态 |
| 社区趋势 | 持续增长 | 逐渐衰退（被 Ceph RGW 替代） |

### MinIO vs SeaweedFS

| 维度 | MinIO | SeaweedFS |
|------|-------|-----------|
| 定位 | 纯对象存储 | 对象 + 文件存储 |
| S3 兼容 | 核心设计目标 | 附加功能（S3 网关模式） |
| 文件系统挂载 | 不支持（需 s3fs-fuse 等第三方） | 原生支持 FUSE 挂载 |
| 小文件场景 | 优秀（Erase Coding 优化） | 优秀（Volume Server 设计） |
| 元数据 | 内嵌（xl.meta） | 外部（Filer 模式需 MySQL/Redis 等） |

---

## 1.4MinIO 架构概览

### 整体架构
MinIO 采用**无共享（Shared Nothing）**架构，每个节点都是对等的，不存在独立的元数据服务器。这种架构消除了单点瓶颈和单点故障。

```text
┌─────────────────────────────────────────────────┐
│                   S3 客户端                      │
│    (aws-sdk, mc, s3cmd, rclone, Cyberduck...)   │
└────────┬──────────┬──────────┬──────────────────┘
         │ HTTP(S)  │          │
         ▼          ▼          ▼
┌─────────────────────────────────────────────────┐
│               负载均衡 (Nginx/HAProxy)            │
│              (可选，但强烈推荐)                    │
└────────┬──────────┬──────────┬──────────────────┘
         │          │          │
         ▼          ▼          ▼
   ┌─────────┐┌─────────┐┌─────────┐
   │ MinIO   ││ MinIO   ││ MinIO   │  ... (Server Pool 0)
   │ Server 1││ Server 2││ Server N│
   │ /data1  ││ /data1  ││ /data1  │
   │ /data2  ││ /data2  ││ /data2  │
   └─────────┘└─────────┘└─────────┘
         │          │          │
         └──────────┼──────────┘
              内部 Gossip 协议
         (节点发现、状态同步、故障检测)
```

### 组件说明

| 组件 | 职责 |
|------|------|
| MinIO Server | 核心服务进程，处理所有 S3 请求、管理数据分布、执行纠删码编解码。单二进制文件 `minio`（~100MB）。 |
| MinIO Client (mc) | 命令行管理工具，提供 `mc ls`、`mc cp`、`mc mirror`、`mc admin` 等命令。对标 AWS CLI。 |
| MinIO Console | Web 管理控制台，提供图形化 Bucket 管理、用户/策略配置、监控面板。 |
| MinIO Operator | Kubernetes Operator，管理 MinIO Tenant（租户）的生命周期（创建、扩缩容、升级、备份）。 |
| KES | Key Encryption Service，用于 SSE-KMS 场景的密钥管理服务，支持 HashiCorp Vault 等后端。 |

### Server Pool（服务器池）概念

从 MinIO RELEASE.2021-04-22T15-44-28Z 开始，引入 Server Pool 概念：

- 一个 MinIO 集群可以由多个 Server Pool 组成
- 每个 Pool 是一组独立的 Erasure Coding Set
- 不同 Pool 的节点数、磁盘数可以不同（通常是相同的）
- 扩容通过添加新 Pool 实现（旧 Pool 不参与重平衡）

```text
Cluster
├── Pool 0: 4 nodes × 4 drives = 16 drives (EC: 8+8)
├── Pool 1: 4 nodes × 4 drives = 16 drives (EC: 8+8)  ← 扩容
└── Pool 2: 8 nodes × 2 drives = 16 drives (EC: 8+8)  ← 再次扩容
```

### 数据写入流程（简化）

```text
客户端 PUT /bucket/object
       │
       ▼
  任意 MinIO 节点（通过负载均衡路由）
       │
       ▼
  计算哈希 → 确定对象归属的 EC Set
       │
       ▼
  将对象按 EC 条带大小（默认 64MB）分块
       │
       ▼
  对每块数据进行 EC 编码：数据块 + 奇偶校验块
       │
       ▼
  将编码后的分片写入对应节点/磁盘（并行写入）
       │
       ▼
  等待 N/2+1 个写入确认 → 返回成功
```

### 数据读取流程（简化）

```text
客户端 GET /bucket/object
       │
       ▼
  任意 MinIO 节点
       │
       ▼
  查询对象元数据（xl.meta）→ 确定分片位置
       │
       ▼
  从各节点并行读取数据分片（只需读取 N/2 个分片即可恢复）
       │
       ▼
  EC 解码 → 重组原始数据
       │
       ▼
  流式返回给客户端
```

---

## 1.5MinIO版本历史

| 版本 | 发布时间 | 关键特性 |
|------|----------|----------|
| 2016 早期版本 | 2016 年 | 初始开源版本，支持单机模式、Erasure Coding、S3 API 基础兼容 |
| 2017 版本 | 2017 年 | 分布式模式稳定、S3 V4 签名、Bucket 通知基础支持 |
| 2018 版本 | 2018 年 | 对象锁 (Object Lock)、生命周期管理、联邦模式 (Gateway 模式) |
| 2019 版本 | 2019 年 | IAM 增强、TLS 配置简化、Prometheus 指标暴露 |
| 2020 版本 | 2020 年 | 支持 LDAP/OIDC、KMS/KES 集成、复制 (Replication) |
| 2021.04 | 2021 年 4 月 | **Server Pool 架构**引入，支持多 Pool 扩容 |
| 2022 版本 | 2022 年 | Site Replication（站点复制）、Tiering（分层存储）、Console 重大改进 |
| 2023 版本 | 2023 年 | Bulk Replication、对象过期增强、性能持续优化、CRD Operator 成熟 |
| 2024 版本 | 2024 年 | 性能大幅提升（面向 NVMe + 100Gbps 网络优化）、K8s 更深度集成、KES 与 Vault 集成增强 |
| 2025 版本 | 2025 年 | SSE-KMS 改进、批量操作 API、多站点复制增强、Console 全新 UI |

> **许可证变更**：自 2021 年起，MinIO 将开源协议从 Apache 2.0 逐步调整为 AGPLv3（主要是为了限制云厂商将 MinIO 作为商业服务直接售卖而不贡献回社区）。对于大多数企业内部使用场景，AGPLv3 不存在合规问题。如有商业再分发需求，需获取商业许可。

---

## 1.6MinIO适用场景与不适用场景

### 适用场景

| 场景 | 说明 | 典型案例 |
|------|------|----------|
| **影像/音视频存储** | 海量图片、视频的原始存储与分发 | 社交App图片中心、视频点播平台、医学影像归档 |
| **备份与归档** | 定期备份数据的廉价大规模存储 | 数据库备份、日志归档、合规数据留存 |
| **数据湖** | 大规模结构化/非结构化数据的统一存储底座 | 数据仓库ODS层、AI训练数据集、IoT时序数据 |
| **静态网站托管** | 前端静态资源（HTML/CSS/JS）+ 自定义域名 | 官网、文档站、SPA应用 |
| **云原生应用存储** | Kubernetes环境下的持久化存储 | 容器化应用的文件上传、制品库 |
| **混合云统一存储** | 本地数据中心 + 公有云之间的数据分层与同步 | 本地上传 + 按策略同步到 AWS S3/Glacier |
| **边缘计算** | 边缘节点本地存储，轻量级部署 | 工业IoT数据采集、CDN边缘缓存 |
| **DevOps/CI/CD** | 制品、构建缓存、测试数据存储 | Maven/NPM/Docker Registry 替代后端 |

### 不适用场景

| 场景 | 原因 | 替代方案 |
|------|------|----------|
| **POSIX文件系统需求** | MinIO 是对象存储，不支持文件锁、Append写入、目录遍历等 POSIX 语义 | NFS、CephFS、JuiceFS |
| **极低延迟单机随机读写** | 对象存储在 HTTP 请求路径上有额外开销 | 本地 SSD、SAN |
| **需要存储过程/触发器** | 对象存储无服务端计算能力（虽然支持事件通知触发 Lambda） | 关系型数据库 |
| **频繁修改对象的部分内容** | 对象是不可变的，修改需要重新PUT整个对象 | 块存储、文件系统 |
| **极小文件（< 1KB）且数量极大** | 每个对象的元数据占 2-4KB，小文件过多会导致存储放大 | 数据库、列式存储 |
| **目录层级深度嵌套** | MinIO 是平面命名空间，"目录"只是 Key 前缀的显示效果 | 文件存储 |

### 对象存储中的"目录"机制

MinIO 是平面命名空间，**不存在真正的目录**。`photos/2024/vacation/pic.jpg` 中的 `/` 是 Key 名称的一部分。

- 创建目录：`aws s3api put-object --bucket test --key dir/`（一个大小为 0、以 `/` 结尾的对象）
- 列举"目录下"的对象：`aws s3api list-objects --bucket test --prefix dir/ --delimiter /`
- Console 界面会根据 `/` 展示树形结构，但这只是 UI 层面的模拟

```java
// MinIO Java SDK 模拟目录列举
Iterable<Result<Item>> results = minioClient.listObjects(
    ListObjectsArgs.builder()
        .bucket("mybucket")
        .prefix("dir1/")
        .delimiter("/")       // 使用分隔符模拟目录层级
        .build()
);
for (Result<Item> result : results) {
    Item item = result.get();
    if (item.isDir()) {
        System.out.println("目录: " + item.objectName());
    } else {
        System.out.println("文件: " + item.objectName() + " 大小: " + item.size());
    }
}
```

---

## 1.7S3 API 协议简介

MinIO 的核心价值之一是**兼容 AWS S3 API**。理解 S3 API 有助于深入使用 MinIO。

### S3 API 的基本要素

| 要素 | 说明 | 示例 |
|------|------|------|
| Bucket | 存储桶，对象的容器（全局唯一名称，命名规则：3-63字符，小写，不能以IP格式命名） | `my-bucket` |
| Object Key | 对象在桶内的唯一标识符 | `photos/2024/01/01/pic.jpg` |
| ETag | 对象的 MD5 哈希值（分片上传时是复合MD5） | `"abc123def456"` |
| VersionId | 对象版本标识（开启版本控制后） | `null` 或 UUID |
| Metadata | 用户自定义的键值对（x-amz-meta-前缀） | `x-amz-meta-author: john` |
| ACL | 访问控制列表（MinIO 建议使用 IAM Policy 而非 ACL） | `private`, `public-read` |

### S3 API 核心操作分类

```text
Service API (服务级别)
├── ListBuckets          列举所有桶

Bucket API (桶级别)
├── CreateBucket         创建桶
├── DeleteBucket         删除桶
├── HeadBucket           检查桶是否存在
├── GetBucketLocation    获取桶区域
├── GetBucketVersioning  获取版本控制状态
├── PutBucketVersioning  设置版本控制
├── GetBucketLifecycle   获取生命周期规则
├── PutBucketLifecycle   设置生命周期规则
├── GetBucketPolicy      获取桶策略
├── PutBucketPolicy      设置桶策略
├── GetBucketEncryption  获取默认加密配置
├── PutBucketEncryption  设置默认加密
├── GetBucketReplication 获取复制规则
├── PutBucketReplication 设置复制规则
├── GetBucketNotification 获取通知配置
├── PutBucketNotification 设置通知配置
├── GetObjectLockConfig  获取对象锁配置
├── PutObjectLockConfig  设置对象锁

Object API (对象级别)
├── PutObject            上传对象
├── GetObject            下载对象
├── DeleteObject         删除对象（单个）
├── DeleteObjects        批量删除（Multi-Object Delete）
├── HeadObject           获取对象元数据
├── CopyObject           复制对象
├── ListObjects          列举对象 (V1)
├── ListObjectsV2        列举对象 (V2，推荐)
├── PutObjectRetention   设置对象保留期限
├── GetObjectRetention   获取对象保留期限
├── PutObjectLegalHold   设置对象依法保留
├── GetObjectLegalHold   获取对象依法保留
├── PutObjectTagging     设置对象标签
├── GetObjectTagging     获取对象标签
├── DeleteObjectTagging  删除对象标签
├── SelectObjectContent  对象内容查询（SQL）

Multipart Upload API (分片上传)
├── CreateMultipartUpload 初始化分片上传
├── UploadPart            上传分片
├── UploadPartCopy        从已有对象复制分片
├── CompleteMultipartUpload 完成分片上传
├── AbortMultipartUpload  取消分片上传
├── ListParts             列举已上传分片
├── ListMultipartUploads  列举进行中的分片上传

Presigned URL
├── PresignedPutObject   生成预签名上传URL
├── PresignedGetObject   生成预签名下载URL
```

### 签名版本

MinIO 支持两种 AWS 签名版本：

| 签名版本 | 说明 |
|----------|------|
| Signature V2 | 旧版签名，已基本弃用，MinIO 默认关闭 |
| Signature V4 | 当前标准，支持 Content-SHA256 校验、支持 Chunked Transfer Encoding |

V4 签名流程：
```text
1. 构造 CanonicalRequest (HTTP方法 + URI + QueryString + Headers + PayloadHash)
2. 构造 StringToSign (算法 + 时间戳 + 凭证范围 + CanonicalRequest的SHA256)
3. 计算 Signature (HMAC-SHA256(SigningKey, StringToSign))
4. 拼装 Authorization 头
```

---

## 1.8知识储备

学习 MinIO 之前，建议具备以下基础：

### 必备基础
- **HTTP/REST API**：理解 HTTP 方法（GET/PUT/DELETE/HEAD）、状态码、请求/响应头
- **AWS S3 基础概念**：Bucket、Object、Key、Region、AccessKey/SecretKey 的基本认知
- **Java SE**：基础语法、IO流、文件操作
- **Spring Boot**：基础配置、自动装配、Bean管理
- **Docker**：基础镜像操作（pull、run、compose）
- **Linux**：基础命令（文件管理、进程管理、网络配置）

### 可选基础
- **Kubernetes**：学习 Operator 部署和 Tenant 管理时需要
- **Prometheus + Grafana**：学习监控告警时需要
- **分布式系统理论**：CAP 定理、Quorum 机制、一致性哈希等（理解分布式部署时非常有帮助）
- **Go 语言**：阅读 MinIO 源码时需要（笔记中会尽量用 Java 类比解释）

---

## 1.9本笔记使用的版本

| 组件 | 版本 |
|------|------|
| MinIO Server | RELEASE.2025-02-28T09-55-16Z（2025年最新稳定版） |
| MinIO Client (mc) | 最新稳定版 |
| MinIO Java SDK | 8.5.x（minio Java client） |
| Java | 21 |
| Spring Boot | 3.3.x |
| 构建工具 | Maven |
| IDE | IntelliJ IDEA |
| 操作系统 | Linux (Ubuntu 22.04) / Windows (Docker Desktop) |
