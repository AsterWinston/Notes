# 1. Optional 概述

```
Optional 是 JDK8 引入的容器类（java.util.Optional），用于优雅地处理 null 值。
一个 Optional 对象要么包含一个非空值，要么为空（Optional.empty()）。

设计目的：
  1. 显式表达"值可能为空"的语义，减少空指针异常（NullPointerException）
  2. 强制调用者处理空值情况，编写更健壮的代码
  3. 避免到处写 if (obj != null) 的繁琐判断

注意：Optional 本身是值类型，不要将 Optional 用作类的字段（序列化问题），
     主要用于方法返回值。
```

# 2. 创建 Optional 对象

```
2.1 Optional.of(T value)
    创建一个包含非 null 值的 Optional。
    如果传入 null，立即抛出 NullPointerException。
      Optional<String> opt = Optional.of("Hello");

2.2 Optional.ofNullable(T value)
    创建一个可能为 null 的 Optional。
    如果 value 为 null，返回 Optional.empty()；否则同 Optional.of(value)。
      Optional<String> opt = Optional.ofNullable(null);  // Optional.empty

2.3 Optional.empty()
    创建一个空的 Optional 对象（不包含任何值）。
      Optional<String> opt = Optional.empty();
```

# 3. 判断值是否存在

```
3.1 boolean isPresent()
    是否有值。JDK11 起不推荐使用，建议优先使用 ifPresent 等方法。
      if (opt.isPresent()) {
          String value = opt.get();
      }

3.2 boolean isEmpty()
    JDK11+，是否为空。是 isPresent() 的反义词。
      if (opt.isEmpty()) {
          // 处理空值
      }

3.3 void ifPresent(Consumer<? super T> action)
    如果值存在，则执行指定操作；不存在则什么都不做。
      opt.ifPresent(v -> System.out.println(v));
      opt.ifPresent(System.out::println);  // 方法引用写法

3.4 void ifPresentOrElse(Consumer<? super T> action, Runnable emptyAction)
    JDK9+，如果有值执行 action，否则执行 emptyAction。
      opt.ifPresentOrElse(
          v -> System.out.println("Value: " + v),
          () -> System.out.println("No value")
      );
```

# 4. 获取值

```
4.1 T get()
    获取值。如果为空则抛出 NoSuchElementException。
    JDK10 起不推荐直接使用，建议使用 orElse/orElseGet/orElseThrow。
      String value = opt.get();

4.2 T orElse(T other)
    获取值，如果为空则返回默认值 other。
      String value = opt.orElse("默认值");

    注意：orElse 中的默认值总会计算（即使 Optional 有值）。
    如果默认值的计算开销较大，应该使用 orElseGet。

4.3 T orElseGet(Supplier<? extends T> supplier)
    获取值，如果为空则通过 supplier 生成默认值（惰性求值，只有为空时才计算）。
      String value = opt.orElseGet(() -> computeDefaultValue());
      // 有值时不会执行 computeDefaultValue()

4.4 T orElseThrow()
    JDK10+，获取值，为空则抛出 NoSuchElementException。
      String value = opt.orElseThrow();

4.5 <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier)
    获取值，为空则抛出自定义异常。
      String value = opt.orElseThrow(() -> new IllegalArgumentException("值不能为空"));
```

# 5. 值转换和过滤

```
5.1 <U> Optional<U> map(Function<? super T, ? extends U> mapper)
    如果值存在，应用 mapper 函数转换值，返回新的 Optional。
    如果原值不存在，返回 Optional.empty()。
      Optional<Integer> len = opt.map(String::length);
      // "Hello" → Optional.of(5)
      // Optional.empty() → Optional.empty()

5.2 <U> Optional<U> flatMap(Function<? super T, ? extends Optional<? extends U>> mapper)
    与 map 类似，但 mapper 返回的是 Optional，flatMap 会"拍平"结果（不产生嵌套 Optional）。
      // 假设方法签名：Optional<String> findUser(int id)
      Optional<String> name = Optional.of(1).flatMap(id -> findUser(id));

5.3 Optional<T> filter(Predicate<? super T> predicate)
    如果有值且满足条件则返回该 Optional，否则返回 Optional.empty()。
      Optional<String> opt = Optional.of("hello");
      Optional<String> filtered = opt.filter(s -> s.length() > 3);  // 保留 "hello"
      Optional<String> empty = opt.filter(s -> s.length() > 10);    // Optional.empty

5.4 Stream<T> stream()
    JDK9+，将 Optional 转换为 Stream（有值时为单元素 Stream，无值时为空 Stream）。
    便于与 Stream API 组合使用。
      opt.stream().forEach(System.out::println);
```

# 6. JDK9+ 新增方法

```
6.1 Optional<T> or(Supplier<? extends Optional<? extends T>> supplier)
    如果值存在返回自身，否则返回 supplier 生成的另一个 Optional。
      Optional<String> result = primary.or(() -> secondary);

6.2 void ifPresentOrElse(Consumer<? super T> action, Runnable emptyAction)
    见 3.4。

6.3 Stream<T> stream()
    见 5.4。
```

# 7. 使用示例

## 7.1 传统 vs Optional 对比

```
传统写法（null 检查）：
  public String getCity(User user) {
      if (user != null) {
          Address addr = user.getAddress();
          if (addr != null) {
              City city = addr.getCity();
              if (city != null) {
                  return city.getName();
              }
          }
      }
      return "未知";
  }

Optional 写法：
  public String getCity(User user) {
      return Optional.ofNullable(user)
          .map(User::getAddress)
          .map(Address::getCity)
          .map(City::getName)
          .orElse("未知");
  }
```

## 7.2 实际场景示例

```
场景一：从数据库查询可能不存在的用户
  public Optional<User> findById(long id) {
      User user = database.get(id);
      return Optional.ofNullable(user);
  }

  // 调用方：
  findById(1)
      .map(User::getName)
      .ifPresent(name -> System.out.println("用户名: " + name));

场景二：返回带默认值的配置
  public String getConfig(String key) {
      return Optional.ofNullable(configMap.get(key))
          .orElse("defaultConfig");
  }

场景三：条件过滤
  Optional<String> password = Optional.ofNullable(inputPassword);
  boolean valid = password.filter(p -> p.length() >= 8)
                          .filter(p -> p.matches(".*[A-Z].*"))
                          .isPresent();

场景四：扁平化链式调用
  public Optional<String> getManagerEmail(Employee emp) {
      return Optional.ofNullable(emp)
          .flatMap(Employee::getDepartment)
          .flatMap(Department::getManager)
          .map(Employee::getEmail);
  }
```

# 8. 最佳实践与反模式

```
8.1 推荐做法
  ✅ 方法返回值使用 Optional 表示可能返回 null
  ✅ 使用 Optional 链式调用替代多层 if 嵌套
  ✅ 使用 orElseGet() 替代 orElse() 当默认值计算开销大时

8.2 不推荐做法
  ❌ 将 Optional 用作类的字段（可能导致序列化问题，且违反了 Optional 的设计意图）
  ❌ 将 Optional 用作方法参数（增加不必要的复杂性，调用者仍需要处理 null Optional）
  ❌ 调用 Optional.get() 而不先检查（还不如直接用 null，失去了 Optional 的意义）
  ❌ Optional.of(null) （直接抛 NPE，毫无意义）
  ❌ 在集合或数组中存放 Optional（会大大增加内存开销）

8.3 总结
  Optional 是方法返回类型的合适选择，不是字段或参数类型的合适选择。
  主要收益在于迫使方法调用者思考"值为空时该怎么办"。
```

# 9. 基本类型的 Optional（JDK8）

```
为减少装箱开销，JDK8 还提供了三种基本类型 Optional：
  OptionalInt    // int 专用
  OptionalLong   // long 专用
  OptionalDouble // double 专用

用法与 Optional 类似，但获取值用 getAsInt()/getAsLong()/getAsDouble()。
例如：
  OptionalInt opt = OptionalInt.of(42);
  int value = opt.orElse(0);
  opt.ifPresent(v -> System.out.println(v));
```
