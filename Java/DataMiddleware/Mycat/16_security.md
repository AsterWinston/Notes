
# 1. 用户认证

## 1.1 conf/users.json

```json
{
  "users": {
    "root": {
      "password": "123456",
      "schemas": ["order_db", "user_db"],
      "ipWhiteList": "192.168.1.0/24,10.0.0.100"
    },
    "readonly": {
      "password": "readonly123",
      "schemas": ["order_db"],
      "readOnly": true
    },
    "app_user": {
      "password": "app123",
      "schemas": ["order_db"]
    }
  }
}
```

| 配置项 | 说明 |
|--------|------|
| `password` | 明文或加密密码 |
| `schemas` | 可访问的逻辑 Schema 列表 |
| `ipWhiteList` | IP 白名单（CIDR 格式） |
| `readOnly` | 只读用户（所有写操作被拒绝） |
| `maxCon` | 该用户最大连接数 |

# 2. IP 白名单

```json
{
  "users": {
    "app_user": {
      "password": "app123",
      "schemas": ["order_db"],
      "ipWhiteList": "192.168.1.0/24,10.0.0.0/8"
    }
  }
}
```

支持格式：
- 单个 IP：`192.168.1.100`
- CIDR 网段：`192.168.1.0/24`
- 多个条目：逗号分隔
- 通配：不设置或 `*` 表示所有 IP

# 3. 密码加密

```bash
# Mycat2 提供密码加密工具
cd mycat/lib
java -cp mycat2-1.22-release-jar-with-dependencies.jar \
     io.mycat.util.SecurityUtil \
     my_password

# 输出：
# 加密后：AES:xxxxxxxxxxxxxxxxxxxx
```

```json
{
  "users": {
    "root": {
      "password": "AES:xxxxxxxxxxxxxxxxxxxx",
      "encryptType": "AES"
    }
  }
}
```

# 4. 权限控制

## 4.1 Schema 级别

```json
{
  "users": {
    "order_admin": {
      "password": "admin123",
      "schemas": ["order_db"]       // 只能访问 order_db
    }
  }
}
```

## 4.2 读写权限

```json
{
  "users": {
    "report_user": {
      "password": "report123",
      "schemas": ["order_db"],
      "readOnly": true              // 只读
    }
  }
}
```

## 4.3 连接限制

```json
{
  "users": {
    "app_user": {
      "password": "app123",
      "schemas": ["order_db"],
      "maxCon": 100,                // 最大连接数
      "maxConnectionsPerIP": 10     // 单 IP 最大连接数
    }
  }
}
```

# 5. SQL 注入防护

Mycat2 内置基础 SQL 注入防护：

```java
// 检测常见 SQL 注入模式
public class SQLInjectionGuard {

    private static final Pattern[] DANGEROUS_PATTERNS = {
        Pattern.compile("'.*OR.*'.*=", Pattern.CASE_INSENSITIVE),   // OR 1=1
        Pattern.compile("'.*UNION.*SELECT", Pattern.CASE_INSENSITIVE), // UNION SELECT
        Pattern.compile("'.*--", Pattern.CASE_INSENSITIVE),         // SQL 注释注入
        Pattern.compile("';\\s*DROP", Pattern.CASE_INSENSITIVE),    // DROP TABLE
    };

    public boolean isSafe(String sql) {
        for (Pattern pattern : DANGEROUS_PATTERNS) {
            if (pattern.matcher(sql).find()) {
                return false;
            }
        }
        return true;
    }
}
```

**注意**：Mycat2 的 SQL 注入防护是**基础的**，不能替代 Web 应用层的安全措施（如参数化查询、ORM 框架）。

# 6. SSL/TLS 加密

```yaml
# mycat.yml
server:
  ssl:
    enabled: true
    clientAuth: false
    keyStorePath: conf/ssl/mycat2.keystore
    keyStorePassword: mycat123
    trustStorePath: conf/ssl/mycat2.truststore
    trustStorePassword: mycat123
```

```bash
# 生成自签名证书
keytool -genkey -alias mycat2 \
  -keyalg RSA -keysize 2048 \
  -keystore mycat2.keystore \
  -storepass mycat123 \
  -dname "CN=mycat2.example.com"
```

# 7. 审计日志

```yaml
server:
  audit:
    enabled: true
    logFile: /var/log/mycat2/audit.log
    logFormat: "json"
    includeSQL: true          # 是否记录 SQL
    includeParams: false      # 是否记录参数值（敏感）
    maxSQLLength: 1024
```

```json
// audit.log 示例
{
  "timestamp": "2024-01-15T10:30:00.123",
  "user": "app_user",
  "ip": "192.168.1.100",
  "schema": "order_db",
  "sqlType": "SELECT",
  "sql": "SELECT * FROM orders WHERE id = ?",
  "shards": ["dn0"],
  "rt": 25,
  "rows": 1
}
```

# 8. 后端连接安全

```json
{
  "datasources": {
    "ds1": {
      "user": "mycat",
      "password": "mycat123",
      "security": {
        "ssl": true,
        "sslMode": "PREFERRED"
      }
    }
  }
}
```

后端 MySQL 安全建议：
1. Mycat2 使用专用数据库账号（最小权限）
2. Mycat2 账号仅允许从 Mycat2 所在 IP 连接
3. 后端 MySQL 开启 `require_secure_transport`
4. 定期轮换数据库密码

# 9. 安全检查清单

```
□ users.json 中密码使用加密存储
□ 生产环境密码不要使用默认值
□ root 用户仅限管理 IP 访问
□ 应用用户设置 readOnly、ipWhiteList
□ 启用审计日志
□ 管理端口 (9066) 不对外暴露
□ 限制单用户最大连接数
□ 监控异常连接尝试
□ 定期检查未授权 Schema 访问
□ SSL/TLS 加密敏感数据传输
```
