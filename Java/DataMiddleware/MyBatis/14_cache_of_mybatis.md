```
MyBatis的缓存
缓存：cache
缓存的作⽤：通过减少IO的⽅式，来提⾼程序的执⾏效率。
mybatis的缓存：将select语句的查询结果放到缓存（内存）当中，下⼀次还是这条select语句的话，直接从缓存中取，不再查数据库。⼀⽅⾯是减少了IO。另⼀⽅⾯不再执⾏繁琐的查找算法。效率⼤⼤提升。
mybatis缓存包括：

⼀级缓存：将查询到的数据存储到SqlSession中。
⼆级缓存：将查询到的数据存储到SqlSessionFactory中。
或者集成其它第三⽅的缓存：⽐如EhCache【Java语⾔开发的】、Memcache【C语⾔开发的】等。
缓存只针对于DQL语句，也就是说缓存机制只对应select语句。
```

# 1. 一级缓存

## 1.1 ⼀级缓存默认是开启的。不需要做任何配置。
```
原理：只要使⽤同⼀个SqlSession对象执⾏同⼀条SQL语句，就会⾛缓存。

测试

@Test
public void testSelectByCollection(){
    SqlSession sqlSession = SqlSessionUtil.openSession();
    ClazzMapper mapper = sqlSession.getMapper(ClazzMapper.class);
    Clazz clazz1 = mapper.selectByCollection(1001);
    System.out.println(clazz1);
    Clazz clazz2 = mapper.selectByCollection(1001);
    System.out.println(clazz2);
    sqlSession.close();
}

执行结果，只执行了一次sql语句，第二次是直接从缓存中拿到数据输出
```
## 1.2 什么情况下不会⾛缓存？
```
第⼀种：不同的SqlSession对象。
第⼆种：查询条件变化了。
```
## 1.3 ⼀级缓存失效情况包括两种
```
第⼀种：第⼀次查询和第⼆次查询之间，⼿动清空了⼀级缓存。
第⼆种：第⼀次查询和第⼆次查询之间，执⾏了增删改操作。【这个增删改和哪张表没有关系，只要有insert delete update操作，⼀级缓存就失效。】
```
## 1.4 关闭一级缓存
```
方法 1：使用不同的 SqlSession（最彻底的方式）

方法 2：设置 localCacheScope = STATEMENT（官方推荐的“弱化”方式）
<settings>
    <setting name="localCacheScope" value="STATEMENT"/>
</settings>
含义：
默认是 SESSION：同一 SqlSession 内，操作共享一级缓存。
STATEMENT：每执行一次查询就清空缓存 → 相当于让一级缓存几乎“失效”。

方法 3：手动清空缓存
sqlSession.clearCache();

方法 4：让查询变为 UPDATE / INSERT / DELETE 导致缓存自动清空
sqlSession.update(...); // 清空缓存
sqlSession.selectOne(...); // 必然重新查库

方法 5：在 mapper 上使用 flushCache="true"
如果一定要让某个 select 不用一级缓存：
<select id="selectById" flushCache="true">
    select * from student where id = #{id}
</select>
效果：
每次执行该语句前都会 clearCache()
强制重新查库
```

# 2. 二级缓存
```
⼆级缓存的范围是SqlSessionFactory。
```
## 2.1 使⽤⼆级缓存需要具备以下⼏个条件：
```
1.全局性地开启或关闭所有映射器配置⽂件中已配置的任何缓存。默认就是true，⽆需设置。
<setting name="cacheEnabled" value="true">
2.在需要使⽤⼆级缓存的SqlMapper.xml⽂件中添加配置：
<!--默认情况下，二级缓存是开启的。只要在对应的SqlMapperx.xml文件中添加如下标签，就表示此文件中使用二级缓存-->
<cache/>
<cache/>标签位置要求：<cache/> 必须放在 <mapper> 内 最顶层位置；必须在 所有 SQL 语句之前；<resultMap>、<select>、<insert>、<update>、<delete> 都必须在 <cache/> 之后
3.使⽤⼆级缓存的实体类对象必须是可序列化的，也就是必须实现java.io.Serializable接⼝
4.SqlSession对象关闭或提交之后，⼀级缓存中的数据才会被写⼊到⼆级缓存当中。此时⼆级缓存才可⽤。
```
## 2.2 ⼆级缓存的失效
```
只要两次查询之间出现了增删改操作。⼆级缓存就会失效。【⼀级缓存也会失效】
```
## 2.3 ⼆级缓存的相关配置（<cache/>标签的属性）
```
1.eviction：指定从缓存中移除某个对象的淘汰算法。默认采⽤LRU策略。
LRU：Least Recently Used。最近最少使⽤。优先淘汰在间隔时间内使⽤频率最低的对象。(其实还有⼀种淘汰算法LFU，最不常⽤。)
FIFO：First In First Out。⼀种先进先出的数据缓存器。先进⼊⼆级缓存的对象最先被淘汰。
SOFT：软引⽤。淘汰软引⽤指向的对象。具体算法和JVM的垃圾回收算法有关。
WEAK：弱引⽤。淘汰弱引⽤指向的对象。具体算法和JVM的垃圾回收算法有关。
2.flushInterval：
⼆级缓存的刷新时间间隔。单位毫秒。如果没有设置。就代表不刷新缓存，只要内存⾜够⼤，⼀直会向⼆级缓存中缓存数据。除⾮执⾏了增删改。
3.readOnly：
true：多条相同的sql语句执⾏之后返回的对象是共享的同⼀个。性能好。但是多线程并发可能会存在安全问题。
false：多条相同的sql语句执⾏之后返回的对象是副本，调⽤了clone⽅法。性能⼀般。但安全。
4.size：默认1024
设置⼆级缓存中最多可存储的java对象数量。
5.type：PERPETUAL是默认值，对应类org.apache.ibatis.cache.impl.PerpetualCache，这是默认缓存实现，永久缓存，回收依赖 eviction 策略
实现了 org.apache.ibatis.cache.Cache 接口就可以用于这个配置，只需要把类的全限定类名指定即可。
6.blocking：默认false
true：当多个线程同时访问同一个缓存 key 并且缓存未命中时，只有第一个线程去数据库查询，其他线程等待该线程查询完毕并把结果放入缓存
false：所有线程都会并发去数据库查询，容易造成“缓存击穿”
```

# 3. MyBatis集成EhCache（二级缓存实现）
```
集成EhCache是为了代替mybatis⾃带的⼆级缓存。⼀级缓存是⽆法替代的。
mybatis对外提供了接⼝，也可以集成第三⽅的缓存组件。⽐如EhCache、Memcache等。都可以。
EhCache是Java写的。Memcache是C语⾔写的。所以mybatis集成EhCache较为常⻅，按照以下步骤操作，就可以完成集成：
```
## 3.1 第⼀步：引⼊mybatis整合ehcache的依赖。
```
<!--mybatis集成ehcache的组件，本质上这就是一个适配器包，依靠依赖传递就可以将核心类库引入-->
<dependency>
    <groupId>org.mybatis.caches</groupId>
    <artifactId>mybatis-ehcache</artifactId>
    <version>1.2.2</version>
</dependency>
<!--本质上需要这个核心类库，靠着依赖传递，适配器包引入后就可以实现将核心类库一并引入-->
<!--<dependency>
    	<groupId>net.sf.ehcache</groupId>
    	<artifactId>ehcache</artifactId>
    	<version>2.10.9.2</version>
	</dependency>-->
<!--ehcache需要slf4j的⽇志组件,log4j不好使-->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.11</version>
    <scope>test</scope>
</dependency>
```
## 3.2 第⼆步：在类的根路径下新建ehcache.xml⽂件，并提供以下配置信息。
```
<?xml version="1.0" encoding="UTF-8"?>
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
         updateCheck="false">
    <!--磁盘存储:将缓存中暂时不使⽤的对象,转移到硬盘,类似于Windows系统的虚拟内存-->
    <diskStore path="d:/ehcache"/>

    <!--defaultCache：默认的管理策略-->
    <!--eternal：设定缓存的elements是否永远不过期。如果为true，则缓存的数据始终有效，如果为false那么还要根据timeToIdleSeconds，timeToLiveSeconds判断-->
    <!--maxElementsInMemory：在内存中缓存的element的最⼤数⽬-->
    <!--overflowToDisk：如果内存中数据超过内存限制，是否要缓存到磁盘上-->
    <!--diskPersistent：是否在磁盘上持久化。指重启jvm后，数据是否有效。默认为false-->
    <!--timeToIdleSeconds：对象空闲时间(单位：秒)，指对象在多⻓时间没有被访问就会失效。只对eternal为false的有效。默认值0，表示⼀直可以访问-->
    <!--timeToLiveSeconds：对象存活时间(单位：秒)，指对象从创建到失效所需要的时间。只对eternal为false的有效。默认值0，表示⼀直可以访问-->
    <!--memoryStoreEvictionPolicy：缓存的3 种清空策略-->
    <!--FIFO：first in first out (先进先出)-->
    <!--LFU：Less Frequently Used (最少使⽤).意思是⼀直以来最少被使⽤的。缓存的元素有⼀个hit 属性，hit 值最⼩的将会被清出缓存-->
    <!--LRU：Least Recently Used(最近最少使⽤). (ehcache 默认值).缓存的元素有⼀个时间戳，当缓存容量满了，⽽⼜需要腾出地⽅来缓存新的元素的时候，那么现有缓存元素中时间戳离当前时间最远的元素将被清出缓存-->
    <defaultCache eternal="false" maxElementsInMemory="1000" overflowToDisk="false" diskPersistent="false"
                  timeToIdleSeconds="0" timeToLiveSeconds="600" memoryStoreEvictionPolicy="LRU"/>

</ehcache>
```
## 3.3 第三步：修改SqlMapper.xml⽂件中的标签，添加type属性。
```
<cache type="org.mybatis.caches.ehcache.EhcacheCache"/>
```
## 3.4 第四步：编写测试程序使⽤。
```
```

# 4. 其他二级缓存实现
```
如memcache也可以集成到mbyatis中
```

# 5. Cache 接口源码分析

```java
/**
 * Cache 接口 — MyBatis 所有缓存实现的顶层接口
 * 源码：org.apache.ibatis.cache.Cache
 *
 * MyBatis 使用了装饰器模式来实现缓存的多种功能组合。
 * 核心实现：
 *   PerpetualCache（基础实现：HashMap）
 *     装饰器：
 *       LruCache（LRU 淘汰策略）
 *       FifoCache（FIFO 淘汰策略）
 *       SoftCache（软引用）
 *       WeakCache（弱引用）
 *       ScheduledCache（定时刷新）
 *       SerializedCache（序列化，用于读写安全的副本返回）
 *       LoggingCache（日志记录）
 *       SynchronizedCache（同步锁）
 *       TransactionalCache（事务缓存，二级缓存的事务包装）
 *       BlockingCache（阻塞缓存，防止缓存击穿）
 */
public interface Cache {

    /** 缓存唯一标识（通常为 namespace） */
    String getId();

    /** 向缓存中放入一个对象 */
    void putObject(Object key, Object value);

    /** 从缓存中获取一个对象 */
    Object getObject(Object key);

    /** 移除缓存中的一个对象 */
    Object removeObject(Object key);

    /** 清空缓存 */
    void clear();

    /** 获取缓存中对象的数量 */
    int getSize();

    /** 获取读写锁（可选） */
    default ReadWriteLock getReadWriteLock() { return null; }
}
```

## 5.1 PerpetualCache — 基础实现

```java
/**
 * PerpetualCache — 最基本的缓存实现
 * 内部使用 HashMap 存储，永不过期
 * MyBatis 的一级缓存直接使用 PerpetualCache
 */
public class PerpetualCache implements Cache {

    private final String id;
    // ★ 简单的 HashMap ★
    private final Map<Object, Object> cache = new HashMap<>();

    public PerpetualCache(String id) {
        this.id = id;
    }

    @Override
    public void putObject(Object key, Object value) {
        cache.put(key, value);
    }

    @Override
    public Object getObject(Object key) {
        return cache.get(key);
    }

    @Override
    public int getSize() {
        return cache.size();
    }

    @Override
    public void clear() {
        cache.clear();
    }
}
```

## 5.2 LruCache 装饰器

```java
/**
 * LruCache 通过装饰器模式在 PerpetualCache 基础上增加了 LRU 淘汰策略
 */
public class LruCache implements Cache {

    private final Cache delegate;                     // 被装饰的缓存
    private Map<Object, Object> keyMap;               // 记录访问顺序
    private Object eldestKey;                         // 最老的 key

    public LruCache(Cache delegate) {
        this.delegate = delegate;
        setSize(1024);  // 默认最大 1024 个对象
    }

    @Override
    public void putObject(Object key, Object value) {
        delegate.putObject(key, value);
        cycleKeyList(key);  // 更新访问记录
    }

    @Override
    public Object getObject(Object key) {
        keyMap.get(key);  // ★ 触发 LinkedHashMap 的访问排序 ★
        return delegate.getObject(key);
    }

    private void cycleKeyList(Object key) {
        keyMap.put(key, key);
        if (eldestKey != null) {
            delegate.removeObject(eldestKey);  // ★ 淘汰最老的对象 ★
            eldestKey = null;
        }
    }
}
```

## 5.3 CacheKey — 缓存 Key 的计算

```java
/**
 * CacheKey 用于唯一标识一次缓存查询
 * 源码：org.apache.ibatis.cache.CacheKey
 *
 * hashCode 的计算基于：
 *   1. multiplier（乘数因子，默认 37）
 *   2. hashcode（基于所有 update() 传入的值累加）
 *   3. checksum（所有 update() 对象 hashCode 的和）
 *   4. count（update() 调用的次数）
 *
 * update(Object) 方法每调用一次，都将一个新的因素加入计算
 * BaseExecutor.createCacheKey() 会连续 update 这些值：
 *   - MappedStatement 的 id
 *   - RowBounds 的 offset 和 limit
 *   - SQL 语句
 *   - 每个参数值
 *   - 环境 ID
 */
public class CacheKey implements Cloneable, Serializable {

    private static final int DEFAULT_MULTIPLIER = 37;
    private static final int DEFAULT_HASHCODE = 17;

    private final int multiplier;
    private int hashcode;
    private long checksum;
    private int count;

    // 存储 update 传入的对象（用于 equals 比较）
    private List<Object> updateList;

    public CacheKey() {
        this.hashcode = DEFAULT_HASHCODE;
        this.multiplier = DEFAULT_MULTIPLIER;
        this.count = 0;
        this.updateList = new ArrayList<>();
    }

    /**
     * 每次 update 会更新 hashcode，确保最终的 hashCode 由所有因素共同决定
     */
    public void update(Object object) {
        int baseHashCode = object == null ? 1 : ArrayUtil.hashCode(object);
        count++;
        checksum += baseHashCode;
        baseHashCode *= count;
        hashcode = multiplier * hashcode + baseHashCode;
        updateList.add(object);
    }

    @Override
    public boolean equals(Object object) {
        if (this == object) return true;
        if (!(object instanceof CacheKey)) return false;

        final CacheKey cacheKey = (CacheKey) object;
        if (hashcode != cacheKey.hashcode) return false;
        if (checksum != cacheKey.checksum) return false;
        if (count != cacheKey.count) return false;

        // 逐个比较 updateList 中的元素
        for (int i = 0; i < updateList.size(); i++) {
            Object thisObject = updateList.get(i);
            Object thatObject = cacheKey.updateList.get(i);
            if (!ArrayUtil.equals(thisObject, thatObject)) {
                return false;
            }
        }
        return true;
    }
}
```

## 5.4 TransactionalCache — 事务缓存

```java
/**
 * TransactionalCache 是二级缓存的事务包装
 * 源码：org.apache.ibatis.cache.decorators.TransactionalCache
 *
 * 为什么需要 TransactionalCache？
 *   二级缓存是跨 SqlSession 的（跨事务的），
 *   在一个事务中修改了数据但还没提交时，不能将未确认的数据放入共享的二级缓存。
 *   所以需要一个"事务缓冲"机制。
 *
 * 核心机制：
 *   - putObject() 时不直接写入真正的缓存，而是暂存在 entriesToAddOnCommit 中
 *   - commit() 时：将 entriesToAddOnCommit 写入真正的缓存
 *   - rollback() 时：丢弃 entriesToAddOnCommit
 *   - 事务进行中的 getObject()：会忽略未提交的查询结果
 */
public class TransactionalCache implements Cache {

    private final Cache delegate;                          // 真正的缓存对象

    // 事务提交时需要加入缓存的条目（暂存）
    private final Map<Object, Object> entriesToAddOnCommit;

    // 事务提交时需要从缓存中清除的 key（由增删改操作触发）
    private final Set<Object> entriesMissedInCache;

    @Override
    public void putObject(Object key, Object object) {
        // ★ 不直接写入 delegate，先暂存 ★
        entriesToAddOnCommit.put(key, object);
    }

    @Override
    public void commit() {
        if (clearOnCommit) {
            delegate.clear();
        }
        // ★ commit 时才将暂存的数据写入真正的缓存 ★
        flushPendingEntries();
        reset();
    }

    private void flushPendingEntries() {
        for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
            delegate.putObject(entry.getKey(), entry.getValue());  // ★ 写入真正的缓存 ★
        }
    }
}
```

## 5.5 二级缓存的完整装饰链

```
<cache/> 标签配置后，MyBatis 会构建如下装饰链：

  SynchronizedCache              ← 线程安全包装（最外层）
    → LoggingCache              ← 日志记录
      → SerializedCache         ← 对象序列化副本（readWrite=true 时）
        → LruCache             ← LRU 淘汰（或 FifoCache/SoftCache/WeakCache）
          → PerpetualCache     ← 基础 HashMap 实现（最内层）

一级缓存没有这些装饰器，直接就是 PerpetualCache（存在 BaseExecutor.localCache）
```