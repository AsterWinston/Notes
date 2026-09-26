# 1.File对象

```
File 对象是 java.io 包下用于表示文件和目录路径名的抽象表示。
File 对象不代表文件本身，只是一个路径的抽象封装。
```

## 1.1构造

```
1.1.1 public File(String pathname)
      // 通过路径名字符串创建 File 对象
      File f = new File("D:\\data\\test.txt");

1.1.2 public File(String parent, String child)
      // 通过父路径 + 子路径创建
      File f = new File("D:\\data", "test.txt");

1.1.3 public File(File parent, String child)
      // 通过父 File 对象 + 子路径创建
      File parent = new File("D:\\data");
      File f = new File(parent, "test.txt");
```

## 1.2 路径概念

```
1.2.1 绝对路径（Absolute Path）
  从根目录开始的完整路径。
  Windows：D:\data\test.txt
  Linux：/home/user/test.txt

1.2.2 相对路径（Relative Path）
  相对于当前工作目录的路径。
  在 IDE 中运行，当前目录通常是项目根目录。
  File f = new File("test.txt");  // 相对于当前工作目录

1.2.3 规范路径（Canonical Path）
  解析了 . 和 .. 后的绝对路径。
  getAbsolutePath() 可能包含 . 和 ..
  getCanonicalPath() 会解析回实际的路径（可能抛出 IOException）。

1.2.4 路径分隔符
  Windows：\（反斜杠，Java 中需要转义写 \\）
  Linux/Mac：/（正斜杠）
  Java 中的推荐写法：使用正斜杠 /（Java 在所有平台都能识别），
  或使用 File.separator 动态获取。
```

## 1.3遍历

```
1.3.1 public static File[] listRoots()
    列出可用的文件系统根（Windows 返回 C:\, D:\ 等盘符）。

1.3.2 public String[] list()
    获取当前路径下所有内容的名称数组（只返回文件名/目录名的字符串）。

1.3.3 public String[] list(FilenameFilter filter)
    利用文件名过滤器获取当前路径下所有内容的名称。

1.3.4 public File[] listFiles()
    获取当前路径下所有内容的 File 对象数组（推荐使用）。

1.3.5 public File[] listFiles(FileFilter filter)
    利用文件过滤器获取当前路径下内容。
    示例：
      File[] files = dir.listFiles(f -> f.isFile() && f.getName().endsWith(".txt"));

1.3.6 public File[] listFiles(FilenameFilter filter)
    利用文件名过滤器获取当前路径下内容。
    示例：
      File[] files = dir.listFiles((d, name) -> name.endsWith(".java"));
```

## 1.4File的常见成员方法（判断、获取）

```
1.4.1 public boolean isDirectory()    // 判断是否为目录（文件夹）
1.4.2 public boolean isFile()         // 判断是否为文件
1.4.3 public boolean exists()         // 判断文件或目录是否存在
1.4.4 public boolean isHidden()       // 判断是否为隐藏文件
1.4.5 public boolean isAbsolute()     // 判断是否为绝对路径
1.4.6 public boolean canRead()        // 判断是否可读
1.4.7 public boolean canWrite()       // 判断是否可写
1.4.8 public boolean canExecute()     // 判断是否可执行
1.4.9 public long length()            // 返回文件的字节数（目录返回 0 或不确定值）
1.4.10 public String getAbsolutePath() // 返回绝对路径名
1.4.11 public String getPath()         // 返回构造时使用的路径
1.4.12 public String getName()         // 返回文件/目录名（带后缀）
1.4.13 public String getParent()       // 返回父目录路径（字符串）
1.4.14 public File getParentFile()     // 返回父目录的 File 对象
1.4.15 public long lastModified()      // 返回最后修改时间的毫秒值
1.4.16 public long getTotalSpace()     // 返回分区总空间（字节）
1.4.17 public long getFreeSpace()      // 返回分区未分配空间（字节）
1.4.18 public long getUsableSpace()    // 返回分区可用空间（字节）
1.4.19 public String getCanonicalPath() throws IOException // 返回规范路径
```

## 1.5File的常见成员方法（创建、删除）

```
1.5.1 public boolean createNewFile() throws IOException
    创建一个新的空的文件，父级路径必须存在，否则报错。
    如果文件已存在，返回 false（不报错）。

1.5.2 public boolean mkdir()
    创建单级文件夹。父级路径必须存在。
    创建成功返回 true，已存在或失败返回 false。

1.5.3 public boolean mkdirs()
    创建多级文件夹（包括不存在的父目录）。推荐使用此方法。
    创建成功返回 true。

1.5.4 public boolean delete()
    删除文件或空文件夹（只能删除空文件夹，不放入回收站）。
    不建议直接删除有内容的数据，删除成功返回 true。

1.5.5 public boolean renameTo(File dest)
    重命名文件/文件夹（实质是移动+重命名）。
    相当于 Linux 的 mv 命令。成功返回 true。

1.5.6 public boolean setReadOnly()
    设置文件为只读。

1.5.7 public boolean setLastModified(long time)
    设置最后修改时间。
```

## 1.6 递归遍历和删除文件夹

```
递归遍历文件夹示例：
  思路：获取目录下所有 File，判断是文件则处理，是目录则递归进入。

  public static void traverseDir(File dir) {
      if (!dir.isDirectory()) return;
      File[] files = dir.listFiles();
      if (files == null) return;
      for (File f : files) {
          if (f.isFile()) {
              System.out.println("文件: " + f.getAbsolutePath());
          } else if (f.isDirectory()) {
              System.out.println("目录: " + f.getAbsolutePath());
              traverseDir(f);  // 递归遍历子目录
          }
      }
  }

递归删除文件夹（包含所有内容）：
  思路：先删除内部所有内容，再删除空文件夹。

  public static void deleteDir(File dir) {
      if (!dir.isDirectory()) return;
      File[] files = dir.listFiles();
      if (files != null) {
          for (File f : files) {
              if (f.isFile()) {
                  f.delete();
              } else if (f.isDirectory()) {
                  deleteDir(f);  // 先递归删除子目录内容
              }
          }
      }
      dir.delete();  // 删除已清空的目录
  }
```

# 2.NIO.2 — Path 接口和 Files 工具类（JDK7+）

```
NIO.2（New I/O 2）是 JDK7 引入的全新文件操作 API，位于 java.nio.file 包。
相比于传统 File 类，NIO.2 提供了更现代、更高效的文件操作方式。

File vs NIO.2 对比：
  File：传统 API，功能有限，异常信息不明确
  NIO.2：现代 API，功能丰富，异常信息详细，支持异步操作
```

## 2.1Path 接口（替代 File 的路径抽象）

```
Path 是 NIO.2 中的路径接口，比 File 更灵活。

2.1.1 获取 Path 对象：
  使用 Paths.get()（JDK7）或 Path.of()（JDK11+）：
    Path p1 = Paths.get("D:\\data", "test.txt");
    Path p2 = Path.of("D:\\data", "test.txt");   // JDK11+

2.1.2 Path 的常用方法：
  toString()              // 返回路径字符串
  getFileName()           // 返回文件名或最底层目录名
  getParent()             // 返回父路径
  getRoot()               // 返回根路径（如 D:\ 或 /）
  getNameCount()          // 返回路径中元素的个数
  getName(int index)      // 返回指定索引的元素
  resolve(String other)   // 拼接路径（自动处理分隔符）
  resolvesibling(String)  // 拼接兄弟路径
  relativize(Path other)  // 获取相对路径
  normalize()             // 去除路径中的 . 和 ..
  toAbsolutePath()        // 转为绝对路径
  toFile()                // 转为 File 对象
  startsWith(Path other)  // 判断是否以指定路径开头
  endsWith(Path other)    // 判断是否以指定路径结尾

Path 与 File 互转：
  File → Path：file.toPath()
  Path → File：path.toFile()
```

## 2.2Files 工具类

```
Files 是 NIO.2 中的工具类，提供了大量静态方法进行文件操作。

2.2.1 文件/目录判断和获取
  exists(path)           // 判断是否存在
  isDirectory(path)      // 判断是否为目录
  isRegularFile(path)    // 判断是否为普通文件
  isHidden(path)         // 判断是否隐藏
  isReadable(path)       // 是否可读
  isWritable(path)       // 是否可写
  isExecutable(path)     // 是否可执行
  size(path)             // 返回文件大小（字节），目录抛出异常
  getLastModifiedTime(path)  // 获取最后修改时间

2.2.2 文件创建和删除
  createFile(path)                 // 创建文件（已存在抛异常）
  createDirectory(path)            // 创建单级目录
  createDirectories(path)          // 创建多级目录（推荐）
  createTempFile(prefix, suffix)   // 创建临时文件
  createTempDirectory(prefix)      // 创建临时目录
  delete(path)                     // 删除文件/空目录（不存在抛异常）
  deleteIfExists(path)             // 存在则删除（更安全）
  copy(source, target, options...)  // 复制文件/目录
  move(source, target, options...)  // 移动/重命名文件

2.2.3 文件读写（更便捷）
  readString(path)                  // JDK11+，读整个文件为字符串
  readString(path, charset)         // 指定编码读取
  writeString(path, str)            // JDK11+，写字符串到文件
  writeString(path, str, options)  // 追加模式等
  readAllBytes(path)                // 读取全部字节
  write(path, bytes[])              // 写入字节
  readAllLines(path)                // 读取全部行到 List<String>
  newBufferedReader(path)           // 创建 BufferedReader
  newBufferedWriter(path)           // 创建 BufferedWriter
  newInputStream(path)              // 创建 InputStream
  newOutputStream(path)             // 创建 OutputStream
  lines(path)                       // 返回 Stream<String>（按行），配合 Stream API 使用

2.2.4 目录遍历
  list(path)              // 列出所有内容（返回 Stream<Path>），推荐用于大目录
  walk(path, maxDepth)    // 递归遍历（返回 Stream<Path>），
                          //    注意关闭流，或使用 try-with-resources
  find(path, maxDepth, matcher)  // 递归查找匹配条件的文件
  newDirectoryStream(path)       // 高效遍历（返回 DirectoryStream）
  newDirectoryStream(path, glob) // 按 glob 模式遍历

  递归遍历示例：
    try (Stream<Path> stream = Files.walk(Paths.get("D:\\data"), 3)) {
        stream.filter(Files::isRegularFile)
              .forEach(System.out::println);
    }

2.2.5 文件属性
  getAttribute(path, "size")             // 获取指定属性
  setAttribute(path, "dos:hidden", true) // 设置属性
  readAttributes(path, BasicFileAttributes.class) // 读取基本属性

2.2.6 符号链接相关
  createSymbolicLink(link, target)  // 创建符号链接
  isSymbolicLink(path)              // 判断是否为符号链接
  readSymbolicLink(path)            // 读取符号链接目标
```

## 2.3StandardOpenOption（文件打开选项）

```
用于 Files 的 write、newInputStream/newOutputStream 等方法。

标准选项：
  StandardOpenOption.READ        // 以读模式打开
  StandardOpenOption.WRITE       // 以写模式打开
  StandardOpenOption.CREATE      // 文件不存在则创建
  StandardOpenOption.CREATE_NEW  // 文件不存在则创建，已存在则抛异常
  StandardOpenOption.APPEND      // 追加模式（默认覆盖）
  StandardOpenOption.TRUNCATE_EXISTING  // 清空已存在的文件内容
  StandardOpenOption.DELETE_ON_CLOSE    // 关闭时删除文件（临时文件）
```
