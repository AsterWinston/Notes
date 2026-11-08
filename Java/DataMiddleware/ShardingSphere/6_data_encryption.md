数据加密解决的是"**库里落盘的数据不能是明文敏感信息**"这一合规刚需（等保、GDPR、个保法）。ShardingSphere 的方案是**字段级透明加解密**：应用照常读写明文列，内核在 SQL 改写阶段把明文列换成密文列、对写入参数加密、对读取结果解密，业务代码完全无感。它不是数据库加密（TDE），而是应用侧的列级加密，密钥掌握在应用/密钥服务手里而非 DBA 手里。本篇讲加解密改写原理、算法家族、最棘手的"密文怎么支持 LIKE/等值检索"，以及密钥管理纪律。配置为 5.5.x。

# 1. 概述

ShardingSphere 提供**字段级透明加密**，数据在存储时加密，读取时解密，应用层无感知。

```
Application                    ShardingSphere                    MySQL
   │                                │                              │
   │ INSERT name='Alice'            │                              │
   │ ───────────────────────────→   │                              │
   │                                │ Encrypt('Alice') → 'xxx'     │
   │                                │ ─────────────────────────→   │ INSERT name_cipher='xxx'
   │                                │                              │
   │ SELECT name                    │                              │
   │ ←───────────────────────────   │ Decrypt('xxx') → 'Alice'    │ SELECT name_cipher
```

**透明**的代价是**能力边界**：加密列丧失了明文可提供的绝大多数 SQL 语义——不能排序、不能模糊匹配、不能范围查询、不能当分片键。ShardingSphere 用"多列冗余"的思路部分抢救这些能力（第 4 节），这也是数据加密规则最容易被误解的地方。

> **设计争议**：字段加密放**应用层（ShardingSphere）**还是**数据库层（TDE/列加密函数）**？应用层密钥可控、跨库统一、DBA 看不到明文，但加解密算力占用应用 JVM、且必须保证所有写入路径都过 ShardingSphere（有旁路导入就漏明文）；数据库层对应用零侵入、备份/复制天然带密文，但密钥托管在 DB 且不同数据库能力不齐。合规要求"运维不可见明文"时通常选应用层加密。

# 2. 加密配置

5.5.x 用**嵌套结构**（`cipher`/`assistedQuery`/`likeQuery` 各是一个 `name + encryptorName` 对象），取代了早期 `cipherColumn` 扁平写法：

```yaml
rules:
  - !ENCRYPT
    tables:
      t_user:
        columns:
          user_name:
            cipher:
              name: user_name_cipher       # 密文列
              encryptorName: aes_encryptor
          phone:
            cipher:
              name: phone_cipher
              encryptorName: aes_encryptor
          email:
            cipher:
              name: email_cipher
              encryptorName: aes_encryptor
            assistedQuery:                 # 等值检索辅助列（确定性摘要）
              name: email_query
              encryptorName: md5_encryptor
          address:
            cipher:
              name: address_cipher
              encryptorName: aes_encryptor
            likeQuery:                     # 模糊检索辅助列
              name: address_like
              encryptorName: char_digest_like
    encryptors:
      aes_encryptor:
        type: AES
        props:
          aes-key-value: 1234567890abcdef
          digest-algorithm-name: SHA-1     # 密钥派生摘要
      md5_encryptor:
        type: MD5
      char_digest_like:
        type: CHAR_DIGEST_LIKE
```

> **注意**：早期 5.0 文档里能看到的 `cipherColumn: user_name_cipher` + `assistedQueryColumn: ...` 扁平写法，在 5.5.x 已换成上面 `cipher.name` / `assistedQuery.name` 结构，从旧版本迁移要改写。加密规则里逻辑列名（`user_name`）与密文列名（`user_name_cipher`）都指向真实表，物理表需自己建好密文列。

# 3. 加密算法

| 算法 type | 说明 | 可逆 | 特点 |
|-----------|------|------|------|
| `AES` | 对称加密 | 可解密 | 通用；需 `aes-key-value`，可选 `digest-algorithm-name` 派生 |
| `RC4` | 流加密 | 可解密 | 性能高，安全性弱于 AES |
| `SM4` | 国密对称加密 | 可解密 | 国产化替代 AES |
| `MD5` | 单向摘要 | 不可逆 | 密码类，或做等值检索辅助摘要 |
| `SM3` | 国密摘要 | 不可逆 | 替代 MD5 |
| `CHAR_DIGEST_LIKE` | 逐字符摘要 | 不可逆 | 专为 `LIKE '%x%'` 设计（第 4.2） |

```yaml
encryptors:
  sm4_encryptor:
    type: SM4
    props:
      sm4-key-value: 1234567890abcdef
```

> **注意**：RC4、MD5 在安全审计中通常被判"弱算法"，生产加密列建议 AES-256 或国密 SM4；摘要算法只用于检索辅助/密码场景，不要拿来"加密"需要还原的数据。

# 4. 检索方案：等值、LIKE、辅助查询器

加密字段无法直接 LIKE（密文不保持原文的字符特征）——这是加密表最大的功能损失，ShardingSphere 用辅助列分级解决。

## 4.1 等值查询（自动支持，无需辅助列）

ShardingSphere 内置 AES 加密器是**确定性**的（同明文同密文，由 `aes-key-value` + 摘要派生固定密钥流），`WHERE phone = '明文'` 会被改写成 `WHERE phone_cipher = Encrypt('明文')` 直接命中密文列：

```sql
-- 逻辑 SQL
SELECT * FROM t_user WHERE phone = '13800001111';
-- 改写后的物理 SQL（参数被就地加密）
SELECT * FROM t_user WHERE phone_cipher = 'encrypted_value';
```

> **设计争议**：确定性是功能换安全的交易——**同明文同密文**意味着密文列暴露频率信息，低熵字段（手机号、身份证）可被频次分析/彩虹表攻击。缓解手段：高敏列改用 `assistedQuery`（MD5 摘要可配 `salt`）做等值检索、密文列本身不落可猜测值；若自定义 `Encryptor` 插件引入随机 IV，等值查询会**直接失效**——这是"本地能查、线上查不到"的经典根因，实现加密器时必须明确是否要支持密文等值。

## 4.2 模糊查询（CHAR_DIGEST_LIKE 辅助列）

`LIKE '%关键词%'` 用逐字符摘要辅助列把"子串匹配"翻译成"摘要集合的连续匹配"：

```sql
-- ❌ 无法工作：密文 LIKE 无意义
SELECT * FROM t_user WHERE address LIKE '%朝阳%';

-- ✓ 走 likeQuery 辅助列：明文逐字符摘要后可做 LIKE 改写
SELECT * FROM t_user WHERE address LIKE '%朝阳%';
-- 内核把 LIKE 参数转成 address_like 列上的摘要匹配片段
```

配置见第 2 节 `likeQuery`。CHAR_DIGEST_LIKE 牺牲部分精确度（字符级摘要可能碰撞），但把"完全不能模糊搜"变成"能模糊搜、代价可控"。

```yaml
columns:
  address:
    cipher: { name: address_cipher, encryptorName: aes_encryptor }
    likeQuery: { name: address_like, encryptorName: char_digest_like }
```

> **设计考量**：
>
> - **辅助列 = 冗余存储 + 泄露面扩大**：assistedQuery/likeQuery 存的是摘要，本质是"给检索让路而多留一份可反查的线索"。低熵字段（手机号）的 MD5 可被彩虹表还原，辅助列要配加盐/慢摘要思路权衡。
> - **排序、范围、GROUP BY 加密列一律无解**：密文序 = 随机序，需要按加密字段排序/区间时，只能解密后在应用层做，或该字段干脆不加密——提前想清楚"哪些列要参与计算"比事后补救重要。

# 5. 加解密的处理流程

```
SELECT * FROM t_user WHERE email = 'alice@example.com'

1. Parse：解析 WHERE 条件
2. Rewrite：
   - email → email_cipher（密文列改写）
   - 'alice@example.com' → Encrypt('alice@example.com')
   - 改写为：SELECT email_cipher FROM t_user WHERE email_cipher = 'encrypted_value'
3. Route → Execute → Merge
4. Result Decrypt：email_cipher → email（结果集按逻辑列名解密返回）
```

**关键机制**：改写阶段同时替换 **列引用** 和 **参数值**——SELECT 投影里把明文列换成密文列（或辅助列）、WHERE 里等值参数就地加密、INSERT 里同时写入明文列映射的密文列；归并阶段把密文列按 `cipher→逻辑列` 映射解密回填。因为都发生在 SQL 层，ORM 拿到的 ResultSet 列名仍是逻辑明文列名，应用无感。

# 6. 与分片组合

```yaml
rules:
  - !SHARDING
    tables:
      t_user:
        actualDataNodes: ds${0..1}.t_user
        databaseStrategy:
          standard:
            shardingColumn: user_id      # 分片键用未加密的 user_id
            shardingAlgorithmName: mod
  - !ENCRYPT
    tables:
      t_user:
        columns:
          phone:
            cipher: { name: phone_cipher, encryptorName: aes_encryptor }
    encryptors:
      aes_encryptor:
        type: AES
        props:
          aes-key-value: 1234567890abcdef

  # shardingAlgorithms 段省略
```

**注意**：**分片键不能是加密列**——路由需要明文参与算法计算，密文既不确定也未必可算。所以按手机号查用户这类需求，要么把手机号所在表按 user_id 分片 + 加密列走辅助检索，要么单独建"手机号→user_id"映射表。

# 7. 存量明文在线迁移

对已有明文数据，推荐用 **Proxy 数据迁移（Migration）Pipeline** 而不是手写 UPDATE，因为它支持增量双写、一致性校验、可回滚：

```sql
-- 1. 物理表先加密文列
ALTER TABLE t_user ADD COLUMN phone_cipher VARCHAR(255);

-- 2. 用迁移规则把 phone 明文读入、加密写入 phone_cipher，并校验
--    （5.x DistSQL，详见 10_governance 迁移章节）
REGISTER MIGRATION SOURCE STORAGE UNIT ds0 (...);
MIGRATE TABLE ds0.t_user INTO ds0.t_user_encrypt;   -- 结合 !ENCRYPT 规则

-- 3. 验证一致后，应用侧切到只读写密文列，最后删明文列（谨慎）
-- ALTER TABLE t_user DROP COLUMN phone;
```

手写 `UPDATE t_user SET phone_cipher = AES_ENCRYPT(...)` 只能处理静态小表，且 `AES_ENCRYPT` 与 ShardingSphere 的 AES 编码方式（Base64、密钥派生）**不一定兼容**，直接混用会导致 ShardingSphere 解不出密文。迁移务必用 ShardingSphere 自己的加密器。

> **警告**：删除明文列前必须确认——备份中明文是否也已处理、历史 binlog/从库是否残留、迁移校验 100% 通过。合规上"曾经明文存过"和"现在仍明文存着"是两个量级的风险。

# 8. 性能影响

| 操作 | 额外开销 | 来源 |
|------|---------|------|
| INSERT | +5-10% | 每加密列一次对称加密 |
| SELECT | +3-8% | 结果集逐行解密 |
| 等值查询 | 接近普通 | 仅参数一次加密 |
| LIKE 查询 | +10-20% | 辅助摘要列计算 |

加解密是 CPU 密集且**不可下推到数据库**的操作，列多、行多的宽表导出会明显变慢，这类批处理建议走未加密的分析副本。

# 9. 最佳实践

1. **加密列不做分片键/排序键/范围键**：需要参与计算的列要么不加密，要么另建明文辅助。
2. **索引建在密文/辅助列上**：对 `phone_cipher`、`email_query` 建索引，明文列不建。
3. **密钥管理走 KMS**：`aes-key-value` 绝不明文写在配置/代码/Git 里，通过外部密钥服务或加密配置项注入。
4. **辅助列按需启用**：只对确有等值/模糊检索需求的列加 assistedQuery/likeQuery，能少一份摘要就少一份泄露面。
5. **算法用强算法**：加密列 AES-256/SM4，摘要列避免裸 MD5（可加盐），满足审计。
6. **密钥轮换预案**：换密钥意味着全量重加密，规划期就要留出双密钥并存的迁移窗口。

# 10. 小结

| 主题 | 一句话结论 |
|------|-----------|
| 定位 | 应用侧字段级透明加解密，密钥在应用不在 DB |
| 配置 | 5.5.x 用 `cipher/assistedQuery/likeQuery` 嵌套结构 |
| 算法 | AES/SM4 可逆加密，MD5/SM3 摘要，CHAR_DIGEST_LIKE 管模糊搜 |
| 等值 | 确定性算法自动支持；非确定走 assistedQuery |
| 边界 | 排序/范围/GROUP BY 加密列无解；分片键不能加密 |
| 迁移 | 用 Migration Pipeline，别手搓 AES_ENCRYPT |
| 纪律 | 密钥进 KMS、辅助列按需、强算法、预留轮换 |

下一篇（7_shadow_database）看影子库：如何在生产环境给压测流量开一条隔离的数据通路。
