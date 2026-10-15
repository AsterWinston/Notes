# 1. 拦截器（Interceptor）

拦截器（Interceptor）是 Hibernate 提供的一种原生扩展机制，允许开发者在持久化操作（如 save、delete、update、load）执行前后织入自定义逻辑。它工作在 ORM 内部生命周期，比 JPA 的 EntityListeners 层次更低、控制更精细。

## 1.1 Interceptor 接口

`org.hibernate.Interceptor` 是 Hibernate 拦截器的核心接口，定义了一系列回调方法。开发者通过实现此接口，可以在实体生命周期关键节点注入自定义行为。

**核心方法一览：**

| 方法 | 触发时机 | 返回值说明 |
|------|---------|-----------|
| `onSave()` | 执行 save/persist 之前 | `true` 表示拦截器已处理保存，Hibernate 跳过默认逻辑 |
| `onDelete()` | 执行 delete/remove 之前 | `true` 表示拦截器已处理删除 |
| `onFlushDirty()` | 脏实体被 flush 时（update） | `true` 表示拦截器已处理更新 |
| `onLoad()` | 实体从 ResultSet 加载后 | `true` 表示拦截器已自己完成封装 |
| `findDirty()` | 脏检查阶段，判断实体是否有变更 | 返回脏属性索引数组（null 表示使用默认脏检查） |
| `onCollectionUpdate()` | 集合元素变更时 | void，不拦截默认行为 |
| `onCollectionRemove()` | 集合元素被移除时 | void |
| `onCollectionRecreate()` | 集合被重建时 | void |
| `instantiate()` | 需要实例化实体但尚无构造器可用 | 返回实体实例 |
| `getEntity()` | 通过实体名称获取实例 | 返回实体实例 |
| `preFlush()` | flush 开始前 | void |
| `postFlush()` | flush 结束后 | void |
| `onPrepareStatement()` | SQL 语句准备阶段 | 可修改 SQL |

**Interceptor 的作用范围：**

| 作用级别 | 配置方式 | 生命周期 | 共享性 |
|---------|---------|---------|--------|
| SessionFactory 级别 | `Configuration.setInterceptor()` | 整个 SessionFactory | 所有 Session 共用同一个实例（线程不安全） |
| Session 级别 | `SessionFactory.withOptions().interceptor()` | 单个 Session | 每个 Session 独立实例 |

> **注意**：SessionFactory 级别的 Interceptor 被所有 Session 共享，必须保证线程安全，通常不推荐在生产环境直接使用。Session 级别的 Interceptor 更加灵活和安全。

**完整示例——审计日志拦截器：**

```java
import org.hibernate.*;
import org.hibernate.type.Type;
import java.io.Serializable;
import java.time.LocalDateTime;
import java.util.Iterator;

public class AuditLogInterceptor implements Interceptor {

    private static final String AUDIT_USER = "auditUser";

    public boolean onSave(Object entity, Serializable id, Object[] state,
                          String[] propertyNames, Type[] types) {
        if (entity instanceof AuditableEntity) {
            AuditableEntity auditable = (AuditableEntity) entity;
            auditable.setCreatedTime(LocalDateTime.now());
            auditable.setCreatedBy(getCurrentUser());
            System.out.println("[审计] 新建实体: " + entity.getClass().getSimpleName()
                + ", ID=" + id);
        }
        return false; // false 表示让 Hibernate 继续默认处理
    }

    public boolean onFlushDirty(Object entity, Serializable id,
                                Object[] currentState, Object[] previousState,
                                String[] propertyNames, Type[] types) {
        if (entity instanceof AuditableEntity) {
            AuditableEntity auditable = (AuditableEntity) entity;
            auditable.setUpdatedTime(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
            System.out.println("[审计] 更新实体: " + entity.getClass().getSimpleName()
                + ", ID=" + id);
        }
        return false;
    }

    public boolean onDelete(Object entity, Serializable id, Object[] state,
                            String[] propertyNames, Type[] types) {
        System.out.println("[审计] 删除实体: " + entity.getClass().getSimpleName()
            + ", ID=" + id);
        // 可在此写入删除审计表
        return false;
    }

    public boolean onLoad(Object entity, Serializable id, Object[] state,
                          String[] propertyNames, Type[] types) {
        // 加载后不做额外处理
        return false;
    }

    public int[] findDirty(Object entity, Serializable id,
                           Object[] currentState, Object[] previousState,
                           String[] propertyNames, Type[] types) {
        // 返回 null 使用 Hibernate 默认脏检查
        return null;
    }

    // 集合相关回调
    public void onCollectionUpdate(Object collection, Serializable key) { }
    public void onCollectionRemove(Object collection, Serializable key) { }
    public void onCollectionRecreate(Object collection, Serializable key) { }

    // 实体实例化回调
    public Object instantiate(String entityName, EntityMode entityMode,
                               Serializable id) {
        return null; // null 表示使用默认实例化
    }

    public Object getEntity(String entityName, Serializable id) {
        return null;
    }

    public String getEntityName(Object object) {
        return null;
    }

    public void preFlush(Iterator entities) {
        System.out.println("[审计] 开始 flush，处理实体数: " + getEntityCount(entities));
    }

    public void postFlush(Iterator entities) {
        System.out.println("[审计] flush 完成");
    }

    public void onPrepareStatement(String sql) {
        // 可在此记录或修改 SQL —— 慎用
    }

    public Boolean isTransient(Object entity) {
        return null; // null 表示使用 Hibernate 默认判断
    }

    public String onPrepareStatement(String sql) {
        return sql;
    }

    // 获取当前用户（可从 ThreadLocal、SecurityContext 等获取）
    private String getCurrentUser() {
        return AuditContextHolder.getCurrentUser();
    }

    private int getEntityCount(Iterator entities) {
        int count = 0;
        while (entities.hasNext()) {
            count++;
            entities.next();
        }
        return count;
    }
}

// Session 级别使用
Session session = sessionFactory
    .withOptions()
    .interceptor(new AuditLogInterceptor())
    .openSession();
```

## 1.2 EmptyInterceptor

实现完整 `Interceptor` 接口需要覆盖大量方法，多数场景下只需关心其中几个。`org.hibernate.EmptyInterceptor` 提供了所有方法的空实现（返回 null 或 false），让开发者只重写关心的方法即可。

**示例——自动设置 createTime、updateTime：**

```java
import org.hibernate.EmptyInterceptor;
import org.hibernate.type.Type;
import java.io.Serializable;
import java.time.LocalDateTime;

public class TimestampInterceptor extends EmptyInterceptor {

    @Override
    public boolean onSave(Object entity, Serializable id, Object[] state,
                          String[] propertyNames, Type[] types) {
        return updateTimestamps(entity, state, propertyNames, true);
    }

    @Override
    public boolean onFlushDirty(Object entity, Serializable id,
                                Object[] currentState, Object[] previousState,
                                String[] propertyNames, Type[] types) {
        return updateTimestamps(entity, currentState, propertyNames, false);
    }

    private boolean updateTimestamps(Object entity, Object[] state,
                                     String[] propertyNames, boolean isNew) {
        if (!(entity instanceof Timestamped)) {
            return false;
        }
        boolean changed = false;
        for (int i = 0; i < propertyNames.length; i++) {
            if ("createTime".equals(propertyNames[i]) && isNew) {
                state[i] = LocalDateTime.now();
                changed = true;
            }
            if ("updateTime".equals(propertyNames[i])) {
                state[i] = LocalDateTime.now();
                changed = true;
            }
        }
        return changed;
    }
}
```

`EmptyInterceptor` 大幅减少了样板代码。在审计、时间戳、权限控制等场景下，通常只需覆盖 `onSave`、`onFlushDirty`、`onDelete` 三个方法。

## 1.3 Session-scoped Interceptor

Session 级别的拦截器是推荐的使用方式，通过 `SessionFactory.withOptions().interceptor()` 指定，确保每个 Session 拥有独立的 Interceptor 实例。

**Session 级别使用方式：**

```java
Session session = sessionFactory
    .withOptions()
    .interceptor(new AuditLogInterceptor())
    .openSession();
Transaction tx = session.beginTransaction();

Book book = new Book();
book.setTitle("Hibernate 高级进阶");
session.persist(book);  // 触发 onSave

tx.commit();
session.close();
```

**多租户场景的典型应用——注入租户 ID：**

```java
public class TenantInterceptor extends EmptyInterceptor {

    private final String tenantId;

    public TenantInterceptor(String tenantId) {
        this.tenantId = tenantId;
    }

    @Override
    public boolean onSave(Object entity, Serializable id, Object[] state,
                          String[] propertyNames, Type[] types) {
        if (entity instanceof TenantAware) {
            for (int i = 0; i < propertyNames.length; i++) {
                if ("tenantId".equals(propertyNames[i])) {
                    state[i] = tenantId;
                    return true;
                }
            }
        }
        return false;
    }

    @Override
    public boolean onFlushDirty(Object entity, Serializable id,
                                Object[] currentState, Object[] previousState,
                                String[] propertyNames, Type[] types) {
        // 确保租户 ID 不被篡改
        if (entity instanceof TenantAware) {
            for (int i = 0; i < propertyNames.length; i++) {
                if ("tenantId".equals(propertyNames[i])) {
                    currentState[i] = tenantId;
                    return true;
                }
            }
        }
        return false;
    }
}

// 在每个请求中动态创建带租户上下文的 Session
public Session openTenantSession(String tenantId) {
    return sessionFactory
        .withOptions()
        .interceptor(new TenantInterceptor(tenantId))
        .openSession();
}
```

这种模式在多租户 SaaS 应用中非常常见：每个请求携带当前租户 ID，通过拦截器自动注入到所有持久化实体中，保证租户数据隔离。

## 1.4 与 JPA EntityListeners 的对比

| 对比维度 | Interceptor | JPA EntityListeners |
|---------|------------|---------------------|
| **标准** | Hibernate 原生，非 JPA 标准 | JPA 规范标准 |
| **工作层次** | ORM 内部层级，接近 JDBC | 实体生命周期层级 |
| **操作能力** | 可访问实体状态数组（state）、类型信息，可修改 SQL | 只能访问实体对象，无法直接操作 SQL |
| **注册方式** | 编程式（代码注册） | 声明式（@EntityListeners 注解） |
| **方法开销** | 需实现大量接口方法（EmptyInterceptor 可简化） | 仅需关注需要的生命周期回调 |
| **可移植性** | 与 Hibernate 强耦合 | 标准 JPA，可跨 ORM 移植 |
| **适用场景** | 低层次拦截（时间戳、租户注入、审计 SQL 修改） | 高层次业务逻辑（验证、通知、日志记录） |
| **粒度** | 可按 Session / SessionFactory 配置 | 按实体类配置 |
| **对 state[] 的访问** | 支持（可精确修改属性值） | 不支持 |

**选用建议：**

- 需要操作底层状态数组、修改 SQL、或跨多种实体统一拦截 → **Interceptor**
- 需要遵循 JPA 标准、跨 ORM 框架移植、简单的生命周期回调 → **EntityListeners**
- 多租户、时间戳自动填充等低层次逻辑 → **Interceptor** 控制力更强
- 触发邮件、消息通知、缓存清理等业务逻辑 → **EntityListeners** 更清晰

---

# 2. 事件系统（Event / Listener）

Hibernate 事件系统是比 Interceptor 更现代、更强大的扩展机制。它将每个持久化操作抽象为对应的事件类型，开发者实现对应的监听器接口并注册即可。事件系统是 Hibernate 内部实现的核心，Interceptor 本质上也是事件系统的上层封装。

## 2.1 事件类型体系

Hibernate 的事件类型覆盖了所有持久化操作。每个事件都有 Pre / Post 两个版本（部分有更多子类型），形成完整的前置/后置拦截体系。

**完整事件列表：**

```java
// ─── 插入事件 ───
PreInsertEvent          // 实体插入前（此时 ID 可能尚未分配）
PostInsertEvent         // 实体插入后（ID 已确定）
PostCommitInsertEvent   // 插入事务提交后（仅 JPA）

// ─── 更新事件 ───
PreUpdateEvent          // 实体更新前
PostUpdateEvent         // 实体更新后
PostCommitUpdateEvent   // 更新事务提交后（仅 JPA）

// ─── 删除事件 ───
PreDeleteEvent          // 实体删除前
PostDeleteEvent         // 实体删除后
PostCommitDeleteEvent   // 删除事务提交后（仅 JPA）

// ─── 加载事件 ───
PreLoadEvent            // 实体数据从 ResultSet 读取后、对象填充前
PostLoadEvent           // 实体对象完全构建后

// ─── 集合事件 ───
PreCollectionRecreateEvent   // 集合被整体重建前
PostCollectionRecreateEvent  // 集合被整体重建后
PreCollectionRemoveEvent     // 集合元素被移除前
PostCollectionRemoveEvent    // 集合元素被移除后
PreCollectionUpdateEvent     // 集合元素被更新前
PostCollectionUpdateEvent    // 集合元素被更新后

// ─── Flush 事件 ───
FlushEvent              // flush 操作触发
FlushEntityEvent        // 单个实体在 flush 中被处理时
AutoFlushEvent          // 自动 flush（查询前）
PreFlushEvent           // flush 开始前（不同层级）

// ─── 脏检查事件 ───
DirtyCheckEvent         // 脏检查执行时

// ─── 其他操作事件 ───
EvictEvent              // 实体被从 Session 中驱逐
LockEvent               // 悲观锁获取
RefreshEvent            // 实体从数据库刷新
ReplicateEvent          // 实体跨数据库复制
MergeEvent              // merge 操作
PersistEvent            // persist 操作
SaveOrUpdateEvent       // saveOrUpdate 操作
DeleteEvent             // delete 操作

// ─── 事务相关事件 ───
FlushBeginEvent         // 事务 flush 开始
FlushEndEvent           // 事务 flush 结束
InitializeCollectionEvent // 集合初始化触发
```

事件监听器和对应的接口命名规则非常简单：事件类名去掉 `Event` 后缀，加上 `EventListener` 后缀，例如 `PreInsertEvent` 对应的监听器接口为 `PreInsertEventListener`。

## 2.2 监听器接口

每个事件类型对应一个独立的监听器接口，接口只有一个方法，签名清晰：

| 事件 | 监听器接口 | 方法签名 |
|------|-----------|---------|
| PreInsertEvent | PreInsertEventListener | `boolean onPreInsert(PreInsertEvent event)` |
| PostInsertEvent | PostInsertEventListener | `void onPostInsert(PostInsertEvent event)` |
| PreUpdateEvent | PreUpdateEventListener | `boolean onPreUpdate(PreUpdateEvent event)` |
| PostUpdateEvent | PostUpdateEventListener | `void onPostUpdate(PostUpdateEvent event)` |
| PreDeleteEvent | PreDeleteEventListener | `boolean onPreDelete(PreDeleteEvent event)` |
| PostDeleteEvent | PostDeleteEventListener | `void onPostDelete(PostDeleteEvent event)` |
| PreLoadEvent | PreLoadEventListener | `void onPreLoad(PreLoadEvent event)` |
| PostLoadEvent | PostLoadEventListener | `void onPostLoad(PostLoadEvent event)` |
| FlushEntityEvent | FlushEntityEventListener | `void onFlushEntity(FlushEntityEvent event)` |
| DirtyCheckEvent | DirtyCheckEventListener | `void onDirtyCheck(DirtyCheckEvent event)` |

**Pre* 与 Post* 的区别：**

- **Pre* 系列**返回 `boolean`：返回 `true` 表示监听器已处理该操作，Hibernate 将**跳过**后续的默认处理逻辑（必须慎重）；返回 `false` 表示继续执行 Hibernate 默认行为。
- **Post* 系列**返回 `void`：纯粹的后置通知，不能阻止或改变操作结果。

**事件对象中的关键属性：**

所有事件对象都包含以下核心信息，以 `PreInsertEvent` 为例：

```java
public class PreInsertEvent extends AbstractEvent {
    private Object entity;          // 实体对象
    private Serializable id;        // 实体 ID
    private Object[] state;         // 属性值数组（按属性名顺序）
    private String[] propertyNames; // 属性名数组
    private EntityPersister persister;  // 实体持久化器（可访问元数据）
    private EventSource source;     // 事件来源（即 SessionImplementor）
}
```

`state` 数组和 `propertyNames` 数组是事件系统的核心价值所在：你可以通过索引精确读取和**修改**任意属性的值，这是 EntityListeners 无法做到的低层次控制。

## 2.3 注册监听器

### 2.3.1 方式一：Integrator（推荐）

`org.hibernate.integrator.spi.Integrator` 是 Hibernate 的 SPI 扩展点，在 SessionFactory 构建阶段自动装配。通过 Java SPI（`ServiceLoader`）机制自动发现，无需修改业务代码。

**步骤1：实现 Integrator 接口**

```java
import org.hibernate.boot.Metadata;
import org.hibernate.engine.spi.SessionFactoryImplementor;
import org.hibernate.event.service.spi.EventListenerRegistry;
import org.hibernate.event.spi.EventType;
import org.hibernate.integrator.spi.Integrator;
import org.hibernate.service.spi.SessionFactoryServiceRegistry;

public class AuditEventListenerIntegrator implements Integrator {

    @Override
    public void integrate(Metadata metadata,
                          SessionFactoryImplementor sessionFactory,
                          SessionFactoryServiceRegistry serviceRegistry) {
        // 从 ServiceRegistry 获取 EventListenerRegistry
        EventListenerRegistry listenerRegistry = serviceRegistry
            .getService(EventListenerRegistry.class);

        // 注册监听器到对应事件类型
        listenerRegistry.appendListeners(
            EventType.PRE_INSERT, new AuditPreInsertListener());
        listenerRegistry.appendListeners(
            EventType.PRE_UPDATE, new AuditPreUpdateListener());
        listenerRegistry.appendListeners(
            EventType.POST_DELETE, new AuditPostDeleteListener());
    }

    @Override
    public void disintegrate(Metadata metadata,
                              SessionFactoryImplementor sessionFactory,
                              SessionFactoryServiceRegistry serviceRegistry) {
        // SessionFactory 关闭时的清理工作
    }
}
```

**步骤2：在 `META-INF/services/` 下创建 SPI 配置文件**

文件路径：`src/main/resources/META-INF/services/org.hibernate.integrator.spi.Integrator`

文件内容：

```
com.example.hibernate.AuditEventListenerIntegrator
```

Java SPI 会在类路径下搜索所有 `META-INF/services/org.hibernate.integrator.spi.Integrator` 文件，读取其中的类名并自动实例化注册。无需任何额外配置。

**步骤3：实现具体的事件监听器**

```java
import org.hibernate.event.spi.*;

public class AuditPreInsertListener implements PreInsertEventListener {

    private static final long serialVersionUID = 1L;

    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        String[] propertyNames = event.getPropertyNames();
        Object[] state = event.getState();

        for (int i = 0; i < propertyNames.length; i++) {
            if ("createdAt".equals(propertyNames[i])) {
                state[i] = java.time.LocalDateTime.now();
            }
            if ("createdBy".equals(propertyNames[i])) {
                state[i] = AuditContextHolder.getCurrentUser();
            }
        }
        return false; // false → 让 Hibernate 继续执行
    }
}

public class AuditPreUpdateListener implements PreUpdateEventListener {

    private static final long serialVersionUID = 1L;

    @Override
    public boolean onPreUpdate(PreUpdateEvent event) {
        String[] propertyNames = event.getPropertyNames();
        Object[] state = event.getState();

        for (int i = 0; i < propertyNames.length; i++) {
            if ("updatedAt".equals(propertyNames[i])) {
                state[i] = java.time.LocalDateTime.now();
            }
            if ("updatedBy".equals(propertyNames[i])) {
                state[i] = AuditContextHolder.getCurrentUser();
            }
        }
        return false;
    }
}

public class AuditPostDeleteListener implements PostDeleteEventListener {

    private static final long serialVersionUID = 1L;

    @Override
    public void onPostDelete(PostDeleteEvent event) {
        System.out.println("[事件审计] 实体已删除: "
            + event.getEntity().getClass().getSimpleName()
            + ", ID=" + event.getId());
    }

    @Override
    public boolean requiresPostCommitHandling(EntityPersister persister) {
        // 此处为 false 表示不需要提交后处理
        return false;
    }
}
```

### 2.3.2 方式二：直接通过 EventListenerRegistry（不推荐）

也可以直接在代码中获取 `EventListenerRegistry` 并注册监听器，但这种方式绕过了 Integrator 的标准化机制，丧失了 SPI 自动发现的优势，通常只在集成测试或特殊场景下使用：

```java
// 不推荐：硬编码注册
SessionFactoryImplementor sfImpl = (SessionFactoryImplementor) sessionFactory;
ServiceRegistry serviceRegistry = sfImpl.getServiceRegistry();
EventListenerRegistry listenerRegistry = serviceRegistry
    .getService(EventListenerRegistry.class);
listenerRegistry.appendListeners(
    EventType.PRE_INSERT, new AuditPreInsertListener());
```

这种方式耦合度高，不便于维护，推荐始终使用 Integrator + SPI 方式。

**监听器注册方法说明：**

`EventListenerRegistry` 提供了两种注册方式：

| 方法 | 行为 | 适用场景 |
|------|------|---------|
| `appendListeners()` | 追加到已有监听器列表末尾 | 大多数场景，不影响内置监听器 |
| `prependListeners()` | 插入到已有监听器列表头部 | 需要比内置监听器更早执行 |

## 2.4 替代旧 Interceptor 的示例

如果项目中已有大量基于 `Interceptor` 的代码，建议逐步迁移到事件系统。事件系统提供了更细粒度的控制和类型安全。

**旧代码——基于 Interceptor 的审计逻辑：**

```java
public class LegacyAuditInterceptor extends EmptyInterceptor {
    @Override
    public boolean onSave(Object entity, Serializable id, Object[] state,
                          String[] propertyNames, Type[] types) {
        // 设置创建时间...
        return false;
    }

    @Override
    public boolean onFlushDirty(Object entity, Serializable id,
                                Object[] currentState, Object[] previousState,
                                String[] propertyNames, Type[] types) {
        // 设置更新时间...
        return false;
    }
}
```

**新代码——迁移到事件监听器 + Integrator：**

```java
// 将业务逻辑拆分到独立的监听器中，职责更单一
public class AuditPreInsertListener implements PreInsertEventListener {
    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        setProperty(event.getState(), event.getPropertyNames(),
            "createdAt", LocalDateTime.now());
        setProperty(event.getState(), event.getPropertyNames(),
            "createdBy", AuditContextHolder.getCurrentUser());
        return false;
    }

    private void setProperty(Object[] state, String[] propertyNames,
                              String propertyName, Object value) {
        for (int i = 0; i < propertyNames.length; i++) {
            if (propertyName.equals(propertyNames[i])) {
                state[i] = value;
                return;
            }
        }
    }
}
```

迁移后的优势：每个监听器负责一个明确的关注点（PreInsert / PreUpdate / PostDelete），更容易测试、更容易维护。

---

# 3. JPA 实体监听器（EntityListeners）

JPA 实体监听器是 Java EE / Jakarta EE 标准的一部分，通过注解声明式地在实体生命周期钩子中执行自定义逻辑。它与 Interceptor / EventListener 不同，完全遵循 JPA 规范，不依赖 Hibernate 专有 API。

## 3.1 生命周期回调注解

JPA 定义了以下生命周期回调注解，标注在实体方法或外部监听器类方法上：

| 注解 | 触发时机 | 典型用途 |
|------|---------|---------|
| `@PrePersist` | `EntityManager.persist()` 调用后、INSERT SQL 执行前 | 设置创建时间、验证数据完整性 |
| `@PostPersist` | INSERT SQL 执行后、事务提交前 | 发送通知、记录日志 |
| `@PreUpdate` | 脏实体 flush 时、UPDATE SQL 执行前 | 设置更新时间 |
| `@PostUpdate` | UPDATE SQL 执行后、事务提交前 | 缓存失效、消息发布 |
| `@PreRemove` | `EntityManager.remove()` 调用后、DELETE SQL 执行前 | 级联处理、删除前校验 |
| `@PostRemove` | DELETE SQL 执行后、事务提交前 | 清理关联数据 |
| `@PostLoad` | 实体从数据库加载（或二级缓存获取）后 | 初始化计算字段、解密敏感数据 |

**回调方法签名要求：**

```java
// 有效的方法签名
void anyMethodName()                    // 无参，无返回值
public void anyMethodName()             // 访问修饰符任意
void anyMethodName(Object entity)       // 可选接收实体参数（不常用）
```

- 方法必须返回 void
- 方法可以有零个或一个 Object 参数
- 方法不能声明受检异常（checked exception），可抛出运行时异常（会回滚事务）
- 一个方法上可以标注多个回调注解

**实体内部回调 vs 外部监听器类：**

```java
// 方式1：在实体内部定义回调方法
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private LocalDateTime createdAt;

    @PrePersist
    public void beforePersist() {
        this.createdAt = LocalDateTime.now();
        System.out.println("Product 即将被持久化: " + this.name);
    }

    @PostLoad
    public void afterLoad() {
        System.out.println("Product 已加载: " + this.name);
    }
}
```

内部回调适合简单的、与实体紧密相关的逻辑。对于跨实体共享的逻辑（如审计），应使用外部监听器。

## 3.2 @EntityListeners

`@EntityListeners` 用于声明外部监听器类，将回调逻辑与实体解耦，支持复用。

**声明外部监听器：**

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface EntityListeners {
    Class<?>[] value();  // 监听器类数组
}
```

**完整示例——审计字段自动填充监听器：**

```java
import javax.persistence.*;
import java.time.LocalDateTime;

// 审计接口（标记可审计的实体）
public interface Auditable {
    void setCreatedAt(LocalDateTime dateTime);
    void setCreatedBy(String user);
    void setUpdatedAt(LocalDateTime dateTime);
    void setUpdatedBy(String user);
}

// 外部监听器类
public class AuditEntityListener {

    @PrePersist
    public void prePersist(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            String currentUser = AuditContextHolder.getCurrentUser();
            LocalDateTime now = LocalDateTime.now();
            auditable.setCreatedAt(now);
            auditable.setCreatedBy(currentUser);
            // 首次创建也填充更新字段
            auditable.setUpdatedAt(now);
            auditable.setUpdatedBy(currentUser);
        }
    }

    @PreUpdate
    public void preUpdate(Object entity) {
        if (entity instanceof Auditable) {
            Auditable auditable = (Auditable) entity;
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(AuditContextHolder.getCurrentUser());
        }
    }
}

// 实体使用监听器
@Entity
@EntityListeners(AuditEntityListener.class)
public class Book implements Auditable {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;
    private LocalDateTime createdAt;
    private String createdBy;
    private LocalDateTime updatedAt;
    private String updatedBy;

    // getters / setters 省略
}
```

**多个监听器（执行顺序）：**

```java
@Entity
@EntityListeners({AuditEntityListener.class, ValidationListener.class})
public class Book { ... }
```

多个监听器按声明顺序从左到右执行。当涉及继承时，父类的监听器先执行。

**@ExcludeSuperclassListeners 和 @ExcludeDefaultListeners：**

| 注解 | 作用 |
|------|------|
| `@ExcludeSuperclassListeners` | 排除父类上声明的 @EntityListeners，只执行当前类的监听器 |
| `@ExcludeDefaultListeners` | 排除 ORM 配置文件（orm.xml）中全局定义的默认监听器 |

```java
@Entity
@EntityListeners(ChildListener.class)
@ExcludeSuperclassListeners  // 父类的 EntityListeners 不执行
public class ChildEntity extends ParentEntity {
    // ...
}
```

**监听器执行总顺序：**

```
1. 默认全局监听器（orm.xml 中定义，按声明顺序）
2. 排除默认监听器？→ 跳过 1
3. 父类 @EntityListeners（从顶级父类到当前类，按声明顺序）
4. 排除父类监听器？→ 跳过 3
5. 当前类 @EntityListeners（按声明顺序）
6. 实体内部标记的 @PreXxx / @PostXxx 方法
```

## 3.3 @EntityListeners vs Interceptor vs EventListener 对比

| 对比维度 | EntityListeners | Interceptor | EventListener |
|---------|----------------|------------|---------------|
| **标准** | JPA 规范（JSR 338） | Hibernate 原生 | Hibernate 原生 |
| **层次** | 实体生命周期级 | ORM 内部级 | ORM 内部级（最底层） |
| **注册方式** | 注解 `@EntityListeners` | 编程式 `withOptions()` | SPI `Integrator` |
| **可移植性** | 高（可跨 JPA 实现） | 低 | 低 |
| **操作能力** | 通过实体 setter | 通过 state[] 数组 | 通过 event.getState() 数组 |
| **修改 SQL** | 不支持 | 支持（onPrepareStatement） | 不支持 |
| **Pre返回bool控制** | 无（void） | 返回 boolean | 返回 boolean |
| **学习成本** | 低 | 中 | 中高 |
| **推荐场景** | 标准业务逻辑、跨ORM | 快速原型、轻量拦截 | 插件开发、框架级扩展 |

**选用决策流程：**

```
需要跨 ORM 框架移植?
├─ 是 → EntityListeners（JPA 标准）
└─ 否 → 需要修改底层 state[] 数组或 SQL?
        ├─ 是 → 需要 SPI 自动发现?
        │       ├─ 是 → EventListener + Integrator
        │       └─ 否 → Interceptor (Session-scoped)
        └─ 否 → EntityListeners（够用且标准）
```

- **大多数业务场景** → EntityListeners（简单、标准、可移植）
- **需要操作 state[] 数组精确控制** → EventListener（更细粒度）
- **小团队快速开发** → Interceptor（配置简单，EmptyInterceptor 省代码）
- **开发 Hibernate 插件或框架** → EventListener（最低层次、最大控制力）

---

# 4. 过滤器（Filter）

过滤器（Filter）是 Hibernate 提供的一种**声明式条件注入**机制，允许在不修改查询代码的前提下，自动为实体和关联导航附加额外的 WHERE 条件。它通过注解定义过滤规则，运行时动态启用/禁用，是实现软删除、多租户、行级数据权限的优雅方案。

## 4.1 @FilterDef / @Filter

### 4.1.1 @FilterDef（定义过滤器）

`@FilterDef` 通常标注在包级别（`package-info.java`）或实体类上，用于定义一个过滤器的名称和参数：

```java
@FilterDef(
    name = "tenantFilter",           // 过滤器名称
    parameters = {
        @ParamDef(name = "tenantId", type = String.class)  // 参数定义
    },
    defaultCondition = "tenant_id = :tenantId"  // 默认条件（可选）
)
@FilterDef(
    name = "softDeleteFilter",
    defaultCondition = "deleted = false"
)
package com.example.entity;
```

### 4.1.2 @Filter（应用过滤器到实体）

`@Filter` 标注在实体类或集合字段上，引用已定义的 `@FilterDef`：

```java
@Entity
@FilterDef(
    name = "tenantFilter",
    parameters = @ParamDef(name = "tenantId", type = String.class),
    defaultCondition = "tenant_id = :tenantId"
)
@FilterDef(name = "softDeleteFilter", defaultCondition = "deleted = false")
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
@Filter(name = "softDeleteFilter", condition = "deleted = false")
public class Project {
    @Id @GeneratedValue
    private Long id;
    private String name;
    private String tenantId;
    private boolean deleted;

    @OneToMany(mappedBy = "project")
    @Filter(name = "softDeleteFilter", condition = "deleted = false")
    private List<Task> tasks;
    // ...
}
```

`@Filter` 注解的关键属性：

| 属性 | 说明 |
|------|------|
| `name` | 引用 `@FilterDef` 定义的过滤器名称（必填） |
| `condition` | SQL WHERE 子句片段（可覆盖 `@FilterDef.defaultCondition`） |
| `deduceAliasInjectionPoints` | Hibernate 6+ 特性，自动推断条件中实体别名位置 |

## 4.2 动态启用/禁用过滤器

过滤器默认是关闭的，必须在运行时通过 Session 显式启用并设置参数：

```java
Session session = sessionFactory.openSession();

// 1. 启用过滤器并设置参数
session.enableFilter("softDeleteFilter");   // 无参过滤器
session.enableFilter("tenantFilter")
       .setParameter("tenantId", "T-001");  // 有参过滤器

// 2. 执行业务操作（所有加载/导航查询自动附加上滤条件）
List<Project> projects = session.createQuery(
    "from Project", Project.class).getResultList();
    // HQL/JPQL 不自动应用过滤器，需手动加条件

Project project = session.get(Project.class, 1L);
    // → SELECT * FROM project WHERE id = 1 AND deleted = false AND tenant_id = 'T-001'

project.getTasks().size();  // 触发集合加载
    // → SELECT * FROM task WHERE project_id = 1 AND deleted = false

// 3. 禁用过滤器
session.disableFilter("tenantFilter");

// 4. 获取已启用的过滤器（用于检查状态或修改参数）
Filter enabledFilter = session.getEnabledFilter("softDeleteFilter");
if (enabledFilter != null) {
    // 过滤器已启用
}
```

**过滤器生效范围：**

- `session.load()` / `session.get()` → 自动应用
- 关联导航（如 `project.getTasks()`）→ 自动应用
- HQL / JPQL 查询 → **不自动应用**（需手动在 WHERE 子句中添加）
- Criteria 查询 → **不自动应用**（需手动添加 Restriction）

## 4.3 典型应用场景

### 4.3.1 场景一：软删除（Soft Delete）

```java
// package-info.java 或实体类
@FilterDef(name = "softDeleteFilter", defaultCondition = "is_deleted = false")

@Entity
@Table(name = "users")
@Filter(name = "softDeleteFilter", condition = "is_deleted = false")
public class User {
    @Id @GeneratedValue
    private Long id;
    private String username;
    @Column(name = "is_deleted")
    private boolean deleted;
    // ...
}

// 使用：始终启用软删除过滤器
@Repository
public class UserRepository {
    @PersistenceContext
    private EntityManager entityManager;

    public User findById(Long id) {
        Session session = entityManager.unwrap(Session.class);
        session.enableFilter("softDeleteFilter");
        return session.get(User.class, id);
        // → SELECT * FROM users WHERE id = ? AND is_deleted = false
    }

    // 如果需要查询包括已删除的数据（如管理员界面）
    public User findByIdIncludingDeleted(Long id) {
        Session session = entityManager.unwrap(Session.class);
        // 不启用过滤器即可
        return session.get(User.class, id);
    }
}
```

### 4.3.2 场景二：多租户（Multi-Tenancy）

```java
@FilterDef(
    name = "tenantFilter",
    parameters = @ParamDef(name = "tenantId", type = String.class),
    defaultCondition = "tenant_id = :tenantId"
)

@Entity
@Table(name = "orders")
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Order {
    @Id @GeneratedValue
    private Long id;
    private String orderNo;
    @Column(name = "tenant_id")
    private String tenantId;
    private BigDecimal amount;
}

// 请求拦截器中启用租户过滤器
@Component
public class TenantFilterInterceptor implements HandlerInterceptor {

    @Autowired
    private EntityManagerFactory emf;

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        String tenantId = request.getHeader("X-Tenant-Id");
        if (tenantId != null) {
            Session session = emf.createEntityManager()
                .unwrap(Session.class);
            session.enableFilter("tenantFilter")
                  .setParameter("tenantId", tenantId);
        }
        return true;
    }
}
```

### 4.3.3 场景三：数据权限（行级权限）

```java
@FilterDef(
    name = "deptFilter",
    parameters = @ParamDef(name = "deptIds", type = Integer[].class)  // Hibernate 6 ; String in older
)

@Entity
@Table(name = "documents")
@Filter(name = "deptFilter", condition = "dept_id IN (:deptIds)")
public class Document {
    @Id @GeneratedValue
    private Long id;
    private String title;
    @Column(name = "dept_id")
    private Integer deptId;
    // ...
}

// 使用时注入当前用户拥有权限的部门 ID 列表
List<Integer> userDeptIds = getCurrentUserDeptIds();  // [1, 3, 5]
session.enableFilter("deptFilter")
       .setParameterList("deptIds", userDeptIds);
```

### 4.3.4 场景四：多语言（国际化）

```java
@FilterDef(
    name = "localeFilter",
    parameters = @ParamDef(name = "locale", type = String.class),
    defaultCondition = "locale = :locale"
)

@Entity
@Table(name = "i18n_messages")
@Filter(name = "localeFilter", condition = "locale = :locale")
public class I18nMessage {
    @Id @GeneratedValue
    private Long id;
    @Column(name = "msg_key")
    private String key;
    private String locale;  // en_US, zh_CN
    private String content;
}

// 使用
session.enableFilter("localeFilter")
       .setParameter("locale", "zh_CN");
```

### 4.3.5 完整示例——软删除 + 多租户组合：

```java
@FilterDef(name = "softDeleteFilter", defaultCondition = "deleted = false")
@FilterDef(
    name = "tenantFilter",
    parameters = @ParamDef(name = "tenantId", type = String.class),
    defaultCondition = "tenant_id = :tenantId"
)

@Entity
@Table(name = "products")
@Filter(name = "softDeleteFilter", condition = "deleted = false")
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private BigDecimal price;
    private String tenantId;
    private boolean deleted;

    @OneToMany(mappedBy = "product")
    @Filter(name = "softDeleteFilter", condition = "deleted = false")
    private List<Sku> skus;
}

// 统一启用过滤器（可封装为工具方法）
public static void enableCommonFilters(Session session, String tenantId) {
    session.enableFilter("softDeleteFilter");
    session.enableFilter("tenantFilter")
           .setParameter("tenantId", tenantId);
}

// 在 Service 层使用
@Transactional
public Product getProduct(Long productId) {
    Session session = entityManager.unwrap(Session.class);
    enableCommonFilters(session, TenantContext.getCurrentTenant());
    return session.get(Product.class, productId);
    // SQL: SELECT * FROM products
    //      WHERE id = ? AND deleted = false AND tenant_id = ?
}
```

## 4.4 @FilterJoinTable

`@FilterJoinTable` 用于过滤多对多关联的中间表。当中间表本身需要软删除或多租户过滤时使用。

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @ManyToMany
    @JoinTable(
        name = "user_role",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    @FilterJoinTable(
        name = "softDeleteFilter",
        condition = "is_deleted = false"
    )
    @FilterJoinTable(
        name = "tenantFilter",
        condition = "tenant_id = :tenantId"
    )
    private Set<Role> roles;
}
```

`@FilterJoinTable` 仅影响关联表的查询，不会作用于关联实体（Role）本身。如果同时需要过滤 Role 实体，还需在 Role 类上添加 `@Filter`。

## 4.5 过滤器与查询

**核心原则：过滤器自动应用于加载操作和关联导航，但 HQL/JPQL/Criteria 查询不会自动应用过滤器。**

```java
// 1. session.get() / session.load() → 过滤器自动生效
Product p = session.get(Product.class, 1L);
// SQL: SELECT ... FROM products WHERE id=? AND deleted=false AND tenant_id=?

// 2. HQL/JPQL → 过滤器不生效，需手动处理
List<Product> list = session.createQuery(
    "from Product", Product.class).getResultList();
// SQL: SELECT ... FROM products  ← 没有自动附加 WHERE 条件！

// 3. HQL 中手动应用过滤
List<Product> list = session.createQuery(
    "from Product p where p.deleted = false and p.tenantId = :tenantId",
    Product.class)
    .setParameter("tenantId", TenantContext.getCurrentTenant())
    .getResultList();

// 4. 过滤关联集合 → 启动过滤器后导航
session.enableFilter("softDeleteFilter");
Product p = session.get(Product.class, 1L);
p.getSkus().size();  // → 关联加载时自动应用过滤器
// SQL: SELECT ... FROM skus WHERE product_id=? AND deleted=false
```

**查询中启用过滤器的方式总结：**

| 查询方式 | 过滤器是否生效 | 替代方案 |
|---------|--------------|---------|
| `session.get()` / `session.load()` | 是 | — |
| 关联导航（延迟加载集合） | 是（如果导航前已启用） | — |
| HQL / JPQL | 否 | 手动 WHEN 条件 + 命名参数 |
| Criteria API | 否 | 手动添加 `Restrictions` 或 `Predicate` |
| Native SQL | 否 | 手动 WHERE 条件 |
| `@NamedQuery` | 否 | 使用命名参数在调用时注入 |

---

# 5. 自定义类型（UserType）

当 Hibernate 内置的 Java 类型到 JDBC 类型的映射无法满足需求时，可以通过实现 `UserType` 接口来自定义映射逻辑。典型场景包括：JSON 列映射、加密字段、枚举自定义映射、复合类型等。

## 5.1 UserType 接口详解

`org.hibernate.usertype.UserType` 是自定义类型的核心接口，定义了从 JDBC ResultSet 读取到 Java 对象的双向转换逻辑。

**核心方法一览：**

| 方法 | 职责 | 关键要点 |
|------|------|---------|
| `nullSafeGet()` | 从 ResultSet 读取列值并转为 Java 对象 | 处理 null 情况 |
| `nullSafeSet()` | 将 Java 对象转为 PreparedStatement 参数 | 处理 null 情况 |
| `returnedClass()` | 返回映射的 Java 类型 | 必须与实体属性类型一致 |
| `sqlTypes()` | 返回对应的 JDBC 类型数组 | 例如 `{Types.VARCHAR}` |
| `equals()` | 判断两个对象是否相等（脏检查用） | 必须正确实现 |
| `hashCode()` | 计算哈希码 | 与 equals 一致 |
| `deepCopy()` | 深拷贝对象（快照用） | 可变类型必须深拷贝 |
| `assemble()` | 从二级缓存反序列化 | 与 `disassemble` 配对 |
| `disassemble()` | 序列化到二级缓存 | 与 `assemble` 配对 |
| `replace()` | merge 时替换值 | mutable 类型需返回深拷贝 |
| `isMutable()` | 是否可变 | 影响脏检查策略 |

**完整示例——JSON 列类型（Map<String, Object> ↔ JSONB）：**

```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.hibernate.HibernateException;
import org.hibernate.engine.spi.SharedSessionContractImplementor;
import org.hibernate.usertype.UserType;
import java.io.Serializable;
import java.sql.*;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;

public class JsonbUserType implements UserType<Map<String, Object>> {

    private static final ObjectMapper mapper = new ObjectMapper();
    private static final TypeReference<Map<String, Object>> TYPE_REF
        = new TypeReference<Map<String, Object>>() {};
    private static final int SQL_TYPE = Types.OTHER; // PostgreSQL JSONB

    @Override
    public int getSqlType() {
        return SQL_TYPE;
    }

    @Override
    public Class<Map<String, Object>> returnedClass() {
        return (Class<Map<String, Object>>) (Class<?>) Map.class;
    }

    @Override
    public boolean equals(Map<String, Object> x, Map<String, Object> y) {
        return Objects.equals(x, y);
    }

    @Override
    public int hashCode(Map<String, Object> x) {
        return (x != null) ? x.hashCode() : 0;
    }

    @Override
    public Map<String, Object> nullSafeGet(ResultSet rs, int position,
                                            SharedSessionContractImplementor session,
                                            Object owner) throws SQLException {
        String json = rs.getString(position);
        if (json == null || rs.wasNull()) {
            return new HashMap<>();
        }
        try {
            return mapper.readValue(json, TYPE_REF);
        } catch (JsonProcessingException e) {
            throw new HibernateException("JSON 反序列化失败", e);
        }
    }

    @Override
    public void nullSafeSet(PreparedStatement st, Map<String, Object> value,
                             int index, SharedSessionContractImplementor session)
            throws SQLException {
        if (value == null) {
            st.setNull(index, SQL_TYPE);
        } else {
            try {
                String json = mapper.writeValueAsString(value);
                st.setObject(index, json, SQL_TYPE);
            } catch (JsonProcessingException e) {
                throw new HibernateException("JSON 序列化失败", e);
            }
        }
    }

    @Override
    public Map<String, Object> deepCopy(Map<String, Object> value) {
        if (value == null) return null;
        try {
            String json = mapper.writeValueAsString(value);
            return mapper.readValue(json, TYPE_REF);
        } catch (JsonProcessingException e) {
            throw new HibernateException("JSON 深拷贝失败", e);
        }
    }

    @Override
    public boolean isMutable() {
        return true; // Map 是可变的
    }

    @Override
    public Serializable disassemble(Map<String, Object> value) {
        if (value == null) return null;
        try {
            return (Serializable) mapper.writeValueAsString(value);
        } catch (JsonProcessingException e) {
            throw new HibernateException("JSON 序列化失败", e);
        }
    }

    @Override
    public Map<String, Object> assemble(Serializable cached, Object owner) {
        if (cached == null) return null;
        try {
            return mapper.readValue((String) cached, TYPE_REF);
        } catch (JsonProcessingException e) {
            throw new HibernateException("JSON 反序列化失败", e);
        }
    }

    @Override
    public Map<String, Object> replace(Map<String, Object> original,
                                        Map<String, Object> target,
                                        Object owner) {
        return deepCopy(original);
    }
}
```

**使用方式：**

```java
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Column(columnDefinition = "jsonb")
    @Type(JsonbUserType.class)
    private Map<String, Object> attributes;  // {"color": "red", "size": "XL", "weight": 1.5}
}
```

## 5.2 CompositeUserType

当多个数据库列需要映射到一个 Java 对象时，使用 `CompositeUserType`。典型的场景是货币金额（amount + currency → MonetaryAmount）。

**完整示例——MonetaryAmount 复合类型：**

```java
import org.hibernate.HibernateException;
import org.hibernate.engine.spi.*;
import org.hibernate.usertype.CompositeUserType;
import java.io.Serializable;
import java.math.BigDecimal;
import java.sql.*;
import java.util.Objects;

// 值对象
public class MonetaryAmount implements Serializable {
    private final BigDecimal amount;
    private final String currency;  // CNY, USD, EUR

    public MonetaryAmount(BigDecimal amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof MonetaryAmount)) return false;
        MonetaryAmount that = (MonetaryAmount) o;
        return Objects.equals(amount, that.amount)
            && Objects.equals(currency, that.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}

// CompositeUserType 实现
public class MonetaryAmountUserType implements CompositeUserType<MonetaryAmount> {

    @Override
    public Class<MonetaryAmount> returnedClass() {
        return MonetaryAmount.class;
    }

    @Override
    public String[] getPropertyNames() {
        return new String[] { "amount", "currency" };
    }

    @Override
    public Type[] getPropertyTypes() {
        return new Type[] { StandardType.BIG_DECIMAL, StandardType.STRING };
    }

    @Override
    public Object getPropertyValue(MonetaryAmount component, int property) {
        if (component == null) return null;
        switch (property) {
            case 0: return component.getAmount();
            case 1: return component.getCurrency();
            default: throw new IllegalArgumentException("Invalid property: " + property);
        }
    }

    @Override
    public void setPropertyValue(MonetaryAmount component, int property, Object value) {
        throw new UnsupportedOperationException("MonetaryAmount is immutable");
    }

    @Override
    public MonetaryAmount nullSafeGet(ResultSet rs, int position,
                                       SharedSessionContractImplementor session,
                                       Object owner) throws SQLException {
        BigDecimal amount = rs.getBigDecimal(position);
        String currency = rs.getString(position + 1);
        // 注意：position 是起始位置，后续列需要 +1
        if (amount == null && currency == null) return null;
        return new MonetaryAmount(amount, currency);
    }

    @Override
    public void nullSafeSet(PreparedStatement st, MonetaryAmount value,
                             int index, SharedSessionContractImplementor session)
            throws SQLException {
        if (value == null) {
            st.setNull(index, Types.NUMERIC);
            st.setNull(index + 1, Types.VARCHAR);
        } else {
            st.setBigDecimal(index, value.getAmount());
            st.setString(index + 1, value.getCurrency());
        }
    }

    @Override
    public boolean equals(MonetaryAmount x, MonetaryAmount y) {
        return Objects.equals(x, y);
    }

    @Override
    public int hashCode(MonetaryAmount x) {
        return (x != null) ? x.hashCode() : 0;
    }

    @Override
    public MonetaryAmount deepCopy(MonetaryAmount value) {
        return value; // 不可变对象，直接返回
    }

    @Override
    public boolean isMutable() {
        return false;
    }

    @Override
    public Serializable disassemble(MonetaryAmount value,
                                     SharedSessionContractImplementor session) {
        return (Serializable) deepCopy(value);
    }

    @Override
    public MonetaryAmount assemble(Serializable cached,
                                    SharedSessionContractImplementor session,
                                    Object owner) {
        return (MonetaryAmount) cached;
    }
}

// 使用
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Columns(columns = {
        @Column(name = "price_amount"),
        @Column(name = "price_currency")
    })
    @Type(MonetaryAmountUserType.class)
    private MonetaryAmount price;
}
```

**CompositeUserType 与 @Embeddable 的对比：**

| 对比维度 | CompositeUserType | @Embeddable |
|---------|-------------------|-------------|
| **控制力** | 完全控制 JDBC 级映射 | Hibernate 自动处理 |
| **列到属性的映射** | 手动控制（列名任意） | 基于命名约定或 @Column 注解 |
| **HQL 查询** | 不支持直接属性路径查询 | 支持（如 `price.amount`） |
| **实现复杂度** | 高（需实现多个方法） | 低（注解驱动） |
| **适用场景** | 遗留数据库、非标准映射、值类型不可变 | 标准场景、需要查询组件内属性 |

## 5.3 EnhancedUserType

`EnhancedUserType` 扩展了 `UserType`，增加了 XML 序列化支持。在 Hibernate 5+ 中使用较少，因为 XML 映射已逐渐被注解取代。

```java
import org.hibernate.usertype.EnhancedUserType;

public class JsonEnhancedUserType implements EnhancedUserType<JsonNode> {
    // ... 除 UserType 方法外，还需实现 toXMLString / fromXMLString
    @Override
    public String toSqlString(JsonNode value) {
        return "'" + value.toString() + "'";
    }

    @Override
    public String toXMLString(JsonNode value) {
        return value.toString();
    }

    @Override
    public Object fromXMLString(String xmlValue) {
        return mapper.readTree(xmlValue);
    }
}
```

绝大多数场景下，标准的 `UserType` 已经足够。只有在需要导出 hbm.xml 配置文件时才需要 `EnhancedUserType`。

## 5.4 ParameterizedType

`ParameterizedType` 允许在 `@Type` 注解中传入参数，实现可配置的自定义类型。

**完整示例——支持不同算法的加密字段类型：**

```java
import org.hibernate.annotations.Parameter;
import org.hibernate.usertype.ParameterizedType;
import java.util.Properties;

public class EncryptedStringUserType implements UserType<String>, ParameterizedType {

    private String algorithm;
    private String secretKey;

    @Override
    public void setParameterValues(Properties parameters) {
        this.algorithm = parameters.getProperty("algorithm", "AES");
        this.secretKey = parameters.getProperty("secretKey");
        if (this.secretKey == null) {
            throw new IllegalArgumentException("secretKey 参数必须指定");
        }
    }

    @Override
    public int getSqlType() {
        return Types.VARCHAR;
    }

    @Override
    public Class<String> returnedClass() {
        return String.class;
    }

    @Override
    public boolean equals(String x, String y) {
        return Objects.equals(x, y);
    }

    @Override
    public int hashCode(String x) {
        return (x != null) ? x.hashCode() : 0;
    }

    @Override
    public String nullSafeGet(ResultSet rs, int position,
                               SharedSessionContractImplementor session,
                               Object owner) throws SQLException {
        String encrypted = rs.getString(position);
        if (encrypted == null) return null;
        return decrypt(encrypted);
    }

    @Override
    public void nullSafeSet(PreparedStatement st, String value, int index,
                             SharedSessionContractImplementor session)
            throws SQLException {
        if (value == null) {
            st.setNull(index, Types.VARCHAR);
        } else {
            st.setString(index, encrypt(value));
        }
    }

    @Override
    public String deepCopy(String value) {
        return value; // String 不可变
    }

    @Override
    public boolean isMutable() {
        return false;
    }

    @Override
    public Serializable disassemble(String value) {
        return (Serializable) value;
    }

    @Override
    public String assemble(Serializable cached, Object owner) {
        return (String) cached;
    }

    @Override
    public String replace(String original, String target, Object owner) {
        return original;
    }

    private String encrypt(String plainText) {
        // 使用 algorithm 和 secretKey 加密
        // 实际实现略
        return plainText; // 占位
    }

    private String decrypt(String cipherText) {
        // 使用 algorithm 和 secretKey 解密
        return cipherText; // 占位
    }
}

// 使用 @Type 注解传入参数
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    @Column(name = "phone")
    @Type(value = EncryptedStringUserType.class, parameters = {
        @Parameter(name = "algorithm", value = "AES"),
        @Parameter(name = "secretKey", value = "${encrypt.phone.key}")
    })
    private String phone;

    @Column(name = "id_card")
    @Type(value = EncryptedStringUserType.class, parameters = {
        @Parameter(name = "algorithm", value = "DES"),
        @Parameter(name = "secretKey", value = "${encrypt.idcard.key}")
    })
    private String idCard;
}
```

`ParameterizedType` 让同一个 `UserType` 实现类可以为不同字段提供不同的行为，大幅提升复用性。

## 5.5 Hibernate 6 中的新类型系统

Hibernate 6 重构了类型系统，将 Java 类型和 JDBC 类型的映射分离为两个独立的概念：

```
旧模型（Hibernate 5.x）:  UserType（一体两面：既管 Java 类型，又管 JDBC 映射）
                     ↓
新模型（Hibernate 6.x）:  JavaType（描述 Java 类型） + JdbcType（描述 JDBC 映射）
```

**新注解：**

| 注解 | 作用 |
|------|------|
| `@JavaType` | 指定自定义 Java 类型描述符 |
| `@JdbcType` | 指定自定义 JDBC 类型描述符 |
| `@JdbcTypeCode` | 使用 JDBC 类型代码指定 JDBC 类型 |
| `@Type` | 同时指定 JavaType 和 JdbcType（兼容旧写法） |

**Maven 坐标变化（Hibernate 6）：**

```xml
<!-- Hibernate 6 核心 -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>6.x</version>
</dependency>
```

**自定义 JavaType 实现（Hibernate 6）：**

```java
import org.hibernate.type.descriptor.WrapperOptions;
import org.hibernate.type.descriptor.java.AbstractClassJavaType;
import org.hibernate.type.descriptor.java.ImmutableMutabilityPlan;
import org.hibernate.type.descriptor.jdbc.VarcharJdbcType;

public class PhoneNumberJavaType extends AbstractClassJavaType<PhoneNumber> {

    public static final PhoneNumberJavaType INSTANCE = new PhoneNumberJavaType();

    public PhoneNumberJavaType() {
        super(PhoneNumber.class, ImmutableMutabilityPlan.instance());
    }

    @Override
    public String toString(PhoneNumber value) {
        return (value != null) ? value.getFullNumber() : null;
    }

    @Override
    public PhoneNumber fromString(CharSequence string) {
        return (string != null) ? PhoneNumber.parse(string.toString()) : null;
    }

    @Override
    public <X> X unwrap(PhoneNumber value, Class<X> type, WrapperOptions options) {
        if (value == null) return null;
        if (String.class.isAssignableFrom(type)) {
            return (X) value.getFullNumber();
        }
        throw new UnsupportedOperationException(
            "Cannot unwrap PhoneNumber to " + type);
    }

    @Override
    public <X> PhoneNumber wrap(X value, WrapperOptions options) {
        if (value == null) return null;
        if (value instanceof String) {
            return PhoneNumber.parse((String) value);
        }
        throw new UnsupportedOperationException(
            "Cannot wrap " + value.getClass());
    }
}

// 使用时注册到 MetadataBuilderContributor
public class TypeContributor implements MetadataBuilderContributor {
    @Override
    public void contribute(MetadataBuilder metadataBuilder) {
        metadataBuilder.applyBasicType(
            new BasicTypeImpl<>(
                PhoneNumberJavaType.INSTANCE,
                VarcharJdbcType.INSTANCE
            ),
            "phone_number"
        );
    }
}
```

Hibernate 6 的新类型系统更清晰、更可组合。对于新项目，建议直接使用 JavaType + JdbcType 分离模式；已有 UserType 的项目可以继续运行（Hibernate 6 兼容旧接口）。

---

# 6. 多租户（Multi-tenancy）

多租户（Multi-tenancy）是指在单一应用实例中为多个客户（租户）提供服务的架构模式。Hibernate 原生支持三种多租户隔离策略：独立数据库（DATABASE）、独立 Schema（SCHEMA）和鉴别列（DISCRIMINATOR）。

## 6.1 多租户策略方案

### 6.1.1 DATABASE 策略

每个租户使用独立的数据库实例，物理隔离级别最高，也是最安全的多租户方案。

**核心接口：**

- `MultiTenantConnectionProvider`：根据租户 ID 提供对应的数据库连接
- `CurrentTenantIdentifierResolver`：从当前请求上下文中解析租户 ID

**完整实现代码：**

```java
import org.hibernate.engine.jdbc.connections.spi.AbstractMultiTenantConnectionProvider;
import org.hibernate.engine.jdbc.connections.spi.ConnectionProvider;
import org.hibernate.engine.jdbc.connections.spi.MultiTenantConnectionProvider;
import org.hibernate.service.UnknownUnwrapTypeException;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

// 1. 多租户连接提供者
public class SchemaMultiTenantConnectionProvider
        extends AbstractMultiTenantConnectionProvider<String> {

    private final Map<String, DataSource> dataSourceMap = new ConcurrentHashMap<>();

    public SchemaMultiTenantConnectionProvider(Map<String, DataSource> dataSources) {
        this.dataSourceMap.putAll(dataSources);
    }

    @Override
    protected ConnectionProvider getAnyConnectionProvider() {
        // 返回任意一个 ConnectionProvider（用于元数据操作等）
        return new DataSourceConnectionProvider(
            dataSourceMap.values().iterator().next());
    }

    @Override
    protected ConnectionProvider selectConnectionProvider(String tenantId) {
        DataSource ds = dataSourceMap.get(tenantId);
        if (ds == null) {
            throw new IllegalArgumentException("未知的租户: " + tenantId);
        }
        return new DataSourceConnectionProvider(ds);
    }

    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        DataSource ds = dataSourceMap.get(tenantId);
        if (ds == null) {
            throw new SQLException("找不到租户 " + tenantId + " 的数据源");
        }
        return ds.getConnection();
    }

    @Override
    public void releaseConnection(String tenantId, Connection connection)
            throws SQLException {
        connection.close();
    }

    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) {
        return false;
    }

    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        throw new UnknownUnwrapTypeException(unwrapType);
    }

    private static class DataSourceConnectionProvider implements ConnectionProvider {
        private final DataSource dataSource;

        DataSourceConnectionProvider(DataSource dataSource) {
            this.dataSource = dataSource;
        }

        @Override
        public Connection getConnection() throws SQLException {
            return dataSource.getConnection();
        }

        @Override
        public void closeConnection(Connection conn) throws SQLException {
            conn.close();
        }

        @Override
        public boolean supportsAggressiveRelease() {
            return true;
        }

        @Override
        public boolean isUnwrappableAs(Class<?> unwrapType) {
            return false;
        }

        @Override
        public <T> T unwrap(Class<T> unwrapType) {
            return null;
        }
    }
}

// 2. 当前租户标识解析器（从请求上下文获取租户 ID）
public class TenantIdentifierResolver
        implements CurrentTenantIdentifierResolver<String> {

    @Override
    public String resolveCurrentTenantIdentifier() {
        String tenantId = TenantContext.getCurrentTenant();
        if (tenantId == null) {
            throw new IllegalStateException(
                "无法解析当前租户 ID，请确保请求中包含了租户信息");
        }
        return tenantId;
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true; // 验证已有 Session 的租户是否一致
    }
}

// 3. ThreadLocal 租户上下文
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void setCurrentTenant(String tenant) {
        CURRENT_TENANT.set(tenant);
    }

    public static String getCurrentTenant() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}

// 4. Web 请求拦截器（设置租户上下文）
@Component
public class TenantInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        String tenantId = request.getHeader("X-Tenant-Id");
        TenantContext.setCurrentTenant(tenantId);
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler, Exception ex) {
        TenantContext.clear();
    }
}
```

**Hibernate 配置（Spring Boot）：**

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        multi_tenancy: DATABASE
        multi_tenant_connection_provider: com.example.SchemaMultiTenantConnectionProvider
        tenant_identifier_resolver: com.example.TenantIdentifierResolver
```

### 6.1.2 SCHEMA 策略

每个租户使用同一数据库中的独立 Schema，Hibernate 会在 SQL 语句中自动添加 Schema 前缀。

**优势**：相比 DATABASE 策略成本更低（共享数据库实例），相比 DISCRIMINATOR 策略隔离性更佳（各自拥有独立的表结构）。

**实现示例：**

```java
// SCHEMA 策略的 MultiTenantConnectionProvider（共享同一数据源）
public class SchemaMultiTenantConnectionProvider
        implements MultiTenantConnectionProvider<String> {

    private final DataSource dataSource;

    public SchemaMultiTenantConnectionProvider(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public Connection getAnyConnection() throws SQLException {
        return dataSource.getConnection();
    }

    @Override
    public void releaseAnyConnection(Connection connection) throws SQLException {
        connection.close();
    }

    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        Connection connection = dataSource.getConnection();
        // 设置 Schema（不同数据库 API 不同）
        connection.setSchema(tenantId);  // JDBC 4.1+
        // 或者: connection.createStatement()
        //          .execute("SET search_path TO " + tenantId);  // PostgreSQL
        return connection;
    }

    @Override
    public void releaseConnection(String tenantId, Connection connection)
            throws SQLException {
        connection.setSchema(null);  // 恢复默认 Schema
        connection.close();
    }

    @Override
    public boolean supportsAggressiveRelease() {
        return true;
    }

    @Override
    public boolean isUnwrappableAs(Class<?> unwrapType) {
        return false;
    }

    @Override
    public <T> T unwrap(Class<T> unwrapType) {
        return null;
    }
}

// TenantIdentifierResolver 与 DATABASE 策略相同
```

**Hibernate 配置：**

```properties
hibernate.multi_tenancy = SCHEMA
hibernate.multi_tenant_connection_provider = com.example.SchemaMultiTenantConnectionProvider
hibernate.tenant_identifier_resolver = com.example.TenantIdentifierResolver
```

在 SCHEMA 策略下，Hibernate 生成的 SQL 会自动带上 Schema 前缀：

```sql
-- 租户 T-001
SELECT * FROM "T-001".users WHERE ...

-- 租户 T-002
SELECT * FROM "T-002".users WHERE ...
```

### 6.1.3 DISCRIMINATOR 策略

所有租户共享同一张表，通过一个鉴别列（如 `tenant_id`）区分数据，这是成本最低但隔离性最弱的策略。

> **注意**：Hibernate 6 的 DISCRIMINATOR 策略并非通过内置多租户机制实现，而是利用 `@Filter` 机制。这与第 4 节的过滤器内容相呼应。

```java
// 不依赖 Hibernate 多租户特性，使用 Filter 实现 DISCRIMINATOR 隔离

// 定义过滤器
@FilterDef(
    name = "tenantFilter",
    parameters = @ParamDef(name = "tenantId", type = String.class),
    defaultCondition = "tenant_id = :tenantId"
)

@Entity
@Table(name = "users")
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class User {
    @Id @GeneratedValue
    private Long id;
    private String name;
    @Column(name = "tenant_id")
    private String tenantId;
    private String email;
}

// 在请求处理中启用过滤器
public Session openTenantAwareSession(String tenantId) {
    Session session = sessionFactory.openSession();
    session.enableFilter("tenantFilter")
           .setParameter("tenantId", tenantId);
    return session;
}
```

**三种策略的对比：**

| 维度 | DATABASE | SCHEMA | DISCRIMINATOR |
|------|---------|--------|---------------|
| **隔离级别** | 最高（物理隔离） | 高（逻辑隔离） | 低（行级隔离） |
| **安全性** | 极高 | 高 | 中（依赖应用层控制） |
| **数据库成本** | 高（每个租户独立数据库） | 中（共享数据库，独立 Schema） | 低（完全共享） |
| **扩展性** | 高（可独立扩展） | 中 | 中 |
| **运维复杂度** | 高 | 中 | 低 |
| **备份/恢复粒度** | 租户级 | Schema 级 | 表级 |
| **适用场景** | 金融、医疗等强隔离需求 | 企业 SaaS，中等隔离需求 | 小型 SaaS、开发/测试环境 |
| **实现方式** | MultiTenantConnectionProvider | MultiTenantConnectionProvider | @Filter + @FilterDef |

## 6.2 多租户的最佳实践

### 6.2.1 租户数据隔离保证

1. **永远不要在代码中硬编码跳过租户过滤**，即使是为了调试
2. **在 Service 层入口统一设置租户上下文**，而非在每个 DAO 方法中分散设置
3. **租户 ID 应从认证令牌中解析**，不可从前端直接传入（防止租户 ID 篡改）
4. **定期审计**：编写测试用例验证租户数据隔离，确保不会出现跨租户数据泄漏

### 6.2.2 连接池配置

DATABASE 策略下需为每个租户配置独立的连接池：

```java
// 为每个租户创建独立的 HikariCP 连接池
public Map<String, DataSource> buildTenantDataSources(Map<String, TenantConfig> tenants) {
    Map<String, DataSource> dataSources = new HashMap<>();
    for (Map.Entry<String, TenantConfig> entry : tenants.entrySet()) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(entry.getValue().getJdbcUrl());
        config.setUsername(entry.getValue().getUsername());
        config.setPassword(entry.getValue().getPassword());
        config.setMaximumPoolSize(10);  // 每个租户的连接池上限
        config.setMinimumIdle(2);
        dataSources.put(entry.getKey(), new HikariDataSource(config));
    }
    return dataSources;
}
```

### 6.2.3 缓存隔离

无论是二级缓存还是自定义应用缓存，都必须按租户隔离：

```xml
<!-- Ehcache 按租户配置独立 Region -->
<cache name="com.example.User" maxEntriesLocalHeap="10000" timeToLiveSeconds="600"/>
<!-- 关键：确保缓存的 key 包含租户 ID，否则会跨租户泄漏 -->
```

在 Hibernate 二级缓存中，实体的缓存 key 默认只包含实体类型和 ID，所以不同数据库（DATABASE 策略）自然隔离。但如果使用 DISCRIMINATOR 策略（共享数据库），必须确保二级缓存能够区分租户：

```java
// Hibernate 6 中的租户级二级缓存配置
// application.yml
hibernate:
  cache:
    use_second_level_cache: true
    region:
      factory_class: jcache
  javax:
    cache:
      uri: classpath:ehcache.xml
  multi_tenancy: DISCRIMINATOR
  # 租户 ID 会被自动纳入缓存 key
```

---

# 7. Hibernate Envers（审计/数据版本控制）

Hibernate Envers 是 Hibernate 的审计模块，自动为 `@Audited` 实体创建历史版本表（`_AUD`），记录每一次 INSERT、UPDATE、DELETE 的快照数据，无需在业务代码中写任何审计逻辑。

## 7.1 Envers 概述

Envers 的核心概念：

```
业务表:                            审计表:
┌─ users ─────────┐              ┌─ users_aud ────────────────────┐
│ id  │ name │ age │              │ id │ rev │ revtype │ name │ age │
├─────┼──────┼─────┤              ├────┼─────┼─────────┼──────┼─────┤
│  1  │ 张三  │  25 │  ──新增──→  │  1 │  1  │    0    │ 张三  │  25 │
│  1  │ 张三  │  26 │  ──修改──→  │  1 │  2  │    1    │ 张三  │  26 │
│  1  │ NULL │ NULL│  ──删除──→  │  1 │  3  │    2    │ NULL │ NULL│
└─────┴──────┴─────┘              └────┴─────┴─────────┴──────┴─────┘

全局版本表:
┌─ revinfo ───────────┐
│ rev │ revtstmp      │
├─────┼───────────────┤
│  1  │ 1640000000000 │
│  2  │ 1640000001000 │
│  3  │ 1640000002000 │
└─────┴───────────────┘
```

**变更类型（revtype）：**

| revtype | 含义 | 说明 |
|---------|------|------|
| 0 | ADD | 实体首次插入 |
| 1 | MOD | 实体被修改 |
| 2 | DEL | 实体被删除 |

**依赖引入：**

```xml
<!-- Maven -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-envers</artifactId>
    <version>6.x</version>
</dependency>
```

```groovy
// Gradle
implementation 'org.hibernate.orm:hibernate-envers:6.x'
```

## 7.2 @Audited

### 7.2.1 基本使用

```java
import org.hibernate.envers.Audited;
import org.hibernate.envers.NotAudited;

@Entity
@Audited  // 审计实体所有字段
public class Customer {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @NotAudited  // 排除不需要审计的字段（如大文本、密码等）
    @Column(length = 4000)
    private String password;

    private String email;
    private Integer age;

    @OneToMany(mappedBy = "customer")
    @Audited  // 关联实体也可审计（需在关联实体上也标注 @Audited）
    private List<Order> orders;
}

@Entity
@Audited
public class Order {
    @Id @GeneratedValue
    private Long id;
    private String orderNo;
    private BigDecimal amount;

    @ManyToOne
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

### 7.2.2 @AuditTable 指定审计表名

```java
@Entity
@Table(name = "users")
@Audited
@AuditTable("users_history")  // 自定义审计表名，默认是 users_aud
public class User {
    @Id @GeneratedValue
    private Long id;
    private String username;
}
```

### 7.2.3 @AuditOverride / @AuditOverrides

嵌入组件（@Embeddable）默认会被审计，如需排除其中的某个字段：

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;
}

@Entity
@Audited
@AuditOverride(forClass = Address.class, name = "zipCode", isAudited = false)
public class Company {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Embedded
    private Address address;
}
```

**注意**：不对 `@Embeddable` 施加 `@Audited`，实体级的 `@Audited` 已生效。如果要排除嵌入组件中的某个字段，使用 `@AuditOverride`。

## 7.3 查询审计数据

### 7.3.1 AuditReader 获取

```java
import org.hibernate.envers.AuditReader;
import org.hibernate.envers.AuditReaderFactory;

Session session = sessionFactory.openSession();
AuditReader auditReader = AuditReaderFactory.get(session);

// 或通过 JPA
EntityManager em = entityManagerFactory.createEntityManager();
AuditReader auditReader = AuditReaderFactory.get(em);
```

### 7.3.2 查询某实体的所有历史版本

```java
// forRevisionsOfEntity： 查询指定实体的所有历史修订
List<Object[]> revisions = auditReader.createQuery()
    .forRevisionsOfEntity(Customer.class, false, true)  // (实体类, selectDeleted, selectEntities)
    .add(AuditEntity.id().eq(customerId))
    .getResultList();

for (Object[] row : revisions) {
    Customer customer = (Customer) row[0];      // 实体快照
    DefaultRevisionEntity rev = (DefaultRevisionEntity) row[1];  // 版本信息
    RevisionType revType = (RevisionType) row[2];                 // 变更类型

    System.out.println("版本 " + rev.getId()
        + ", 类型 " + revType  // ADD / MOD / DEL
        + ", 时间 " + rev.getRevisionDate()
        + ", 名称 " + customer.getName());
}
```

### 7.3.3 查询某个版本的所有变更实体

```java
// forEntitiesModifiedAtRevision： 查询某个修订版本中所有变更的实体
List<Object> entities = auditReader.createQuery()
    .forEntitiesModifiedAtRevision(Customer.class, 3)  // 版本号 3
    .getResultList();

for (Object entity : entities) {
    Customer customer = (Customer) entity;
    System.out.println(customer.getName());
}
```

### 7.3.4 查询某实体的某个历史版本（精确快照）

```java
// find：查询某个实体在指定版本的历史快照
Customer historicalCustomer = auditReader.find(
    Customer.class,    // 实体类型
    customerId,        // 实体 ID
    2                  // 版本号
);

System.out.println("历史上版本 2 的名称: " + historicalCustomer.getName());
```

### 7.3.5 查询某实体在指定日期的版本

```java
// 查询实体在 2024-01-15 10:00:00 时的状态
Customer customer = auditReader.createQuery()
    .forRevisionsOfEntity(Customer.class, true, true)
    .add(AuditEntity.id().eq(customerId))
    .add(AuditEntity.revisionProperty("timestamp")
        .le(Date.from(LocalDateTime.of(2024, 1, 15, 10, 0)
            .atZone(ZoneId.systemDefault()).toInstant()).getTime()))
    .addOrder(AuditEntity.revisionNumber().desc())
    .setMaxResults(1)
    .getSingleResult();
```

## 7.4 修订实体定制

默认的修订信息只包含修订号和时间戳。通过自定义 `@RevisionEntity`，可以添加操作用户、IP 地址等审计信息。

**完整示例：**

```java
import org.hibernate.envers.RevisionEntity;
import org.hibernate.envers.RevisionListener;
import org.hibernate.envers.RevisionNumber;
import org.hibernate.envers.RevisionTimestamp;
import javax.persistence.*;
import java.util.Date;

// 1. 自定义修订实体
@Entity
@Table(name = "custom_revinfo")
@RevisionEntity(CustomRevisionListener.class)
public class CustomRevisionEntity {

    @Id
    @GeneratedValue
    @RevisionNumber
    private Long rev;           // 修订号

    @RevisionTimestamp
    @Temporal(TemporalType.TIMESTAMP)
    private Date revtstmp;      // 修订时间戳

    @Column(name = "user_name")
    private String userName;    // 操作用户

    @Column(name = "ip_address")
    private String ipAddress;   // 操作 IP

    @Column(name = "request_url")
    private String requestUrl;  // 请求 URL

    @Column(name = "user_agent")
    private String userAgent;   // 浏览器 User-Agent

    // getters / setters...
}

// 2. 自定义 RevisionListener
public class CustomRevisionListener implements RevisionListener {

    @Override
    public void newRevision(Object revisionEntity) {
        CustomRevisionEntity revEntity = (CustomRevisionEntity) revisionEntity;

        // 从请求上下文获取审计信息
        revEntity.setUserName(SecurityContext.getCurrentUser());
        revEntity.setIpAddress(ServletContext.getClientIp());
        revEntity.setRequestUrl(ServletContext.getRequestUrl());
        revEntity.setUserAgent(ServletContext.getUserAgent());
    }
}

// 3. 查询时获取自定义修订信息
AuditReader reader = AuditReaderFactory.get(session);
List<Object[]> revisions = reader.createQuery()
    .forRevisionsOfEntity(Customer.class, false, false)
    .add(AuditEntity.id().eq(customerId))
    .getResultList();

for (Object[] row : revisions) {
    CustomRevisionEntity rev = (CustomRevisionEntity) row[1];
    RevisionType revType = (RevisionType) row[2];

    System.out.println("修订 " + rev.getRev()
        + ", 用户 " + rev.getUserName()
        + ", 类型 " + revType);
}
```

## 7.5 策略配置

Envers 提供了几种审计策略，通过 `hibernate.properties` 或 `persistence.xml` 配置：

```properties
# 审计策略
# default: 每次版本记录全部实体快照（冗余但查询快）
# validity: 记录有效时间范围（节省空间，按日期查询更快）
org.hibernate.envers.audit_strategy = org.hibernate.envers.strategy.ValidityAuditStrategy

# 删除时是否保留数据（默认 false，删除后审计表中记录被清空）
org.hibernate.envers.store_data_at_delete = true

# 是否审计嵌入组件修改（默认 true）
org.hibernate.envers.do_not_audit_optimistic_locking_field = true

# 修订字段名（默认 rev）
org.hibernate.envers.revision_field_name = rev

# 修订类型字段名（默认 revtype）
org.hibernate.envers.revision_type_field_name = revtype

# 自动生成审计表 DDL
# hibernate.hbm2ddl.auto = update  # 在整体 Hibernate 配置中
```

**Spring Boot 配置示例：**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
      org:
        hibernate:
          envers:
            audit_strategy: org.hibernate.envers.strategy.ValidityAuditStrategy
            store_data_at_delete: true
            do_not_audit_optimistic_locking_field: true
```

**两种审计策略对比：**

| 策略 | 原理 | 查询效率 | 存储占用 | 适用场景 |
|------|------|---------|---------|---------|
| **default** | 每个版本记录当前实体的所有属性值（全字段冗余） | 查单个历史版本快（直接取） | 高 | 实体属性较少、主要查询特定版本快照 |
| **validity（有效性）** | 只记录变更差异，添加有效性起止时间 | 查某个时间点的状态复杂（需范围查询） | 低 | 实体属性多、主要查询某时间点的状态 |

---

# 8. 字节码增强（Bytecode Enhancement）

Hibernate 字节码增强（Bytecode Enhancement）是在编译期或类加载期对实体类字节码进行修改，以提供更高效的运行时行为。相比传统的代理（Proxy），增强直接在实体类中注入逻辑，减少了代理对象的开销和限制。

## 8.1 增强能力

字节码增强可提供以下能力：

| 能力 | 说明 | 默认是否需要增强 |
|------|------|----------------|
| **属性级懒加载** | `@Basic(fetch = LAZY)` 真正生效 | 是（必须） |
| **属性级脏检查** | 追踪个别属性变更，而非整个实体快照对比 | 是（可选但推荐） |
| **双向关联自动同步** | 设置一端的关联时自动同步另一端 | 是（可选） |
| **@LazyToOne(NO_PROXY)** | 无代理懒加载，避免类继承限制 | 是（必须） |

### 8.1.1 属性级懒加载

```java
@Entity
public class Document {
    @Id @GeneratedValue
    private Long id;
    private String title;

    @Basic(fetch = FetchType.LAZY)  // 需字节码增强才能真正延迟加载
    @Column(columnDefinition = "TEXT")
    private String content;         // 大文本字段

    @Basic(fetch = FetchType.LAZY)
    @Lob
    private byte[] attachment;      // 大附件
}

// 不使用字节码增强时，即使声明 LAZY，content 和 attachment 也会被立即加载
// 增强后，只有访问 document.getContent() 时才触发 SELECT
```

### 8.1.2 脏检查优化

增强后，Hibernate 可以通过实体内部的 tracker 精确知道哪些属性被修改，避免全字段对比：

```
未增强:    实体 ↔ 加载快照（Object[]）→ 逐字段 equals 比较 → 探测脏字段
已增强:    属性 setter → tracker 记录 → 仅对比被修改的字段
```

### 8.1.3 双向关联自动同步

```java
@Entity
public class Parent {
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    private List<Child> children = new ArrayList<>();
}

@Entity
public class Child {
    @ManyToOne
    @JoinColumn(name = "parent_id")
    private Parent parent;
}

// 未增强时，必须手动同步双向关联：
// child.setParent(parent);
// parent.getChildren().add(child);

// 增强后，只需单向设置：
child.setParent(parent);  // 会自动将 child 添加到 parent.children
```

### 8.1.4 @LazyToOne(NO_PROXY) 实现

```java
@OneToOne(fetch = FetchType.LAZY)
@LazyToOne(LazyToOneOption.NO_PROXY)  // 需要字节码增强
private UserProfile profile;
```

传统的 `@OneToOne` 懒加载使用 Proxy（JDK 动态代理或 CGLIB），要求实体类不能是 final、方法不能是 final。`NO_PROXY` 模式直接在增强后的实体类中嵌入懒加载逻辑，没有这些限制。

## 8.2 增强配置

### 8.2.1 Maven 插件配置

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.hibernate.orm.tooling</groupId>
            <artifactId>hibernate-enhance-maven-plugin</artifactId>
            <version>6.x</version>
            <executions>
                <execution>
                    <goals>
                        <goal>enhance</goal>
                    </goals>
                    <configuration>
                        <!-- 启用属性级懒加载 -->
                        <enableLazyInitialization>true</enableLazyInitialization>
                        <!-- 启用脏检查优化 -->
                        <enableDirtyTracking>true</enableDirtyTracking>
                        <!-- 启用双向关联自动同步 -->
                        <enableAssociationManagement>true</enableAssociationManagement>
                        <!-- 启用扩展的增强功能 -->
                        <enableExtendedEnhancement>false</enableExtendedEnhancement>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 8.2.2 Gradle 配置

```groovy
// Kotlin DSL
plugins {
    id("org.hibernate.orm") version "6.x"
}

hibernate {
    enhancement {
        enableLazyInitialization = true
        enableDirtyTracking = true
        enableAssociationManagement = true
        enableExtendedEnhancement = false
    }
}
```

### 8.2.3 @LazyGroup 分组懒加载

`@LazyGroup` 允许多个字段共享同一个懒加载组，一次数据库访问加载组内所有字段：

```java
@Entity
public class Product {
    @Id @GeneratedValue
    private Long id;

    @Basic(fetch = FetchType.LAZY)
    @LazyGroup("largeFields")
    @Column(columnDefinition = "TEXT")
    private String description;

    @Basic(fetch = FetchType.LAZY)
    @LazyGroup("largeFields")
    @Column(columnDefinition = "TEXT")
    private String specifications;

    @Basic(fetch = FetchType.LAZY)
    @LazyGroup("images")
    private byte[] thumbnailImage;

    @Basic(fetch = FetchType.LAZY)
    @LazyGroup("images")
    private byte[] fullImage;
}

// 访问任意 "largeFields" 组的字段 → 触发一条 SQL 同时加载 description + specifications
// 访问任意 "images" 组的字段 → 触发一条 SQL 同时加载 thumbnailImage + fullImage
```

### 8.2.4 验证增强是否生效

```java
// 方法一：检查实体类是否实现了 ManagedEntity 接口
Product product = new Product();
if (product instanceof org.hibernate.engine.spi.ManagedEntity) {
    System.out.println("字节码增强已生效");
}

// 方法二：检查类中是否包含 $$_hibernate_ 前缀的方法/字段
// javap -p target/classes/com/example/Product.class | grep hibernate

// 方法三：通过日志验证
// logging.level.org.hibernate.orm.bytecode = DEBUG
```

增强后的类会在编译时被修改，常见的增强效果标志：实现了 `ManagedEntity` 和 `PersistentAttributeInterceptable` 接口。

---

# 9. Hibernate Validator 集成

Hibernate Validator 是 Jakarta Bean Validation（JSR 380）的参考实现。它与 Hibernate ORM 深度集成，可以实现实体字段的自动验证，甚至将验证约束反映到数据库 DDL 中。

## 9.1 Bean Validation 与 Hibernate 的协作

### 9.1.1 自动验证时机

Hibernate 在以下时机自动触发 Bean Validation：

| 时机 | JPA 事件 | 触发条件 |
|------|---------|---------|
| `persist()` | PrePersist | 默认触发 |
| `merge()` 中的更新 | PreUpdate | 默认触发 |
| `remove()` | PreRemove | 默认触发（较少使用） |

```java
@Entity
public class Employee {
    @Id @GeneratedValue
    private Long id;

    @NotBlank(message = "姓名不能为空")
    @Size(min = 2, max = 50, message = "姓名长度 2~50 个字符")
    private String name;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Min(value = 18, message = "年龄不能小于 18")
    @Max(value = 65, message = "年龄不能大于 65")
    private Integer age;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;
}

// 使用
EntityManager em = entityManagerFactory.createEntityManager();
em.getTransaction().begin();

Employee emp = new Employee();
emp.setName("A");   // 不满足 @Size

try {
    em.persist(emp);  // 自动触发验证 → 抛出 ConstraintViolationException
    em.getTransaction().commit();
} catch (ConstraintViolationException e) {
    e.getConstraintViolations().forEach(v ->
        System.out.println(v.getPropertyPath() + ": " + v.getMessage())
    );
}
```

### 9.1.2 hibernate.validator.apply_to_ddl

将验证约束自动映射为数据库 DDL 约束（列长度、非空等）：

```properties
# 开启 DDL 级验证约束
hibernate.validator.apply_to_ddl = true

# 配置选项（Spring Boot）:
spring.jpa.properties.jakarta.persistence.validation.mode = AUTO
```

**效果示例：**

```java
@Size(max = 100)
private String name;
// DDL: name VARCHAR(100)  ← 自动设置列长度为 100

@NotNull
private String email;
// DDL: email VARCHAR(255) NOT NULL  ← 自动添加 NOT NULL 约束
```

### 9.1.3 验证分组

针对不同场景使用不同的验证规则：

```java
// 验证分组接口
public interface CreateGroup { }
public interface UpdateGroup { }
public interface DeleteGroup { }

@Entity
public class Product {
    @Id
    @NotNull(groups = UpdateGroup.class, message = "更新时 ID 不能为空")
    @Null(groups = CreateGroup.class, message = "创建时 ID 必须为 null")
    @GeneratedValue
    private Long id;

    @NotBlank(groups = {CreateGroup.class, UpdateGroup.class})
    private String name;

    @NotNull(groups = CreateGroup.class, message = "创建时必须指定价格")
    private BigDecimal price;
}

// 使用分组验证
@Service
public class ProductService {

    @PersistenceContext
    private EntityManager em;

    @Transactional
    public void createProduct(Product product) {
        // 创建时验证 CreateGroup
        validate(product, CreateGroup.class);
        em.persist(product);
    }

    @Transactional
    public void updateProduct(Product product) {
        // 更新时验证 UpdateGroup
        validate(product, UpdateGroup.class);
        em.merge(product);
    }

    private void validate(Object obj, Class<?>... groups) {
        Validator validator = Validation.buildDefaultValidatorFactory()
            .getValidator();
        Set<ConstraintViolation<Object>> violations = validator
            .validate(obj, groups);
        if (!violations.isEmpty()) {
            throw new ConstraintViolationException(violations);
        }
    }
}
```

### 9.1.4 自定义验证器

```java
import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = {PhoneValidator.class})
@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPhone {
    String message() default "手机号格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 验证器实现
public class PhoneValidator implements
        ConstraintValidator<ValidPhone, String> {
    private static final Pattern PHONE_PATTERN =
        Pattern.compile("^1[3-9]\\d{9}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isEmpty()) {
            return true; // null 应该由 @NotNull 处理
        }
        return PHONE_PATTERN.matcher(value).matches();
    }
}

// 使用自定义注解
@Entity
public class Contact {
    @Id @GeneratedValue
    private Long id;

    @ValidPhone(message = "请填写有效的手机号")
    private String phone;
}
```

---

# 10. 其他高级特性

以下高级特性虽然使用频率不如前 9 节，但在特定场景下至关重要。

## 10.1 Hibernate Spatial（地理空间数据）

Hibernate Spatial 是 Hibernate 对地理空间数据的支持模块，可用于处理 Geometry 类型（Point、LineString、Polygon 等）和空间查询。

**依赖：**

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-spatial</artifactId>
    <version>6.x</version>
</dependency>
```

**Geometry 类型映射：**

```java
import org.locationtech.jts.geom.*;

@Entity
@Table(name = "stores")
public class Store {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Column(columnDefinition = "geometry(Point,4326)")  // EPSG:4326
    private Point location;  // 经纬度坐标

    @Column(columnDefinition = "geometry(Polygon,4326)")
    private Polygon serviceArea;  // 服务范围

    @Column(columnDefinition = "geometry(MultiPolygon,4326)")
    private MultiPolygon deliveryAreas;  // 多个配送区域
}
```

**空间查询：**

```java
import org.hibernate.spatial.SpatialFunction;

// 1. 查询给定点附近 5km 范围内的门店
String hql = "from Store s where within(s.location, :circle) = true";
Geometry circle = createCircle(lng, lat, 5000);  // 半径 5000 米
List<Store> stores = session.createQuery(hql, Store.class)
    .setParameter("circle", circle)
    .getResultList();

// 2. 查询配送区域包含指定点的门店
String hql = "from Store s where intersects(s.serviceArea, :point) = true";
Point point = geometryFactory.createPoint(new Coordinate(lng, lat));
List<Store> stores = session.createQuery(hql, Store.class)
    .setParameter("point", point)
    .getResultList();

// 3. 使用 within 函数（JPQL）
List<Store> stores = session.createQuery(
    "select s from Store s where within(:point, s.serviceArea) = true",
    Store.class)
    .setParameter("point", point)
    .getResultList();
```

**空间方言配置：**

| 数据库 | Dialect 类 |
|--------|-----------|
| PostgreSQL + PostGIS | `PostgreSQLPG95Dialect` 或 `PostgreSQLDialect` (Hibernate 6+) |
| MySQL 8+ | `MySQL8SpatialDialect` |
| Oracle | `OracleSpatial10gDialect` |

```yaml
spring:
  jpa:
    database-platform: org.hibernate.spatial.dialect.postgis.PostgisDialect
```

## 10.2 Hibernate Reactive

Hibernate Reactive 是 Hibernate 6+ 的响应式扩展，基于 Vert.x / Mutiny 实现非阻塞数据库访问。

**依赖：**

```xml
<dependency>
    <groupId>org.hibernate.reactive</groupId>
    <artifactId>hibernate-reactive-core</artifactId>
    <version>2.x</version>
</dependency>
```

**核心接口——Mutiny.Session：**

```java
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.Multi;
import org.hibernate.reactive.mutiny.Mutiny;

@ApplicationScoped
public class ReactiveBookService {

    @Inject
    Mutiny.SessionFactory sessionFactory;

    // 查询单个实体（返回 Uni）
    public Uni<Book> findBook(Long id) {
        return sessionFactory.withSession(session ->
            session.find(Book.class, id)
        );
    }

    // 查询列表（返回 Uni<List>）
    public Uni<List<Book>> findBooksByAuthor(String author) {
        return sessionFactory.withSession(session ->
            session.createQuery(
                "from Book b where b.author = :author", Book.class)
                .setParameter("author", author)
                .getResultList()
        );
    }

    // 保存（链式组合）
    public Uni<Book> createBook(Book book) {
        return sessionFactory.withTransaction((session, tx) ->
            session.persist(book)
                .call(() -> session.flush())
                .replaceWith(book)
        );
    }

    // 事务性操作
    public Uni<Void> updateBookTitle(Long id, String newTitle) {
        return sessionFactory.withTransaction((session, tx) ->
            session.find(Book.class, id)
                .invoke(book -> book.setTitle(newTitle))
                .replaceWithVoid()
        );
    }

    // 响应式流式查询
    public Multi<Book> streamAllBooks() {
        return sessionFactory.withSession(session ->
            session.createQuery("from Book", Book.class)
                .getResultStream()
        );
    }
}
```

**与传统 Hibernate 的区别：**

| 维度 | 传统 Hibernate | Hibernate Reactive |
|------|---------------|-------------------|
| **阻塞模型** | 同步阻塞（JDBC） | 非阻塞异步（Vert.x） |
| **返回值** | 直接返回对象或 List | Uni / Multi（或 CompletionStage） |
| **Session** | `org.hibernate.Session` | `Mutiny.Session` / `Stage.Session` |
| **事务管理** | 声明式 `@Transactional` | 显式 `.withTransaction()` |
| **数据库驱动** | JDBC 驱动 | Vert.x Reactive 数据库客户端 |
| **连接池** | HikariCP / DBCP2 | Reactive SQL Client 连接池 |
| **适用场景** | 传统 Servlet 容器 | 响应式应用（Quarkus、Vert.x） |

**配置文件示例：**

```yaml
# application.yml (Quarkus 风格)
quarkus:
  hibernate-orm:
    dialect: org.hibernate.dialect.PostgreSQLDialect
  datasource:
    reactive:
      url: vertx-reactive:postgresql://localhost:5432/mydb
      max-size: 20
```

## 10.3 自定义命名策略完整实现

Hibernate 提供两个层次的命名策略，允许完全掌控表名和列名的生成逻辑。

### 10.3.1 ImplicitNamingStrategy

控制当未显式指定表名/列名时，从 Java 标识符自动推断的规则：

```java
import org.hibernate.boot.model.naming.ImplicitNamingStrategyJpaCompliantImpl;
import org.hibernate.boot.model.naming.ImplicitEntityNameSource;
import org.hibernate.boot.model.naming.ImplicitAnyKeyColumnNameSource;
import org.hibernate.boot.model.naming.Identifier;

public class CustomImplicitNamingStrategy
        extends ImplicitNamingStrategyJpaCompliantImpl {

    @Override
    public Identifier determinePrimaryTableName(
            ImplicitEntityNameSource source) {
        // 所有表名加 "t_" 前缀
        String className = source.getEntityNaming().getClassName();
        return toIdentifier("t_" + toSnakeCase(className),
            source.getBuildingContext());
    }

    @Override
    public Identifier determineAnyKeyColumnName(
            ImplicitAnyKeyColumnNameSource source) {
        // 外键列：关联表名 + "_" + 主键列
        String tableName = source.getBuildingContext()
            .getMetadataBuildingContext().getMetadataCollector()
            .getEntityBinding(source.getNamingPath().getPrefix())
            .getTable().getName();
        return toIdentifier(tableName + "_id",
            source.getBuildingContext());
    }

    private String toSnakeCase(String camelCase) {
        return camelCase.replaceAll("([a-z])([A-Z])", "$1_$2").toLowerCase();
    }
}
```

### 10.3.2 PhysicalNamingStrategy

控制最终写入数据库的物理名称，是最常用的命名策略扩展点：

```java
import org.hibernate.boot.model.naming.Identifier;
import org.hibernate.boot.model.naming.PhysicalNamingStrategy;
import org.hibernate.engine.jdbc.env.spi.JdbcEnvironment;

public class CustomPhysicalNamingStrategy implements PhysicalNamingStrategy {

    private static final String TABLE_PREFIX = "t_";
    private static final String COLUMN_PREFIX = "c_";

    @Override
    public Identifier toPhysicalCatalogName(
            Identifier logicalName, JdbcEnvironment jdbcEnvironment) {
        return apply(logicalName, jdbcEnvironment);
    }

    @Override
    public Identifier toPhysicalSchemaName(
            Identifier logicalName, JdbcEnvironment jdbcEnvironment) {
        return apply(logicalName, jdbcEnvironment);
    }

    @Override
    public Identifier toPhysicalTableName(
            Identifier logicalName, JdbcEnvironment jdbcEnvironment) {
        if (logicalName == null) return null;
        String name = logicalName.getText();
        if (!name.startsWith(TABLE_PREFIX)) {
            name = TABLE_PREFIX + toSnakeCase(name);
        }
        return Identifier.toIdentifier(name, logicalName.isQuoted());
    }

    @Override
    public Identifier toPhysicalSequenceName(
            Identifier logicalName, JdbcEnvironment jdbcEnvironment) {
        return logicalName;
    }

    @Override
    public Identifier toPhysicalColumnName(
            Identifier logicalName, JdbcEnvironment jdbcEnvironment) {
        if (logicalName == null) return null;
        String name = logicalName.getText();
        // 已有下划线的名称视为已转换（如 @Column 显式指定）
        if (name.contains("_")) return logicalName;
        return Identifier.toIdentifier(
            COLUMN_PREFIX + toSnakeCase(name), logicalName.isQuoted());
    }

    private Identifier apply(Identifier logicalName, JdbcEnvironment env) {
        if (logicalName == null) return null;
        return Identifier.toIdentifier(
            toSnakeCase(logicalName.getText()), logicalName.isQuoted());
    }

    private String toSnakeCase(String camelCase) {
        return camelCase
            .replaceAll("([a-z])([A-Z])", "$1_$2")
            .replaceAll("([A-Z])([A-Z][a-z])", "$1_$2")
            .toLowerCase();
    }
}
```

**配置：**

```yaml
spring:
  jpa:
    hibernate:
      naming:
        implicit-strategy: com.example.CustomImplicitNamingStrategy
        physical-strategy: com.example.CustomPhysicalNamingStrategy
```

## 10.4 自定义 Dialect 扩展

Hibernate Dialect 不仅负责 SQL 方言适配，还可以注册自定义 SQL 函数、覆盖分页行为和类型映射。

**完整示例——扩展 PostgreSQL Dialect：**

```java
import org.hibernate.dialect.PostgreSQLDialect;
import org.hibernate.dialect.function.StandardSQLFunction;
import org.hibernate.type.StandardBasicTypes;

public class ExtendedPostgreSQLDialect extends PostgreSQLDialect {

    public ExtendedPostgreSQLDialect() {
        super();

        // 1. 注册自定义 PostgreSQL 函数
        registerFunction("date_trunc", new StandardSQLFunction(
            "date_trunc", StandardBasicTypes.TIMESTAMP));
        registerFunction("array_agg", new StandardSQLFunction(
            "array_agg"));
        registerFunction("jsonb_extract_path_text",
            new StandardSQLFunction("jsonb_extract_path_text",
            StandardBasicTypes.STRING));
        registerFunction("ilike", new StandardSQLFunction("ilike"));
        registerFunction("regexp_replace", new StandardSQLFunction(
            "regexp_replace", StandardBasicTypes.STRING));

        // 2. 注册自定义 SQL 函数
        registerFunction("group_concat",
            new SQLFunctionTemplate(StandardBasicTypes.STRING,
                "string_agg(?1, ?2)"));
    }

    // 3. 覆盖分页行为（添加自定义分页逻辑）
    @Override
    public String getLimitString(String sql, boolean hasOffset) {
        // PostgreSQL 默认行为已足够，此处仅为示例
        // 可在此添加 SQL 注释用于 DBA 追踪
        return sql + (hasOffset
            ? " limit ? offset ? /* custom pagination */"
            : " limit ? /* custom pagination */");
    }

    // 4. 覆盖类型映射
    @Override
    public String getTypeName(int code, long length,
                               int precision, int scale) {
        // 自定义类型映射：将 Java String 默认映射为 VARCHAR 而非 TEXT
        switch (code) {
            case Types.VARCHAR:
                return (length > 16384) ? "text" : "varchar(" + length + ")";
            default:
                return super.getTypeName(code, length, precision, scale);
        }
    }
}

// HQL 中使用自定义函数
String hql = "select date_trunc('month', p.createdAt) as month, " +
             "count(*) from Product p group by date_trunc('month', p.createdAt)";
```

**多数据源的 Dialect 配置：**

```java
// 当应用中存在多个数据源时，不同数据源可能需要不同的 Dialect

// PostgreSQL 数据源
@Bean
@Primary
public LocalContainerEntityManagerFactoryBean primaryEntityManager() {
    Map<String, Object> props = new HashMap<>();
    props.put("hibernate.dialect", ExtendedPostgreSQLDialect.class.getName());
    // ... 其他配置
}

// MySQL 数据源
@Bean
public LocalContainerEntityManagerFactoryBean secondaryEntityManager() {
    Map<String, Object> props = new HashMap<>();
    props.put("hibernate.dialect", CustomMySQLDialect.class.getName());
    // ... 其他配置
}
```

**Dialect 中注册 SQL 函数的作用：**

注册后在 HQL/JPQL/Criteria 中可以直接使用函数名调用，Hibernate 会自动将其翻译为对应数据库的 SQL 函数：

```java
// Criteria 中使用 date_trunc
List<Object[]> result = session.createQuery(
    "select function('date_trunc', 'month', p.createdAt), count(p) " +
    "from Product p " +
    "group by function('date_trunc', 'month', p.createdAt)",
    Object[].class)
    .getResultList();
// 生成 SQL: SELECT date_trunc('month', created_at), count(*) FROM t_product GROUP BY date_trunc('month', created_at)
```
