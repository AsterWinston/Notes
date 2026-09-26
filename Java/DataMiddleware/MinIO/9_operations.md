# 1.mc 命令行工具详解

## 1.1mc 别名管理

```bash
# 添加 MinIO 别名
mc alias set myminio http://localhost:9000 minioadmin minioadmin
mc alias set prod-minio https://prod-minio.example.com:9000 ${AK} ${SK}

# 列出所有别名
mc alias list

# 删除别名
mc alias remove myminio
```

## 1.2Bucket 管理

```bash
# 创建 Bucket
mc mb myminio/my-new-bucket

# 列出 Bucket
mc ls myminio

# 查看 Bucket 信息（大小和对象数）
mc du myminio/mybucket

# 删除 Bucket（必须为空）
mc rb myminio/my-bucket

# 强制删除 Bucket（包括所有对象）
mc rb --force myminio/my-bucket

# 查看 Bucket 生命周期配置
mc ilm ls myminio/mybucket

# 导出 Bucket 生命周期配置
mc ilm export myminio/mybucket --alias myminio
```

## 1.3对象操作

```bash
# 上传文件
mc cp /path/to/local/file.txt myminio/mybucket/

# 上传目录（递归）
mc cp --recursive /path/to/local/dir/ myminio/mybucket/dir/

# 下载文件
mc cp myminio/mybucket/file.txt /path/to/local/

# 移动/重命名对象
mc mv myminio/mybucket/old-name.txt myminio/mybucket/new-name.txt

# 删除对象
mc rm myminio/mybucket/file.txt

# 递归删除
mc rm --recursive myminio/mybucket/logs/

# 强制删除（不确认）
mc rm --recursive --force myminio/mybucket/tmp/

# 查看对象内容（前几行）
mc head myminio/mybucket/file.txt

# 查看对象信息
mc stat myminio/mybucket/file.txt

# 查找对象
mc find myminio/mybucket --name "*.jpg"
mc find myminio/mybucket --newer-than 7d  # 最近7天的对象
mc find myminio/mybucket --older-than 30d  # 30天前的对象
```

## 1.4数据同步（mirror）

```bash
# 本地 → MinIO（单向镜像）
mc mirror /data/local/photos/ myminio/mybucket/photos/

# MinIO → 本地（单向镜像）
mc mirror myminio/mybucket/backup/ /data/local/restore/

# MinIO → MinIO（跨集群同步）
mc mirror myminio/source-bucket/ myminio-remote/target-bucket/

# 选项:
#   --overwrite   覆盖目标文件（即使已存在）
#   --remove      删除目标中存在但源中不存在的文件（真正镜像）
#   --watch       持续监控并自动同步
#   --fake        干运行（不实际传输）

# 持续监控同步（类似 rsync --daemon）
mc mirror --watch /data/incoming/ myminio/mybucket/incoming/
```

## 1.5集群管理（mc admin）

```bash
# 查看集群信息
mc admin info myminio

# JSON 格式输出
mc admin info myminio --json

# 查看集群拓扑
mc admin topology myminio

# 查看服务状态
mc admin service status myminio

# 重启服务
mc admin service restart myminio

# 停止服务（谨慎）
mc admin service stop myminio

# 查看策略列表
mc admin policy list myminio

# 用户管理
mc admin user list myminio
mc admin user add myminio alice Password123!
mc admin user info myminio alice
mc admin user enable myminio alice
mc admin user disable myminio alice
mc admin user remove myminio alice

# 群组管理
mc admin group list myminio
mc admin group add myminio developers alice bob
mc admin group info myminio developers

# 配置管理
mc admin config get myminio        # 查看所有配置
mc admin config set myminio api requests_max=0  # 修改配置
```

## 1.6数据修复（mc admin heal）

```bash
# 扫描并修复所有对象（递归）
mc admin heal --recursive myminio

# 修复特定 Bucket
mc admin heal --recursive myminio/mybucket

# 仅修复（不扫描扫描——扫描已由后台 Scanner 完成）
mc admin heal --recursive --scan=deep myminio/mybucket

# 查看挂起的修复任务
mc admin heal --recursive --pending myminio
```

## 1.7Bucket 配置管理

```bash
# 版本控制
mc version enable myminio/mybucket       # 启用
mc version suspend myminio/mybucket      # 挂起
mc version info myminio/mybucket         # 查看状态

# 对象锁
mc retention set myminio/mybucket GOVERNANCE 30d
mc legalhold set myminio/mybucket/photos/important.jpg

# 加密
mc encrypt set SSE-S3 myminio/mybucket   # 设置默认加密
mc encrypt info myminio/mybucket         # 查看加密配置
mc encrypt clear myminio/mybucket        # 清除加密配置

# 桶策略
mc anonymous set public myminio/mybucket
mc anonymous set private myminio/mybucket
mc anonymous set download myminio/mybucket

# 事件通知
mc event add myminio/mybucket arn:minio:sqs::PRIMARY:webhook \
  --event "put,delete"
mc event list myminio/mybucket
mc event remove myminio/mybucket arn:minio:sqs::PRIMARY:webhook
```

---

# 2.备份与恢复

## 2.1使用 mc mirror 备份

```bash
# 全量备份到另一个 MinIO 集群
mc mirror --watch myminio/prod-bucket backup-minio/prod-bucket-backup

# 备份到本地磁盘
mc mirror --recursive myminio/prod-bucket /backup/minio/$(date +%Y%m%d)/

# 增量备份（利用 S3 复制）
mc replicate add myminio/prod-bucket --remote-bucket backup-bucket \
  --priority 1
```

## 2.2使用 rclone 备份

rclone 支持多后端，适合将 MinIO 数据备份到其他云存储：

```bash
# 安装 rclone
curl https://rclone.org/install.sh | sudo bash

# 配置 rclone 远程
rclone config
# 选择 s3 类型，输入 MinIO endpoint/access_key/secret_key

# 同步 MinIO 到 AWS S3
rclone sync minio:my-bucket s3:my-aws-bucket --progress

# 同步 MinIO 到本地
rclone sync minio:my-bucket /backup/minio --progress

# 增量同步
rclone copy minio:my-bucket /backup/minio --progress
```

## 2.3灾难恢复

```text
灾难恢复场景:

场景1: 单个 Bucket 数据损坏
  1. 从备份集群恢复该 Bucket
  2. mc mirror backup-minio/bucket prod-minio/bucket --overwrite

场景2: 整个站点故障
  1. 在新站点部署 MinIO 集群
  2. 如果配置了 Site Replication: 新集群自动同步
  3. 如果没有: 从备份恢复 mc mirror backup/新集群

场景3: format.json 丢失导致集群无法启动
  1. format.json 记录了集群拓扑（节点-UUID映射）
  2. 从每个磁盘根目录备份中恢复 format.json
  3. 如果所有磁盘的 format.json 都丢失 → 数据永久不可恢复
  4. 预防措施: 定期备份所有节点第一个磁盘的 format.json

场景4: 批量误删
  1. 如果开启了版本控制: 删除的是删除标记，可恢复
  2. 如果开启了复制但目标端未同步删除: 从目标端恢复
  3. 如果以上都不满足: 只能从备份恢复
```

## 2.4备份策略建议

```text
备份层级:

L1 实时备份（秒级）
  方法: Bucket Replication / Site Replication
  目的: 防止硬件故障
  RPO: < 1 秒, RTO: < 1 分钟
  
L2 周期备份（日级）
  方法: mc mirror (cron job)
  目的: 防止误删、逻辑错误
  RPO: 24 小时, RTO: 1-4 小时（取决于数据量）
  
L3 异地灾备（周期）
  方法: mc mirror → 远程站点 或 AWS S3
  目的: 防止站点级灾难
  RPO: 24 小时, RTO: 4-24 小时

L4 合规归档（年级）
  方法: mc cp + WORM 对象锁
  目的: 满足法规要求
  永久保留
```

---

# 3.数据迁移

## 3.1从其他 S3 存储迁移到 MinIO

```bash
# 使用 mc mirror 从 AWS S3 迁移
mc alias set s3 https://s3.amazonaws.com ${AWS_AK} ${AWS_SK}
mc alias set myminio http://minio:9000 ${MINIO_AK} ${MINIO_SK}

# 镜像迁移
mc mirror s3/source-bucket myminio/target-bucket --watch

# 使用 rclone
rclone sync s3:source-bucket minio:target-bucket --progress
```

## 3.2从文件系统迁移到 MinIO

```bash
# 上传本地 NAS/NFS 到 MinIO
mc mirror /mnt/nas/data/ myminio/mybucket/data/

# 保留原有目录结构
mc mirror --recursive /mnt/nas/data/ myminio/mybucket/nas-data/

# 仅上传新文件（跳过已存在的）
mc mirror --recursive --overwrite=false /mnt/nas/data/ myminio/mybucket/
```

## 3.3迁移后验证

```bash
# 验证对象数量
echo "源: $(mc ls --recursive s3/source-bucket | wc -l)"
echo "目标: $(mc ls --recursive myminio/target-bucket | wc -l)"

# 验证总大小
mc du s3/source-bucket
mc du myminio/target-bucket

# 抽样验证文件完整性
mc stat s3/source-bucket/important-file.pdf
mc stat myminio/target-bucket/important-file.pdf
# 比对 ETag
```

---

# 4.升级策略

## 4.1升级注意事项

| 要点 | 说明 |
|------|------|
| 备份 | 升级前备份所有节点的 format.json |
| 版本兼容性 | 检查 Release Notes 中的 Breaking Changes |
| 滚动升级 | 逐节点升级，每次验证节点重新加入集群 |
| 回退计划 | 准备旧版本二进制文件用于紧急回退 |
| 时间窗口 | 选择低业务时段升级 |

## 4.2滚动升级步骤

```bash
# 1. 验证集群当前状态
mc admin info myminio
# 确保所有节点在线，磁盘健康

# 2. 备份所有节点的 format.json
scp minio-1:/mnt/disk1/.minio.sys/format.json /backup/format-minio-1.json
# (对所有节点重复)

# 3. 升级第一个节点
# 在 minio-1 上:
sudo systemctl stop minio
# 备份旧二进制
sudo cp /usr/local/bin/minio /usr/local/bin/minio.old
# 下载新版本
sudo wget https://dl.min.io/server/minio/release/linux-amd64/minio \
  -O /usr/local/bin/minio
sudo chmod +x /usr/local/bin/minio
# 启动
sudo systemctl start minio

# 4. 观察 minio-1 重新加入集群
mc admin info myminio
# 确保 minio-1 重新在线

# 5. 对 minio-2, minio-3, minio-4 重复步骤 3-4
# (每次间隔 5-10 分钟，确保上一节点完全恢复)

# 6. 最终验证
mc admin info myminio
# 所有节点在线，版本一致
```

## 4.3使用 Docker 升级

```bash
# 更新镜像
docker pull minio/minio:latest

# 使用 Docker Compose 滚动升级
# 编辑 docker-compose.yml 更新镜像版本
docker compose up -d --no-deps minio1
# 依次升级 minio2, minio3, minio4
```

---

# 5.容量规划

## 5.1容量计算公式

```text
有效存储容量 = 总物理磁盘容量 × (k/N) − 元数据开销

其中:
  k = 数据块数量
  N = 总磁盘数 (k + m)
  元数据开销 ≈ 2%~5%（取决于平均对象大小）

示例:
  32 块磁盘 × 4TB = 128TB 物理容量
  EC: 16+16, 存储效率: 50%
  有效容量: 128TB × 50% − 5%元数据 ≈ 60.8TB

  如果平均对象大小为 1MB, 元数据开销约 32 × 65536 × 8KB ≈ 16GB
  如果平均对象大小为 10KB, 元数据开销可能超过 10%（存储放大严重）
```

## 5.2容量增长预估

```text
当前数据量: 10TB
月增长率: 15%
 
6个月后: 10 × (1.15)^6  ≈ 23TB
12个月后: 10 × (1.15)^12 ≈ 53TB
24个月后: 10 × (1.15)^24 ≈ 285TB

建议扩容时机: 使用率达到 60-70% 时开始规划
```

## 5.3容量优化策略

- **对象压缩**：对文本类数据使用 Gzip 压缩后上传
- **生命周期管理**：自动清理过期数据
- **版本控制管理**：定期清理非当前版本（30天后删除）
- **去重**：应用层去重（MinIO 本身不提供文件级去重，因为 S3 协议限制）
- **存储分层**：冷数据迁移到低成本存储

---

# 6.常见故障排查

## 6.1节点无法加入集群

```bash
# 症状: 新节点或重启后的节点长时间处于 Offline 状态

# 排查步骤:
# 1. 检查网络连通性
ping minio-1.example.com
telnet minio-1.example.com 9000

# 2. 检查 DNS 解析
nslookup minio-1.example.com

# 3. 检查时间同步
ntpq -p
# 如果偏差 > 5 秒，调整 NTP 设置

# 4. 检查 format.json 一致性
cat /mnt/disk1/.minio.sys/format.json
# 与正常节点比对 UUID

# 5. 检查日志
journalctl -u minio -f

# 6. 检查防火墙规则
sudo firewall-cmd --list-all
```

## 6.2磁盘空间不足

```bash
# 检查各节点容量
mc admin info myminio

# 查找大文件和过期对象
mc find myminio/mybucket --larger-than 100MB
mc find myminio/mybucket --older-than 90d

# 紧急清理:
# 1. 删除不需要的临时文件
mc rm --recursive --force myminio/mybucket/tmp/

# 2. 强制过期非当前版本（如果开启了版本控制）
# 通过生命周期规则自动清理

# 3. 清理未完成的分片上传
# 查看: 
mc admin config get myminio api
# 设置生命周期规则清理 7 天前的未完成分片
```

## 6.3访问被拒绝 (AccessDenied)

```bash
# 检查用户凭证
mc admin user info myminio alice

# 检查用户策略
mc admin policy list myminio
mc admin user info myminio alice

# 检查 Bucket 策略
mc anonymous info myminio/mybucket

# 检查对象锁状态（WORM 对象无法删除）
mc legalhold info myminio/mybucket/object-key

# 检查 IAM 策略是否生效
mc admin policy info myminio readonly
```

## 6.4上传大文件失败

```bash
# 1. 检查网络超时设置
# 客户端: 增加 writeTimeout / readTimeout
# 反向代理: 增加 proxy_read_timeout

# 2. 检查 Nginx 配置
# nginx.conf
proxy_connect_timeout 300;
proxy_send_timeout 300;
proxy_read_timeout 300;
client_max_body_size 0;  # 不限制上传大小
proxy_request_buffering off;

# 3. 检查 MinIO API 请求超时
mc admin config get myminio api

# 4. 使用分片上传（防止单次请求超时）
# 参见 4_java_sdk.md 第 10 节
```

## 6.5集群性能下降

```bash
# 1. 检查是否有磁盘故障
mc admin info myminio | grep -i "offline\|down\|healing"

# 2. 检查是否有正在进行的 Heal 操作
mc admin heal --recursive --pending myminio

# 3. 检查磁盘 I/O
iostat -x 1

# 4. 检查网络
sar -n DEV 1 10

# 5. 查看慢请求
mc admin trace myminio --verbose
```

---

# 7.生产环境检查清单

## 7.1部署前检查

```text
硬件:
  □ 每节点最少 4 块磁盘（推荐 8-16 块）
  □ 所有磁盘型号和容量一致
  □ 使用 NVMe 或企业级 SSD（生产环境）
  □ 确认磁盘直接挂载（JBOD），不使用 RAID
  □ 网络带宽满足要求（节点间 >= 25Gbps）
  □ 配置负载均衡器（Nginx/HAProxy/F5）
  □ 配置 NTP 时间同步

操作系统:
  □ 使用 XFS 文件系统（推荐）
  □ 挂载选项: noatime,nodiratime
  □ 配置内核参数（sysctl）
  □ 设置文件描述符限制（>= 1000000）
  □ 配置防火墙规则
  □ 关闭透明大页（THP）
  □ 设置 CPU 性能模式

MinIO 配置:
  □ Root 用户使用强密码
  □ 配置 TLS/SSL 证书
  □ 设置 MINIO_ROOT_USER 和 MINIO_ROOT_PASSWORD
  □ 配置 systemd 服务（自动重启）

监控:
  □ 配置 Prometheus 指标采集
  □ 导入 Grafana Dashboard
  □ 设置告警规则
  □ 配置审计日志采集

安全:
  □ 创建独立 IAM 用户/服务账号（不用 root）
  □ 应用最小权限策略
  □ 对关键 Bucket 启用对象锁
  □ 对关键 Bucket 启用版本控制
```

## 7.2部署后验证

```text
  □ 所有节点在线 (mc admin info)
  □ 所有磁盘健康 (Drives: N/N OK)
  □ 上传测试文件成功
  □ 下载测试文件成功
  □ 验证对象 ETag 正确
  □ 验证预签名 URL 功能
  □ 验证 Console 可访问
  □ 验证 HTTPS 正常工作
  □ 验证 IAM 权限生效
  □ 备份 format.json
  □ 配置备份策略并验证
  □ 模拟磁盘故障并验证自动修复
```
