`java.io.File` 是 Java 里操作文件系统的入口类，但它封装的只是**一条路径**，而不是磁盘上的文件本身——这是所有"File 相关的困惑"的总根源。本篇先讲透 `File` 的路径语义（绝对路径、相对路径、规范路径到底差在哪）、判断获取方法、目录遍历与递归删除，再讲 JDK7 引入的 NIO.2（`Path` + `Files`）为什么在各方面都更值得用。真正的数据读写（输入输出流）留到第 16 篇。

# 1. File：路径的抽象，不是文件本身

理解 `File` 的第一句话是：**`File` 对象封装的是一条路径，它不要求这条路径对应的文件真实存在。** 它就像通讯录里的一个条目——写下一个地址并不意味着那里真的住着人。

```java
import java.io.File;
import java.io.FileInputStream;

public class FileNature {
    public static void main(String[] args) throws Exception {
        // 下面这个路径在磁盘上根本不存在，但创建 File 对象一行都不会报错
        File f = new File("D:/不存在的目录/不存在的文件.txt");
        System.out.println("exists() = " + f.exists());      // false —— 只有主动询问才知道
        System.out.println("getName()= " + f.getName());     // 纯字符串截取，照样能算出来
        System.out.println("length() = " + f.length());      // 文件不存在时返回 0

        // 对比：FileInputStream 会立刻去操作系统打开文件，不存在就抛异常
        try {
            new FileInputStream("D:/不存在的目录/不存在的文件.txt");
        } catch (java.io.FileNotFoundException e) {
            System.out.println("FileInputStream 直接抛异常：" + e.getMessage());
        }
    }
}
/* 输出：
exists() = false
getName()= 不存在的文件.txt
length() = 0
FileInputStream 直接抛异常：D:\不存在的目录\不存在的文件.txt (系统找不到指定的路径。)
*/
```

`File` 与"流"最本质的区别就在这里：

| 类型 | 代表什么 | 构造时是否访问磁盘 | 路径不存在时 |
|------|---------|-------------------|-------------|
| `File` | 一条**路径**（字符串的封装） | **否**，纯内存对象 | **什么也不发生** |
| `InputStream` / `OutputStream` | 一个**已打开的文件句柄** | **是** | 抛 `FileNotFoundException` |

> **注意**：正因为 `File` 不校验路径合法性，`new File("??***")`、`new File("")` 都不会报错。大量"文件明明存在却读不到"的问题，根源都是把 `File` 当成了文件本身，而没有用 `exists()`、`canRead()` 去主动验证。记住：**`File` 不会主动告诉你任何事，你必须逐个问它。**

## 1.1 三种构造方法

三个常用构造方法的区别只在于"路径怎么拼出来"，拼完之后行为完全一致。

| 构造方法 | 说明 | 适用场景 |
|----------|------|----------|
| `File(String pathname)` | 直接给完整路径字符串 | 路径已经是一个完整字符串 |
| `File(String parent, String child)` | 父路径字符串 + 子路径字符串 | 父目录与文件名分开存放 |
| `File(File parent, String child)` | 父 `File` 对象 + 子路径字符串 | **推荐**，父目录本身还需要用 `File` 操作时 |

```java
import java.io.File;

public class FileCtor {
    public static void main(String[] args) {
        // Windows 下反斜杠是转义字符，必须写成两个 \\
        File f1 = new File("D:\\data\\test.txt");
        File f2 = new File("D:\\data", "test.txt");
        File f3 = new File(new File("D:\\data"), "test.txt");

        System.out.println(f1.getPath());
        System.out.println(f2.getPath());
        System.out.println(f3.getPath());
    }
}
/* 输出（三个完全一致）：
D:\data\test.txt
D:\data\test.txt
D:\data\test.txt
*/
```

两个容易被忽略的细节：

1. **子路径可以是多级**，`new File("D:/data", "2026/01/test.txt")` 完全合法，但 `File` **不会替你把中间目录建出来**——这只是拼字符串；
2. 父路径传空串 `""` 时等价于只有子路径；父路径传 `null` 时，等价于 `File(String child)` 的语义。

## 1.2 绝对路径、相对路径与规范路径

这是初学者最容易困惑、面试也最常问的一块。三种"路径"必须分清：

| 概念 | 含义 | 谁决定 |
|------|------|--------|
| **绝对路径** | 从根（盘符或 `/`）开始的完整路径，在任何位置执行都指向同一处 | 调用方给定 |
| **相对路径** | 相对**当前工作目录**的路径，换个目录执行就指向别处 | 调用方给定 |
| **规范路径** | 去掉 `.`、`..`、解析符号链接后的**唯一**绝对路径 | 由 `getCanonicalPath()` 计算 |

三个 API 的区别是整个 `File` 类里最值得记住的一组：

| 方法 | 返回内容 | 访问磁盘 | 抛异常 |
|------|----------|---------|--------|
| `getPath()` | **原样返回**构造时传入的字符串 | 否 | 否 |
| `getAbsolutePath()` | 相对路径则拼上工作目录；**但不解析 `.` 和 `..`** | 否 | 否 |
| `getCanonicalPath()` | 解析 `.`、`..`、符号链接、盘符大小写后的**唯一**路径 | **是** | **是**（`IOException`） |

```java
import java.io.File;
import java.io.IOException;

public class PathDemo {
    public static void main(String[] args) throws IOException {
        // user.dir 就是"当前工作目录"，是所有相对路径的基准
        System.out.println("当前工作目录 = " + System.getProperty("user.dir"));

        File f1 = new File("a.txt");
        File f2 = new File("./a.txt");
        File f3 = new File("sub/../a.txt");

        System.out.println("f1.getPath()          -> " + f1.getPath());
        System.out.println("f1.getAbsolutePath()  -> " + f1.getAbsolutePath());
        System.out.println("f2.getAbsolutePath()  -> " + f2.getAbsolutePath());
        System.out.println("f3.getAbsolutePath()  -> " + f3.getAbsolutePath());
        System.out.println("f3.getCanonicalPath() -> " + f3.getCanonicalPath());
    }
}
/* 输出：
当前工作目录 = D:\github\Notes
f1.getPath()          -> a.txt
f1.getAbsolutePath()  -> D:\github\Notes\a.txt
f2.getAbsolutePath()  -> D:\github\Notes\.\a.txt
f3.getAbsolutePath()  -> D:\github\Notes\sub\..\a.txt
f3.getCanonicalPath() -> D:\github\Notes\a.txt
*/
```

> **陷阱**：`getAbsolutePath()` **不是"规范化"**。上面 `f2`、`f3` 的结果里 `.\` 和 `sub\..` 原封不动地留着。它们是**合法的绝对路径**，也能正常打开文件，但一旦拿来做字符串比较（判断"两个路径是不是同一个文件"）就会出错。要比较请用 `getCanonicalPath()`，或者更可靠地用 `Files.isSameFile(a, b)`——它会真正去磁盘查文件标识。

## 1.3 相对路径相对的是"当前工作目录"

这是初学者最常踩的坑，值得单独一节讲透。

**相对路径的基准是当前工作目录（Current Working Directory），也就是 `System.getProperty("user.dir")`。它跟你的 `.java` 源文件放在哪个目录、跟编译出的 `.class` 放在哪个目录，都没有任何关系。**

```java
// 假设源码在 D:\proj\src\demo\A.java，编译产物在 D:\proj\out\demo\A.class
// 在 D:\proj 目录下执行命令：java -cp out demo.A
File f = new File("config.properties");
System.out.println(f.getAbsolutePath());
// 输出：D:\proj\config.properties   （既不是 src 目录，也不是 out 目录）
```

不同运行方式下的工作目录完全不同：

| 运行方式 | 默认工作目录 |
|----------|-------------|
| 命令行 `java -cp out demo.A` | 执行命令时所在的目录 |
| IDEA 运行 `main` 方法 | **项目根目录**（可在运行配置里改，宏是 `$PROJECT_DIR$`） |
| IDEA 运行单元测试 | **模块目录**，与跑 `main` 不一致——这正是"测试过了上线就挂"的常见原因 |
| 打包成 jar 后双击运行 | 由启动方式决定，**随机性很大** |
| Spring Boot 启动脚本 | 脚本里 `cd` 到哪就是哪 |
| Windows 计划任务 / systemd 服务 | 通常不是应用目录，**极易踩坑** |

> **行业实践**：生产代码里**永远不要依赖 `user.dir`**。需要读配置文件时，从 classpath 读（`getClass().getClassLoader().getResourceAsStream("config.properties")`）；需要读写业务文件时，用 `Path.of(System.getProperty("user.home"), "app", "data")` 这类**绝对路径**，或者做成配置项由部署环境注入。判断一个项目的成熟度，看它有没有到处写 `new File("xxx")` 就够了。

## 1.4 路径分隔符与跨平台

Java 里"分隔符"有两种，含义完全不同，千万别混：

| 常量 | 值（Windows） | 值（Linux/macOS） | 作用 |
|------|--------------|------------------|------|
| `File.separator` | `\` | `/` | **路径内部**各级之间的分隔 |
| `File.pathSeparator` | `;` | `:` | **多个路径之间**的分隔，如 `PATH`、`-cp` |

```java
import java.io.File;
import java.nio.file.Path;

public class SeparatorDemo {
    public static void main(String[] args) {
        // 三种拼路径方式，第三种最好
        String p1 = "data" + File.separator + "2026" + File.separator + "note.txt";
        File   p2 = new File("data", "2026" + File.separator + "note.txt");
        Path   p3 = Path.of("data", "2026", "note.txt");   // resolve 自动加分隔符

        System.out.println(p1);
        System.out.println(p2.getPath());
        System.out.println(p3);
        System.out.println("路径分隔符 = '" + File.separatorChar
                           + "'  多路径分隔符 = '" + File.pathSeparatorChar + "'");
    }
}
/* 输出（Windows）：
data\2026\note.txt
data\2026\note.txt
data\2026\note.txt
路径分隔符 = '\'  多路径分隔符 = ';'
*/
```

记三条结论就够了：

1. **Windows 同时接受 `/` 和 `\`**，Java 内部会把 `/` 统一规范成 `\`，所以 `new File("D:/data/test.txt")` 在 Windows 上完全正常；
2. **Linux/macOS 只认 `/`**，那里写 `\` 会被当成普通文件名字符；
3. 因此**写死 `/` 的代码在三大平台都能跑，写死 `\` 的代码一到 Linux 就崩**。这就是"推荐用 `/` 或 `File.separator`"的原因。

> **注意**：`File.separator` 是 **`String`**，`File.separatorChar` 才是 `char`。循环里做字符拼接时用后者可以省掉一次字符串拼接。

# 2. File 的判断与获取方法

这一组方法全部是"查询"语义：只读磁盘状态，不改动任何东西。它们的共同点是**即使路径不存在也不抛异常**，只返回 `false` 或 `0`。

## 2.1 判断类方法

| 方法 | 说明 | 路径不存在时 |
|------|------|-------------|
| `boolean exists()` | 文件或目录是否存在 | `false` |
| `boolean isFile()` | 是否是**普通文件** | `false` |
| `boolean isDirectory()` | 是否是**目录** | `false` |
| `boolean isHidden()` | 是否隐藏（Windows 看隐藏属性，Unix 看是否以 `.` 开头） | `false` |
| `boolean isAbsolute()` | 构造时给的是不是绝对路径（**纯字符串判断，不访问磁盘**） | 视字符串而定 |
| `boolean canRead()` | 是否可读 | `false` |
| `boolean canWrite()` | 是否可写 | `false` |
| `boolean canExecute()` | 是否可执行 | `false` |

```java
import java.io.File;

public class JudgeDemo {
    public static void main(String[] args) {
        File f = new File("D:/data/test.txt");
        // 先 exists() 再判断类型：否则"不存在"和"不是目录"都是 false，区分不出来
        if (f.exists()) {
            System.out.println(f.isDirectory() ? "是个目录" : "是个文件");
        } else {
            System.out.println("路径不存在");       // 输出这句
        }
    }
}
```

> **陷阱**：`isFile()` 和 `isDirectory()` **都对不存在的路径返回 `false`**。所以只写 `if (isDirectory()) ... else if (isFile()) ...` 并不能覆盖所有情况——"路径不存在"会掉进两个 `false` 之间的缝隙里，被静默忽略。先 `exists()` 再判断类型是唯一稳妥的写法。

## 2.2 获取类方法

| 方法 | 说明 |
|------|------|
| `String getName()` | 最后一级的名字（含后缀），**纯字符串截取** |
| `String getPath()` | 构造时传入的原样字符串 |
| `String getAbsolutePath()` | 拼上工作目录的绝对路径，不解析 `.` / `..` |
| `String getCanonicalPath()` | 规范路径，会抛 `IOException` |
| `String getParent()` | 父目录字符串，**没有父级时返回 `null`** |
| `File getParentFile()` | 父目录的 `File` 对象，同样可能为 `null` |
| `long length()` | 文件字节数；**目录返回 0 或实现相关值，不要用来判断目录大小** |
| `long lastModified()` | 最后修改时间的毫秒值 |
| `long getTotalSpace()` | 所在分区总容量（字节） |
| `long getFreeSpace()` | 所在分区**未分配**空间 |
| `long getUsableSpace()` | 所在分区**当前用户可用**空间 |

```java
import java.io.File;
import java.time.Instant;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;

public class GetDemo {
    public static void main(String[] args) {
        File f = new File("D:/data/test.txt");
        System.out.println("getName   = " + f.getName());         // test.txt
        System.out.println("getParent = " + f.getParent());       // D:\data
        System.out.println("length    = " + f.length());          // 不存在时 0

        // 目录的 length() 是 0，别拿它算目录大小
        System.out.println("目录 length = " + new File("D:/data").length());   // 0

        // 时间戳转可读时间
        Instant instant = Instant.ofEpochMilli(f.lastModified());          // 时间轴上的一个点
        System.out.println("修改时间  = " + instant.atZone(ZoneId.systemDefault())
                .format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));

        File root = new File("D:/");
        System.out.println("总容量 " + root.getTotalSpace() / 1024 / 1024 / 1024 + " GB");
        System.out.println("可用   " + root.getUsableSpace() / 1024 / 1024 / 1024 + " GB");
    }
}
```

> **注意**：`getFreeSpace()` 与 `getUsableSpace()` 的区别在于**权限配额**——前者是文件系统层面的空闲块数，后者是当前用户实际能用的空间。判断"磁盘够不够写"应当用 `getUsableSpace()`。

# 3. File 的创建、删除与重命名

## 3.1 创建文件与目录

| 方法 | 创建对象 | 父目录必须存在？ | 已存在时 | 返回值 |
|------|---------|----------------|---------|--------|
| `createNewFile()` | 文件 | **是** | 不报错，返回 `false` | `boolean`，抛 `IOException` |
| `mkdir()` | **单级**目录 | **是** | 返回 `false` | `boolean` |
| `mkdirs()` | **多级**目录 | 否 | 返回 `false` | `boolean` |

```java
import java.io.File;
import java.io.IOException;

public class CreateDemo {
    public static void main(String[] args) throws IOException {
        // mkdirs 会把中间缺失的 2026、01 一并创建
        File dir = new File("D:/data/2026/01");
        System.out.println("建多级目录 = " + dir.mkdirs());   // 第一次 true；已存在则 false

        // createNewFile 要求父目录已存在，所以顺序永远是 mkdirs -> createNewFile
        File file = new File(dir, "note.txt");
        System.out.println("建文件     = " + file.createNewFile());  // true；已存在则 false

        // 对照：mkdir 遇到多级路径会失败
        System.out.println("mkdir 多级 = " + new File("D:/x/y/z").mkdir());  // false，且什么都不建
    }
}
```

三条必须记住的坑：

1. **`createNewFile()` 的父目录不存在会抛 `IOException`（`NoSuchFileException`）**，它不会替你建父目录。标准顺序永远是 `mkdirs()` → `createNewFile()`。
2. **`mkdir()` 遇到多级路径直接失败**。`new File("D:/a/b/c").mkdir()` 在 `D:/a` 不存在时返回 `false` 并且什么都不建。**一律用 `mkdirs()`**，它建单级目录也完全正常。
3. 两个方法的返回值都是 `boolean` 而非 `void`，且**"已存在"和"失败"都返回 `false`**，靠返回值区分不了。想区分就先 `exists()`，或者改用 `Files.createFile()`——已存在时它抛 `FileAlreadyExistsException`，语义清楚得多。

## 3.2 删除与重命名

| 方法 | 说明 |
|------|------|
| `boolean delete()` | 删除文件或**空目录**；不经过回收站；失败只返回 `false`，**不告诉你为什么** |
| `boolean renameTo(File dest)` | 重命名或移动（相当于 `mv`）；跨分区移动可能失败 |
| `boolean setReadOnly()` | 设为只读 |
| `boolean setLastModified(long time)` | 设置最后修改时间 |
| `boolean setWritable(boolean)` / `setReadable(boolean)` / `setExecutable(boolean)` | 设置权限 |

```java
import java.io.File;

public class DeleteDemo {
    public static void main(String[] args) {
        File f = new File("D:/data/test.txt");
        System.out.println("删除文件   = " + f.delete());     // 文件不存在也是 false

        File d = new File("D:/data/empty");
        System.out.println("删除空目录 = " + d.delete());     // 目录里有东西就 false

        // renameTo 实质是"移动 + 改名"，同目录下换个名字就是重命名
        File src = new File("D:/data/a.txt");
        System.out.println("重命名     = " + src.renameTo(new File("D:/data/b.txt")));

        // 只精确到秒的文件系统（部分 FAT、某些网盘同步目录）可能设置不生效
        src.setLastModified(System.currentTimeMillis());
    }
}
```

> **陷阱**：`delete()` 的返回值信息量极低——**文件不存在、目录非空、文件被其他进程占用（Windows 上极其常见）、权限不足，全都是 `false`**。排查时不要靠猜，改用 `Files.delete(path)`：它会对每种情况抛不同的异常（`NoSuchFileException`、`DirectoryNotEmptyException`、`AccessDeniedException`），一眼就能定位。如果不想写 `try/catch`，用 `Files.deleteIfExists(path)`，返回 `boolean` 表示"实际删掉了没有"。

> **注意**：`delete()` **不进回收站**，删了就真没了。任何涉及用户数据的删除操作，都应该先做备份，或者先"软删除"（改名到 `.trash` 目录，定时清理）。

# 4. 目录遍历：list 与 listFiles

`File` 提供两套遍历 API：返回**字符串名**的 `list*` 和返回 **`File` 对象**的 `listFiles*`。**实际开发一律用 `listFiles`**——拿到字符串之后往往还要 `new File(dir, name)` 拼回去，纯属浪费。

| 方法 | 返回 | 说明 |
|------|------|------|
| `File[] listRoots()` | `File[]` | 列出文件系统根（Windows 是各个盘符，Linux 是 `/`） |
| `String[] list()` | `String[]` | 当前目录下所有内容的**名称** |
| `String[] list(FilenameFilter)` | `String[]` | 带文件名过滤器的名称数组 |
| `File[] listFiles()` | `File[]` | 当前目录下所有内容的 **`File` 对象**（**推荐**） |
| `File[] listFiles(FileFilter)` | `File[]` | 用 `File` 对象判断的过滤器 |
| `File[] listFiles(FilenameFilter)` | `File[]` | 用文件名判断的过滤器 |

```java
import java.io.File;

public class RootsDemo {
    public static void main(String[] args) {
        for (File root : File.listRoots()) {
            System.out.println("根 = " + root);
        }
    }
}
/* 输出（Windows 装了 C、D 两个盘）：
根 = C:\
根 = D:\
*/
```

## 4.1 两种过滤器的区别

这是本节最容易混淆的点。`FileFilter` 和 `FilenameFilter` 都是**函数式接口**，但参数不同，能力也不同：

| 接口 | 抽象方法 | 能拿到什么 | 适用 |
|------|---------|-----------|------|
| `FileFilter` | `boolean accept(File pathname)` | 完整的 `File` 对象 | **推荐**，可判断类型、大小、时间 |
| `FilenameFilter` | `boolean accept(File dir, String name)` | 父目录 + 文件**名字符串** | 只按名字筛选时 |

```java
import java.io.File;

public class FilterDemo {
    public static void main(String[] args) {
        File dir = new File("D:/data");

        // FileFilter：能拿到 File，可以判断"是文件且以 .txt 结尾"
        File[] txts = dir.listFiles(f -> f.isFile() && f.getName().endsWith(".txt"));

        // FilenameFilter：只有名字字符串，判断不了类型（目录也能叫 xxx.java）
        File[] javas = dir.listFiles((d, name) -> name.endsWith(".java"));

        // 注意两者都可能返回 null，必须判空（见 4.2）
        System.out.println("txt 文件数  = " + (txts == null ? -1 : txts.length));
        System.out.println("java 名字数 = " + (javas == null ? -1 : javas.length));
    }
}
```

> **最佳实践**：需要"类型 + 名字"双重条件时用 `FileFilter`；只按后缀筛选时两者都行，但 **`FileFilter` 更不容易写错**，因为 `FilenameFilter` 拿不到 `File`，无法排除同名目录，也无法判断文件大小、修改时间。

## 4.2 listFiles 返回 null：最常见的一处 NPE

**这是 `File` 类最容易导致线上 `NullPointerException` 的地方。** `listFiles()` 只有在**该路径确实是存在且可读的目录**时才返回非 `null`：

| 情形 | `listFiles()` 返回 |
|------|-------------------|
| 路径不存在 | `null` |
| 路径存在但是个**普通文件** | `null` |
| 目录存在但**当前用户没有读权限** | `null` |
| 其他 IO 错误 | `null` |

```java
import java.io.File;

public class ListNullDemo {
    public static void main(String[] args) {
        File notExist = new File("no_such_dir");
        File aFile    = new File("no_such_dir/x.txt");
        File realFile = new File("ListNullDemo.java");   // 磁盘上真实存在的文件

        System.out.println("不存在的目录 listFiles()   -> " + notExist.listFiles());
        System.out.println("对普通文件调用 listFiles() -> " + aFile.listFiles());
        System.out.println("对真实文件调用 listFiles() -> " + realFile.listFiles());
    }
}
/* 输出：
不存在的目录 listFiles()   -> null
对普通文件调用 listFiles() -> null
对真实文件调用 listFiles() -> null
*/
```

所以**任何 `listFiles()` 的返回值都必须判 `null`**：

```java
File[] files = dir.listFiles();
if (files == null) {                  // 目录不存在、不是目录、或没有读权限
    System.out.println("无法列出：" + dir);
    return;
}
for (File f : files) {
    System.out.println(f.getName());
}
```

> **陷阱**：`Files.list()`（NIO.2）返回 `Stream<Path>`，遇到不存在或无权限的路径会**直接抛 `IOException`**（`NotDirectoryException`、`AccessDeniedException`），而不是给个 `null`。这正是"异常优于返回特殊值"的正面例子——问题在调用点立刻暴露，而不是拖到后面某行才 NPE。

# 5. 递归遍历与递归删除

## 5.1 递归遍历目录树

思路只有一句话：**拿到目录下所有 `File`，是文件就处理，是目录就自己调用自己。**

```java
import java.io.File;

public class WalkDemo {
    // 打印一棵目录树，depth 用来控制缩进
    public static void traverse(File dir, int depth) {
        if (!dir.isDirectory()) return;          // 不是目录就没有继续的必要

        File[] files = dir.listFiles();
        if (files == null) return;               // 必须判 null，见 4.2

        String indent = "  ".repeat(depth);      // 每深一层多两个空格
        for (File f : files) {
            if (f.isFile()) {
                System.out.println(indent + "文件 " + f.getName()
                                   + " (" + f.length() + " 字节)");
            } else if (f.isDirectory()) {
                System.out.println(indent + "目录 " + f.getName() + "/");
                traverse(f, depth + 1);          // 递归进入子目录
            }
        }
    }

    public static void main(String[] args) {
        traverse(new File("D:/data"), 0);
    }
}
/* 输出（D:/data 下有 2026/01/note.txt 和 readme.md）：
目录 2026/
  目录 01/
    文件 note.txt (1024 字节)
文件 readme.md (256 字节)
*/
```

> **注意**：递归时的**终止条件必须是"不是目录就返回"**，而不是"是文件就返回"。因为符号链接、设备文件等既不是普通文件也不是目录，只判断"是文件"会让它们进入递归分支，导致意外行为。

## 5.2 递归删除非空目录

`delete()` 只能删空目录，所以删除整棵树只有一条路：**后序遍历**——先把里面清空，再删自己。

```java
import java.io.File;

public class DeleteDirDemo {
    /** 删除目录及其全部内容，全部成功返回 true */
    public static boolean deleteDir(File dir) {
        if (dir == null || !dir.exists()) return true;   // 本来就没有，视为成功
        if (dir.isFile()) return dir.delete();           // 传进来的是文件，直接删

        File[] files = dir.listFiles();
        if (files != null) {
            for (File f : files) {
                // 文件和目录都交给 deleteDir，逻辑更统一
                if (!deleteDir(f)) return false;         // 有一个删不掉就整体失败
            }
        }
        return dir.delete();                             // 此时目录已空，可以删了
    }

    public static void main(String[] args) throws Exception {
        File target = new File("D:/data/tmp");
        // 危险操作：真实项目务必先做路径白名单校验，防止误删
        String cano = target.getCanonicalPath();
        if (!cano.startsWith("D:\\data\\")) {
            throw new IllegalStateException("拒绝删除白名单之外的路径：" + cano);
        }
        System.out.println("删除结果 = " + deleteDir(target));
    }
}
```

> **陷阱**：递归删除**不进回收站**，一旦参数写错（把 `D:/data/tmp` 写成 `D:/`），后果不可逆。"递归删除参数写错"在工业界是真实发生过多起灾难性事故的。生产代码里请务必在方法开头做两件事：**用 `getCanonicalPath()` 校验白名单前缀**，并**拒绝根目录**。这不是小题大做。

## 5.3 递归的代价：栈溢出

递归写得优雅，但**每层递归都占用一个栈帧**。JVM 的线程栈深度有限（默认几百 KB 到 1 MB，可用 `-Xss` 调整），目录层级过深就会抛 `StackOverflowError`：

```text
栈帧 1   traverse(D:\data)
栈帧 2     traverse(D:\data\level1)
栈帧 3       traverse(D:\data\level1\level2)
 ...           ...
栈帧 N         traverse(...\levelN)  ── 超过栈深度上限，抛 StackOverflowError
```

四种遍历方案的对比：

| 方案 | 底层实现 | 深目录 | 控制粒度 | 需关闭资源 |
|------|---------|--------|---------|-----------|
| 手写递归 | 方法调用栈 | **有栈溢出风险** | 只能全遍历或提前 return | 无 |
| 显式栈 + 循环 | `ArrayDeque`（堆内存） | 安全 | 略好 | 无 |
| `Files.walk` | 迭代式文件树遍历器 | **安全** | 好（配合 Stream 短路） | **需 `try-with-resources`** |
| `Files.walkFileTree` | 访问者模式，迭代式 | **安全** | **最好**（可跳过子树、可处理失败） | 无需 |

> **最佳实践**：**现代 Java 里不要手写递归遍历**。用 `Files.walk(dir)` 拿到 `Stream<Path>`，它内部是迭代实现，不受栈深限制，还能直接接上 Stream API 做过滤、映射、统计。真正需要"遇到某个目录就跳过整个子树""单个文件失败了继续往下走"这类细粒度控制时，用 `Files.walkFileTree`（见 8.2）。

# 6. NIO.2 的 Path 接口

## 6.1 为什么 JDK7 要引入 Path

`File` 有三个绕不开的短板，它们在 JDK1.4 引入 NIO 时就被公认为设计缺陷。但 `File` 是 `java.io` 的公开 API，改不得，只能推倒重来：

| 短板 | 具体表现 |
|------|---------|
| **能力太弱** | 想读整个文件？自己写循环；想复制？自己写循环；想要文件属主、符号链接、ACL？**完全没有** |
| **错误信息太少** | `delete()` 返回 `false`、`mkdirs()` 返回 `false`，**不告诉你为什么失败**；`renameTo` 跨平台行为还不一致 |
| **无法表达非默认文件系统** | 无法操作 Zip 内部条目、内存文件系统、远端存储 |

JDK7 的 NIO.2 给出了答案：`java.nio.file.Path`（路径抽象，替代 `File`）+ `java.nio.file.Files`（工具类，替代 `File` 上那一堆方法）。

```text
        JDK1.0 的 java.io.File                JDK7 的 java.nio.file
     ┌────────────────────────┐           ┌────────────────────────────┐
     │ File                   │           │ Path   （路径抽象）          │
     │  ├ 路径字符串           │           │  ├ 由 FileSystem 抽象出实现  │
     │  ├ 查询 / 创建 / 删除   │  ──▶      │  │   可指向 Zip、内存文件系统 │
     │  ├ 目录遍历             │           │  └ 与 File 可用 toPath/toFile│
     │  └ 失败只返回 false     │           │      互转                    │
     └────────────────────────┘           │ Files  （静态工具类）        │
                                          │  ├ 读写 / 复制 / 移动        │
                                          │  ├ 遍历 walk / walkFileTree  │
                                          │  └ 失败抛具体异常            │
                                          └────────────────────────────┘
```

## 6.2 获取 Path 对象

```java
import java.io.File;
import java.nio.file.Path;
import java.nio.file.Paths;

public class PathCreate {
    public static void main(String[] args) {
        Path p1 = Paths.get("D:\\data", "test.txt");   // JDK7 的写法
        Path p2 = Path.of("D:/data", "test.txt");      // JDK11+ 推荐，语义完全相同
        Path p3 = Path.of("D:/data/test.txt");         // 一次性给完整路径

        System.out.println(p1.equals(p2));             // true：Path 重写了 equals，按路径元素比较

        File f  = p1.toFile();                         // Path → File，兼容老 API 时用
        Path p4 = f.toPath();                          // File → Path
        System.out.println(p4.equals(p1));             // true
    }
}
/* 输出：
true
true
*/
```

`Path` 与 `File` 的一个关键差异是 **`Path` 重写了 `equals()`**，按路径元素逐个比较；`File` 没有重写 `equals()`，两个内容完全相同的 `File` 对象也永远不相等。不过跨平台比较（Windows 路径大小写不敏感、Linux 敏感）仍然要靠 `Files.isSameFile(a, b)`——它会真正去磁盘查文件标识。

## 6.3 Path 的常用方法

| 方法 | 说明 | 例子（`Path.of("/a/b/c.txt")`） |
|------|------|-------------------------------|
| `toString()` | 返回路径字符串 | `/a/b/c.txt` |
| `getFileName()` | 最后一级 | `c.txt` |
| `getParent()` | 父路径，**无父级返回 `null`** | `/a/b` |
| `getRoot()` | 根，没有则 `null` | `/` |
| `getNameCount()` | 元素个数 | `3`（`a`、`b`、`c.txt`） |
| `getName(int i)` | 第 i 个元素（从 0 开始） | `getName(0)` → `a` |
| `resolve(String/Path)` | **拼接**路径，自动处理分隔符 | `p.resolve("d.txt")` → `/a/b/c.txt/d.txt` |
| `resolveSibling(String)` | 拼接**兄弟**路径，替换最后一级 | → `/a/b/d.txt` |
| `relativize(Path)` | 求相对路径 | `p.relativize(Path.of("/a/x"))` → `../../x` |
| `normalize()` | 去掉 `.` 和 `..`（**纯字符串计算，不访问磁盘**） | `Path.of("/a/./b/../c").normalize()` → `/a/c` |
| `toAbsolutePath()` | 转绝对路径 | 相对路径则拼上 `user.dir` |
| `toRealPath()` | 解析符号链接后的**真实**路径（**访问磁盘，抛 `IOException`**） | 相当于 `File.getCanonicalPath()` |
| `startsWith(Path)` / `endsWith(Path)` | 前缀 / 后缀判断（**按路径元素，不是字符串**） | `p.endsWith("c.txt")` → `true` |
| `toFile()` | 转成 `File` | |
| `iterator()` | 逐级迭代，可配合 for-each 遍历每一级 | |

```java
import java.nio.file.Path;

public class PathMethodDemo {
    public static void main(String[] args) throws Exception {
        Path p = Path.of("D:/data/./2026/../test.txt");
        System.out.println("normalize     = " + p.normalize());       // 纯计算，文件不必存在
        System.out.println("toAbsolutePath= " + p.toAbsolutePath());  // 保留 . 和 ..
        // toRealPath 会真正解析，文件不存在时抛 NoSuchFileException
        System.out.println("toRealPath    = " + p.toRealPath());
    }
}
/* 输出（文件不存在时最后一行抛异常）：
normalize     = D:\data\test.txt
toAbsolutePath= D:\data\.\2026\..\test.txt
Exception in thread "main" java.nio.file.NoSuchFileException: D:\data\.\2026\..\test.txt
*/
```

`normalize()` 与 `toRealPath()` 的区别值得单独强调：

| 维度 | `normalize()` | `toRealPath()` |
|------|--------------|----------------|
| 访问磁盘 | 否 | **是** |
| 解析 `..` | 是（纯字符串） | 是 |
| 解析符号链接 | **否** | **是** |
| 文件不存在时 | 正常工作 | 抛 `NoSuchFileException` |
| 异常 | 无 | `IOException` |

> **陷阱**：`normalize()` 只做字符串层面的化简，路径里有**符号链接**时结果可能与真实位置不符。例如 `/a/link` 是指向 `/x/y` 的软链，那么 `/a/link/../b` 的 `normalize()` 结果是 `/a/b`，而真实位置是 `/x/b`。要拿真实位置只能用 `toRealPath()`。

# 7. Files 工具类

`Files` 是 NIO.2 的核心工具类，**全部方法都是静态的**，第一个参数基本都是 `Path`。它把 `File` 做不到的事一次性补齐了。

## 7.1 判断与查询

| 方法 | 说明 |
|------|------|
| `exists(Path, LinkOption...)` | 是否存在（默认**跟随符号链接**，传 `NOFOLLOW_LINKS` 可改） |
| `notExists(Path, LinkOption...)` | 是否**确定不存在**（权限不足时 `exists` 和 `notExists` **都返回 `false`**） |
| `isDirectory(Path, LinkOption...)` | 是否目录 |
| `isRegularFile(Path, LinkOption...)` | 是否普通文件（**比 `!isDirectory` 准确**，能排除设备文件、符号链接） |
| `isHidden(Path)` | 是否隐藏 |
| `isReadable` / `isWritable` / `isExecutable` | 权限判断 |
| `isSymbolicLink(Path)` | 是否符号链接 |
| `size(Path)` | 文件字节数。**目录上行为是实现相关的**，Windows 下返回 `0`，不要依赖 |
| `getLastModifiedTime(Path)` | 返回 `FileTime` |
| `readAttributes(Path, Class)` | 批量读取属性，见 7.5 |

> **陷阱**：`exists()` 与 `notExists()` 都可能返回 `false`——当**无法确定是否存在**时（典型是无权限访问父目录）两者都是 `false`。所以不能写 `if (!Files.notExists(p))` 来当 `exists` 用。规范写法是 `if (Files.exists(p)) { ... } else if (Files.notExists(p)) { ... } else { /* 无法确定 */ }`。

## 7.2 创建与删除

| 方法 | 说明 | 已存在时 |
|------|------|---------|
| `createFile(Path, FileAttribute...)` | 创建空文件 | **抛 `FileAlreadyExistsException`** |
| `createDirectory(Path, FileAttribute...)` | 创建单级目录 | 抛 `FileAlreadyExistsException`；父目录不存在抛 `NoSuchFileException` |
| `createDirectories(Path, FileAttribute...)` | 创建多级目录（**推荐**） | **不报错**，直接返回 |
| `delete(Path)` | 删除文件或空目录 | 不存在抛 `NoSuchFileException`；非空目录抛 `DirectoryNotEmptyException` |
| `deleteIfExists(Path)` | 存在则删除 | 返回 `boolean`，不存在不报错 |

```java
import java.io.IOException;
import java.nio.file.*;

public class FileCreateDemo {
    public static void main(String[] args) throws IOException {
        Path dir = Path.of("D:/data/2026/01");
        Files.createDirectories(dir);                 // 已存在也不报错，可反复调用

        Path f = dir.resolve("note.txt");             // resolve 自动处理分隔符
        Files.createFile(f);                          // 已存在会抛 FileAlreadyExistsException
        System.out.println("创建成功 " + f);

        // 删除：不存在会抛 NoSuchFileException，所以用之前先判存在或直接用 deleteIfExists
        System.out.println("删除结果 = " + Files.deleteIfExists(Path.of("D:/data/ghost.txt")));  // false
    }
}
```

## 7.3 读写：一行搞定

这是 `Files` 相比 `File` + 流最直观的进步：**绝大多数"读一个文件""写一个文件"的场景，一行就够。**

| 方法 | 起始版本 | 说明 |
|------|---------|------|
| `readString(Path)` | **JDK11+** | 读整个文件为 `String`（**UTF-8**） |
| `readString(Path, Charset)` | JDK11+ | 指定编码读为 `String` |
| `writeString(Path, CharSequence, OpenOption...)` | **JDK11+** | 写字符串到文件 |
| `readAllBytes(Path)` | JDK7+ | 读全部字节为 `byte[]` |
| `readAllLines(Path)` | JDK7+ | 读全部行到 `List<String>`（**UTF-8**） |
| `readAllLines(Path, Charset)` | JDK7+ | 指定编码 |
| `write(Path, byte[], OpenOption...)` | JDK7+ | 写字节 |
| `write(Path, Iterable<? extends CharSequence>, Charset, OpenOption...)` | JDK7+ | **逐行写**，每行自动加平台换行符 |
| `lines(Path, Charset)` | JDK8+ | 返回 `Stream<String>`，**懒加载，必须关闭** |
| `newBufferedReader(Path, Charset)` | JDK7+ | 得到 `BufferedReader`（默认 UTF-8） |
| `newBufferedWriter(Path, Charset, OpenOption...)` | JDK7+ | 得到 `BufferedWriter` |
| `newInputStream(Path, OpenOption...)` | JDK7+ | 得到 `InputStream` |
| `newOutputStream(Path, OpenOption...)` | JDK7+ | 得到 `OutputStream` |
| `mismatch(Path, Path)` | JDK12+ | 返回第一个不同字节的下标，完全相同返回 `-1` |

> **注意**：`Files.readAllLines(path)`、`Files.readString(path)`、`Files.newBufferedReader(path)` **默认统一用 UTF-8**，与平台默认字符集无关。这在中文 Windows 上尤其要记住——`new FileReader(path)` 用的是平台默认（中文 Windows 上是 GBK），而 `Files.newBufferedReader(path)` 用的是 UTF-8，读同一个文件可能一个正常一个乱码。

```java
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.util.List;

public class FilesDemo {
    public static void main(String[] args) throws IOException {
        Path base = Path.of("nio_demo");
        Files.createDirectories(base.resolve("a/b"));            // 多级目录一次建好

        Path f = base.resolve("a/b/note.txt");
        Files.writeString(f, "第一行\n第二行\n", StandardCharsets.UTF_8);

        System.out.println("readString   -> " + Files.readString(f).replace("\n", "\\n"));
        System.out.println("readAllLines -> " + Files.readAllLines(f).size() + " 行");
        System.out.println("size         -> " + Files.size(f) + " 字节");

        // 追加：必须同时给 CREATE，否则文件不存在时抛 NoSuchFileException
        Files.writeString(f, "第三行\n", StandardCharsets.UTF_8,
                          StandardOpenOption.APPEND, StandardOpenOption.CREATE);
        System.out.println("追加后行数   -> " + Files.readAllLines(f).size());

        // 逐行写：List 的每个元素占一行，自动补平台换行符
        Files.write(base.resolve("lines.txt"),
                    List.of("甲", "乙", "丙"), StandardCharsets.UTF_8);
        System.out.println("lines.txt 内容 -> "
                           + Files.readString(base.resolve("lines.txt")).replace("\n", "\\n"));
    }
}
/* 输出：
readString   -> 第一行\n第二行\n
readAllLines -> 2 行
size         -> 20 字节
追加后行数   -> 3
lines.txt 内容 -> 甲\r\n乙\r\n丙\r\n
*/
```

> **陷阱**：`Files.writeString(f, "第三行\n", ...)` 或 `Files.newBufferedWriter(f, UTF_8, StandardOpenOption.APPEND)` **只给 `APPEND` 而不给 `CREATE`** 时，文件不存在会抛 `NoSuchFileException`。原因是 `APPEND` 的语义是"追加到末尾"，它本身不负责创建文件。稳妥写法永远是 **`APPEND` + `CREATE` 一起给**。

> **最佳实践**：`Files.write(path, lines)` 的重载会**自动在每行末尾补 `System.lineSeparator()`**，直接传一个 `List<String>` 就能生成规范的多行文本文件，不必自己拼 `\n`。反过来，读文件优先用 `Files.readAllLines`，它会自动识别 `\n`、`\r\n`、`\r` 三种换行符。

## 7.4 复制、移动与临时文件

这三个操作在 `File` 里要么没有，要么靠 `renameTo` 勉强凑合（跨分区直接失败），`Files` 把它们做成了一等公民。

| 方法 | 说明 |
|------|------|
| `copy(Path src, Path dst, CopyOption...)` | 复制文件或目录 |
| `move(Path src, Path dst, CopyOption...)` | 移动 / 重命名 |
| `createTempFile(String prefix, String suffix)` | 在系统临时目录创建临时文件 |
| `createTempFile(Path dir, String prefix, String suffix, FileAttribute...)` | 在指定目录创建 |
| `createTempDirectory(String prefix)` | 创建临时目录 |
| `createSymbolicLink(Path link, Path target)` | 创建符号链接 |
| `readSymbolicLink(Path link)` | 读取符号链接指向的目标 |

`CopyOption` 的常用实现分两组：

| 选项 | 属于 | 含义 |
|------|------|------|
| `StandardCopyOption.REPLACE_EXISTING` | `copy` / `move` | 目标已存在则**覆盖**（不加会抛 `FileAlreadyExistsException`） |
| `StandardCopyOption.COPY_ATTRIBUTES` | `copy` | 一并复制文件属性（时间戳、权限） |
| `StandardCopyOption.ATOMIC_MOVE` | `move` | **原子移动**：要么完全成功要么完全失败，不会留下半个文件 |
| `LinkOption.NOFOLLOW_LINKS` | 两者 | 不跟随符号链接，操作链接本身 |

```java
import java.io.IOException;
import java.nio.file.*;

public class CopyMoveDemo {
    public static void main(String[] args) throws IOException {
        Path src = Path.of("D:/data/a.txt");
        Path dst = Path.of("D:/backup/a_backup.txt");

        Files.createDirectories(dst.getParent());            // 目标目录必须已存在
        Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
        System.out.println("复制后大小 = " + Files.size(dst));

        // 移动 / 重命名：同目录换个名字就是重命名
        Files.move(dst, dst.resolveSibling("a_renamed.txt"),
                   StandardCopyOption.REPLACE_EXISTING);

        // 临时文件：进程退出不会自动删除，必须自己清理
        Path tmp = Files.createTempFile("report-", ".csv");
        System.out.println("临时文件 = " + tmp);
        Files.deleteIfExists(tmp);                           // 用完立刻删，别留垃圾
    }
}
```

> **注意**：`ATOMIC_MOVE` 是"安全写文件"的标准手法——**先写临时文件，写完整了再原子移动替换目标文件**，这样任何时刻读到的都是完整文件，绝不会读到写了一半的内容。配置中心、日志轮转、缓存快照都在用这个模式。但要注意：**跨文件系统的 `ATOMIC_MOVE` 会抛 `AtomicMoveNotSupportedException`**，必须做降级处理。

> **陷阱**：`Files.copy` 复制**目录**时只会创建一个空目录，**不会递归复制里面的内容**。要复制整棵目录树，得配合 `Files.walk` 自己遍历，或者用 Apache Commons IO 的 `FileUtils.copyDirectory`。

## 7.5 文件属性

`File` 只能拿到大小、时间等寥寥几项，`Files.readAttributes` 能**一次读出整组属性**，避免多次系统调用。

| 方法 | 说明 |
|------|------|
| `readAttributes(Path, Class<A>)` | 读取一组属性，返回属性视图对象 |
| `readAttributes(Path, String, LinkOption...)` | 按名字读，如 `"size,lastModifiedTime"` |
| `getAttribute(Path, String)` | 读单个属性，如 `"size"` |
| `setAttribute(Path, String, Object)` | 设置属性，如 `"dos:hidden"`、`"posix:permissions"` |
| `getPosixFilePermissions(Path)` / `setPosixFilePermissions(...)` | 读写 POSIX 权限（**仅 Linux/macOS**） |
| `getOwner(Path)` / `setOwner(Path, UserPrincipal)` | 读写属主 |

```java
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;

public class AttrDemo {
    public static void main(String[] args) throws Exception {
        Path p = Path.of("D:/data/a.txt");
        BasicFileAttributes attr =
                Files.readAttributes(p, BasicFileAttributes.class);   // 一次系统调用读一批

        System.out.println("是否目录   = " + attr.isDirectory());
        System.out.println("大小       = " + attr.size());
        System.out.println("创建时间   = " + attr.creationTime());
        System.out.println("修改时间   = " + attr.lastModifiedTime());
        System.out.println("访问时间   = " + attr.lastAccessTime());

        // 读单个属性也行，但要按名字拼字符串，不如上面的类型安全
        System.out.println("size 单项  = " + Files.getAttribute(p, "size"));
    }
}
```

`BasicFileAttributes` 只是最通用的一层，实际有四类视图：

| 视图 | 额外提供 |
|------|---------|
| `BasicFileAttributes` | 时间、大小、类型、文件键 |
| `DosFileAttributes` | 只读、隐藏、系统、存档（**Windows**） |
| `PosixFileAttributes` | 属主、组、权限（Linux/macOS） |
| `AclFileAttributes` | 访问控制列表（Windows） |

> **注意**：`Files.getAttribute(p, "dos:hidden")` 这类调用**强依赖文件系统**，在 Linux 上会抛 `UnsupportedOperationException`。跨平台代码要么 `try/catch` 包住，要么直接用 `Files.isHidden(p)` 这种已屏蔽平台差异的通用方法。

# 8. 文件树遍历：walk 与 walkFileTree

`Files` 提供两个遍历整棵树的方法，**都是迭代实现，不会栈溢出**。

## 8.1 Files.walk：流式遍历

```java
import java.nio.file.*;
import java.util.stream.Stream;

public class WalkStreamDemo {
    public static void main(String[] args) throws Exception {
        Path root = Path.of("D:/data");
        // walk 返回的 Stream<Path> 持有打开的目录句柄，必须放进 try-with-resources 关闭
        try (Stream<Path> stream = Files.walk(root, 3)) {         // 3 = 最大深度
            stream.filter(Files::isRegularFile)                    // 只要普通文件
                  .forEach(p -> System.out.println(p.toAbsolutePath()));
        }
    }
}
```

| 方法 | 说明 |
|------|------|
| `walk(Path start, FileVisitOption...)` | 深度不限 |
| `walk(Path start, int maxDepth, FileVisitOption...)` | 限定最大深度 |
| `find(Path, int maxDepth, BiPredicate<Path, BasicFileAttributes>)` | 带属性条件的查找 |
| `list(Path dir)` | **只列一层**，返回 `Stream<Path>`，大目录比 `listFiles` 更省内存 |
| `newDirectoryStream(Path dir)` | 返回 `DirectoryStream<Path>`，**只需遍历一次时的最省内存方案** |
| `newDirectoryStream(Path dir, String glob)` | 按 glob 模式过滤，如 `"*.java"` |

```java
import java.nio.file.*;
import java.util.stream.Stream;

public class StreamListDemo {
    public static void main(String[] args) throws Exception {
        Path dir = Path.of("D:/data");

        // list：只列一层，返回 Stream，同样必须关闭
        try (Stream<Path> s = Files.list(dir)) {
            System.out.println("一层条目数 = " + s.count());
        }

        // newDirectoryStream：同样是"一层"，但它不生成 Stream 中间对象，
        // 遍历超大目录时内存占用最低；用 for-each 迭代即可，关闭由 try 负责
        try (DirectoryStream<Path> ds = Files.newDirectoryStream(dir, "*.java")) {
            for (Path p : ds) {
                System.out.println("匹配到 " + p.getFileName());
            }
        }
    }
}
```

> **注意**：`DirectoryStream` **只能正向迭代一次**，不是 `Collection`，没有 `size()`、`get()`，也不能重复遍历。它的定位非常明确：**"只要扫一遍，扫完就走"的场景内存占用最低。** 需要多次遍历或随机访问时，仍然要用 `Files.list` 收集到集合，或者退回 `File.listFiles()`。

> **陷阱**：`Files.walk` 和 `Files.lines` 返回的 `Stream` **背后持有打开的文件句柄**，不关闭会造成句柄泄漏——在 Windows 上表现为"文件明明没被用却删不掉"。**必须放进 `try-with-resources`**。这是 NIO.2 里最常被忽略的资源泄漏点。

> **注意**：`Files.walk` **默认不跟随符号链接**。需要跨链接遍历时传 `FileVisitOption.FOLLOW_LINKS`，但要小心**循环链接导致无限遍历**，此时必须设置 `maxDepth`。

## 8.2 Files.walkFileTree：访问者模式

`walkFileTree` 用**访问者模式**，通过 `FileVisitor` 的回调在每个节点做处理，能表达 `walk` 表达不了的控制流。

| 回调方法 | 触发时机 | 返回值含义 |
|---------|---------|-----------|
| `preVisitDirectory` | **进入**目录前 | `CONTINUE` 继续 / `SKIP_SUBTREE` 跳过整个子树 / `TERMINATE` 终止 |
| `visitFile` | 访问到文件 | `CONTINUE` / `TERMINATE` |
| `visitFileFailed` | 访问失败（权限、IO 错误） | `CONTINUE` / `TERMINATE` / `SKIP_SUBTREE` |
| `postVisitDirectory` | **离开**目录后 | 常用于后序删除 |

`SimpleFileVisitor` 是 `FileVisitor` 的空实现（所有方法默认 `CONTINUE`），继承它只需要覆写关心的方法。

```java
import java.io.IOException;
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;

public class DeleteTreeDemo {
    /** 用 walkFileTree 删除整棵目录树：后序删除，天然正确 */
    public static void deleteTree(Path root) throws IOException {
        Files.walkFileTree(root, new SimpleFileVisitor<Path>() {
            @Override
            public FileVisitResult visitFile(Path file, BasicFileAttributes attrs)
                    throws IOException {
                Files.delete(file);                     // 先删文件
                return FileVisitResult.CONTINUE;
            }

            @Override
            public FileVisitResult visitFileFailed(Path file, IOException exc) {
                System.out.println("跳过无法访问的 " + file + "：" + exc.getClass().getSimpleName());
                return FileVisitResult.CONTINUE;        // 单个失败不中断整体
            }

            @Override
            public FileVisitResult postVisitDirectory(Path dir, IOException exc)
                    throws IOException {
                if (exc != null) throw exc;             // 遍历该目录时出过错，向上抛
                Files.delete(dir);                      // 子内容已清空，再删目录
                return FileVisitResult.CONTINUE;
            }
        });
    }

    public static void main(String[] args) throws IOException {
        deleteTree(Path.of("D:/data/tmp"));
        System.out.println("目录树已删除");
    }
}
```

## 8.3 递归 vs 流式：怎么选

| 维度 | 手写递归 | `Files.walk` | `Files.walkFileTree` |
|------|---------|-------------|---------------------|
| 代码量 | 少 | **最少** | 中 |
| 栈溢出风险 | **有** | 无 | 无 |
| 跳过子树 | 需自己写 | 不支持 | **支持（`SKIP_SUBTREE`）** |
| 失败处理 | 需自己写 | 抛异常中断整个流 | **`visitFileFailed` 逐个处理** |
| 需要关闭 | 否 | **是** | 否 |
| 适合场景 | 教学、层级很浅 | **查询、统计、筛选** | 删除、复制、带容错的批量处理 |

> **行业实践**：**"读"用 `Files.walk`，"改"用 `Files.walkFileTree`。** 前者借助 Stream API 写起来最简洁，适合"找文件、算大小、收集路径"；后者能精确控制"进入 / 离开 / 失败"三个时机，适合删除、移动、批量重命名这类**会改变文件系统状态**的操作。

# 9. StandardOpenOption：文件打开选项

`StandardOpenOption` 是 `Files.newOutputStream`、`Files.newInputStream`、`Files.write`、`Files.writeString`、`FileChannel.open` 等方法共用的**打开方式枚举**。它决定了"文件不存在怎么办""已有内容保不保留""能不能读"这些关键语义。

| 选项 | 含义 | 关键点 |
|------|------|--------|
| `READ` | 以**读**方式打开 | `newInputStream` 的隐含默认 |
| `WRITE` | 以**写**方式打开 | `newOutputStream` / `write` 的隐含默认 |
| `CREATE` | 不存在则**创建** | 已存在时**忽略**，不报错 |
| `CREATE_NEW` | 不存在则创建，**已存在则抛异常** | 想"绝不覆盖"时用它，抛 `FileAlreadyExistsException` |
| `APPEND` | 写到**文件末尾** | 需配 `WRITE`；想让"文件不存在也能建"要再加 `CREATE` |
| `TRUNCATE_EXISTING` | 打开时**清空**已有内容 | 只对已有文件有意义 |
| `DELETE_ON_CLOSE` | 关闭时删除文件 | 临时文件的简陋版，尽力而为，JVM 崩溃时不保证 |
| `SYNC` / `DSYNC` | 每次写都同步落盘 | **极慢**，只在数据绝不能丢的场景用 |

各方法**不给选项时的默认行为**（最容易记错的一组）：

| 调用 | 默认选项 | 效果 |
|------|---------|------|
| `Files.newInputStream(p)` | `READ` | 读；不存在抛 `NoSuchFileException` |
| `Files.newOutputStream(p)` | `WRITE, CREATE, TRUNCATE_EXISTING` | **覆盖写**，不存在则创建 |
| `Files.write(p, bytes)` | `WRITE, CREATE, TRUNCATE_EXISTING` | **覆盖写** |
| `Files.writeString(p, s)` | `WRITE, CREATE, TRUNCATE_EXISTING` | **覆盖写** |
| `Files.newBufferedWriter(p, cs)` | `WRITE, CREATE, TRUNCATE_EXISTING` | **覆盖写** |

> **陷阱**：很多人以为 `Files.write` 是"追加"，其实它**默认清空原文件**。想保留原内容必须显式传 `APPEND`，而且记得补 `CREATE`：

```java
import java.io.BufferedWriter;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;

public class AppendDemo {
    public static void main(String[] args) throws Exception {
        Path log = Path.of("D:/data/app.log");

        // ✗ 错误理解：以为这是追加。实际每次运行都会把日志清空
        Files.writeString(log, "新日志\n");

        // ✓ 正确写法一：追加 + 不存在则创建（两个选项缺一不可）
        Files.writeString(log, "新日志\n", StandardCharsets.UTF_8,
                          StandardOpenOption.APPEND, StandardOpenOption.CREATE);

        // ✓ 正确写法二：用 newBufferedWriter 组合
        try (BufferedWriter bw = Files.newBufferedWriter(log, StandardCharsets.UTF_8,
                StandardOpenOption.APPEND, StandardOpenOption.CREATE)) {
            bw.write("新日志\n");
        }

        System.out.println(Files.readString(log).replace("\n", "\\n"));
    }
}
/* 输出：
新日志\n新日志\n
*/
```

**选项的合法组合**有两条必须记住：

1. `APPEND` 与 `TRUNCATE_EXISTING` **语义冲突**，同时传会抛 `IllegalArgumentException`；
2. `READ` 与 `TRUNCATE_EXISTING` 同时传时，`TRUNCATE_EXISTING` 会被**静默忽略**（只读打开谈不上清空）。

```java
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.util.List;

public class OptionConflict {
    public static void main(String[] args) throws Exception {
        Path p = Path.of("D:/data/app.log");
        try {
            Files.write(p, List.of("x"), StandardCharsets.UTF_8,
                        StandardOpenOption.APPEND, StandardOpenOption.TRUNCATE_EXISTING);
        } catch (Exception e) {
            System.out.println(e.getClass().getName());
        }
    }
}
/* 输出：
java.lang.IllegalArgumentException
*/
```

再看一个 `CREATE_NEW` 的典型用法——**生成唯一文件，绝不覆盖已有数据**：

```java
import java.io.IOException;
import java.nio.file.*;

public class LockDemo {
    public static void main(String[] args) {
        Path lock = Path.of("D:/data/app.lock");
        try {
            // CREATE_NEW 保证"文件已存在就失败"，天然适合做单实例锁
            Files.writeString(lock, String.valueOf(ProcessHandle.current().pid()),
                              StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE);
            System.out.println("拿到锁，PID 已写入");
        } catch (FileAlreadyExistsException e) {
            System.out.println("已有实例在运行，启动中止");       // 第二次运行输出这句
        } catch (IOException e) {
            System.out.println("其他 IO 错误：" + e.getMessage());
        }
    }
}
```

# 10. File 与 Path / Files 的全面对比

| 维度 | `java.io.File`（JDK1.0） | `Path` + `Files`（JDK7+） |
|------|------------------------|--------------------------|
| 所在包 | `java.io` | `java.nio.file` |
| 失败反馈 | 返回 `false`，**无原因** | **抛具体异常**（`NoSuchFileException` 等） |
| 读整个文件 | 要自己写循环 | `Files.readAllLines` / `readString`（JDK11+） |
| 写整个文件 | 要自己写循环 | `Files.write` / `writeString`（JDK11+） |
| 复制 | **无** | `Files.copy` |
| 移动 / 重命名 | `renameTo`，跨平台行为不一致 | `Files.move`，支持 `ATOMIC_MOVE` |
| 临时文件 | **无** | `Files.createTempFile` / `createTempDirectory` |
| 遍历目录 | `listFiles`，返回数组，**可能为 `null`** | `Files.list` / `walk`，返回 `Stream`，**不关闭会泄漏句柄** |
| 文件属性 | 大小、时间、权限几项 | `readAttributes` 批量读，含属主、ACL |
| 符号链接 | **无法识别** | `isSymbolicLink` / `createSymbolicLink` / `readSymbolicLink` |
| 默认字符集 | `FileReader` / `FileWriter` 用**平台默认** | `Files.*` 的文本方法统一用 **UTF-8** |
| 路径比较 | 未重写 `equals`（按对象比） | 重写 `equals`（按路径元素比），`Files.isSameFile` 查真实文件 |
| 扩展性 | 写死本地文件系统 | 可指向 Zip、内存文件系统等 |

**选择建议：**

| 场景 | 推荐 |
|------|------|
| 新建项目、纯文件读写 | **一律用 `Path` + `Files`** |
| 需要与老 API 交互（`Properties.load`、`Desktop.open` 等） | 用 `path.toFile()` 转换，或先找是否已有 `Path` 重载 |
| 只是传一个路径字符串、不做任何 IO | 两者都行，`Path` 更符合现代风格 |
| 递归删除、批量变更 | `Files.walkFileTree` |

> **行业实践**：**新代码不要再用 `File` 做文件操作。** `Path` / `Files` 在异常信息、编码默认值、跨平台一致性上都明显更好——尤其在"到底为什么失败"这件事上，`NoSuchFileException`、`AccessDeniedException`、`DirectoryNotEmptyException` 比一个 `false` 有用得多。`File` 保留下来主要是为了兼容历史 API，遇到时用 `toPath()` / `toFile()` 互转即可。

> **注意**：`File` 与 `Path` 的互转**是有损的**——`Path` 可以指向 Zip 内部条目或非默认文件系统的路径，这种 `Path` 调用 `toFile()` 会抛 `UnsupportedOperationException`。本地文件系统上两者等价，跨文件系统时 `toFile()` 不保证成功。

# 11. 小结

| 主题 | 一句话要点 |
|------|-----------|
| `File` 的本质 | **只是路径的抽象**，`new File("不存在")` 不报错，必须主动用 `exists()` 验证 |
| 三种路径 | `getPath()` 原样返回；`getAbsolutePath()` 拼工作目录但**保留 `.` / `..`**；`getCanonicalPath()` 才唯一（会访问磁盘、抛 `IOException`） |
| 相对路径基准 | 是**当前工作目录** `System.getProperty("user.dir")`，与源码/class 文件位置无关；生产代码不要依赖它 |
| 分隔符 | 路径内用 `File.separator`，多路径间用 `File.pathSeparator`；**写死 `/` 最安全** |
| 创建删除 | 多级目录用 `mkdirs()`（不是 `mkdir()`）；`delete()` 只能删空目录且失败原因不明 |
| `listFiles` 的坑 | 路径不存在、不是目录、无权限时**都返回 `null`**，不判空必 NPE |
| 递归的代价 | 每层递归占一个栈帧，深目录会 `StackOverflowError`；改用 `Files.walk` / `walkFileTree` |
| NIO.2 | `Path` 表达路径，`Files` 提供静态工具方法；**文本方法默认 UTF-8**，失败抛具体异常 |
| 文件树遍历 | 查询统计用 `Files.walk`（**必须 `try-with-resources` 关闭**），增删改用 `Files.walkFileTree` |
| `StandardOpenOption` | `Files.write` 默认是**覆盖**不是追加；追加要 `APPEND + CREATE`；`APPEND` 与 `TRUNCATE_EXISTING` 冲突 |
| 选型 | **新代码用 `Path` + `Files`，`File` 只用于兼容老 API** |

下一篇（第 16 篇）进入 IO 流：`File` 只解决了"文件在哪"，真正把数据读进来、写出去的是 `InputStream` / `OutputStream` / `Reader` / `Writer` 这套流体系，那里会讲清缓冲、编码、序列化与打印流的原理和坑。
