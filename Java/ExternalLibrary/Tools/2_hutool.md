# 1. 概述与简介

Hutool 是一个功能丰富的 Java 工具类库，由国人开发维护（作者：路小磊，Looly），旨在简化 Java 开发中的常见操作。它通过静态方法封装，提供了大量实用工具类，覆盖字符串处理、集合操作、日期处理、文件 I/O、加密解密、HTTP 请求、反射、JSON 处理等多个领域。

## 1.1 Hutool 的设计哲学

- **简单易用**：所有 API 都是静态方法，一行代码解决问题
- **功能全面**：一个依赖解决 90% 的工具类需求
- **无额外依赖**：核心模块 `hutool-core` 无任何第三方依赖
- **中文文档**：提供完善的中文文档和中文注释
- **自动适配**：根据运行环境自动选择最佳实现

## 1.2 模块架构

Hutool 采用模块化设计，主要模块包括：

| 模块 | 说明 |
|------|------|
| hutool-core | 核心工具类（字符串、集合、日期、反射、IO 等） |
| hutool-http | HTTP 客户端 |
| hutool-crypto | 加密解密（对称、非对称、摘要） |
| hutool-json | JSON 处理 |
| hutool-db | 数据库操作（JDBC 封装） |
| hutool-cache | 缓存工具 |
| hutool-setting | 配置文件工具 |
| hutool-log | 日志门面 |
| hutool-system | 系统信息获取 |
| hutool-poi | Office 文档处理（Excel、Word） |
| hutool-extra | 扩展模块（模版引擎、邮件、FTP 等） |

## 1.3 Maven 依赖

```xml
<!-- 按需引入各个模块 -->
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-core</artifactId>
    <version>5.8.25</version>
</dependency>
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-http</artifactId>
    <version>5.8.25</version>
</dependency>
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-crypto</artifactId>
    <version>5.8.25</version>
</dependency>
```

或者引入完整模块（包含全部子模块）：

```xml
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.8.25</version>
</dependency>
```

---

# 2. 核心功能与 API

## 2.1 字符串工具 - StrUtil

StrUtil 是使用频率最高的工具类之一，覆盖了几乎所有常见的字符串操作。

```java
public class StrUtilExample {

    public static void main(String[] args) {
        // ========== 判空 ==========
        StrUtil.isEmpty(null);      // true
        StrUtil.isEmpty("");        // true
        StrUtil.isEmpty("  ");      // false（空格不算空）
        StrUtil.isBlank(null);      // true
        StrUtil.isBlank("  ");      // true（空白字符算空）
        StrUtil.isNotBlank("abc");  // true
        StrUtil.hasBlank("a", " ", "c");  // true
        StrUtil.isAllNotEmpty("a", "b");  // true

        // ========== 去除 ==========
        StrUtil.removePrefix("abcde", "abc");  // "de"
        StrUtil.removeSuffix("abcde", "de");   // "abc"
        StrUtil.removeAll("aabbcc", "b");      // "aacc"
        StrUtil.cleanBlank(" a  b ");          // "ab"（去除空白字符）
        StrUtil.trim("  abc  ");               // "abc"

        // ========== 截取 ==========
        StrUtil.sub("abcdefg", 2, 5);     // "cde"
        StrUtil.subBefore("a_b_c", "_", false);  // "a"
        StrUtil.subAfter("a_b_c", "_", false);   // "b_c"
        StrUtil.subBetween("[abc]", "[", "]");   // "abc"
        StrUtil.subSuf("abcdef", 3);       // "def"
        StrUtil.subPre("abcdef", 3);       // "abc"

        // ========== 格式化 ==========
        StrUtil.format("你好，{}！今天是{}", "张三", "周一");
        // "你好，张三！今天是周一"
        StrUtil.fillBefore("1", '0', 5);  // "00001"
        StrUtil.fillAfter("1", '0', 5);   // "10000"

        // ========== 转换 ==========
        StrUtil.toCamelCase("user_name");      // "userName"
        StrUtil.toUnderlineCase("userName");   // "user_name"
        StrUtil.bytes("hello");                // byte[]
        StrUtil.uuid();                        // 32位UUID（无横线）

        // ========== 其他常用 ==========
        StrUtil.repeat("ab", 3);           // "ababab"
        StrUtil.count("abcaba", "a");      // 3
        StrUtil.reverse("abc");            // "cba"
        StrUtil.wrap("abc", "\"");         // "\"abc\""
        StrUtil.indexOf("abcabc", "a", 2); // 从index 2开始找第一个a -> 3
        StrUtil.padPre("1", 5, "0");       // "00001"（左补齐）
        StrUtil.padAfter("1", 5, "0");     // "10000"（右补齐）
        StrUtil.upperFirst("hello");       // "Hello"
        StrUtil.lowerFirst("Hello");       // "hello"
        StrUtil.equalsAny("a", "a", "b", "c");  // true
        StrUtil.startWithAny("abc", "ab", "cd"); // true
    }
}
```

## 2.2 集合工具 - CollUtil

CollUtil 提供了集合操作的大量便捷方法：

```java
public class CollUtilExample {

    public static void main(String[] args) {
        // ========== 创建集合 ==========
        List<String> list1 = CollUtil.newArrayList("a", "b", "c");
        List<String> list2 = CollUtil.of("a", "b", "c");   // 更简洁
        Set<String> set = CollUtil.newHashSet("a", "b");
        Map<String, String> map = CollUtil.newHashMap();

        // ========== 判空 ==========
        CollUtil.isEmpty(list1);    // false
        CollUtil.isNotEmpty(list1); // true
        CollUtil.isEmpty(null);     // true

        // ========== 集合操作 ==========
        List<String> list = CollUtil.newArrayList("a", "b", "c", "d", "e");

        // 交集
        List<String> intersection = CollUtil.intersection(
            CollUtil.newArrayList("a", "b", "c"),
            CollUtil.newArrayList("b", "c", "d")
        );  // ["b", "c"]

        // 并集
        List<String> union = CollUtil.union(
            CollUtil.newArrayList("a", "b"),
            CollUtil.newArrayList("b", "c")
        );  // ["a", "b", "c"]

        // 差集
        List<String> disjunction = CollUtil.disjunction(
            CollUtil.newArrayList("a", "b", "c"),
            CollUtil.newArrayList("b", "c", "d")
        );  // ["a", "d"]

        // 补集（减去）
        List<String> subtract = CollUtil.subtract(
            CollUtil.newArrayList("a", "b", "c"),
            CollUtil.newArrayList("b")
        );  // ["a", "c"]

        // ========== 分组 ==========
        List<User> users = getUserList();
        Map<String, List<User>> groupByCity = CollUtil.groupByField(users, "city");

        // ========== 提取 ==========
        // 提取某字段
        List<String> names = CollUtil.getFieldValues(users, "name");
        // 提取并去重
        Set<String> cities = CollUtil.fieldAsSet(users, "city");

        // ========== 分页 ==========
        int page = 1, pageSize = 10;
        List<String> pageResult = CollUtil.page(page, pageSize, list);

        // ========== 排序 ==========
        CollUtil.sort(list, Comparator.naturalOrder());
        CollUtil.sortByProperty(list, "name");  // 按属性排序

        // ========== 其他常用 ==========
        CollUtil.join(list, ",");         // "a,b,c,d,e"
        CollUtil.count(list, s -> s.startsWith("a"));  // 符合条件的元素个数

        // 分割集合为多个子集合
        List<List<String>> split = CollUtil.split(list, 2);
        // [["a", "b"], ["c", "d"], ["e"]]

        // 获取集合中指定元素
        String last = CollUtil.getLast(list);    // "e"
        String first = CollUtil.getFirst(list);  // "a"
        String random = CollUtil.randomEle(list); // 随机一个
    }

    static class User {
        private String name;
        private String city;
        // getter/setter
    }
}
```

## 2.3 Map 工具 - MapUtil

```java
public class MapUtilExample {

    public static void main(String[] args) {
        // ========== 创建 ==========
        Map<String, Object> map = MapUtil.builder()
                .put("name", "张三")
                .put("age", 25)
                .put("city", "北京")
                .build();

        // ========== 判空 ==========
        MapUtil.isEmpty(map);     // false
        MapUtil.isNotEmpty(map);  // true

        // ========== 操作 ==========
        // 批量获取
        List<Object> values = MapUtil.getValues(map, "name", "age");

        // 排序
        Map<String, Object> sorted = MapUtil.sort(map);

        // 反转键值
        Map<Object, String> reverse = MapUtil.reverse(
            MapUtil.of("key1", "value1", "key2", "value2")
        );

        // ========== 转换 ==========
        // Map 转 Bean
        User user = MapUtil.toObjectIgnoreCase(
            MapUtil.of("Name", "张三", "Age", 25), User.class);

        // ========== 常用快捷操作 ==========
        // 创建不可变 Map（一个元素）
        Map<String, Object> single = MapUtil.of("key", "value");
        // 创建不可变 Map（多个元素）
        Map<String, Object> multi = MapUtil.of("k1", "v1", "k2", "v2");

        // 获取值，带默认值
        String value = MapUtil.getStr(map, "name");
        Integer age = MapUtil.getInt(map, "age");
        Object nameOrDef = MapUtil.get(map, "nickname", "默认昵称");

        // 获取 key 数组/值数组
        Set<String> keys = MapUtil.keySet(map);
    }
}
```

## 2.4 Bean 工具 - BeanUtil

```java
public class BeanUtilExample {

    public static void main(String[] args) {
        User user = new User("张三", 25, "北京");

        // ========== Bean 转 Map ==========
        Map<String, Object> map = BeanUtil.beanToMap(user);

        // 下划线形式 key
        Map<String, Object> map2 = BeanUtil.beanToMap(user, true, false);

        // ========== Map 转 Bean ==========
        Map<String, Object> sourceMap = MapUtil.builder()
                .put("name", "李四")
                .put("age", 30)
                .put("city", "上海")
                .build();
        User user2 = BeanUtil.toBean(sourceMap, User.class);

        // ========== Bean 复制（属性拷贝） ==========
        UserVO vo = new UserVO();
        BeanUtil.copyProperties(user, vo);

        // 忽略某些字段
        BeanUtil.copyProperties(user, vo, "age");

        // ========== 获取/设置属性 ==========
        String name = BeanUtil.getProperty(user, "name");
        BeanUtil.setProperty(user, "name", "王五");

        // ========== 判断 ==========
        BeanUtil.isBean(User.class);             // true
        BeanUtil.isNotEmpty(user, "name");       // true

        // ========== 字段操作 ==========
        // 获取所有字段（包括继承的）
        Field[] fields = BeanUtil.getFields(User.class);
        // 获取 getter/setter 方法
        Method getter = BeanUtil.getGetter(User.class, "name");
        Method setter = BeanUtil.getSetter(User.class, "name");

        // ========== 填充 ==========
        // 用 Map 填充 Bean（忽略大小写）
        Map<String, Object> data = MapUtil.of("NAME", "赵六", "AGE", 35);
        User user3 = BeanUtil.fillBeanWithMap(data, new User(), true);
    }
}
```

## 2.5 日期处理 - DateUtil

DataUtil 是 Hutool 中功能最强的工具类之一，解决了 Java 8 之前日期 API 的痛点。

```java
public class DateUtilExample {

    public static void main(String[] args) {
        // ========== 当前时间 ==========
        Date now = DateUtil.date();                        // 当前时间
        Date now2 = DateUtil.dateSecond();                 // 精确到秒
        long currentMillis = DateUtil.current();           // 当前毫秒时间戳

        // ========== 格式化 ==========
        String format1 = DateUtil.format(now, "yyyy-MM-dd HH:mm:ss");
        String format2 = DateUtil.formatDate(now);         // yyyy-MM-dd
        String format3 = DateUtil.formatTime(now);         // HH:mm:ss
        String format4 = DateUtil.formatDateTime(now);     // yyyy-MM-dd HH:mm:ss

        // ========== 解析 ==========
        Date parsed1 = DateUtil.parse("2024-01-15");
        Date parsed2 = DateUtil.parse("2024-01-15 14:30:00");
        Date parsed3 = DateUtil.parse("2024/01/15", "yyyy/MM/dd");

        // ========== 日期时间计算 ==========
        Date yesterday = DateUtil.yesterday();
        Date tomorrow = DateUtil.tomorrow();
        Date lastWeek = DateUtil.lastWeek();
        Date nextWeek = DateUtil.nextWeek();
        Date lastMonth = DateUtil.lastMonth();

        // 偏移
        Date offsetDay = DateUtil.offsetDay(now, 5);     // 5天后
        Date offsetHour = DateUtil.offsetHour(now, -3);  // 3小时前
        Date offsetMinute = DateUtil.offsetMinute(now, 30);
        Date offsetMonth = DateUtil.offsetMonth(now, 1);

        // ========== 计算间隔 ==========
        Date start = DateUtil.parse("2024-01-01");
        Date end = DateUtil.parse("2024-01-15");

        long betweenDay = DateUtil.betweenDay(start, end, true);    // 14天
        long betweenHour = DateUtil.betweenHour(start, end);        // 336小时
        long betweenMinute = DateUtil.betweenMinute(start, end);
        long betweenSecond = DateUtil.betweenSecond(start, end);
        long betweenWeek = DateUtil.betweenWeek(start, end, true);

        // ========== 年龄计算 ==========
        Date birthDate = DateUtil.parse("1990-05-15");
        int age = DateUtil.ageOfNow(birthDate);  // 从出生到现在的年龄

        // ========== 星座/生肖 ==========
        String zodiac = DateUtil.getZodiac(DateUtil.parse("1990-05-15"));  // 金牛座
        String chineseZodiac = DateUtil.getChineseZodiac(1990);            // 马

        // ========== 时间范围 ==========
        // 一天的开始和结束
        Date beginOfDay = DateUtil.beginOfDay(now);     // 00:00:00
        Date endOfDay = DateUtil.endOfDay(now);         // 23:59:59
        Date beginOfMonth = DateUtil.beginOfMonth(now);
        Date endOfMonth = DateUtil.endOfMonth(now);

        // ========== 判断 ==========
        DateUtil.isInRange(now, start, end);      // 是否在时间范围内
        DateUtil.isSameDay(now, now);              // 是否同一天
        DateUtil.isSameMonth(now, now);
        DateUtil.isSameTime(now, now);

        // ========== 星期 ==========
        DayOfWeek dayOfWeek = DateUtil.dayOfWeek(now);   // DayOfWeek 枚举
        String chineseDayOfWeek = DateUtil.dayOfWeekEnum(now).toChinese();  // "星期一"

        // ========== Date <=> LocalDateTime ==========
        // Date 转 LocalDateTime
        LocalDateTime ldt = DateUtil.toLocalDateTime(now);
        // LocalDateTime 转 Date
        Date fromLocal = DateUtil.date(LocalDateTime.now());

        // ========== 时间字符串 ==========
        // "刚刚"、"3分钟前" 等人类可读格式
        String humanReadable = DateUtil.formatBetween(DateUtil.parse("2024-01-01"), DateUtil.date());
        // "14天前"
    }
}
```

## 2.6 文件 I/O - FileUtil 和 IoUtil

```java
public class FileIOExample {

    public static void main(String[] args) {
        // ========== FileUtil ==========
        // 创建文件/目录
        File file = FileUtil.touch("D:/test/example.txt");
        File dir = FileUtil.mkdir("D:/test/subdir");

        // 写文件
        FileUtil.writeUtf8String("Hello Hutool", file);
        FileUtil.writeString("Hello 中文", file, CharsetUtil.CHARSET_UTF_8);
        FileUtil.writeLines(CollUtil.newArrayList("a", "b", "c"), file);

        // 追加内容
        FileUtil.appendString("追加内容", file);
        FileUtil.appendLines(CollUtil.newArrayList("line1", "line2"), file);

        // 读文件
        String content = FileUtil.readUtf8String(file);
        List<String> lines = FileUtil.readUtf8Lines(file);
        byte[] bytes = FileUtil.readBytes(file);

        // 文件信息
        long size = FileUtil.size(file);          // 文件大小（字节）
        String readableSize = FileUtil.readableFileSize(file);  // "1.5KB"
        String ext = FileUtil.extName(file);      // "txt"
        String name = FileUtil.getName(file);     // "example.txt"
        String prefix = FileUtil.mainName(file);  // "example"（不含扩展名）

        // 文件操作
        FileUtil.copy(file, new File("D:/test/copy.txt"), true);
        FileUtil.move(file, new File("D:/test/moved.txt"), true);
        FileUtil.rename(file, "renamed.txt", true);
        FileUtil.del("D:/test/subdir");           // 删除文件或目录

        // 目录遍历
        List<File> files = FileUtil.loopFiles("D:/test");
        List<File> txtFiles = FileUtil.loopFiles("D:/test", file -> file.getName().endsWith(".txt"));

        // 判断
        FileUtil.isFile(file);         // true
        FileUtil.isDir(dir);           // true
        FileUtil.isDirectoryEmpty(dir);
        FileUtil.exist(file);

        // 路径
        String normalizedPath = FileUtil.normalize("D:/test//subdir/../file.txt");
        // ========== IoUtil ==========
        // 从流读取字符串
        InputStream in = new FileInputStream(file);
        String str = IoUtil.readUtf8(in);

        // 从流读取 byte[]
        byte[] data = IoUtil.readBytes(new FileInputStream(file));

        // 写到流
        IoUtil.write(new FileOutputStream(file), CharsetUtil.CHARSET_UTF_8, false, "content");

        // 复制流
        IoUtil.copy(new FileInputStream(file), new FileOutputStream("D:/test/copy2.txt"));

        // 关闭（自动处理 null 和异常）
        IoUtil.close(in);
    }
}
```

## 2.7 HTTP 工具 - HttpUtil

```java
public class HttpUtilExample {

    public static void main(String[] args) {
        // ========== GET 请求 ==========
        // 简单 GET
        String result1 = HttpUtil.get("https://api.example.com/users");

        // GET 带参数
        Map<String, Object> params = MapUtil.builder()
                .put("page", 1)
                .put("size", 10)
                .build();
        String result2 = HttpUtil.get("https://api.example.com/users", params);

        // ========== POST 请求 ==========
        // 简单 POST（表单）
        String result3 = HttpUtil.post("https://api.example.com/login",
                MapUtil.of("username", "admin", "password", "123456"));

        // POST JSON 体
        String jsonBody = JSONUtil.toJsonStr(MapUtil.of("name", "张三"));
        String result4 = HttpUtil.post("https://api.example.com/users", jsonBody);

        // ========== 链式构建 (HttpRequest) ==========
        String result5 = HttpRequest.get("https://api.example.com/users")
                .header("Authorization", "Bearer xxx")
                .header("Accept", "application/json")
                .form("page", 1)
                .form("size", 10)
                .timeout(5000)
                .execute()
                .body();

        // POST JSON
        String result6 = HttpRequest.post("https://api.example.com/users")
                .header("Content-Type", "application/json")
                .body(JSONUtil.toJsonStr(MapUtil.of("name", "张三")))
                .timeout(5000)
                .execute()
                .body();

        // ========== HttpResponse 处理 ==========
        HttpResponse response = HttpRequest.get("https://api.example.com/data")
                .execute();

        int status = response.getStatus();
        String body = response.body();
        Map<String, List<String>> headers = response.headers();
        String contentType = response.header("Content-Type");

        // 下载文件
        File file = HttpUtil.downloadFileFromUrl("https://example.com/file.zip",
                FileUtil.file("D:/download/"));

        // 检查 URL 是否有效
        boolean isValid = HttpUtil.isHttps("https://example.com");
    }
}
```

## 2.8 加密解密 - SecureUtil

```java
public class SecureUtilExample {

    public static void main(String[] args) {
        // ========== 摘要算法（MD5、SHA） ==========
        String md5 = SecureUtil.md5("hello");                    // 32位小写
        String md5_16 = SecureUtil.md5("hello").substring(8, 24); // 16位
        String sha1 = SecureUtil.sha1("hello");
        String sha256 = SecureUtil.sha256("hello");

        // ========== 对称加密 ==========
        // AES 加密
        String content = "需要加密的敏感信息";
        String key = "1234567890abcdef";  // 16位密钥

        AES aes = SecureUtil.aes(key.getBytes());
        String encryptHex = aes.encryptHex(content);     // 加密为十六进制字符串
        String decryptStr = aes.decryptStr(encryptHex);  // 解密

        // DES 加密
        DES des = SecureUtil.des();
        String desEncrypt = des.encryptBase64(content);

        // ========== 非对称加密 ==========
        // RSA 加密
        RSA rsa = SecureUtil.rsa();

        // 生成公私钥
        String privateKey = rsa.getPrivateKeyBase64();
        String publicKey = rsa.getPublicKeyBase64();

        // 公钥加密，私钥解密
        String rsaEncrypt = rsa.encryptBase64(content, KeyType.PublicKey);
        String rsaDecrypt = rsa.decryptStr(rsaEncrypt, KeyType.PrivateKey);

        // ========== 签名 ==========
        Sign sign = SecureUtil.sign(SignAlgorithm.SHA256withRSA);
        byte[] signed = sign.sign(content.getBytes());
        boolean verified = sign.verify(content.getBytes(), signed);

        // ========== Base64 ==========
        String base64 = SecureUtil.base64("hello");
        String decoded = SecureUtil.base64DecodeStr(base64);
    }
}
```

## 2.9 JSON 工具 - JSONUtil

```java
public class JSONUtilExample {

    public static void main(String[] args) {
        User user = new User("张三", 25, "北京");

        // ========== 对象转 JSON 字符串 ==========
        String json = JSONUtil.toJsonStr(user);
        // {"name":"张三","age":25,"city":"北京"}

        // 格式化
        String prettyJson = JSONUtil.toJsonPrettyStr(user);

        // ========== JSON 字符串转对象 ==========
        User parsedUser = JSONUtil.toBean(json, User.class);

        // 转 List
        String jsonArray = "[{\"name\":\"张三\"},{\"name\":\"李四\"}]";
        List<User> users = JSONUtil.toList(jsonArray, User.class);

        // ========== JSONObject / JSONArray ==========
        JSONObject obj = JSONUtil.parseObj(json);
        String name = obj.getStr("name");
        int age = obj.getInt("age");

        JSONArray arr = JSONUtil.parseArray(jsonArray);
        JSONObject first = arr.getJSONObject(0);

        // ========== 创建 JSON ==========
        JSONObject newObj = JSONUtil.createObj()
                .set("name", "李四")
                .set("age", 30);
        JSONArray newArr = JSONUtil.createArray()
                .set(newObj);

        // ========== XML 转 JSON ==========
        String xml = "<root><name>张三</name><age>25</age></root>";
        JSONObject xmlJson = JSONUtil.parseFromXml(xml);

        // ========== JSON 转 XML ==========
        String toXml = JSONUtil.toXmlStr(newObj);

        // ========== 判断 ==========
        JSONUtil.isJson(json);        // true
        JSONUtil.isJsonObj(json);     // true
        JSONUtil.isJsonArray(jsonArray); // true
    }
}
```

## 2.10 反射工具 - ReflectUtil

```java
public class ReflectUtilExample {

    public static void main(String[] args) {
        User user = new User("张三", 25, "北京");

        // ========== 创建实例 ==========
        User newUser = ReflectUtil.newInstance(User.class);
        User withArgs = ReflectUtil.newInstance(User.class, "张三", 25, "北京");

        // ========== 调用方法 ==========
        Object result = ReflectUtil.invoke(user, "getName");
        ReflectUtil.invoke(user, "setName", "李四");

        // 调用静态方法
        ReflectUtil.invokeStatic(ReflectUtil.getMethod(User.class, "staticMethod"));

        // ========== 获取/设置字段 ==========
        Object fieldValue = ReflectUtil.getFieldValue(user, "name");
        ReflectUtil.setFieldValue(user, "name", "王五");

        // ========== 获取信息 ==========
        // 获取所有字段（包括继承）
        Field[] fields = ReflectUtil.getFields(User.class);

        // 获取所有方法
        Method[] methods = ReflectUtil.getMethods(User.class);

        // 获取指定方法
        Method getNameMethod = ReflectUtil.getMethod(User.class, "getName");

        // ========== 获取类信息 ==========
        Package pkg = ReflectUtil.getPackage(User.class);
    }
}
```

## 2.11 类型转换 - Convert

```java
public class ConvertExample {

    public static void main(String[] args) {
        // ========== 基本类型转换 ==========
        int intVal = Convert.toInt("123");
        long longVal = Convert.toLong("123456");
        double doubleVal = Convert.toDouble("3.14");
        boolean boolVal = Convert.toBool("true");
        String str = Convert.toStr(123);

        // 带默认值
        int intDef = Convert.toInt("abc", 0);  // 转换失败返回 0

        // ========== 数组转换 ==========
        String[] strArray = {"1", "2", "3"};
        Integer[] intArray = Convert.toIntArray(strArray);
        long[] longArray = Convert.toLongArray(strArray);

        // int[] <=> Integer[]
        int[] primitiveInts = {1, 2, 3};
        Integer[] wrapperInts = Convert.toIntArray(primitiveInts);

        // ========== 集合转换 ==========
        List<String> list = Convert.toList(String.class, "a", "b", "c");

        // 数组转 List
        List<String> listFromArray = Convert.toList(String.class, strArray);

        // ========== 编码转换 ==========
        // 数字转中文大写（金额）
        String money = Convert.digitToChinese(12345.67);
        // "壹万贰仟叁佰肆拾伍元陆角柒分"

        // 16进制与字符串互转
        String hexStr = Convert.toHex("hello", CharsetUtil.CHARSET_UTF_8);
        String original = Convert.hexToStr(hexStr, CharsetUtil.CHARSET_UTF_8);

        // unicode 与字符串互转
        String unicodeStr = Convert.strToUnicode("中文");
        String chinese = Convert.unicodeToStr(unicodeStr);
    }
}
```

## 2.12 ID 生成 - IdUtil

```java
public class IdUtilExample {

    public static void main(String[] args) {
        // ========== UUID ==========
        String uuid = IdUtil.randomUUID();    // 带横线：550e8400-e29b-41d4-a716-446655440000
        String uuidSimple = IdUtil.simpleUUID();  // 不带横线：550e8400e29b41d4a716446655440000
        String fastUUID = IdUtil.fastUUID();  // 性能更优的实现

        // ========== 雪花算法（Snowflake） ==========
        // 参数：workerId（机器ID 0-31），datacenterId（数据中心ID 0-31）
        Snowflake snowflake = IdUtil.getSnowflake(1, 1);
        long snowId = snowflake.nextId();          // long 类型
        String snowIdStr = snowflake.nextIdStr();  // 字符串类型

        // 也可通过 IdUtil 直接获取
        long id = IdUtil.getSnowflakeNextId();
        String idStr = IdUtil.getSnowflakeNextIdStr();

        // ========== 对象 ID ==========
        // 基于 JVM 唯一对象 ID
        String objectId = IdUtil.objectId();

        // ========== nanoId（更短、更安全的 ID） ==========
        // 需要引入 hutool-core 5.8+ 或单独依赖
        // String nanoId = IdUtil.nanoId();
    }
}
```

### 2.12.1 Snowflake 算法原理

Snowflake 生成的 ID 是一个 64 位的 long，结构如下：

```
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
|       |                  |                |        |            |
1bit    41bit 时间戳       5bit 数据中心ID  5bit 机器ID   12bit 序列号
（固定0）  （毫秒级）         （0-31）       （0-31）     （0-4095）
```

- 1bit：符号位，固定为 0
- 41bit：时间戳（毫秒），可以使用 69 年
- 10bit：工作机器 ID（5bit 数据中心 + 5bit 机器ID）
- 12bit：序列号，同一毫秒内可生成 4096 个 ID

---

# 3. 高级特性

## 3.1 缓存工具 - CacheUtil

```java
public class CacheUtilExample {

    public static void main(String[] args) {
        // FIFO 缓存（先进先出）
        Cache<String, String> fifoCache = CacheUtil.newFIFOCache(3);
        fifoCache.put("a", "1");
        fifoCache.put("b", "2");
        fifoCache.put("c", "3");
        fifoCache.put("d", "4");  // 超过容量，移除 "a"
        System.out.println(fifoCache.get("a"));  // null

        // LRU 缓存（最近最少使用）
        Cache<String, String> lruCache = CacheUtil.newLRUCache(3);
        lruCache.put("a", "1");
        lruCache.put("b", "2");
        lruCache.put("c", "3");
        lruCache.get("a");  // 使用 "a"，使其成为最近使用
        lruCache.put("d", "4");  // 移除 "b"（最少使用）
        System.out.println(lruCache.get("b"));  // null

        // LFU 缓存（最不经常使用）
        Cache<String, String> lfuCache = CacheUtil.newLFUCache(3);

        // 定时缓存（TTL）
        Cache<String, String> timedCache = CacheUtil.newTimedCache(5000);  // 5秒过期
        timedCache.put("key", "value");
        Thread.sleep(6000);
        System.out.println(timedCache.get("key"));  // null
    }
}
```

## 3.2 配置文件工具 - Setting

```java
public class SettingExample {

    public static void main(String[] args) {
        // 读取 .setting 文件（类似 .properties，但支持分组）
        // config.setting 文件内容：
        // [db]
        // url = jdbc:mysql://localhost:3306/test
        // username = root
        // password = 123456
        //
        // [redis]
        // host = localhost
        // port = 6379

        Setting setting = new Setting("config.setting");

        // 读取配置
        String dbUrl = setting.getStr("db.url");
        String dbUser = setting.get("db.username", "defaultUser");  // 带默认值
        Integer redisPort = setting.getInt("redis.port");

        // 自动重载（文件修改后自动加载）
        setting.autoReload(true);

        // 转换为 Bean
        DbConfig dbConfig = setting.toBean("db", DbConfig.class);

        // ========== Properties 文件 ==========
        // Hutool 也支持标准的 .properties 文件
        Props props = new Props("application.properties");
        String value = props.getStr("key");
    }
}
```

## 3.3 日志门面 - Log

```java
public class LogExample {

    private static final Log log = Log.get();

    public static void main(String[] args) {
        // 自动适配底层日志框架（Log4j、Logback、SLF4J、JDK Logging 等）
        log.info("用户 {} 登录成功", "张三");
        log.error("发生异常", new RuntimeException("test"));
        log.warn("配置未找到，使用默认值");
        log.debug("debug 信息");

        // 如果是 SLF4J，会自动桥接到 SLF4J
        // 如果没有任何日志框架，使用 JDK Logging
    }
}
```

## 3.4 扩展模块 - hutool-extra

```java
public class ExtraExample {

    public static void main(String[] args) {
        // ========== 邮件 ==========
        MailUtil.send("receiver@example.com", "主题", "内容", true);

        // ========== 模板引擎 ==========
        // 支持 FreeMarker、Velocity、Beetl、Thymeleaf、Enjoy
        TemplateEngine engine = TemplateUtil.createEngine();
        Template template = engine.getTemplate("templates/hello.ftl");
        String result = template.render(MapUtil.of("name", "张三"));

        // ========== FTP ==========
        // FTP ftp = Ftp.create(host, port, user, password);
        // ftp.upload("/remote/path", FileUtil.file("local/file.txt"));
        // ftp.download("/remote/path", FileUtil.file("local/download"));

        // ========== 二维码 ==========
        // QrCodeUtil.generate("content", 300, 300, FileUtil.file("qrcode.png"));
    }
}
```

---

# 4. 底层原理

## 4.1 Hutool 与 Apache Commons / Guava 的设计差异

| 特性 | Hutool | Apache Commons | Guava |
|------|--------|---------------|-------|
| 设计哲学 | 中式 API，静态方法，极简 | 面向对象，工具类丰富 | Google 系，函数式风格 |
| 核心依赖 | hutool-core 无任何第三方依赖 | 各模块相互依赖 | 仅 Guava 自身 |
| 方法命名 | 中文式英文命名（isEmpty/isBlank） | 较完整但冗长 | 简洁但偏函数式 |
| 日期处理 | 极其丰富（DateUtil） | 一般（DateUtils） | 较弱（主要在集合） |
| HTTP 支持 | 内置（HttpUtil） | 需单独引入（HttpClient） | 无 |
| 加密支持 | 内置（SecureUtil） | 需引入 Codec | 有 Hashing 但较弱 |
| JSON 支持 | 内置（JSONUtil） | 无 | 无 |
| 更新频率 | 活跃（国内社区） | 稳定但更新少 | 稳定 |
| 学习成本 | 低（API 简洁直观） | 中 | 中（函数式概念多） |

## 4.2 Hutool 的核心设计模式

### 4.2.1 工厂模式

Hutool 大量使用静态工厂方法，例如 `SecureUtil.aes()`、`IdUtil.getSnowflake()` 等，简化对象创建。

### 4.2.2 建造者模式

```java
HttpRequest.post(url)
    .header("Content-Type", "application/json")
    .body(json)
    .timeout(5000)
    .execute();
```

### 4.2.3 适配器模式

日志模块适配了多种日志框架（Log4j、Logback、SLF4J、JDK Logging）。

## 4.3 Hutool 的分层架构

```
应用程序代码
     ↑  ↓
┌────────────────┐
│   Hutool API   │  ← 静态方法入口（StrUtil、FileUtil 等）
├────────────────┤
│   核心实现层    │  ← 具体实现类（StrFormatter、FileTypeUtil 等）
├────────────────┤
│   工具方法层    │  ← 底层通用方法
├────────────────┤
│   JDK 标准库    │  ← java.lang, java.io, java.nio 等
└────────────────┘
```

---

# 5. 性能优化

## 5.1 对象复用

```java
// 错误：频繁创建
for (int i = 0; i < 10000; i++) {
    String json = JSONUtil.toJsonStr(user);  // 每次都创建新的 JSON 配置
}

// 正确：复用 JSON 配置
JSONConfig config = JSONConfig.create()
        .setIgnoreNullValue(true)
        .setDateFormat("yyyy-MM-dd");
for (int i = 0; i < 10000; i++) {
    String json = JSONUtil.toJsonStr(user, config);
}
```

## 5.2 文件操作的缓冲区

```java
// 大文件处理时指定缓冲区大小
FileUtil.readLines(file, CharsetUtil.CHARSET_UTF_8, (line, index) -> {
    // 逐行处理，不加载全部到内存
});
```

## 5.3 避免不必要的装箱拆箱

```java
// Hutool 的 Convert 会自动处理类型转换缓存
// 内部使用 TypeConverter 注册表缓存转换器
```

## 5.4 缓存策略

```java
// TimedCache 适合需要 TTL 的场景
// LRUCache 适合固定大小的热点数据缓存
// FIFOCache 适合先进先出淘汰策略
```

---

# 6. 最佳实践与常见坑

## 6.1 最佳实践

### 6.1.1 按需引入模块

不要引入 `hutool-all` 过大依赖，按需引入子模块：

```xml
<!-- 只引入需要的模块 -->
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-core</artifactId>
</dependency>
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-http</artifactId>
</dependency>
```

### 6.1.2 使用 BeanUtil 的注意事项

```java
// BeanUtil.copyProperties 是浅拷贝
User original = new User("张三", 25, "北京");
User target = new User();
BeanUtil.copyProperties(original, target);
target.setName("李四");
System.out.println(original.getName());  // 张三（String 不可变，安全）

// 但嵌套对象是浅拷贝
class Department {
    private List<User> users;
}
Department src = new Department();
Department dst = new Department();
BeanUtil.copyProperties(src, dst);
// dst.getUsers() == src.getUsers()  // true（同一个引用）
```

### 6.1.3 日期格式统一

```java
// 全局日期格式常量
public class DatePattern {
    public static final String NORM_DATETIME_PATTERN = "yyyy-MM-dd HH:mm:ss";
}
```

### 6.1.4 链式调用使用技巧

```java
HttpRequest.post("https://api.example.com/data")
    .header("Authorization", "Bearer " + token)
    .form("key1", "value1")
    .form("key2", "value2")
    .timeout(3000)
    .execute()
    .body();
```

### 6.1.5 安全编码

```java
// 使用 SecureUtil 处理敏感信息
String password = "user_password";
String hashed = SecureUtil.sha256(password);  // 不要明文存储
```

## 6.2 常见坑

### 6.2.1 坑 1：JSONUtil 对 null 的处理

```java
// JSONUtil.toJsonStr 默认不忽略 null 值
// 输出: {"name":"张三","age":null}
JSONUtil.toJsonStr(new User("张三", null, "北京"));

// 需要忽略 null 时需要配置
JSONConfig config = JSONConfig.create().setIgnoreNullValue(true);
JSONUtil.toJsonStr(new User("张三", null, "北京"), config);
// 输出: {"name":"张三","city":"北京"}
```

### 6.2.2 坑 2：FileUtil 写文件覆盖

```java
// FileUtil.writeXXX 系列方法会覆盖已有文件
FileUtil.writeUtf8String("内容1", file);
FileUtil.writeUtf8String("内容2", file);  // 覆盖为"内容2"

// 追加内容需要用 appendXXX
FileUtil.appendString("内容3", file);   // 文件内容变为 "内容2内容3"
```

### 6.2.3 坑 3：DateUtil 不同版本兼容性

```java
// Hutool 5.x 对 DateUtil 做了较大改动
// 某些 4.x 的方法在 5.x 中已废弃或移除
// 升级时注意检查 API 变更

// 5.x 推荐使用 DateUtil.toLocalDateTime() 替代部分旧 API
```

### 6.2.4 坑 4：HttpUtil 默认超时

```java
// HttpUtil 默认无超时限制！
// 生产环境必须设置超时
String result = HttpRequest.get("https://api.example.com")
        .timeout(5000)  // 连接超时 5 秒
        .execute()
        .body();
```

### 6.2.5 坑 5：类型转换异常

```java
// Convert.toInt 转换失败返回 null（对象类型）或 0（基本类型）
// 不是抛异常，注意判空
Integer val = Convert.toInt("abc");  // null
int val2 = Convert.toInt("abc");    // 0（自动拆箱）

// 如果希望转换失败抛异常，使用 Integer.parseInt
```

### 6.2.6 坑 6：雪花算法 ID 重复

```java
// 同一个 workerId + datacenterId 组合在高并发下
// 或时钟回拨时可能产生重复 ID
// 解决方案：
// 1. 确保 workerId 和 datacenterId 全局唯一
// 2. 使用 Zookeeper 等协调服务自动分配
// 3. 使用支持时钟回拨的改进版雪花算法
```

---

# 7. 与其他技术对比

## 7.1 Hutool vs Apache Commons

| 对比项 | Hutool | Apache Commons |
|--------|--------|---------------|
| 模块数量 | 约 20 个 | 40+ 个 |
| 依赖关系 | 无额外依赖 | 模块间相互依赖 |
| API 风格 | 中文思维，符合国人习惯 | 标准的英文命名 |
| JSON | 内置 | 无 |
| HTTP | 内置 | 需 HttpClient |
| 加密 | 内置 | 需 Codec |
| 文件操作 | 更简洁 | 较详细 |
| 学习曲线 | 低 | 中 |

## 7.2 Hutool vs Guava

| 对比项 | Hutool | Guava |
|--------|--------|-------|
| 核心功能 | 工具类 | 集合、缓存、函数式 |
| 集合增强 | CollUtil | 丰富（Multimap、BiMap 等） |
| 缓存 | CacheUtil 简单实现 | 强大（CacheBuilder、LoadingCache） |
| 函数式 | 弱 | 强 |
| 事件总线 | 无 | EventBus |
| 限流 | 无 | RateLimiter |
| 额外依赖 | 无 | 无 |
| 体积 | ~5MB（hutool-all） | ~3MB |

## 7.3 选型建议

- **国内项目、快速开发**：首选 Hutool，API 简洁直观
- **国际项目、大型工程**：Guava（集合、缓存） + Apache Commons（其他）
- **老旧项目维护**：使用项目中已有的工具库，不要混用
- **新项目技术选型**：Hutool 作为基础工具库 + Guava 作为补充（集合）

---

# 8. 面试要点

## 8.1 基础问题

**Q1: Hutool 的核心设计理念是什么？**

A: 通过静态方法封装常见操作，一行代码解决问题。无额外依赖，API 简洁，符合国人使用习惯。

**Q2: StrUtil.isEmpty 和 StrUtil.isBlank 的区别？**

A: `isEmpty` 检查 null 和空字符串（""）；`isBlank` 在 `isEmpty` 基础上增加空白字符检查（空格、制表符、换行等）。`isBlank("\t\n")` 返回 true。

**Q3: Hutool 中 BeanUtil.copyProperties 是深拷贝还是浅拷贝？**

A: 浅拷贝。对于基本类型和 String 安全（不可变类），但嵌套对象的引用会被直接复制。

**Q4: DateUtil 和 Java 8 的 java.time 包的关系？**

A: Hutool 5.x 的 DateUtil 内部支持 `java.time`，提供了 `toLocalDateTime()`、`toLocalDate()` 等转换方法。但 DateUtil 本身操作的是 `java.util.Date`。

**Q5: 雪花算法 ID 的结构是怎样的？**

A: 1bit 符号位 + 41bit 时间戳 + 5bit 数据中心ID + 5bit 机器ID + 12bit 序列号。共 64 位，生成 long 类型 ID。

## 8.2 进阶问题

**Q6: Hutool 的 HttpUtil 和 RestTemplate / HttpClient 相比优缺点？**

A: 优点：API 极其简洁，链式调用方便，无需额外依赖；缺点：功能不如 HttpClient 丰富，缺少连接池高级配置，不适合复杂 HTTP 场景。

**Q7: Hutool 如何自动适配日志框架？**

A: 通过 `StaticLog` 和 `LogFactory`，采用优先顺序：Log4j2 > Logback > Log4j > SLF4J > JDK Logging。通过 `Class.forName()` 检测对应日志类是否存在。

**Q8: 如何扩展 Hutool 的功能？**

A: 继承已有的工具类覆盖方法，或使用 Hutool 的 SPI 机制（如自定义 `LogFactory`）。也可以通过 `hutool-extra` 模块添加对其他库的封装。

**Q9: 如何解决 Hutool JSON 循环引用问题？**

A: Hutool 的 JSONUtil 默认会检测循环引用并抛出异常。可以使用 `JSONConfig` 设置最大递归深度，或手动处理引用关系。

**Q10: 在 Spring Boot 中如何整合 Hutool？**

A: Hutool 是纯工具类库，直接引入 Maven 依赖即可使用，无需任何配置。可以将常用的 Hutool Bean 注册为 Spring Bean 方便注入。
