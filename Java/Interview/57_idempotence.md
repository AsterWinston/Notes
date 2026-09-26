# 1. 知识点
- **幂等概念**: 对同一操作执行一次与执行多次，产生的效果相同。数学定义：f(f(x)) = f(x)
- **HTTP方法幂等性**: GET(查询天然幂等)、PUT(更新，多次请求结果一致)、DELETE(删除，第一次成功后续返回404或无影响)、POST(非幂等，每次请求创建新资源)
- **幂等性场景**: 网络超时重试(客户端/网关重试导致请求重复执行)、用户重复点击(按钮双击、手抖)、MQ消息重复消费(at-least-once语义) 、定时任务重复执行(多实例部署同时触发)
- **Token机制**: 请求前先获取Token(存入Redis)，提交业务时携带Token。服务端校验Token是否存在，存在则删除并执行业务(原子性)，不存在则拒绝(重复请求)
- **Token机制实现**: 使用Lua脚本原子执行"查询Token→删除Token"两步操作，保证高并发下的幂等性判断
- **Token机制流程**: 客户端GET /getToken → 服务端生成UUID存Redis(key=token, value=userId, 设置过期时间5min) → 客户端提交请求携带token → 服务端Lua脚本校验并删除token → token存在则执行业务
- **数据库唯一索引**: 在业务表上建唯一约束(如订单号的unique key)，利用数据库的约束机制防重。INSERT失败(重复)说明已处理过，返回幂等结果
- **数据库唯一索引实现**: `INSERT IGNORE INTO order(id, amount) VALUES(?, ?)` 或 `INSERT ... ON DUPLICATE KEY UPDATE`，根据数据库返回结果判断是否首次插入
- **数据库唯一索引优缺点**: 实现简单，数据库保证强一致性。缺点：只适用于数据库操作、不适合跨多个操作的幂等、海量数据唯一索引影响写入性能
- **状态机幂等**: 业务对象有明确的状态流转路径(如待支付→已支付→已退款)，已到达终态的操作不允许再次执行。在更新前判断前置状态是否允许操作
- **状态机实现**: `UPDATE order SET status='paid' WHERE id=? AND status='pending'`，根据affected_rows判断是否执行成功(状态不符合条件返回0行)
- **乐观锁幂等**: 使用version字段实现，`UPDATE table SET status='paid', version=version+1 WHERE id=? AND version=?`，根据affected_rows判断。第一次成功，重复请求where条件不满足更新0行
- **Redis setnx防重**: 使用`SETNX request_id value`判断是否为第一次请求，设置过期时间。成功则执行，失败则说明已处理过
- **Redis setnx实现**: 请求携带全局唯一requestId，服务端`SETNX requestId timestamp EX 3600`，返回1则首次执行，返回0则重复请求
- **消息表/去重表**: 业务操作前先检查本地消息表，同一消息ID是否已处理。记录消息ID和处理状态(processing/success/failed)，处理中的跳过或等结果
- **全局请求ID方案**: 每次请求生成全局唯一requestId，服务端记录已处理的requestId集合(Redis/DB)，后续相同requestId直接返回之前的处理结果
- **全局请求ID注意事项**: requestId存储在Redis并设置合适过期时间(如7天)，使用布隆过滤器预先过滤不存在的requestId加速判断
- **分布式锁防重**: 获取分布式锁成功执行业务，获取失败说明有并发请求正在处理，返回幂等结果(如"处理中请稍后")
- **分布式锁防重注意**: 锁的粒度要合理(太粗影响并发，太细防不住)，锁的超时要大于业务执行时间(或用看门狗续期)
- **前后端配合防重**: 前端按钮置灰+Loading(点击后disabled禁止二次点击)+防抖(debounce延迟提交)；后端任何情况下都做服务端幂等校验，不能只依赖前端
- **下单三重防重**: (1) 前端: 按钮置灰+loading遮罩，防止用户重复点击; (2) Redis SETNX分布式锁: 防止同一用户多设备同时提交; (3) Redisson分布式锁(WatchDog自动续期): 防止长时间业务执行时锁过期释放，保证资源独占至业务完成

# 2. 面试题

## 2.1 什么是接口幂等性？为什么需要它？
接口幂等性指同一个操作执行一次和多次效果相同。需要幂等性的原因：1)网络不可靠，客户端超时后重试可能导致请求重复执行 2)MQ消息投递可能重复(at-least-once) 3)用户重复操作(双击按钮) 4)服务间RPC调用重试。若接口不幂等，重复扣款、重复创建订单等将导致严重业务问题。

## 2.2 Token机制如何保证幂等性？详细描述流程
Token机制(也称令牌机制)流程：
1) 客户端打开页面时，向服务端请求一个Token(GET /getToken)
2) 服务端生成全局唯一Token(UUID)存入Redis(key=token, value=userId, 过期时间如5分钟)，返回给前端
3) 用户提交表单时，客户端携带该Token
4) 服务端使用Lua脚本原子执行：先GET token，若存在则DEL token并返回成功，否则返回失败
5) 若Lua返回成功(首次请求)，执行业务逻辑；若返回失败(重复请求)，直接返回幂等结果
关键点：Lua脚本保证"查询→删除"的原子性；Token只能使用一次；需设置过期时间防止内存泄漏。

## 2.3 数据库唯一索引实现幂等的优缺点？
优点：实现简单(加个唯一约束即可)、数据库保证强一致性、无需额外组件。
缺点：1)仅适用于插数据库的场景，不能覆盖纯内存操作或外部调用 2)重复插入会抛异常，需要代码处理异常并返回正确结果 3)唯一索引字段需要业务上有唯一标识(如订单号) 4)MySQL单表数据量过大时唯一索引影响写入性能 5)不适合"部分成功"的复杂幂等逻辑。

## 2.4 状态机如何实现幂等？请举例
状态机幂等通过限制状态流转来保证操作的幂等。例如电商订单状态：待支付→已支付→已发货→已完成。更新时带上前置状态条件：
```sql
UPDATE orders SET status='paid', pay_time=now()
WHERE order_id=? AND status='pending'
```
首次请求status从pending变为paid，affected_rows=1。重复请求where条件`status='pending'`不满足，affected_rows=0。业务代码根据受影响行数判断，若为0说明已支付，返回幂等成功。关键：定义清晰的状态流转图，每个操作对应唯一的前置状态和后置状态。

## 2.5 乐观锁和状态机实现幂等的区别？
两者都基于`UPDATE ... WHERE ...`条件，根据affected_rows判断。区别：
- **状态机**：使用业务状态字段(如status='pending')作为条件，业务语义强
- **乐观锁**：使用额外的version字段作为条件，通用性强
- 状态机只关心当前状态，乐观锁关心并发修改次数
- 状态机可能涉及多个状态流转路径，乐观锁每条更新version都变
- 状态机更适合有明确状态迁移的业务(订单、审批)；乐观锁更适合通用并发控制

## 2.6 Redis和数据库做幂等各自适用场景？
**Redis方案(Token/Setnx)** 适用：1)高并发场景(Redis性能优于DB) 2)非数据库操作的幂等(如调用外部API) 3)临时性幂等控制(有过期时间的Token)。**数据库方案(唯一索引/状态机/乐观锁)** 适用：1)需要持久化存储的幂等记录 2)与业务数据强绑定(如订单唯一性) 3)需要强一致性保证。实际生产常用"Redis快速判断(布隆过滤器加速) + 数据库兜底保证"的组合方案。

## 2.7 如何设计一个通用的幂等组件？
1) **注解驱动**：`@Idempotent(key="#request.orderId", expire=5000)`，AOP拦截
2) **幂等Key生成**：支持SpEL表达式从请求参数提取唯一标识(业务单号/用户ID+操作类型)
3) **存储层可插拔**：Redis(高性能)、数据库(持久化)、支持切换
4) **处理流程**：拦截请求→生成幂等Key→SetNX写入(状态=PROCESSING)→执行业务→更新状态=SUCCESS。重复请求发现状态PROCESSING返回"处理中"，SUCCESS返回缓存结果
5) **异常处理**：业务异常时删除幂等记录(允许重试)或保留状态让客户端查询结果
6) **过期清理**：定期清理过期的幂等记录

## 2.8 如何处理MQ消息重复消费的幂等？
1) **消息业务唯一ID**：生产者发送消息时携带业务唯一ID(如订单号+操作类型)，消费者根据此ID做幂等
2) **消息表去重**：消费者通过业务ID查询消息消费记录表，已消费则跳过
3) **Redis SetNX**：消费前SETNX消息ID，失败则跳过
4) **数据库唯一约束**：消费逻辑最终写DB时依赖唯一索引防重
5) **状态机判断**：类似订单/支付场景，根据业务对象当前状态判断是否已消费
注意：分布式事务场景下，还需考虑"消费处理一半失败"的情况，配合本地消息表保证最终一致性。

## 2.9 POST请求如何设计为幂等？
1) **Token机制**：前端提交前获取Token，提交时带Token，服务端校验Token
2) **客户端生成唯一请求ID**：前端生成requestId（如UUID），服务端记录并去重
3) **业务唯一键**：业务数据本身包含唯一约束字段(如订单号)
4) **状态机**：当前置状态不符合时返回幂等结果(如"订单已创建")
5) **请求去重表**：记录(请求ID → 处理结果)，重复请求返回缓存结果
最重要的是服务端不能只依赖前端控制，必须服务端校验。

## 2.10 下单请求如何实现三重防重？
三重防重是生产环境中防止重复下单的标准方案：

**第一重：前端防重**
- 提交按钮点击后立即置灰（disabled=true）
- 添加loading遮罩层，阻断后续点击
- JS防抖（debounce）：短时间内的多次点击只执行最后一次
- 局限性：前端控制只能防普通用户误操作，无法防止脚本/抓包重放

**第二重：Redis SETNX分布式锁**
- Key设计：`order:lock:{userId}:{商品SKU组合标识}`
- 使用`SETNX key value EX 10`（原子操作，设置成功返回1，失败返回0）
- 获取锁成功→执行业务，失败→说明有并发请求正在处理，返回"处理中"
- 适合短时防重，但存在锁超时风险（业务执行超时锁自动释放）

**第三重：Redisson分布式锁（WatchDog自动续期）**
- 使用Redisson的`RLock`，内置WatchDog机制（看门狗）
- 获取锁时设置leaseTime（默认30s），WatchDog每10s自动续期至30s
- 只要业务未执行完，WatchDog一直续期，保证资源独占至业务完成
- 配合业务唯一标识（订单号唯一索引）兜底，最终保证幂等

```java
@Service
public class OrderService {
    
    public Result createOrder(OrderRequest request) {
        // 第二重：Redis SETNX快速拦截
        String lockKey = "order:lock:" + request.getUserId() + ":" + request.getSkuId();
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(10));
        if (!Boolean.TRUE.equals(acquired)) {
            return Result.fail("订单处理中，请勿重复提交");
        }
        
        // 第三重：Redisson锁（WatchDog续期）
        RLock lock = redissonClient.getLock("order:redisson:" + request.getOrderNo());
        try {
            if (!lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                return Result.fail("系统繁忙，请稍后重试");
            }
            // 执行业务逻辑（包含库存扣减、订单创建等）
            return doCreateOrder(request);
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
            // 清理SETNX锁
            redisTemplate.delete(lockKey);
        }
    }
}
```

**三层递进关系**：前端UI拦截（用户层面）→ Redis SETNX快速拦截（并发防重）→ Redisson WatchDog兜底（长事务防重）+ 数据库唯一索引最终保证。
