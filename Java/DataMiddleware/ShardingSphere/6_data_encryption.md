
# 1. 概述

ShardingSphere 提供**字段级透明加密**，数据在存储时加密，读取时解密，应用层无感知。

```
Application                    ShardingSphere                    MySQL
   │                                │                              │
   │ INSERT name='Alice'            │                              │
   │ ───────────────────────────→   │                              │
   │                                │ Encrypt('Alice') → 'xxx'     │
   │                                │ ─────────────────────────→   │
   │                                │                              │ INSERT name='xxx'
   │                                │                              │
   │ SELECT name                    │                              │
   │ ←───────────────────────────   │                              │
   │ 'Alice'                        │ Decrypt('xxx') → 'Alice'     │
```

# 2. 加密配置

```yaml
rules:
  - !ENCRYPT
    tables:
      t_user:
        columns:
          user_name:
            cipherColumn: user_name_cipher     # 密文列
            encryptorName: aes_encryptor
          phone:
            cipherColumn: phone_cipher
            encryptorName: aes_encryptor
          email:
            cipherColumn: email_cipher
            encryptorName: aes_encryptor
            assistedQueryColumn: email_query   # 查询辅助列

    encryptors:
      aes_encryptor:
        type: AES
        props:
          aes-key-value: 1234567890abcdef
```

# 3. 加密算法

| 算法 | 说明 | 特点 |
|------|------|------|
| `AES` | 对称加密 | 可解密，支持 LIKE 查询（辅助列） |
| `MD5` | 单向哈希 | 不可逆，用于密码等不可解密字段 |
| `RC4` | 流加密 | 性能高，安全性较低 |
| `SM3` | 国密哈希 | 国产标准 |
| `SM4` | 国密加密 | 国产标准 |

# 4. 查询辅助器

加密字段无法直接进行 LIKE 查询（密文不保持原文的字符特征）：

```sql
-- ❌ 无法工作：密文 LIKE 无意义
SELECT * FROM t_user WHERE user_name LIKE '%Alice%';

-- ✓ 查询辅助列：对原文做确定性哈希
SELECT * FROM t_user
WHERE email_query = queryAssisted('alice@example.com');
```

```yaml
columns:
  email:
    cipherColumn: email_cipher
    encryptorName: aes_encryptor
    assistedQueryColumn: email_query
    assistedQueryEncryptorName: md5_encryptor
```

# 5. 加密与查询的处理流程

```
SELECT * FROM t_user WHERE email = 'alice@example.com'

1. Parse：解析 WHERE 条件
2. Rewrite：
   - email → email_cipher（加密列改写）
   - WHERE 条件中的 'alice@example.com' → Encrypt('alice@example.com')
   - 改写为：SELECT * FROM t_user WHERE email_cipher = 'encrypted_value'
3. Route → Execute → Merge
4. Result Decrypt：email_cipher → email（解密返回）
```

# 6. 与分片组合

```yaml
rules:
  - !SHARDING
    tables:
      t_user:
        actualDataNodes: ds${0..1}.t_user
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: mod

  - !ENCRYPT
    tables:
      t_user:
        columns:
          phone:
            cipherColumn: phone_cipher
            encryptorName: aes_encryptor
```

**注意**：分片键不能是加密列，否则分片路由无法工作（分片需要原文）。

# 7. 在线加密字段迁移

```sql
-- 添加密文列（如果表已有数据）
ALTER TABLE t_user ADD COLUMN phone_cipher VARCHAR(255);

-- 数据迁移脚本
UPDATE t_user SET phone_cipher = AES_ENCRYPT(phone, 'key');

-- 验证后删除原文列（谨慎）
-- ALTER TABLE t_user DROP COLUMN phone;
```

# 8. 性能影响

| 操作 | 性能开销 | 说明 |
|------|---------|------|
| INSERT | +5-10% | 仅加密操作 |
| SELECT | +3-8% | 解密开销 |
| LIKE 查询 | +10-20% | 辅助列计算 |

# 9. 最佳实践

1. **加密列不能做分片键**：分片需要明文进行路由计算
2. **索引在密文列上**：对 `cipherColumn` 建索引而非原文列
3. **密钥管理**：生产环境通过密钥管理服务（KMS）管理 `aes-key-value`
4. **辅助列策略**：仅对需要 LIKE/等值查询的列使用辅助列
5. **避免加密 ORDER BY**：密文排序无业务意义
