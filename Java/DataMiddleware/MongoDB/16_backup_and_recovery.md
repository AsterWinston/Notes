
# 1. 备份策略概述

MongoDB提供多种备份方式，从逻辑备份到物理备份，各有适用场景：

| 备份方式 | 工具 | 粒度 | 恢复速度 | 适用场景 |
|---------|------|------|---------|---------|
| 逻辑备份 | mongodump/mongorestore | 集合 | 慢 | 小数据集、跨版本迁移 |
| 物理备份 | 文件系统快照 | 全库 | 快 | 大数据集、快速恢复 |
| 文件拷贝 | cp/rsync | 全库 | 中 | 紧急备份 |
| 云端备份 | Ops Manager/Cloud | — | — | 企业生产环境 |
| 副本集从节点备份 | — | — | — | 备份不阻塞主节点 |

# 2. mongodump / mongorestore

## 2.1 基础备份

```bash
# 全量备份所有数据库
mongodump --host localhost --port 27017 --out /backup/$(date +%Y%m%d)

# 备份单个数据库
mongodump --db mydb --out /backup/

# 备份单个集合
mongodump --db mydb --collection users --out /backup/

# 带认证的备份
mongodump --host localhost --port 27017 \
    -u backupUser -p password \
    --authenticationDatabase admin \
    --out /backup/

# 备份时排除某些集合
mongodump --db mydb --excludeCollection logs --out /backup/

# 压缩备份
mongodump --gzip --archive=/backup/mydb.archive
```

## 2.2 高级选项

```bash
# 指定读关注（确保一致备份）
mongodump --readPreference secondary \
    --readConcern majority \
    --db mydb --out /backup/

# 指定查询条件（部分备份）
mongodump --db mydb --collection orders \
    --query '{ "status": "completed" }' \
    --out /backup/

# 限制oplog（用于PITR）
mongodump --oplog --out /backup/
# --oplog会记录备份期间的oplog，用于restore时恢复到特定时间点

# 备份到归档文件
mongodump --archive=/backup/full_backup.archive --gzip

# 查看备份进度
mongodump --db mydb --out /backup/ 2>&1 | tee backup.log
```

## 2.3 基础恢复

```bash
# 全量恢复
mongorestore /backup/20240115/

# 恢复单个数据库
mongorestore --db mydb /backup/mydb/

# 恢复单个集合
mongorestore --db mydb --collection users /backup/mydb/users.bson

# 从归档文件恢复
mongorestore --archive=/backup/full_backup.archive

# 带压缩的归档恢复
mongorestore --gzip --archive=/backup/full_backup.archive.gz

# 恢复时删除已存在的集合数据
mongorestore --drop /backup/mydb/

# 恢复到不同的数据库名
mongorestore --nsFrom "mydb.*" --nsTo "mydb_restored.*" /backup/mydb/
```

## 2.4 时间点恢复（PITR）

```bash
# 1. 备份时开启oplog记录
mongodump --oplog -o /backup/base/

# 2. 恢复基础备份
mongorestore --oplogReplay /backup/base/

# 3. 如果要将oplog应用到特定时间点
# 先分析dump目录下的oplog.bson
bsondump /backup/base/oplog.bson | head -100

# 使用mongorestore的--oplogLimit选项
mongorestore --oplogReplay \
    --oplogLimit "1705651200:1" \
    /backup/base/
```

# 3. 文件系统快照备份

## 3.1 LVM快照（Linux）

```bash
# 前提：数据目录在LVM卷上，且有journal日志
# 1. 创建快照
lvcreate --size 10G --snapshot --name mongo_snap /dev/vg_data/mongo_lv

# 2. 挂载快照并备份
mount /dev/vg_data/mongo_snap /mnt/snap
tar -czf /backup/mongo_$(date +%Y%m%d).tar.gz -C /mnt/snap .

# 3. 卸载并删除快照
umount /mnt/snap
lvremove /dev/vg_data/mongo_snap
```

## 3.2 同步数据文件（需要锁定写操作）

```bash
# 方法一：fsyncLock（锁定写操作，不建议在生产环境长时间使用）
# 1. 锁定
mongosh --eval "db.fsyncLock()"

# 2. 复制数据文件
rsync -av /data/db/ /backup/data_backup/

# 3. 解锁
mongosh --eval "db.fsyncUnlock()"

# 方法二：使用从节点备份（不锁定主节点）
# 1. 在从节点上锁定
mongosh --host secondary --eval "db.fsyncLock()"

# 2. 从节点上复制文件
rsync -av /data/db/ /backup/data_backup/

# 3. 解锁从节点
mongosh --host secondary --eval "db.fsyncUnlock()"
```

# 4. 企业级备份：Ops Manager

Ops Manager（前身为MongoDB Cloud Manager）提供图形化的自动备份管理：

- **增量备份**：仅备份变更的数据块
- **快照保留策略**：自定义保留时长和频率
- **时间点恢复**：支持恢复到任意时间点
- **查询快照**：可以在备份数据上执行查询（Queryable Backup）
- **自动快照**：按计划自动备份（如每6小时增量、每天全量）

# 5. 备份脚本示例

## 5.1 全量备份Shell脚本（Linux）

```bash
#!/bin/bash
# 每日全量备份脚本

BACKUP_DIR="/backup/mongodb"
KEEP_DAYS=7
MONGODUMP="/usr/bin/mongodump"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/mongo_backup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "开始MongoDB备份..."

# 创建备份目录
mkdir -p "$BACKUP_DIR/$DATE"

# 执行备份
$MONGODUMP \
    --host localhost \
    --port 27017 \
    --username backupUser \
    --password "${MONGO_BACKUP_PASSWORD}" \
    --authenticationDatabase admin \
    --oplog \
    --gzip \
    --out "$BACKUP_DIR/$DATE"

if [ $? -eq 0 ]; then
    log "备份成功: $BACKUP_DIR/$DATE"
    
    # 压缩备份文件
    tar -czf "$BACKUP_DIR/mongo_backup_$DATE.tar.gz" -C "$BACKUP_DIR" "$DATE"
    rm -rf "$BACKUP_DIR/$DATE"
    
    # 清理过期备份
    find "$BACKUP_DIR" -name "mongo_backup_*.tar.gz" -mtime +$KEEP_DAYS -delete
    log "已清理${KEEP_DAYS}天前的备份"
else
    log "备份失败！"
    exit 1
fi

log "备份完成"
```

## 5.2 Java自动备份集成

```java
@Component
public class MongoBackupService {
    
    @Value("${mongodb.backup.path}")
    private String backupPath;
    
    @Value("${mongodb.uri}")
    private String mongoUri;
    
    @Scheduled(cron = "0 0 2 * * ?")  // 每天凌晨2点
    public void scheduledBackup() {
        try {
            executeBackup();
            cleanupOldBackups(7);  // 保留7天
        } catch (Exception e) {
            // 发送告警
            log.error("备份失败", e);
        }
    }
    
    public void executeBackup() throws Exception {
        String timestamp = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
        String backupDir = backupPath + "/" + timestamp;
        
        ProcessBuilder pb = new ProcessBuilder(
            "mongodump",
            "--uri", mongoUri,
            "--out", backupDir,
            "--gzip",
            "--oplog"
        );
        pb.redirectErrorStream(true);
        Process process = pb.start();
        
        int exitCode = process.waitFor();
        if (exitCode != 0) {
            throw new RuntimeException("mongodump failed with code " + exitCode);
        }
        
        log.info("备份完成: {}", backupDir);
    }
    
    private void cleanupOldBackups(int keepDays) {
        File backupDir = new File(backupPath);
        LocalDateTime cutoff = LocalDateTime.now().minusDays(keepDays);
        
        if (backupDir.exists()) {
            for (File file : backupDir.listFiles()) {
                if (file.isDirectory()) {
                    try {
                        LocalDateTime fileTime = LocalDateTime.parse(
                            file.getName().substring(0, 15),
                            DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
                        if (fileTime.isBefore(cutoff)) {
                            deleteRecursively(file);
                            log.info("清理过期备份: {}", file.getName());
                        }
                    } catch (Exception ignored) {}
                }
            }
        }
    }
    
    private void deleteRecursively(File file) {
        if (file.isDirectory()) {
            for (File f : file.listFiles()) {
                deleteRecursively(f);
            }
        }
        file.delete();
    }
}
```

# 6. 恢复流程与验证

## 6.1 恢复前检查

```javascript
// 1. 确认备份完整性
// 检查备份文件是否存在且非空

// 2. 确定恢复范围
// 全量恢复 or 增量恢复

// 3. 评估恢复时间
// mongorestore速度约 10,000~50,000 docs/sec（取决于文档大小）
// 大数据集可能需要数小时

// 4. 停机窗口评估
// 是否需要停机维护窗口
```

## 6.2 全量恢复步骤

```bash
# 步骤1: 如果恢复现有数据库，建议先删除目标数据库
mongosh --eval "use targetDb; db.dropDatabase()"

# 步骤2: 执行恢复
mongorestore \
    --host localhost:27017 \
    --username admin --password pass \
    --authenticationDatabase admin \
    --drop \
    --numInsertionWorkersPerCollection 4 \
    /backup/mydb/

# --drop: 删掉已有同名校验数据
# --numInsertionWorkersPerCollection: 并行恢复线程数

# 步骤3: 恢复Oplog（如果备份时使用了--oplog）
mongorestore --oplogReplay /backup/oplog.bson

# 步骤4: 验证恢复结果
mongosh --eval "
    use targetDb;
    print('文档数:' + db.collection.countDocuments({}));
    print('索引数:' + db.collection.getIndexes().length);
"
```

## 6.3 备份恢复测试

```bash
#!/bin/bash
# 定期验证备份可恢复性

RESTORE_DB="backup_test_db"
BACKUP_PATH="/backup/mongodb/latest"

echo "开始备份恢复测试..."

# 恢复到测试数据库
mongorestore \
    --db $RESTORE_DB \
    --drop \
    $BACKUP_PATH

if [ $? -eq 0 ]; then
    # 验证
    COUNT=$(mongosh --quiet --eval "db.getSiblingDB('$RESTORE_DB').users.countDocuments({})")
    echo "恢复成功，users集合文档数: $COUNT"
    
    # 清理测试数据库
    mongosh --quiet --eval "db.getSiblingDB('$RESTORE_DB').dropDatabase()"
else
    echo "备份恢复失败！！！"
    # 发送告警
    exit 1
fi
```

# 7. 灾难恢复场景

## 7.1 单节点故障恢复

```
场景：mongod进程crash或数据文件损坏

恢复步骤：
1. 停止mongod进程
2. 检查journal日志完整性
3. 重启mongod（MongoDB会自动从journal恢复）
4. 如果自动恢复失败：
   - 删除数据目录
   - 从备份恢复
   - 如果有多个从节点，可从从节点同步数据
```

## 7.2 副本集主节点故障

```
场景：主节点宕机，副本集自动选举新主节点

恢复步骤：
1. 副本集自动failover（10秒内）
2. 等待旧主节点恢复
3. 旧主以Secondary身份重新加入
4. 如果oplog不够：执行Initial Sync
```

## 7.3 全副本集故障

```
场景：整个副本集不可用（如数据中心断电）

恢复步骤：
1. 恢复硬件/网络
2. 从备份恢复数据到主节点
3. 重新初始化副本集
4. 添加其他节点并同步
```

## 7.4 人为误删恢复

```bash
# 使用延迟从节点（Delayed Secondary）
# 配置：设置delay从节点比主节点晚1小时
var cfg = rs.conf()
cfg.members[2].secondaryDelaySecs = 3600  # 延迟1小时
rs.reconfig(cfg)

# 误删除后：
# 1. 立即在延迟从节点上stop mongod
# 2. mongodump导出数据
# 3. 恢复到主节点
```

# 8. 备份checklist

**备份策略制定**：
- [ ] 确定RPO（恢复点目标）：能容忍丢失多少数据
- [ ] 确定RTO（恢复时间目标）：能容忍多长时间的停机
- [ ] 根据RPO/RTO选择备份策略（频率、类型）

**定期执行**：
- [ ] 每日全量备份或增量备份
- [ ] 备份文件加密（敏感数据）
- [ ] 备份文件异地存储
- [ ] 定期备份恢复测试（建议每月）

**监控告警**：
- [ ] 备份成功/失败告警
- [ ] 备份文件大小异常告警
- [ ] 备份存储空间告警

**文档化**：
- [ ] 备份恢复操作文档
- [ ] 紧急联系人和升级路径
- [ ] 灾难恢复手册
