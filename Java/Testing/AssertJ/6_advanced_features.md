
此篇笔记收录针对现代 Java 开发结构设计的进阶特性验证方法，极大地降低繁琐的操作及反射遍历的冗长代码。

---

# 1. Java 8 特性支持 (Optional, CompletableFuture, Stream)

从 Java 8 起，一系列的 Monad 包和异步数据流 API 兴起，如果使用传统方式检查异常繁琐，AssertJ 内置了对此的优雅处理。

## 1.1 完美接管 Optional 断言

我们再也不必写类似 `assertTrue(opt.isPresent())` 外加 `assertEquals(val, opt.get())` 这样的结构：

```java
Optional<String> optional = Optional.of("Game of Thrones");

assertThat(optional)
    .isPresent()           // 是否有值
    .contains("Game of Thrones") // 拆包并核对值是否相等
    .hasValueSatisfying(value -> {
        // 利用 Lambda 对拆包后的真实对象做更丰富的下钻验证
        assertThat(value).startsWith("Game"); 
    });

Optional<String> emptyOpt = Optional.empty();
assertThat(emptyOpt).isEmpty(); // 判断是否为空
```

## 1.2 针对 CompletableFuture 的解耦与预判断言

异步多线程任务（如微服务网关并流组合后的校验）往往需要验证。AssertJ 给出了状态捕获和非阻塞式的值测试体系：

```java
CompletableFuture<String> future = CompletableFuture.completedFuture("Done");

// 注意，这个断言不会卡住/阻塞当前线程。而是提取现有的 CompletableFuture 状态去校验
assertThat(future)
    .isDone()                           // 断言其已经执行完毕
    .isNotCancelled()                   // 没有被中断取消
    .isNotCompletedExceptionally()      // 没有走异常抛出路线完成
    .succeedsWithin(Duration.ofSeconds(1)) // 主动阻塞最多等待1秒并校验是否能正常返回值
    .isEqualTo("Done");
```

---

# 2. I/O 与文件系统 (File, Path, InputStream) 断言

对于涉及文件持久化读写、图片上传、下载下载报表解析的测试代码来说，File & Path 极度依赖手工逐字节对比。AssertJ 直接给出了高维语义封装。

```java
import java.io.File;
import java.nio.file.Path;
import java.nio.file.Paths;

// 文件系统与属性断言
File file = new File("src/test/resources/sample.txt");

assertThat(file)
    .exists()
    .isFile() // 或 isDirectory()
    .canRead()
    .hasExtension("txt")
    .hasSize(1024); // 验证文件字节大小

// 真正杀手锏的功能：内容级别的比对！非常适合对冲 CSV、配置文件的改动
File expectedFile = new File("src/test/resources/expected.txt");

assertThat(file)
    .hasContent("Exact content if it is small.") // 针对非常简单的配置本
    .hasSameTextualContentAs(expectedFile)       // 比对两个文件内容是否行行一致！开发神器！
    .content()
    .contains("某个隐藏信息配置");
```

---

# 3. `usingRecursiveComparison()` 深度递归对比（开发必备）

这往往是 AssertJ 框架里面除了 `extracting` 以外第二伟大的发明！

在微服务分层解耦体系里，往往需要做 `Entity`, `DTO`, `VO` 的无脑映射和深拷贝（类似 MapStruct 做的事）。如何测试映射无误？
传统的 `isEqualTo()` 依赖你强制手写所有类的 `hashCode()` 和 `equals()`，但 VO 层有时甚至不想写这些胶水方法。

使用**深度递归比较**可以帮你脱离底层束缚：它利用反射读取目标每个一嵌套字段、集合的叶子节点进行类似 Json Stringify 后的深层次值比对。

```java
UserDTO dto = new UserDTO("Alice", new Address("NYC", "10001"));
UserEntity entity = new UserEntity("Alice", new Address("NYC", "10001"));
// 这两个在内存是完全不同类的不同结构对象

// 传统 isEqualTo() 必然得到 false, 因为重写的类或者地址不是同一个。
// 但我们可以这么玩：
assertThat(dto).usingRecursiveComparison()
               .isEqualTo(entity); // 通过验证！！因为它们内在的字段名称及挂载的值全部一模一样！
```

## 3.1 递归对比的精细化控制

往往业务有些生成属性你是不可测的，比如主键 Id 的自增、UUID 的随机或者 CreatedTime。可以使用 API 针对性的剔除比较范围：

```java
assertThat(dto)
    .usingRecursiveComparison()
    .ignoringFields("id", "createdAt")            // 这里是属性名！忽略这两个字段比对
    .ignoringFieldsMatchingRegexes(".*Date.*")   // 使用正则忽略比对
    .ignoringExpectedNullFields()                // 任何实体对象为 NULL 的部分，都不要拿去匹配 DTO 的值
    .isEqualTo(entity);
```