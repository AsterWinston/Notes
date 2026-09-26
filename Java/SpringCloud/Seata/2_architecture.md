
# 1. 三大角色

Seata 定义了三个核心角色：

```
┌──────────────────────────────────────────────────────┐
│                    Seata 架构                          │
│                                                       │
│   ┌──────────┐      注册/上报       ┌──────────┐      │
│   │    TM    │ ──────────────────→ │    TC    │      │
│   │(Business)│ ←────────────────── │ (Server) │      │
│   └────┬─────┘   提交/回滚/结果     └────┬─────┘      │
│        │                                │             │
│   开启全局事务                    分支注册/上报        │
│        │                                │             │
│        ▼                                ▼             │
│   ┌──────────┐    RPC 调用 xid    ┌──────────┐      │
│   │    RM    │ ←───────────────── │    RM    │      │
│   │(Order)   │                    │(Account) │      │
│   └──────────┘                    └──────────┘      │
│     参与全局事务                     参与全局事务      │
└──────────────────────────────────────────────────────┘
```

| 角色 | 全称 | 职责 | 实例 |
|------|------|------|------|
| **TC** | Transaction Coordinator | 维护全局事务和分支事务状态，驱动全局提交或回滚 | Seata Server |
| **TM** | Transaction Manager | 定义全局事务范围：开启、提交、回滚全局事务 | `@GlobalTransactional` 所在方法 |
| **RM** | Resource Manager | 管理分支事务资源，向 TC 注册分支事务并汇报状态 | 各服务的数据库操作 |

# 2. 事务生命周期

## 2.1 全局事务流程

```
TM (订单服务)                    TC Server                  RM (账户服务)
      │                              │                           │
      │ 1. 开启全局事务               │                           │
      │ ───begin(xid)───→             │                           │
      │                              │                           │
      │ 2. 执行本地业务               │                           │
      │    (INSERT order,             │                           │
      │     proxy 自动注册分支1)       │                           │
      │ ───register branch1──→       │                           │
      │                              │                           │
      │ 3. RPC 调用账户服务           │                           │
      │ ─────────────────────────────┼──→ (xid 通过 attachment 传播)
      │                              │                           │
      │                              │    4. 执行本地业务         │
      │                              │    (UPDATE account,       │
      │                              │     proxy 自动注册分支2)    │
      │                              │ ←──register branch2──      │
      │                              │                           │
      │ 5. 提交全局事务               │                           │
      │ ───commit(xid)───→           │                           │
      │                              │    6. 驱动分支2 提交       │
      │                              │ ───branch commit──→       │
      │                              │    7. 驱动分支1 提交       │
      │         ←────ok────          │ ───→ 异步                  │
```

## 2.2 正常提交流程

```
Phase 1 (Prepare):
  - 各 RM 执行业务 SQL
  - AT 模式：生成 undo_log（前置镜像 + 后置镜像）
  - 向 TC 注册分支事务
  - 本地事务提交（释放 DB 锁）

Phase 2 (Commit):
  - TM 向 TC 发起全局提交
  - TC 通知各 RM 提交分支事务
  - RM 异步删除 undo_log
```

## 2.3 异常回滚流程

```
Phase 1 (Prepare):
  - 同上

Phase 2 (Rollback):
  - TM 向 TC 发起全局回滚
  - TC 通知各 RM 回滚分支事务
  - RM 根据 undo_log 生成反向 SQL 并执行
  - RM 删除 undo_log，汇报回滚完成
```

# 3. xid — 全局事务 ID

```java
// RootContext 是 xid 的存储容器
public class RootContext {
    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    public static String getXID() {
        return CONTEXT_HOLDER.get();
    }

    public static void bind(String xid) {
        CONTEXT_HOLDER.set(xid);
    }

    public static String unbind() {
        String xid = CONTEXT_HOLDER.get();
        CONTEXT_HOLDER.remove();
        return xid;
    }
}
```

xid 格式：`{ip}:{port}:{transactionId}`，例如 `192.168.1.100:8091:2039718144`

# 4. TC Server 架构

```
┌─────────────────────────────────────────┐
│               TC Server                  │
│                                          │
│  ┌─────────────┐   ┌──────────────┐     │
│  │ Core         │   │ Storage      │     │
│  │ (事务协调)    │   │ (文件/DB/Redis)│   │
│  └─────────────┘   └──────────────┘     │
│                                          │
│  ┌─────────────┐   ┌──────────────┐     │
│  │ Discovery    │   │ Config       │     │
│  │ (注册中心)    │   │ (配置中心)    │     │
│  └─────────────┘   └──────────────┘     │
│                                          │
│  ┌─────────────┐   ┌──────────────┐     │
│  │ RPC          │   │ HA Cluster   │     │
│  │ (Netty)      │   │ (Raft/Seata)  │    │
│  └─────────────┘   └──────────────┘     │
└─────────────────────────────────────────┘
```

**核心模块**：
- **Core**：接收 TM/RM 请求，驱动全局事务状态流转
- **Storage**：存储事务会话（file/db/redis 三种模式）
- **Discovery**：TC 服务注册到注册中心，供 Client 发现
- **Config**：从配置中心加载 TC 配置
- **RPC**：基于 Netty 的客户端-服务端通信

# 5. 通信流程

```
TM/RM (Client)                    TC (Server)
     │                                 │
     │ ─── 1. begin(xid) ──────→      │  开启全局事务
     │                                 │
     │ ─── 2. branchRegister ──→      │  注册分支事务
     │                                 │
     │ ─── 3. branchReport ────→      │  上报分支状态
     │                                 │
     │ ─── 4. commit/rollback ─→      │  请求全局提交/回滚
     │                                 │
     │ ←── 5. branchCommit ────       │  驱动分支提交
     │ ←── 6. branchRollback ──       │  驱动分支回滚
```

**注意**：Phase 2 是 TC 主动推送给 RM，而非 RM 拉取。所以 RM 需要保持与 TC 的长连接。

# 6. 数据模型

```java
// 全局事务（TC 维护）
GlobalTransaction {
    String xid;            // 全局事务 ID
    int status;            // Begin/Committing/Committed/Rollbacking/Rollbacked
    List<BranchSession> branches;
    long beginTime;
}

// 分支事务（TC 维护每分支状态）
BranchSession {
    long branchId;         // 分支 ID
    String xid;            // 所属全局事务
    String resourceGroupId;// 资源组
    BranchType branchType; // AT/TCC/Saga/XA
    int status;
    String applicationData;// RM 上报的附加数据
}
```
