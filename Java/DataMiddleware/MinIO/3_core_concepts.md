# 1.Bucket（存储桶）

## 1.1Bucket 的基本概念

Bucket 是 MinIO 中对象的逻辑容器，类似于文件系统中的"顶层目录"或者关系型数据库中的"数据库"。

- Bucket 名称在集群中全局唯一
- 命名规则：
  - 长度 3~63 个字符
  - 只能包含小写字母、数字、连字符（`-`）
  - 必须以字母或数字开头和结尾
  - 不能以 IP 地址格式命名（如 `192.168.1.1`）
  - 不能用 `xn--` 开头（保留给国际化域名）
- Bucket 创建后名称不可修改（但可以删除后重建）

### Bucket 的内部实现

每个 Bucket 在 MinIO 内部对应一个独立的命名空间，实际上映射到数据磁盘上的一个目录。但关键在于：Bucket 的配置（版本控制、生命周期、策略、加密等）存储在 MinIO 内置的 `.minio.sys` 内部 Bucket 中，通过 `xl.meta` 和配置对象来维护。

```text
.minio.sys/config/                     # 集群/Bucket 配置存储
├── config.json/                       # 集群配置
└── buckets/
    └── my-bucket/                     # Bucket 级配置
        ├── lifecycle/                 # 生命周期规则
        ├── notification/              # 通知配置
        ├── replication/               # 复制规则
        ├── policy/                    # 桶策略
        └── versioning/                # 版本控制状态
```

## 1.2Bucket 操作的本质

从 S3 API 角度来看：

| 操作 | PUT Bucket | GET Bucket | DELETE Bucket |
|------|------------|------------|---------------|
| HTTP 方法 | `PUT /bucket` | `GET /bucket` | `DELETE /bucket` |
| 幂等性 | 非幂等（已存在返回409） | 幂等 | 幂等 |
| 内部行为 | 创建命名空间 + 初始化配置 | 列举命名空间中的对象 | 删除命名空间及所有对象 |

```java
// Java SDK 中的 Bucket 操作
MinioClient client = MinioClient.builder()
    .endpoint("http://localhost:9000")
    .credentials("minioadmin", "minioadmin")
    .build();

// 创建 Bucket
client.makeBucket(MakeBucketArgs.builder()
    .bucket("my-bucket")
    .build());

// 检查 Bucket 是否存在
boolean exists = client.bucketExists(BucketExistsArgs.builder()
    .bucket("my-bucket")
    .build());

// 列举所有 Bucket
List<Bucket> buckets = client.listBuckets();
buckets.forEach(b -> System.out.println(b.name() + ": " + b.creationDate()));

// 删除 Bucket（必须为空）
client.removeBucket(RemoveBucketArgs.builder()
    .bucket("my-bucket")
    .build());
```

---

# 2.Object（对象）

## 2.1对象的概念

对象是 MinIO 存储的基本单元，每个对象由三部分构成：

```text
对象 = 数据（任意二进制流）+ 元数据（键值对集合）+ 唯一标识符（Bucket + Object Key）
```

### 对象 Key 的命名

- 长度：1~1024 字节（UTF-8 编码）
- 字符集：Unicode 字符
- 安全字符：`0-9 a-z A-Z ! - _ . * ' ( )`
- 需要 URL 编码的字符：空格、`&`、`@`、`:`、`$`、`+`、`,`、`=`、`/`、`?` 等
- Key 中的 `/` 被 Console 界面解释为目录分隔符

```bash
# 合法 Key 示例
photo.jpg
photos/2024/01/vacation.jpg
documents/reports/2024_Q4_final_v2.pdf
user/1001/avatar.png
data/2024-01-15/logs/access.log
```

### 对象元数据

MinIO 支持两类元数据：

| 类型 | 示例 | 说明 |
|------|------|------|
| 系统元数据 | Content-Type, Content-Length, ETag, LastModified | 系统自动设置 |
| 用户自定义元数据 | x-amz-meta-author, x-amz-meta-version | 以 `x-amz-meta-` 为前缀的自定义键值对 |
| 对象标签 | tag-key=tag-value | S3 支持的标签机制（最多10个标签/对象） |

```java
// 上传对象时设置元数据
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("documents/report.pdf")
        .stream(inputStream, fileSize, -1)
        .contentType("application/pdf")
        .userMetadata(Map.of(
            "author", "张三",
            "department", "研发部",
            "version", "1.0"
        ))
        .tags(Map.of(
            "status", "published",
            "year", "2024"
        ))
        .build()
);
```

## 2.2对象的 ETag

ETag（Entity Tag）是对象的哈希标识，用于验证对象完整性：

- **简单上传**（≤ 16MB 单次 PUT）：ETag = 对象内容的 MD5 哈希值
- **分片上传**（> 16MB 或手动触发）：ETag = 各分片 MD5 的复合哈希值（格式：`"hash-numParts"`）
- **SSE-C 加密上传**：ETag ≠ 原始内容 MD5（加密后才计算）

```java
// 获取对象元数据（含 ETag）
StatObjectResponse stat = minioClient.statObject(
    StatObjectArgs.builder()
        .bucket("my-bucket")
        .object("photo.jpg")
        .build()
);
System.out.println("ETag: " + stat.etag());
System.out.println("Size: " + stat.size());
System.out.println("Content-Type: " + stat.contentType());
System.out.println("Last Modified: " + stat.lastModified());
```

## 2.3对象的不可变性

对象存储的核心原则之一：**对象是不可变的（Immutable）**。

- 不存在"修改对象部分内容"的操作
- 要"修改"一个对象，需要 **PUT 一个新对象 + DELETE 旧对象**（或依赖版本控制保留历史）
- COPY 操作实际是服务器端复制（不占用客户端带宽），但仍是生成新对象

```java
// "修改"对象的等价操作：先上传新版本，再删除旧版本
// 如果开启了版本控制，旧版本会保留，不会被物理删除
minioClient.putObject(PutObjectArgs.builder()
    .bucket("my-bucket")
    .object("photo.jpg")      // 同名 Key，覆盖或生成新版本
    .stream(newInputStream, newSize, -1)
    .build());
```

---

# 3.Erasure Coding（纠删码）原理详解

**这是 MinIO 最核心、最重要的技术。** 理解纠删码是理解 MinIO 整体架构的基础。

## 3.1为什么使用纠删码而不是多副本

| 方案 | 冗余度 | 容错能力 | 存储效率 |
|------|--------|----------|----------|
| 3副本 | 200% | 任意2个副本故障 | 33.3% |
| EC (8+4) | 150% | 任意4个分片故障 | 66.7% |
| EC (8+8) | 200% | 任意8个分片故障 | 50% |
| EC (16+4) | 125% | 任意4个分片故障 | 80% |

> **关键洞察**：纠删码用更少的存储开销，提供与多副本相同甚至更强的数据保护能力。

## 3.2Reed-Solomon 纠删码的数学原理

MinIO 使用 Reed-Solomon（RS）纠删码。RS 码是纠删码领域最经典的算法，其核心思想是**用数据块构建多项式，通过多项式在不同点的取值作为校验块**。

### 核心概念

```text
假设配置为 (k, m)，其中：
  k = 数据块数量 (Data Drives)
  m = 校验块数量 (Parity Drives)
  N = k + m = 总驱动器数

关键性质：
  - 写入 N 个分片（k 个数据 + m 个校验）
  - 读取时：只需要任意 k 个分片即可恢复完整数据
  - 容错：可以容忍任意 m 个分片丢失
  - 存储开销：m/k (即 (N-k)/k)
```

### 编码过程（简化数学描述）

```text
给定数据块 D = [D1, D2, ..., Dk]，每个 Di 是固定大小的块（如 1MB）

1. 构造范德蒙德矩阵 V (N × k)：
   V[i][j] = i^(j-1)    (i = 0, 1, ..., N-1; j = 0, 1, ..., k-1)
   
   即：
   ⎡ 1  0  0  ...  0   ⎤  ← 行 0 ~ k-1 是单位矩阵（数据块自身）
   ⎢ 0  1  0  ...  0   ⎥
   ⎢ ...               ⎥
   ⎢ 0  0  0  ...  1   ⎥
   ⎢ 1  1  1  ...  1   ⎥  ← 行 k 开始是校验行
   ⎢ 1  2  4  ... 2^k-1⎥
   ⎢ ...               ⎥
   ⎣ 1  m  m² ... m^k-1⎦

2. 编码：C = V × D

3. 结果 C = [D1, D2, ..., Dk, P1, P2, ..., Pm]
   - 前 k 个分片 = 原始数据块本身
   - 后 m 个分片 = 校验分片

4. 写入：N 个分片分别写入 N 个磁盘
```

### 解码过程（恢复数据）

```text
当部分分片丢失时：

1. 假设丢失了 e 个分片（e ≤ m）
2. 从 V 中删除对应行，得到 V' (k × k 子方阵)
3. 从 C 中删除对应元素，得到 C' (k × 1)
4. 计算 D = V'⁻¹ × C' （在伽罗瓦域 GF(2^8) 中的矩阵求逆和乘法）
5. D 即为原始数据

关键前提：V' 必须是可逆的（范德蒙德矩阵的任意 k 行子矩阵都是可逆的，满足该性质）
```

### 伽罗瓦域 GF(2^8) — 为什么选择它

纠删码运算在**有限域（伽罗瓦域）GF(2^8)**上进行，原因如下：

- GF(2^8) 中的每个元素正好是一个字节（8位），与计算机字节天然对齐
- 域上的加法 = XOR 运算，硬件加速极快（CPU 单指令完成）
- 域上的乘法 = 通过查表或 Intel SIMD (GFNI) 指令加速
- 矩阵求逆和乘法在有限域上必定有解（满足范德蒙德性质）

```c
// 伽罗瓦域 GF(2^8) 中的加法（就是 XOR）
uint8_t gf_add(uint8_t a, uint8_t b) {
    return a ^ b;  // CPU 单指令: XOR
}

// 伽罗瓦域 GF(2^8) 中的乘法（查表实现）
uint8_t gf_mul(uint8_t a, uint8_t b) {
    // 实际使用查表法或 CPU GFNI 指令
    if (a == 0 || b == 0) return 0;
    return gf_log[(gf_alog[a] + gf_alog[b]) % 255];
}
```

## 3.3MinIO 中 EC 的默认配置

MinIO 根据总驱动器数量自动选择 EC 配置：

| 总驱动器数 | EC 配置 | 数据块 | 校验块 | 存储效率 | 可故障磁盘数 |
|-----------|---------|--------|--------|----------|-------------|
| 4 | 2+2 | 2 | 2 | 50% | 2 |
| 8 | 4+4 | 4 | 4 | 50% | 4 |
| 16 | 8+8 | 8 | 8 | 50% | 8 |
| 24 | 12+12 | 12 | 12 | 50% | 12 |
| 32 | 16+16 | 16 | 16 | 50% | 16 |

**设计原则**：
- 默认 N/2 校验块（保证至少可容忍一半磁盘故障）
- 最大校验块数不超过 16（即使总磁盘数 > 32）
- 最小 EC 集合大小为 4（少于 4 块磁盘无法使用 EC，只能用副本模式）

## 3.4Erasure Set 的划分

MinIO 将总磁盘划分为多个大小相等的 Erasure Set（EC 集合）。每个 Set 独立进行 EC 编解码。

```text
集群: 16 节点 × 16 磁盘 = 256 磁盘
EC 条带大小: 16 (8数据 + 8校验)
Erasure Set 数量: 256 / 16 = 16 个 Set

Set 0:  Disk 1~16  (节点1~16 的第一块盘)
Set 1:  Disk 17~32 (节点1~16 的第二块盘)
...
Set 15: Disk 241~256

每个 Set 包含来自不同节点的磁盘，确保单节点故障不影响 Set 完整性
```

### 为什么跨节点选取磁盘

```text
好的 Set 划分（跨节点）：
Set 0: [Node1:Disk1, Node2:Disk1, ..., Node16:Disk1]
→ 任何单一节点故障，每个 Set 只损失 1 块盘，仍可恢复

坏的 Set 划分（同节点）：
Set 0: [Node1:Disk1, Node1:Disk2, ..., Node1:Disk16]
→ Node1 故障，整个 Set 的 16 块盘全部丢失，数据永久丢失
```

**MinIO 自动处理 Set 划分，无需手动配置。** 但理解这个原理有助于理解为什么分布式 MinIO 要求最少 4 个节点。

## 3.5EC 条带大小与块大小

- **EC 条带大小（Stripe Size）**：一个 EC 编码单元的数据量。MinIO 根据 EC 配置和磁盘块大小自动计算。
- **默认块大小（Block Size）**：10MB（通过 `MINIO_ERASURE_SET_DRIVE_COUNT` 可影响）。
- **大对象**（> 128MB）：按 10MB 为单位分块，每块独立进行 EC 编码。

```text
以 8+8 EC 配置、10MB 块大小为例，上传一个 200MB 的文件：

200MB 文件
    │
    ▼ 分块（10MB/块）
┌──────┐ ┌──────┐ ┌──────┐     ┌──────┐
│ 块1  │ │ 块2  │ │ 块3  │ ... │ 块20 │
│ 10MB │ │ 10MB │ │ 10MB │     │ 10MB │
└──┬───┘ └──┬───┘ └──┬───┘     └──┬───┘
   │        │        │            │
   ▼        ▼        ▼            ▼
EC编码  EC编码  EC编码  ...  EC编码
   │        │        │            │
   ▼        ▼        ▼            ▼
16分片  16分片  16分片  ...  16分片
(8+8)   (8+8)   (8+8)       (8+8)
```

### 小对象的处理

MinIO 对小对象有特殊优化：
- 非常小的对象（< 128KiB）：可能不使用完整的 EC 条带，而是使用更小的块大小，甚至使用副本模式
- 中等对象（128KiB ~ 10MB）：使用一个 EC 条带

---

## 3.6Bit Rot Protection（静默数据损坏检测）

Bit Rot（数据腐化/静默数据损坏）是指磁盘上的数据因物理退化、宇宙射线等自然原因而不知不觉地发生损坏，而磁盘自身又不报告错误。

### MinIO 的检测机制

MinIO 在每个 64MB 数据块的末尾存储一个**高速哈希校验和**（使用 HighwayHash 算法）：

```text
┌────────────────────────────────────────────────────┐
│                     数据块 (≤64MB)                   │
├────────────────────────────────────────────────────┤
│              HighwayHash 校验和 (32字节)              │
└────────────────────────────────────────────────────┘
```

- **写入时**：计算数据块的 HighwayHash 并附带存储在分片中
- **读取时**：重新计算 HighwayHash 并与存储的校验和比对
- **不一致时**：从其他分片自动修复损坏的块（通过 EC 解码恢复）
- **后台扫描**：MinIO 的 Scanner 子系统会定期扫描所有对象，主动检测并修复 Bit Rot

> **HighwayHash** 是 Google 开发的高速非加密哈希算法，设计用于抵御哈希洪泛攻击的同时保持极高的吞吐量（在 x86_64 上可达 10GB/s+）。MinIO 选择它而非 SHA-256，是因为：
> - 非加密场景下不需要密码学强度
> - HighwayHash 比 SHA-256 快约 5~30 倍
> - HighwayHash 具有伪随机性，满足误判率要求

---

# 4.数据分布与哈希机制

## 4.1对象到 EC Set 的路由

MinIO 使用 **SipHash** 哈希算法将对象 Key 路由到特定的 Erasure Set：

```text
1. 计算 SipHash(object-key, cluster-seed) → 64位哈希值
2. 取哈希值 % ErasureSetCount → 确定归属的 EC Set 编号
3. 在该 Set 内有数据驱动器上进行 EC 编码并写入
```

```go
// 伪代码：MinIO 中对象到 ES Set 的路由
func getErasureSetIndex(objectKey string, setCount int) int {
    clusterSeed := getClusterSeed() // 集群创建时生成，确保一致
    hash := sipHash([]byte(objectKey), clusterSeed)
    return int(hash % uint64(setCount))
}
```

### SipHash 选择的原因

- **SipHash** 是速度与安全平衡的哈希算法（非密码学但具备抗哈希碰撞攻击能力）
- 速度与 MurmurHash3 相当，但比后者更安全（MurmurHash3 可被构造碰撞攻击）
- 64位输出足够用于 Set 路由（即使有 65536 个 Set 也不会溢出）

## 4.2数据在 EC Set 内部的分布

在一个 Erasure Set 内部，EC 编码后的 N 个分片按照**轮询（Round-Robin）**或**指定的分布序列**写入 N 个磁盘：

```text
对象: photo.jpg, EC Set 0, EC 配置 8+8

分片分布（根据 xl.meta 中的 distribution 字段确定）:
  分片 0 → Disk 0  (Node1)
  分片 1 → Disk 1  (Node2)
  分片 2 → Disk 2  (Node3)
  ...
  分片 15 → Disk 15 (Node16)

读取时：任意 8 个分片即可重组原数据
```

---

# 5.一致性模型

## 5.1MinIO 的一致性保证

MinIO 提供 **强一致性（Strong Consistency）**：

- **Read-after-Write 强一致**：对象 PUT 成功后，任何客户端立即可以 GET 到最新内容
- **List-after-Write 强一致**：对象 PUT 成功后，ListObjects 立即可见
- **Delete-after-Write 强一致**：对象 DELETE 成功后，GET 立即返回 404

这是通过 **Quorum 写入确认** 和 **全局元数据仲裁** 实现的。

## 5.2读写 Quorum 机制

MinIO 使用多数派（Quorum）机制确保强一致性：

| 操作 | Quorum 要求 | 说明 |
|------|-------------|------|
| PUT (新建) | N/2 + 1 个分片写入成功 | 确保大多数节点包含该数据 |
| DELETE | N/2 + 1 个分片写入成功 | 确保删除信息在大多数节点生效 |
| GET | 需要成功读取至少 k 个分片 | k 为数据块数（8+8配置下需读8个分片） |
| List | N/2 + 1 个节点响应一致 | 确保目录列表的一致性 |

### Quorum 公式的含义

对于 EC 8+8 配置（N=16）：
- **写入 Quorum** = 16/2 + 1 = 9 个分片
- 只要 9 个磁盘写入成功（其中必须包含至少 8 个数据分片的等价信息，实际就是至少有 1 个校验分片也写成功），操作即返回成功
- 剩余的 7 个分片由后台异步补全或通过读取时的 EC 恢复来保证

```text
写入场景分析：
  ┌─ Disk 0  : ✓ (成功)
  ├─ Disk 1  : ✓
  ├─ Disk 2  : ✓
  ├─ Disk 3  : ✓
  ├─ Disk 4  : ✓
  ├─ Disk 5  : ✓
  ├─ Disk 6  : ✓
  ├─ Disk 7  : ✓
  ├─ Disk 8  : ✓  ← 9个成功, Quorum满足, 返回200
  ├─ Disk 9  : ✗ (失败)
  ├─ Disk 10 : ✗
  ├─ Disk 11 : ✗
  ├─ Disk 12 : ✗
  ├─ Disk 13 : ✗
  ├─ Disk 14 : ✗
  └─ Disk 15 : ✗

结果：客户端收到成功响应。
后台：MinIO 的自动修复机制会将缺失的分片写入 Disk 9~15。
```

## 5.3与 AWS S3 的一致性对比

| 操作 | MinIO | AWS S3（旧） | AWS S3（2020后） |
|------|-------|-------------|-----------------|
| PUT 新对象 | 强一致 | 最终一致 | 强一致 |
| PUT 覆盖 | 强一致 | 最终一致 | 强一致 |
| DELETE | 强一致 | 最终一致 | 强一致 |
| List | 强一致 | 最终一致 | 强一致 |

> MinIO 从一开始就实现了强一致性，而 AWS S3 直到 2020 年 12 月才宣布支持强一致性。这是 MinIO 架构优势的体现。

---

# 6.数据存储格式详解

## 6.1磁盘上的物理存储

MinIO 将每个对象的 EC 分片存储在磁盘上，目录结构如下：

```text
/mnt/disk1/data/bucket-name/object-key/
├── part.1           # EC 编码分片 (元数据 + 数据)
│                     # 格式: [xl.meta JSON] + [0x00]*N + [数据块]
└── xl.meta          # 独立元数据文件 (小对象)
```

### part.1 文件格式

```
偏移量 0:  xl.meta JSON 内容
          { "version": "1.0", "format": "xl", "stat": {...}, ... }
          \n (换行分隔符)
偏移量 N:  二进制数据块内容
```

## 6.2xl.meta 详解

`xl.meta` 是 MinIO 对象元数据的核心文件，JSON 格式。以下是完整字段说明：

```json
{
  "version": "1.0",
  "format": "xl",
  "stat": {
    "size": 1048576,                  // 对象原始大小（字节）
    "modTime": "2025-01-15T10:30:00Z" // 最后修改时间（RFC3339）
  },
  "erasure": {
    "data": 8,                        // 数据块数量
    "parity": 8,                      // 校验块数量
    "blockSize": 10485760,            // EC 块大小（10MB）
    "index": 3,                       // 当前分片索引
    "distribution": [0,1,2,...,15],   // 16 个分片到磁盘的映射
    "checksum": [
      {"algorithm": "highwayHash256S", "blocksize": 67108864}
    ]
  },
  "metaUser": {                       // 用户自定义元数据
    "content-type": "image/jpeg",
    "X-Amz-Meta-Author": "john"
  },
  "parts": [                          // 分片上传时记录的各分片信息
    {"number": 1, "size": 5242880, "etag": "abc123"},
    {"number": 2, "size": 5242880, "etag": "def456"}
  ]
}
```

---

# 7.命名空间与"目录"

## 7.1平面命名空间

MinIO 的命名空间是扁平的。`photos/2024/01/vacation.jpg` 只是对象 Key 中的字符序列，`/` 没有文件系统级别的特殊含义。

### ListObjects V2 的"目录"模拟

```java
// 使用 delimiter 和 prefix 模拟目录层级
ListObjectsArgs args = ListObjectsArgs.builder()
    .bucket("my-bucket")
    .prefix("photos/2024/")     // 只列举该"目录"下的对象
    .delimiter("/")             // 按 "/" 分组
    .build();

Iterable<Result<Item>> results = minioClient.listObjects(args);
for (Result<Item> result : results) {
    Item item = result.get();
    if (item.isDir()) {
        // CommonPrefix: "photos/2024/01/"
        // CommonPrefix: "photos/2024/02/"
        System.out.println("子目录: " + item.objectName());
    } else {
        // Object: "photos/2024/cover.jpg"
        System.out.println("文件: " + item.objectName());
    }
}
```

### 分页列举

```java
// 使用 startAfter 和 maxKeys 实现分页
ListObjectsArgs.builder()
    .bucket("my-bucket")
    .prefix("logs/")
    .delimiter("/")
    .maxKeys(100)              // 每页最多返回 100 个对象
    .startAfter("logs/2024-06-01/")  // 从该 Key 之后开始列举（分页游标）
    .build();
```

---

# 8.MinIO 的内部组件与子系统

## 8.1Object Layer（对象层接口）

MinIO 定义了一个抽象的 ObjectLayer 接口，所有对象操作都基于这个接口。这样可以在不改变上层 API 的前提下，替换底层存储实现：

```go
// Go 伪代码：MinIO 核心接口
type ObjectLayer interface {
    // Bucket 操作
    MakeBucket(ctx context.Context, bucket string, opts BucketOptions) error
    GetBucketInfo(ctx context.Context, bucket string) (BucketInfo, error)
    ListBuckets(ctx context.Context) ([]BucketInfo, error)
    DeleteBucket(ctx context.Context, bucket string, opts DeleteBucketOptions) error

    // Object 操作
    PutObject(ctx context.Context, bucket, object string, data *PutObjReader, opts ObjectOptions) (objInfo ObjectInfo, err error)
    GetObject(ctx context.Context, bucket, object string, startOffset int64, length int64, writer io.Writer, etag string, opts ObjectOptions) error
    GetObjectInfo(ctx context.Context, bucket, object string, opts ObjectOptions) (objInfo ObjectInfo, err error)
    DeleteObject(ctx context.Context, bucket, object string, opts ObjectOptions) (objInfo ObjectInfo, err error)

    // 列表操作
    ListObjects(ctx context.Context, bucket, prefix, marker, delimiter string, maxKeys int) (result ListObjectsInfo, err error)
}
```

### 实现体系

```text
ObjectLayer 接口
├── FSObjects        # 单机/单盘模式实现
│   └── 直接使用本地文件系统
└── erasureServerPools  # 分布式 EC 模式实现
    └── 一组 Erasure Set 的集合，每个 Set 独立进行 EC 编解码
```

## 8.2关键内部组件

| 组件 | 职责 |
|------|------|
| **Scanner** | 后台扫描所有对象元数据，检测数据腐化、索引损坏、过期对象清理、容量统计收集 |
| **Healer** | 自动修复损坏/缺失的分片（通过 EC 解码从正常分片恢复），在磁盘更换后自动重建数据 |
| **Decom** | 节点退役（Decommissioning），将退役节点的数据迁移到其他节点 |
| **Replicator** | 处理桶复制和站点复制，将变化同步到远程集群 |
| **Lifecycle** | 执行生命周期规则（过期删除、转换存储层等） |
| **Notifier** | 处理事件通知（对象创建/删除/访问 → Kafka/Webhook 等） |

## 8.3请求处理流程

```text
HTTP 请求到达
    │
    ▼
┌──────────────┐
│  AuthN 层    │  认证：验证签名 (V2/V4)、JWT 解析 (OIDC)、LDAP 验证
├──────────────┤
│  AuthZ 层    │  授权：检查 IAM Policy、Bucket Policy、ACL
├──────────────┤
│  Handler 层  │  路由：根据 HTTP Method + Path 分发到具体处理函数
│  (s3-handlers)│  ├── PutObjectHandler
│              │  ├── GetObjectHandler
│              │  └── ListObjectsHandler ...
├──────────────┤
│  ObjectLayer │  抽象层：调用 ObjectLayer 接口
│  接口层      │
├──────────────┤
│  EC 层       │  编码/解码：数据分块 → EC 编码 → 分发到磁盘
│  (erasure)   │            ← EC 解码 ← 从磁盘读取分片
├──────────────┤
│  Disk I/O    │  物理读写：实际读写磁盘或通过网络读写远程磁盘
└──────────────┘
```

---

# 9.MinIO 的节点发现与集群管理

## 9.1节点发现机制

MinIO 分布式模式通过 **启动命令中的端点列表** 进行节点发现：

```bash
# 4 个节点启动完全相同的命令
minio server \
  http://minio-1.example.com/data \
  http://minio-2.example.com/data \
  http://minio-3.example.com/data \
  http://minio-4.example.com/data
```

- 每个节点启动时，通过端点列表中的主机名找到其他节点
- 使用 **Gossip 协议**（基于 serf）进行节点状态同步
- 使用 **内部 RPC（gRPC-like）** 进行数据复制和协调

## 9.2集群共识（Leaderless 设计）

MinIO **不使用 Raft/Paxos 等共识协议**，这是其简单性的重要来源：

- 没有 Leader 节点，所有节点都是对等的
- 通过 Quorum 机制保证一致性
- 通过 SipHash 决定对象归属的 EC Set，不需要中心化路由表
- 节点启动时通过读取本地磁盘上的 `format.json` 判断自己是否属于某个集群

### format.json

每个磁盘根目录下有 `format.json`，记录集群拓扑信息：

```json
{
  "version": "1",
  "format": "xl",
  "id": "6a28a3c0-...",
  "xl": {
    "version": "3",
    "this": "6a28a3c0-...",
    "sets": [
      [
        "f7b0e1a0-...",
        "c3d8e2b0-...",
        // ... 每个磁盘的 UUID
      ]
    ],
    "distributionAlgo": "SIPMOD+PARITY"
  }
}
```

---

# 10.内部通信协议

MinIO 内部节点间通信通过 gRPC-like 的自定义 RPC 协议进行：

| 通信场景 | 说明 |
|----------|------|
| **磁盘 I/O 代理** | 当一个节点需要写入/读取另一个节点的磁盘时，通过内部 RPC 转发 I/O 请求 |
| **Heal 数据修复** | 检测到数据损坏时，从其他节点读取正常分片，EC 解码后写回损坏节点 |
| **Gossip 状态同步** | 节点状态（在线/离线/磁盘故障等）通过 serf 库实现的 Gossip 协议传播 |
| **Scanner 协同** | 后台扫描任务在各节点间协调，避免重复扫描 |

---

# 11.容量管理

## 11.1容量组成

```text
总物理容量
    │
    ├── EC 校验开销 (N/2 校验 → 50%)
    │   └── 可用数据容量 = 总物理容量 × k/N
    │       (8+8: 50% | 12+4: 75% | 16+4: 80%)
    │
    └── 元数据开销 (~0.5% - 2%)
        ├── xl.meta 文件 (每个对象 2~8KB)
        ├── .minio.sys 内部存储
        └── format.json 等系统文件
```

## 11.2容量监控

```bash
# 命令行查看集群容量
mc admin info myminio
# 输出：
# ●  minio-1.example.com:9000
#    Uptime: 30 days
#    Drives: 4/4 OK
#    Used: 2.5 TiB (55%)

# 查看详细信息
mc admin info myminio --json
```

```java
// Java SDK 中没有直接的容量查询 API
// 通过 admin API 间接获取（需要 MinIO Admin Client）
// 或者通过 Prometheus 指标获取：
// minio_cluster_capacity_usable_total_bytes
// minio_cluster_capacity_usable_free_bytes
```
