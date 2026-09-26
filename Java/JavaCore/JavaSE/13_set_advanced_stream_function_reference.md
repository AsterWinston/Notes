# 1.集合体系结构

```
集合体系结构（包括 Collection、List、Set、Map 等接口及其实现类）
```

# 2.Collection

## 2.1Collection通用方法

```
2.1.1public boolean add(E e)//List系列永远返回true，Set系列如果当前元素与已经存在元素重复返回false（插入失败），不重复返回true（插入成功）
2.1.2public void clear()//
2.1.3public boolean remove(E e)//
2.1.4public boolean contains(Object obj)//此方法内部调用对象的equals方法判断对象是否一致
2.1.5public boolean isEmpty()
2.1.6public int size()
```

## 2.2Collection的遍历方式

### 2.2.1迭代器遍历

```
2.2.1.1Collection集合获取迭代器的方法:Iterator<E> iterator()
```

#### 2.2.1.2Iterator对象中常用方法

```
2.2.1.2.1boolean hasNext()//判断当前位置是否有元素，有元素返回true，没有元素返回false
2.2.1.2.2E next()//获取当前位置的元素，并将迭代器对象移向下一个位置
2.2.1.2.3void remove()//删除 next() 方法返回的最后一个元素
```

#### 2.2.1.3迭代器遍历问题

```
2.2.1.3.1迭代器遍历完毕，指针不会复位（如果需要，重新获取迭代器对象即可）
2.2.1.3.2迭代器遍历时，不能用集合的方法进行增加或者删除（会导致并发修改异常 ConcurrentModificationException），应该使用迭代器自身的 remove() 方法
```

### 2.2.2增强for遍历（适用于集合和数组）

#### 2.2.2.1注意

```
2.2.2.1.1增强for的底层就是迭代器，为了简化迭代器的代码书写的
2.2.2.1.2它是jdk5之后出现的，其内部原理就是一个iterator
2.2.2.1.3所有的单列集合和数组才能用增强for进行遍历
```

#### 2.2.2.2格式

```
for(元素的数据类型 变量名:数组或者单列集合){}//修改变量不会导致数组或者集合被改变
```

### 2.2.3Lambda表达式遍历

#### 2.2.3.1集合内部的方法:default void forEach(Consumer\<? super T\> action){}//重写Consumer里的accept

```
集合内部的方法:default void forEach(Consumer<? super T> action){}//重写Consumer里的accept
```

#### 2.2.3.2底层原理:根据子类的不同调用子类重写的方法，比如ArrayList调用的是普通for遍历方法

```
底层原理:根据子类的不同调用子类重写的方法，比如ArrayList调用的是普通for遍历方法
```

# 3.List

## 3.1方法（相对于父类独有的）

```
3.1.1void add(int index, E element)//在指定索引处添加元素
3.1.2E remove(int index)//删除指定索引处的元素
3.1.3E set(int index, E element)//修改指定索引处的元素，返回被替换的旧元素
3.1.4E get(int index)//获取指定索引处的元素
```

## 3.2遍历方式（相对于父类独有的）

### 3.2.1列表迭代器

#### 3.2.1.1List对象获取ListIterator对象:ListIterator\<E\> listIterator()

```
List对象获取ListIterator对象:ListIterator<E> listIterator()
```

#### 3.2.1.2ListIterator的方法

```
3.2.1.2.1void add(E e)//在当前迭代器位置之前插入元素
3.2.1.2.2boolean hasNext()//
3.2.1.2.3E next()//
3.2.1.2.4void remove()//
3.2.1.2.5void set(E e)//用指定元素替换next或previous返回的最后一个元素
3.2.1.2.6boolean hasPrevious()//判断是否有前一个元素
3.2.1.2.7E previous()//获取前一个元素，并将迭代器对象移向后一个位置
3.2.1.2.8int nextIndex()//返回 next() 元素索引
3.2.1.2.9int previousIndex()//返回 previous() 元素索引
```

### 3.2.2普通for遍历

```
普通for遍历（通过索引）
```

# 4.ArrayList（数组）

```
ArrayList（底层基于数组实现），查询快，增删慢
```

# 5.LinkdList（双向链表）

```
5.1public void addFirst(E e)//在头部添加元素
5.2public void addLast(E e)//在尾部添加元素（等同于 add(E e)）
5.3public E getFirst()//获取头部元素
5.4public E getLast()//获取尾部元素
5.5public E removeFirst()//移除并返回头部元素
5.6public E removeLast()//移除并返回尾部元素
```

# 6.泛型深入

## 6.1泛型:是jdk5引入的特性，可以在编译阶段约束操作的数据类型，并进行检查

```
泛型:是jdk5引入的特性，可以在编译阶段约束操作的数据类型，并进行检查
```

## 6.2泛型的格式:\<数据类型\>

```
泛型的格式:<数据类型>（例如 `List<String>`）
```

## 6.3泛型的细节（Java的泛型是伪泛型）

```
6.3.1泛型中不能写基本数据类型，必须使用对应的包装类
6.3.2指定泛型的具体类型后，传递数据时，可以传入该类类型或者其子类类型（这是针对元素本身，而不是集合的继承关系）
6.3.3如果不写泛型，默认类型是Object，会给出警告
```

## 6.4泛型的定义

### 6.4.1类后面（泛型类）

```
<authority> class <class_name><generic_type_name>{<class_def>}
```

### 6.4.2方法上面（泛型方法）

```
<authority><generic_type_name> <ret_val_type> <func_name>(params){<func_body>}//such as:public<T> void func(T t){}
```

### 6.4.3接口后面（泛型接口）//使用时可以选择在实现类给出具体的类型，或者延续泛型类

```
<authority> interface <interface_name><generic_type>{<interface_def>}
```

## 6.5泛型的继承和通配符

### 6.5.1通配符用于限定泛型方法的参数

```
6.5.1.1<?>//任意类型（无限定通配符）
6.5.1.2<? extends TYPE>//TYPE以及TYPE的子类类型（上界通配符），用于限制读取数据
6.5.1.3<? super TYPE>//TYPE以及TYPE的父类类型（下界通配符），用于限制写入数据
```

### 6.5.2通配符用于限定泛型类或接口（上界限定）

```
6.5.2.1<generic_type_name extends TYPE>//TYPE以及其子类类型
```

# 7.Set系列集合

```
Set系列集合（特点：无序、不重复、无索引）
```

# 8.HashSet（底层为哈希表）（哈希表的组成:jdk8之前为数组+链表，jdk8开始为数组+链表+红黑树）

## 8.1哈希值

```
8.1.1根据hashCode方法计算出来的int类型的整数
8.1.2该方法定义在Object类中，所有对象都可以调用，默认使用地址值进行计算
8.1.3一般情况下，会重写hashCode方法，利用对象内部的属性值计算哈希值
```

## 8.2对象的哈希值特点

```
8.2.1如果没有重写hashCode方法，不同对象计算出的哈希值是不同的
8.2.2如果已经重写hashCode方法，不同对象只要属性值相同，计算出的哈希值就是一样的
8.2.3在小部分情况下，不同的属性值或者不同的地址值计算出来的哈希值也可能一样（哈希冲突）
```

## 8.3重写hashCode和equals

```
重写 hashCode 和 equals 方法以确保自定义对象的唯一性判断逻辑正确。
```

# 9.LinkedHashSet（HashSet的前提下加双向链表记录顺序）

```
LinkedHashSet（在 HashSet 的底层哈希表结构基础上，额外维护了一个双向链表，用于记录元素的插入顺序，保证遍历有序）
```

# 10.TreeSet（底层为红黑树）

## 10.1排序规则

```
10.1.1默认排序/自然排序:Javabean类实现Comparable接口指定比较规则，当实现的compareTo方法返回值小于零，则treeSet认为插入元素比被比较元素小，0则认为已存在，大于零则认为大
10.1.2比较器排序:创建TreeSet对象的时候，传递比较器Comparator指定规则（Comparator是一个函数式接口），默认使用第一种规则，如果指定第二种，则以第二种为准，这种方法常用于Java库中的类将Comparable写好的情况，所以需要自定义时，只能用这种方法来自定义排序，而不是修改源码
```

# 11.双列集合Map

## 11.1特点

```
11.1.1双列集合一次需要存一对数据，分别为键和值
11.1.2键不能重复，值可以重复
11.1.3键和值是一一对应的，每一个键只能找到自己对应的值
11.1.4键+值这个整体称为键值对或者键值对对象，在Java中叫做“Entry对象”
```

## 11.2Map常见API

```
11.2.1V put(K key, V value)//添加元素，可以用于添加，也可以用来覆盖（已经存在对应键时）（有另一个方法可以控制为不覆盖地插入）
11.2.2V remove(Object key)//根据键值删除键值对
11.2.3void clear()//移除所有的键值对
11.2.4boolean containsKey(Object key)//判断集合是否包含指定的键
11.2.5boolean containsValue(Object value)//判断集合是否包含指定的值
11.2.6boolean isEmpty()//判断集合是否为空
11.2.7int size()//集合的长度，也就是集合中键值对的个数
```

## 11.3Map的遍历方式

```
11.3.1键找值:keySet()+get()
11.3.2键值对:entrySet()+getKey()+getValue()
11.3.3Lambda表达式:default void forEach(BiConsumer<? super K, ? super V> action){}
```

# 12.HashMap

## 12.1特点

```
12.1.1HashMap是Map里面的一个实现类
12.1.2Map没有额外的特定方法，直接使用Map中的即可
12.1.3特点都是由键决定的:无序、不重复、无索引
12.1.4HashMap跟HashSet底层原理一样，都是哈希表结构（数组、链表和红黑树，红黑树比较键值用的是哈希值的大小来比较，所以不需要实现Comparable接口）
```

## 12.2注意

```
12.2.1HashMap底层是哈希表结构
12.2.2依赖hashCode方法和equals方法保证键的唯一
12.2.3如果键存储的是自定义对象，需要重写hashCode和equals方法，如果值存储自定义对象，不需要重写hashCode和equals方法
```

# 13.LinkedHashMap

## 13.1特点

```
13.1.1由键决定:有序（按插入顺序）、不重复、无索引
13.1.2底层数据结构是哈希表，只是每个键值对元素又额外地多了一个双链表的机制记录存储顺序
```

# 14.TreeMap

## 14.1特点

```
14.1.1TreeMap跟TreeSet底层原理一样，都是红黑树结构的
14.1.2由键决定特性:不重复、无索引、可排序
14.1.3可排序:对键进行排序
```

## 14.2注意:默认按照键的从小到大进行排序，也可以自己规定键的排序规则

```
14.2.1实现Comparable接口，指定比较规则
14.2.2创建集合时传递Comparator比较器对象，指定比较规则
```

# 15.可变参数（jdk5开始）

## 15.1格式:TYPE...\<var\_name\>

```
格式:TYPE...<var_name>
```

## 15.2作用:在形参中接收不定个数参数

```
作用:在形参中接收不定个数参数
```

## 15.3本质:一个数组

```
本质:一个数组
```

## 15.4注意

```
15.4.1形参列表中可变参数只能有一个
15.4.2可变参数必须放在形参列表的最后面
```

# 16.Collections:工具类

## 16.1常用api

```
16.1.1public static <T> boolean addAll(Collection<T> c, T...elements)//批量添加元素
16.1.2public static void shuffle(List<?> list)//打乱List集合元素的顺序
16.1.3public static <T> void sort(List<T> list)//按元素的自然顺序升序排序
16.1.4public static <T> void sort(List<T> list, Comparator<T> c)//按指定的比较器排序
16.1.5public static <T> int binarySearch(List<T> list, T key)//二分查找（要求列表有序）
16.1.6public static <T> void copy(List<T> dest, List<T> src)//将源列表元素复制到目标列表
16.1.7public static <T> int fill(List<T> list, T obj)//用指定对象替换列表中的所有元素
16.1.8public static <T> void max/min(Collection<T> coll)//根据默认的自然排序获取最大/最小值
16.1.9public static <T> void swap(List<?> list, int i, int j)//交换列表中指定索引的元素
```

# 17.创建不可变集合

```
17.1static <E> List<E> of(E...elements)//创建一个具有指定元素的List集合对象（List类中的方法）
17.2static <E> Set<E> of(E...elements)//创建一个具有指定元素的Set集合对象
17.3static <K,V> Map(K,V) of(E...elements)//创建一个具有指定元素的Map集合对象
17.3.1键不能重复
17.3.2Map里面的of方法，参数是有上限的，最多只能传递20个参数，10个键值对
17.3.3如果要传递多个键值对，可以使用ofEntries()，然后用toArray([T[] t])（这个需求下需要指定类型），将得到的数组放入即可
```

# 18.Map中的copyOf（jdk10）

```
Map中的copyOf(Map<? extends K, ? extends V> map)//创建指定 Map 的不可变副本
```

# 19.Stream

## 19.1获取Stream流

```
19.1.1单列结合:default Stream<E> stream()//Collection中的默认方法
19.1.2双列结合:不能直接获取到Stream流，需要转换为单列集合（如 keySet(), entrySet(), values()）
19.1.3数组:public static<T> Stream(T[] array)//Arrays工具类中的静态方法，可以是普通数据类型的数组，也可以是引用数据类型的数组
19.1.4同类多个数据:public static<T> Stream<T> of(T...vlaues)//Stream接口中的静态方法，如果要传递数组，那么只能是引用数据类型的，不能是普通数据类型的
```

## 19.2Stream流的中间方法

```
19.2.1Stream<T> filter(Predicate<? super T> predicate)//过滤
19.2.2Stream<T> limit(long maxsize)//获取前几个元素
19.2.3Stream<T> skip(long n)//跳过前几个元素
19.2.4Stream<T> distinct()//元素去重，依赖hashCode和equals方法
19.2.5static <T> Stream<T> concat(Stream a, Stream b)//合并a和b两个流为一个流
19.2.6Stream<R> map(Function<? super T, ? extends R> mapper)//转换流中的数据类型 R
```

## 19.3注意

```
19.3.1中间方法，返回新的Stream流，原来的Stream流只能使用一次，建议使用链式编程
19.3.2修改Stream流中的数据，不会影响原来集合或者数组中的数据
```

## 19.4Stream流的终结方法

```
19.4.1void forEach(Consumer action)//遍历
19.4.2long count()//统计
19.4.3toArray([IntFunction<? extends Object[]> if])//收集流中的数据，放到数组中
19.4.4collect(Collector collector)//收集流中的数据，放到集合中（collector通过Collectors类中的静态方法可以获取）
```

# 20.方法引用（用于替换函数式接口中的抽象函数）

## 20.1方法引用的要求

```
20.1.1引用处需要是函数式接口
20.1.2被引用的方法需要已经存在
20.1.3被引用方法的形参和返回值需要跟抽象方法的形参和返回值保持一致
20.1.4被引用方法的功能需要满足当前的要求
```

## 20.2方法引用的分类

### 20.2.1引用静态方法:类名::静态方法（本类或者其他类）

```
引用静态方法:类名::静态方法（本类或者其他类）
```

### 20.2.2引用成员方法

```
20.2.2.1引用其他类的:其他类对象::方法名
20.2.2.2引用本类的:this::方法名（非静态）
20.2.2.3引用父类的:super::方法名（非静态）
```

### 20.2.3引用构造方法:类名::new

```
引用构造方法:类名::new
```

### 20.2.4其他调用方式

```
20.2.4.1使用类名调用成员方法:类名::成员方法（这种方式的规则，被引用方法的形参需要跟抽象方法的第二个到最后一个形参保持一致，抽象方法的第一个参数决定了可以引用哪些类中的方法，这是因为self参数在第一个）
20.2.4.2引用数组的构造方法:数据类型[]::new（这种方法适用于形参为整数，返回值为数组的抽象方法）
```