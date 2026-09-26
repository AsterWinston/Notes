# 1. 知识点
- **Collection体系**：Collection接口（单列集合）→ List（有序可重复）/ Set（无序不可重复）/ Queue（队列）
- **Map体系**：Map接口（双列集合）→ HashMap / LinkedHashMap / TreeMap / Hashtable / ConcurrentHashMap
- **List实现类**：ArrayList（数组）/ LinkedList（双向链表）/ Vector（线程安全已过时）/ CopyOnWriteArrayList（读写分离）
- **Set实现类**：HashSet（HashMap实现）/ LinkedHashSet（LinkedHashMap实现）/ TreeSet（TreeMap实现）/ CopyOnWriteArraySet
- **Queue/Deque**：LinkedList / ArrayDeque / PriorityQueue / BlockingQueue系列
- **ArrayList底层**：Object[]数组，空参构造初始为空数组，首次add扩容为10，后续扩容为原容量的1.5倍（oldCapacity + oldCapacity >> 1）
- **ArrayList扩容**：扩容时创建新数组，调用System.arraycopy复制元素，因此频繁扩容影响性能，建议预估容量
- **fail-fast机制**：集合内部维护modCount（修改次数），迭代器维护expectedModCount，迭代时检测两者不等则抛出ConcurrentModificationException
- **迭代时删除**：集合自身的remove/clear方法会修改modCount，导致正在迭代的迭代器next时抛异常；应使用迭代器的remove()方法（同步expectedModCount）
- **LinkedList**：双向链表，Node节点含prev/item/next，实现了List和Deque接口
- **ArrayList vs LinkedList**：ArrayList按索引O(1)访问，尾插O(1)（均摊），头插/中间插O(n)；LinkedList头尾操作O(1)（Deque），按索引O(n)，内存占用更大（需维护前后指针）
- **Vector**：ArrayList的线程安全版，所有方法加synchronized，现已过时，Java5建议用Collections.synchronizedList或CopyOnWriteArrayList替代
- **HashMap JDK1.7**：数组+链表，头插法（新节点插入链表头部），多线程扩容可能形成死链（环形链表）
- **HashMap JDK1.8**：数组+链表+红黑树（链表长度≥8且数组长度≥64时树化，≤6时退化为链表），尾插法（避免死链）
- **HashMap put流程**：计算key的hashCode → 高16位与低16位异或（扰动函数）→ (n-1) & hash定位桶 → 桶为空直接插入 → 桶非空判断是链表还是树 → 遍历链表/树看是否存在相同key → 有则替换value，无则插入节点 → 插入后判断是否需扩容
- **HashMap扰动函数**：hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)，目的是让hashCode的高位参与低位运算，降低哈希碰撞
- **HashMap容量为2的幂的原因**：取模运算hash%n可用位运算(n-1)&hash替代（当n为2的幂），效率高；扩容时迁移元素方便（原位置或原位置+旧容量）
- **HashMap负载因子默认0.75**：时间与空间的折中，太高碰撞概率大（查询慢），太低浪费空间
- **HashMap扩容**：扩容为原容量的2倍，转移每个节点的hash & oldCap，结果为0留在原位，非0则移到原位置+oldCap
- **HashMap线程不安全表现**：JDK1.7头插法扩容死链导致CPU100%；JDK1.8 put时数据覆盖（两个线程同时put不同key分配到同桶可能丢失一个）
- **ConcurrentHashMap JDK1.7**：分段锁Segment（继承ReentrantLock），16个Segment每个独立加锁，并发度16，put时先hash定位Segment再tryLock
- **ConcurrentHashMap JDK1.8**：CAS + synchronized锁链表头节点/树根节点，不再使用Segment；初始化通过CAS竞争initTable；扩容支持多线程协同transfer；计数使用CounterCell数组（类似LongAdder）
- **ConcurrentHashMap put流程**：spread计算hash → 循环CAS尝试 → 若数组未初始化则initTable → 若桶为空CAS插入 → 若桶Hash为MOVED(-1)则协助扩容 → 否则synchronized锁住桶首节点 → 遍历链表/树查找或插入 → 根据binCount决定是否树化 → addCount计数并判断扩容
- **LinkedHashMap**：继承HashMap，Entry多了before/after指针维护双向链表，维护插入顺序（accessOrder=false默认）或访问顺序（accessOrder=true，LRU缓存）
- **LRU缓存实现**：LinkedHashMap设置accessOrder=true，重写removeEldestEntry(Entry eldest)方法，当size超过容量时返回true自动删除最老元素
- **TreeMap**：红黑树实现，根据key的自然排序（Comparable）或构造时传入的Comparator比较器进行排序
- **HashSet**：底层是HashMap，元素作为key存入，value统一是一个静态final Object PRESENT常量
- **TreeSet**：底层是TreeMap，同样使用PRESENT占位
- **HashTable**：JDK1.0遗留，全表加synchronized锁，效率低；不允许null键/值
- **Collections.unmodifiableXxx**：装饰器模式，返回UnmodifiableCollection/List/Map等，通过final字段持有原始集合，所有修改方法抛UnsupportedOperationException
- **不可变集合（JDK9+）**：List.of() / Set.of() / Map.of() 创建真正的不可变集合（不能包含null元素）
- **CopyOnWriteArrayList**：写时复制，add/set等操作先加ReentrantLock，复制底层数组，新数组上修改，再替换引用；读操作无锁直接读，适合读多写少场景
- **CopyOnWriteArrayList迭代器**：弱一致性（快照读），迭代器创建时持有数组快照，迭代期间修改不影响迭代器
- **BlockingQueue**：阻塞队列接口，支持生产者消费者模式，put(e)队列满时阻塞，take()队列空时阻塞
- **ArrayBlockingQueue**：有界，底层数组，单锁（put和take同一把ReentrantLock）+ 两个Condition(notFull/notEmpty)
- **LinkedBlockingQueue**：可选有界（默认Integer.MAX_VALUE），底层单向链表，双锁（putLock和takeLock分离）提高并发性能
- **PriorityQueue**：无界优先队列，底层Object[]数组实现二叉堆，默认小顶堆；插入siftUp上浮，删除siftDown下沉；非线程安全
- **DelayQueue**：无界阻塞队列，元素必须实现Delayed接口，只有延迟期满的元素才能被取出；内部使用PriorityQueue + ReentrantLock + Condition
- **SynchronousQueue**：没有容量的阻塞队列，每个put必须等待一个take（反之亦然），适合直接传递场景；公平模式使用TransferQueue（FIFO），非公平模式使用TransferStack（LIFO）
- **Iterator vs ListIterator**：Iterator可正向遍历List和Set，有hasNext/next/remove；ListIterator是Iterator子接口，专用于List，支持双向遍历（hasPrevious/previous），支持add/set修改

# 2. 面试题

## 2.1 ArrayList的扩容机制是怎样的？
默认构造函数创建ArrayList时，elementData被赋值为空数组DEFAULTCAPACITY_EMPTY_ELEMENTDATA（区别于有参构造的空数组，用于区分是否为默认构造）。首次add时，容量扩容为DEFAULT_CAPACITY=10。此后每次扩容，新容量 = oldCapacity + (oldCapacity >> 1)，即1.5倍。

扩容流程：add(E e) → ensureCapacityInternal(size + 1) → calculateCapacity → 首次返回10 → ensureExplicitCapacity → 若超出当前容量则grow() → int newCapacity = oldCapacity + (oldCapacity >> 1) → 若newCapacity仍不够则使用需求容量 → 若需求容量超过MAX_ARRAY_SIZE则调hugeCapacity（最大Integer.MAX_VALUE） → Arrays.copyOf创建新数组。

## 2.2 ArrayList的fail-fast机制是什么？
fail-fast是集合的一种错误检测机制。ArrayList内部维护modCount变量记录集合结构修改次数（add/remove等操作会增加）。迭代器创建时记录expectedModCount = modCount。每次迭代调用next()/remove()时检查 expectedModCount == modCount，不等则立即抛ConcurrentModificationException。

注意：fail-fast不能保证100%检测到并发修改，只是尽力而为。解决方案：使用CopyOnWriteArrayList（快照读）或使用迭代器自身的remove()方法。

## 2.3 HashMap的put方法的执行流程？
1. 计算key的hash值：`(key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16)`
2. 检查数组是否未初始化，是则调用resize()初始化
3. 通过 (n-1) & hash 计算桶下标
4. 若桶为空，直接新建Node放入
5. 若桶非空：
   - 检查桶首节点hash和key是否相同（先==判断，再equals判断），相同则替换e
   - 若是TreeNode节点，调用putTreeVal插入红黑树
   - 否则遍历链表，若找到相同key则替换e，否则尾插新节点；插入后若链表长度>=8，调用treeifyBin尝试树化（还要判断数组长度>=64，否则仅扩容）
6. 若e不为空（找到已存在的key），替换value并返回oldValue
7. 修改modCount++，size++，若size > threshold则resize()
8. 返回null（新增）或oldValue（覆盖）

## 2.4 HashMap为什么容量是2的幂？负载因子为什么默认0.75？
**容量为2的幂**：
1. 位运算求余：当n=2^k时，hash % n = hash & (n-1)，位运算效率远高于取模
2. 扩容迁移方便：扩容时将Node的hash & oldCap，结果为0留在原位，非0移到原位+oldCap，不需要重新计算hash
3. 元素分布更均匀：2的幂-1的二进制全是1，与hash做&运算分布更均匀

**负载因子0.75**：
- 经验折中值。若太大（如1.0），虽节约空间但哈希碰撞概率增大，链表/红黑树更长，查询性能下降
- 若太小（如0.3），空间浪费多，频繁扩容
- 0.75在泊松分布下，链表达到8的概率极低（约0.00000006），空间利用率接近75%

## 2.5 HashMap JDK1.7和1.8的主要区别？为什么1.7多线程扩容死循环？
| 特性 | JDK1.7 | JDK1.8 |
|------|--------|--------|
| 数据结构 | 数组+链表 | 数组+链表+红黑树 |
| 插入方式 | 头插法 | 尾插法 |
| 初始化 | inflateTable | 首次put时resize |
| hash算法 | 多次扰动（4次位运算+5次异或） | 一次异或（高位参与低位） |
| 节点类型 | Entry | Node/红黑树TreeNode |
| 扩容条件 | size >= threshold && table[i]!=null | size > threshold |
| 扩容后转移 | 重新遍历计算index | hash & oldCap判断移动 |

1.7死循环原因：并发扩容时，使用头插法反转链表元素顺序，两个线程同时transfer可能导致链表形成环形引用。1.8改用尾插法，不会反转顺序，解决了死循环问题（但仍有数据覆盖问题）。

## 2.6 ConcurrentHashMap在JDK1.7和1.8的实现区别？
**JDK1.7（分段锁）**：
- 数据结构：Segment数组（继承ReentrantLock） + HashEntry数组（每个Segment内是个小HashMap）
- 并发度：默认16个Segment，最大65536，一旦初始化不可扩容Segment数量
- put流程：hash寻Segment → tryLock() → 失败则自旋（默认64次）→ 再失败则lock()阻塞等待 → Segment内put
- 扩容：Segment内部独立扩容（两倍）
- size()：先不加锁统计2次，不一致则全部加锁统计

**JDK1.8（CAS+sync）**：
- 数据结构：Node数组+链表+红黑树（与HashMap结构一致）
- 锁粒度：锁在数组的每个桶的链表头节点（更细）
- put流程：CAS尝试 → 失败则synchronized锁桶首节点 → 遍历链表/树
- 扩容：多线程协同迁移，通过transferIndex分任务
- size()：使用CounterCell数组+baseCount（类似LongAdder），无锁精确
- 初始化：CAS竞争initTable

## 2.7 LinkedHashMap如何实现LRU缓存？
LinkedHashMap继承HashMap，Entry节点额外维护before/after双向链表指针。

默认accessOrder=false维护插入顺序。设置accessOrder=true后，每次get/put操作会将访问的Entry移到双向链表尾部，链表头部即为最久未使用的元素。

实现LRU：设置accessOrder=true，重写removeEldestEntry(Entry eldest)方法：
```java
public class LRUCache<K,V> extends LinkedHashMap<K,V> {
    private int capacity;
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > capacity;
    }
}
```
HashMap的putVal方法在插入新节点后，会回调afterNodeInsertion(evict)，其中检查removeEldestEntry，若返回true则调用removeNode删除链表头部元素。

## 2.8 HashSet如何保证元素唯一？
HashSet底层是HashMap，元素作为key存入，value是统一的静态final Object PRESENT常量。

add(E e)调用map.put(e, PRESENT)，利用HashMap的put机制：如果该key已存在（hash相同且equals为true），则put返回旧value（非null）=> add返回false；如果是新key，put返回null => add返回true。

LinkedHashSet继承HashSet，构造中通过super调用LinkedHashMap构造，使用双向链表维护插入顺序。

## 2.9 CopyOnWriteArrayList的原理和适用场景？
原理：写时复制。内部用volatile Object[] array保存数据。读操作（get/size/contains等）不加锁，直接读当前数组。写操作（add/set/remove等）先获取ReentrantLock锁，复制当前array为新数组，在新数组上修改，最后将array引用指向新数组，释放锁。

优点：读操作完全无锁，性能高，不会抛ConcurrentModificationException。
缺点：写操作需复制全量数组，内存开销大，实时性不是强一致性（迭代器是快照）。
适用场景：读多写少（白名单/黑名单配置、事件监听器列表等），数据量不大的场景。

迭代器的弱一致性：迭代器创建时获取当前数组的快照（array引用赋值给迭代器内部snapshot），后续对集合的修改（通过写时复制建立新数组）不影响迭代器的快照。

## 2.10 HashMap线程不安全有哪些表现？如何解决？
表现：
1. **数据覆盖**：两个线程同时put不同的key但落同一桶，A线程判断桶为空但还未写入时被挂起，B线程完成写入，A线程恢复后用自己数据覆盖B线程的数据
2. **size不准确**：多个线程同时add时size++是复合操作，实际元素数可能小于size
3. **JDK1.7死链**：并发扩容时头插法反转链表形成环形引用，get时死循环CPU100%

解决方案：
- 使用ConcurrentHashMap
- 使用Collections.synchronizedMap(new HashMap<>())
- 使用Hashtable（不推荐，全表锁性能差）

## 2.11 BlockingQueue有哪些实现？说说生产者消费者模式
常用实现：
- **ArrayBlockingQueue**：有界，数组实现，单锁+双Condition（notFull, notEmpty），FIFO
- **LinkedBlockingQueue**：可选有界（默认无界Integer.MAX_VALUE），链表实现，双锁（putLock/takeLock分离），吞吐量通常高于ArrayBlockingQueue
- **SynchronousQueue**：容量为0的阻塞队列，每个put必须对应一个take，不能存储元素，适合任务直接交接的Exchanger场景
- **PriorityBlockingQueue**：无界，基于优先级的数组二叉堆，元素需实现Comparable
- **DelayQueue**：无界，元素需实现Delayed接口，只有延迟期满才能取出

生产者消费者模式：生产者put元素（队列满则阻塞等待），消费者take元素（队列空则阻塞等待）。Condition的signal()/await()实现等待唤醒机制。ArrayBlockingQueue使用notFull.await()和notEmpty.signal()实现线程协调。

## 2.12 Iterator和ListIterator有什么区别？
| Iterator | ListIterator |
|----------|-------------|
| 适用于所有Collection子类(List/Set) | 仅适用于List |
| 只能正向遍历 | 支持双向遍历(hasPrevious/previous) |
| 只有remove()方法 | 有add()/set()/remove()方法 |
| 使用cursor跟踪位置 | 使用cursor和lastRet跟踪位置 |
| 无法获取当前位置索引 | nextIndex()/previousIndex()获取索引 |

ListIterator在遍历时可以add元素（通过列表迭代器添加，插入在cursor之前），可以set替换上次next/previous返回的元素。而Iterator的remove删除的是上次next返回的元素，不支持add。

## 2.13 TreeMap如何实现排序？红黑树的特性是什么？
排序方式：
1. 自然排序：Key必须实现Comparable接口，按照compareTo方法排序
2. 比较器排序：构造TreeMap时传入Comparator实现，按照compare方法排序
3. 优先级：Comparator > Comparable，即如果有Comparator则使用Comparator

红黑树特性：
- 每个节点非红即黑
- 根节点是黑色
- 叶子节点（NIL空节点）是黑色
- 红节点的两个子节点必须都是黑色（不允许连续红色）
- 从任一节点到其每个叶子节点的路径包含相同数量的黑色节点

红黑树不是严格平衡的AVL树，但保证了最长路径不超过最短路径的2倍，查找效率O(log n)，插入/删除最多3次旋转即可恢复平衡（优于AVL）。

## 2.14 PriorityQueue的底层实现原理？
PriorityQueue底层使用Object[]数组实现二叉堆（默认小顶堆），元素必须实现Comparable接口或传入Comparator。堆是一棵完全二叉树，父节点索引为i，左子节点2i+1，右子节点2i+2。

**siftUp（上浮）**：add/offer时，元素添加到数组末尾（堆的最下层最右），然后与父节点比较，若小于父节点则交换，递归上浮直到满足堆性质或到达根节点。时间复杂度O(log n)

**siftDown（下沉）**：poll/remove时，取出根节点（最小元素），将数组最后一个元素放到根位置，然后与左右子节点中较小者比较，若大于则交换，递归下沉直到满足堆性质或到达叶子。时间复杂度O(log n)

## 2.15 Collections.unmodifiableList()的原理和使用场景？
原理：使用装饰器模式。`Collections.unmodifiableList(list)`返回一个UnmodifiableList对象，该对象内部通过final字段持有原始列表引用。读取方法（get/size/iterator等）直接委托给原始列表，但所有修改方法（add/remove/clear/set等）都会直接抛出UnsupportedOperationException。

注意：UnmodifiableCollection只是视图，实际数据仍保存在原始集合中。如果通过原始集合引用修改数据，unmodifiable集合的数据也会变化。

JDK9+推荐使用List.of()/Set.of()/Map.of()创建真正不可变的集合，它们不允许null元素且更高效。

使用场景：返回不可修改的内部数据视图（防御性编程）、方法参数传递后防止被修改、配置性的全局常量集合。
