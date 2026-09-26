# 1. 知识点
- **MinIO简介**: 高性能开源对象存储服务器，兼容Amazon S3 HTTP API，Go语言开发，用于存储非结构化数据(图片、视频、日志、备份等)
- **S3兼容性**: MinIO完全兼容AWS S3 API，使用标准S3 SDK(如AWS Java SDK)即可操作MinIO，便于已有S3应用迁移
- **适用场景**: 文件存储(图片、视频、文档)、数据湖/数据仓库、AI/ML训练数据存储、应用备份、日志归档
- **核心概念-Bucket(桶)**: 类似于文件系统的"目录"或数据库的"数据库"，用于逻辑隔离和组织对象。每个桶有独立的权限策略、生命周期规则
- **核心概念-Object(对象)**: 存储在Bucket中的实际文件数据+元数据。对象由数据(文件内容)、元数据(key-value对如Content-Type)、唯一标识(对象名/Key)组成
- **核心概念-元数据**: 描述对象的信息，如Content-Type、Content-Length、Last-Modified、ETag(MD5)，支持自定义元数据(x-amz-meta-前缀)
- **部署模式-单机(Standalone)**: 单块磁盘(一个Node一个Drive)运行MinIO，适用于开发测试环境，不支持数据冗余和高可用
- **部署模式-纠删码(Erasure Code)**: MinIO核心的数据保护机制，将数据拆分为数据块(data block)和校验块(parity block)，分散存储在不同磁盘/节点。即使部分磁盘损坏也能恢复数据
- **纠删码原理**: 将文件切分为n份数据块，通过矩阵运算生成m份校验块，总共n+m份(分布在不同磁盘)。写入n+m份，读取时需要至少n份(可容忍最多m份丢失)
- **纠删码配置**: 至少需要4个驱动器(node*drive>=4)。默认n/2数据块 + n/2校验块，即容忍n/2个磁盘损坏。如8个驱动器：4数据+4校验，容忍最多4个磁盘损坏
- **纠删码节点要求**: 分布式部署建议4个以上Node，每个Node配置相同数量的Drive。EC计算(n/m配置)由MinIO自动决策
- **分布式MinIO**: 多Node多Drive组成存储集群(如4个Node，每Node 4个Drive = 16个Drive)。使用纠删码保证数据安全，节点间无主从(对等架构)
- **MinIO集群启动**: 需满足`节点数 × 驱动器数 >= 4`且总磁盘数>=4。集群节点身份标识通过启动参数指定所有节点的地址
- **Java SDK(minio)**: Maven引入`io.minio:minio`，创建MinioClient连接，通过`putObject`上传、`getObject`下载、`getPresignedObjectUrl`生成预签名URL
- **上传-PutObjectArgs**: 设置bucket、object名称、文件流(InputStream)、Content-Type、文件大小(-1自动)、元数据
- **下载-GetObjectArgs**: 获取对象流(GetObjectResponse)，同时可获取元数据
- **预签名URL**: `getPresignedObjectUrl(GetPresignedObjectUrlArgs)`生成临时访问链接，不暴露存储认证，有效期内(如7天)可直接访问/下载对象
- **MinIO Admin SDK**: 管理操作SDK，用于管理Bucket、用户策略、配额、监控等。`minio-admin`库提供丰富管理API
- **生命周期管理**: Bucket可配置生命周期规则，自动过期删除或转换对象(如30天后自动删除临时文件、90天后归档)
- **版本控制(Versioning)**: Bucket级别的版本管理，开启后每次PUT操作生成新版本(versionId)，保留历史版本。可配合生命周期管理自动清理旧版本
- **MinIO vs FastDFS**: FastDFS需要Nginx模块配合访问、文件元数据存储在Tracker+Storage；MinIO自带HTTP(S) API/S3兼容、自带Web管理UI、社区更活跃、使用更简单
- **MinIO vs 阿里云OSS**: MinIO是自建对象存储，适合内网环境和数据私密性要求高的场景；OSS是云服务，免运维按量付费。两者S3 API兼容，代码可平滑迁移
- **OSS S3兼容迁移**: 阿里云OSS等云服务由于兼容S3 API，基于MinIO S3 SDK开发的应用只需修改endpoint和凭据即可切换到OSS，无需修改代码逻辑

# 2. 面试题

## 2.1 MinIO的纠删码(Erasure Code)是如何保证数据安全的？
纠删码将文件切分为N份数据块(data block)，通过数学矩阵运算生成M份校验块(parity block)，总共N+M份分布在不同磁盘上。恢复数据需要至少N份(任何N份，不区分数据和校验)。例如8块磁盘(N=4,M=4)：写入8份(4数据+4校验)，只要至少有4块磁盘可用即可恢复数据(容忍最多4块磁盘损坏)。相比多副本(如3副本)，纠删码空间利用率更高(4+4=50%额外开销 vs 3副本200%额外开销)，但计算开销略高。MinIO默认EC配置为N/2数据+N/2校验(空间利用率为50%)。

## 2.2 MinIO分布式集群的部署要求是什么？
1) 最少4个驱动器(总磁盘数>=4)，建议最少4个Node，每个Node配备相同数量Drive
2) 所有节点的MinIO版本一致、硬件配置尽量一致
3) 节点间网络延迟<10ms，建议同机房内部网络
4) 启动参数指定所有节点地址：`minio server http://node1/data http://node2/data http://node3/data http://node4/data`
5) 不需要额外安装Nginx做代理(MinIO自带HTTP Server)
6) 支持动态扩容(增加节点池Pool)，但需用全新Pool(不能在半满的Pool追加节点)
7) 所有磁盘必须裸盘(无文件系统最好)或使用XFS/ext4等支持direct IO的文件系统

## 2.3 如何使用MinIO Java SDK上传文件和生成预签名URL？
```java
MinioClient minioClient = MinioClient.builder()
    .endpoint("http://minio-server:9000")
    .credentials("accessKey", "secretKey")
    .build();

// 上传文件
minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("my-bucket")
        .object("path/to/file.jpg")
        .stream(inputStream, fileSize, -1)
        .contentType("image/jpeg")
        .build()
);

// 生成预签名URL(有效期1小时)
String presignedUrl = minioClient.getPresignedObjectUrl(
    GetPresignedObjectUrlArgs.builder()
        .bucket("my-bucket")
        .object("path/to/file.jpg")
        .method(Method.GET)
        .expiry(3600)
        .build()
);
```

## 2.4 MinIO和FastDFS对比，选哪个？
| 维度 | MinIO | FastDFS |
|------|-------|---------|
| S3兼容 | 完全兼容AWS S3 API | 不兼容，需要自定义SDK或Nginx模块 |
| 部署 | 单文件启动，易部署 | 需部署Tracker+Storage+Nginx模块 |
| 数据保护 | 纠删码(自修复) | 多副本或依赖存储层RAID |
| 管理界面 | 自带Web管理UI | 无内置UI |
| 社区 | 国际化，活跃 | 中文为主，活跃度下降 |
| API | HTTP REST S3 API | 自定义TCP协议+C HTTP代理 |
| 适用 | 新项目、需S3兼容 | 已有FastDFS的老项目 |

推荐新项目选MinIO：S3兼容(标准API, 易迁移)、部署简单、自带UI、社区活跃。

## 2.5 MinIO如何实现桶的版本控制和生命周期管理？
**版本控制**：
```bash
# 开启版本控制
mc version enable myminio/my-bucket
```
开启后每次PUT操作生成新版本，对象有versionId。默认获取最新版本，可通过versionId获取历史版本。

**生命周期**：
```xml
<!-- 配置规则 -->
<LifecycleConfiguration>
  <Rule>
    <ID>expire-temp-files</ID>
    <Filter><Prefix>temp/</Prefix></Filter>
    <Status>Enabled</Status>
    <Expiration>
      <Days>30</Days>  <!-- 30天后删除 -->
    </Expiration>
  </Rule>
</LifecycleConfiguration>
```
生命周期规则支持：过期删除(Expiration)、版本清理(NoncurrentVersionExpiration)、分层转换(Transition，企业版)。对临时文件、日志等设置自动过期清理。

## 2.6 Java应用中如何使用MinIO替代阿里云OSS？
由于两者都兼容S3 API，切换成本极低：
1) 将Maven依赖从`aliyun-sdk-oss`更换为`minio` 或 使用AWS S3 SDK统一操作
2) 修改配置：endpoint改为MinIO地址，accessKey/secretKey改为MinIO凭据
3) 代码层面：如果使用AWS S3 SDK(推荐)，只需改endpoint和credential，代码零修改
4) 注意：MinIO有的功能(如STS临时授权、跨域复制)可能与OSS不一致，需逐个验证
5) 迁移数据：使用`mc mirror`命令或Rclone工具将OSS数据批量迁移到MinIO

## 2.7 MinIO的预签名URL有什么作用？如何控制安全？
预签名URL允许不拥有MinIO凭据的用户在有效期内访问受保护的对象。常见场景：前端直传(生成上传URL给前端)、临时分享文件(生成下载URL发给用户)、限制访问时间。

安全控制：
1) 设置合理的过期时间(如上传URL 10分钟、下载URL 1小时)
2) 服务端鉴权后再生成预签名URL(权限控制放在后端)
3) URL中包含签名签名防篡改(MinIO自动处理)
4) 设置Bucket Policy为私有(默认)，只有预签名URL能访问
5) 生成的URL包含签名哈希，修改任意参数会失效
