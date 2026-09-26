# 1.SqlSource 体系概览

```
SqlSource 是 MyBatis 中表示 SQL 源的抽象接口。
它负责将 Mapper.xml 或注解中的 SQL 语句转换为可执行的 BoundSql。

MyBatis 源码中 SqlSource 的体系结构：

┌──────────────────────────────────────────────────────────────┐
│                       SqlSource (接口)                        │
│                       getBoundSql(parameterObject)            │
├──────────────────────────────────────────────────────────────┤
│          │                    │                    │          │
│          ▼                    ▼                    ▼          │
│   DynamicSqlSource      RawSqlSource       StaticSqlSource   │
│   (动态SQL源)            (静态SQL源)         (完全静态SQL源)    │
│                                                              │
│   每次调用都要重新           解析时一次性处理         直接持有     │
│   解析 #{} 和动态标签       完 #{} 占位符          最终的SQL    │
│                                                              │
│   适用：含有 ${}           适用：只有 #{}          适用：程序化   │
│   或动态标签的SQL           的静态SQL              构建的SQL    │
└──────────────────────────────────────────────────────────────┘
```

# 2.getBoundSql() 的执行流程

```
SqlSource.getBoundSql(parameterObject) 的执行取决于具体实现：

DynamicSqlSource.getBoundSql(parameterObject):
  1. 根据 parameterObject 创建 DynamicContext
  2. 遍历 SqlNode 树，每个 SqlNode 调用 apply(context)
     - TextSqlNode：将静态文本追加到 context
     - IfSqlNode：判断 test 条件，为 true 则递归 apply
     - WhereSqlNode：动态处理 WHERE 子句
     - ForEachSqlNode：遍历集合，拼接 SQL 片段
     - TrimSqlNode：添加前缀/后缀，覆盖前缀/后缀
  3. 从 DynamicContext 中获取拼接完成的 SQL
  4. 创建 SqlSourceBuilder 解析其中的 #{} 
     → 替换为 ?
     → 生成 ParameterMapping 列表
  5. 返回 new StaticSqlSource(config, finalSql, parameterMappings)

RawSqlSource.getBoundSql(parameterObject):
  （构造时已经完成了 #{} 解析，每次调用直接返回）
  1. 从内部的 StaticSqlSource 获取 BoundSql
  2. 直接返回

StaticSqlSource.getBoundSql(parameterObject):
  1. 使用已持有的 sql 和 parameterMappings 直接构建 BoundSql
  2. 直接返回
```

# 3.SqlNode 树 — 动态 SQL 的 AST

```
MyBatis 将动态 SQL 的 XML 标签解析为一棵 SqlNode 树（AST），
每次执行 SQL 时遍历这棵树来动态生成最终的 SQL 语句。

SqlNode 接口：
  boolean apply(DynamicContext context);

  apply() 方法负责将本节点的 SQL 片段追加到 DynamicContext 中。
  返回 true 表示应用成功。

MyBatis 中所有的 SqlNode 类型：

┌─────────────────────────────────────────────────────────────┐
│                    SqlNode (接口)                             │
│                    apply(DynamicContext): boolean            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ├── TextSqlNode         纯文本（可能含 ${}）                 │
│  ├── StaticTextSqlNode   纯文本（不含占位符）                 │
│  │                                                          │
│  ├── IfSqlNode           条件判断 (if)                       │
│  ├── WhereSqlNode        WHERE 智能处理 (where)              │
│  ├── SetSqlNode          SET 智能处理 (set)                  │
│  ├── TrimSqlNode         前缀/后缀处理 (trim)                │
│  ├── ForEachSqlNode      循环遍历 (foreach)                   │
│  ├── ChooseSqlNode       多条件选择 (choose/when/otherwise)   │
│  │                                                          │
│  ├── MixedSqlNode        混合节点（包含多个子节点）             │
│  ├── VarDeclSqlNode      变量声明 (bind)                     │
│  └── ...                                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

# 4.XMLScriptBuilder — SqlNode 树的构建过程

```
XMLScriptBuilder 负责将 Mapper.xml 中的 SQL 标签解析为 SqlNode 树。

源码：org.apache.ibatis.scripting.xmltags.XMLScriptBuilder

解析流程：

  XMLScriptBuilder.parseScriptNode()
    │
    ├── 解析 SQL 标签的子节点（XNode）
    │
    │   对于纯文本节点：
    │     → 判断是否包含 ${} 
    │       → 包含 ${}: 生成 TextSqlNode
    │       → 不含 ${}: 生成 StaticTextSqlNode
    │
    │   对于 XML 元素节点：
    │     → 根据标签名创建对应的 SqlNode:
    │       ├── "if"        → new IfSqlNode()
    │       ├── "where"     → new WhereSqlNode()
    │       ├── "set"       → new SetSqlNode()
    │       ├── "trim"      → new TrimSqlNode()
    │       ├── "foreach"   → new ForEachSqlNode()
    │       ├── "choose"    → new ChooseSqlNode()
    │       ├── "when"      → new IfSqlNode() (作为 choose 的子节点)
    │       ├── "otherwise" → new MixedSqlNode() (作为 choose 的子节点)
    │       └── "bind"      → new VarDeclSqlNode()
    │
    └── 返回 MixedSqlNode (包含所有子节点的混合节点)

  关键方法：parseDynamicTags(XNode node)

  protected MixedSqlNode parseDynamicTags(XNode node) {
      List<SqlNode> contents = new ArrayList<>();
      NodeList children = node.getNode().getChildNodes();
      for (int i = 0; i < children.getLength(); i++) {
          XNode child = node.newXNode(children.item(i));
          if (child.getNode().getNodeType() == Node.TEXT_NODE) {
              // 文本节点处理
              String data = child.getStringBody("");
              TextSqlNode textSqlNode = new TextSqlNode(data);
              if (textSqlNode.isDynamic()) {  // 包含 ${}
                  contents.add(textSqlNode);
              } else {
                  contents.add(new StaticTextSqlNode(data));
              }
          } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) {
              // 元素节点处理 → 递归解析
              String nodeName = child.getNode().getNodeName();
              NodeHandler handler = nodeHandlerMap.get(nodeName);
              if (handler != null) {
                  handler.handleNode(child, contents);
              }
          }
      }
      return new MixedSqlNode(contents);
  }
```

# 5.逐个 SqlNode 源码分析

## 5.1 IfSqlNode — 条件判断

```java
/**
 * IfSqlNode 对应 <if test="xxx"> 标签
 * 源码：org.apache.ibatis.scripting.xmltags.IfSqlNode
 *
 * 核心逻辑：
 *   - 使用 ExpressionEvaluator 解析 test 属性中的 OGNL 表达式
 *   - 如果表达式计算结果为 true，则将 contents 子节点追加到 DynamicContext
 */
public class IfSqlNode implements SqlNode {
    private final ExpressionEvaluator evaluator;
    private final String test;              // test 属性值，如 "brand != null and brand != ''"
    private final SqlNode contents;         // <if> 标签内的 SqlNode 子节点

    public IfSqlNode(SqlNode contents, String test) {
        this.test = test;
        this.contents = contents;
        this.evaluator = new ExpressionEvaluator();
    }

    @Override
    public boolean apply(DynamicContext context) {
        // 使用 OGNL 引擎解析 test 表达式
        // 例如：test = "brand != null and brand != ''"
        // evaluator.evaluateBoolean() 会从 context.getBindings() 中获取变量值
        if (evaluator.evaluateBoolean(test, context.getBindings())) {
            // 条件为 true → 递归应用子节点
            contents.apply(context);
            return true;
        }
        return false;
    }
}
```

## 5.2 WhereSqlNode — 智能 WHERE

```java
/**
 * WhereSqlNode 对应 <where> 标签
 * 源码：org.apache.ibatis.scripting.xmltags.WhereSqlNode
 *
 * 核心逻辑：
 *   1. 创建临时的 DynamicContext 收集子节点的 SQL
 *   2. 获取子节点生成的 SQL 片段
 *   3. 如果 SQL 片段非空：
 *      - 去除开头的 "AND " 或 "OR "
 *      - 添加 "WHERE " 前缀
 *      - 追加到主 DynamicContext
 *   4. 如果 SQL 片段为空：什么也不追加（不会出现空的 WHERE 子句）
 */
public class WhereSqlNode extends TrimSqlNode {

    public WhereSqlNode(Configuration configuration, SqlNode contents) {
        // 调用父类 TrimSqlNode 构造器
        // prefix="WHERE "                      — 添加 WHERE 前缀
        // prefixOverrides="AND |OR "           — 去除开头的 AND 或 OR
        // suffix=""                            — 不添加后缀
        // suffixOverrides=""                   — 不去除后缀
        super(configuration, contents, "WHERE ", "AND |OR ", null, null);
    }
}
```

## 5.3 SetSqlNode — 智能 SET

```java
/**
 * SetSqlNode 对应 <set> 标签
 * 源码：org.apache.ibatis.scripting.xmltags.SetSqlNode
 *
 * 核心逻辑：
 *   1. 收集子节点的 SQL 片段
 *   2. 如果 SQL 片段非空：
 *      - 去除结尾的 ","
 *      - 添加 "SET " 前缀
 *      - 追加到主 DynamicContext
 */
public class SetSqlNode extends TrimSqlNode {

    public SetSqlNode(Configuration configuration, SqlNode contents) {
        // prefix="SET "         — 添加 SET 前缀
        // suffixOverrides=","   — 去除结尾的逗号
        super(configuration, contents, "SET ", null, null, ",");
    }
}
```

## 5.4 ForEachSqlNode — 循环遍历

```java
/**
 * ForEachSqlNode 对应 <foreach> 标签
 * 源码：org.apache.ibatis.scripting.xmltags.ForEachSqlNode
 *
 * 核心流程：
 *   1. 从 context 中通过 OGNL 获取集合对象（collection 属性指定的）
 *   2. 如果集合为 null 或空 → 直接返回
 *   3. 遍历集合中的每个元素：
 *      a. 将当前元素放入 context 的 bindings 中（命名为 item 属性指定值）
 *      b. 将索引放入 context 的 bindings 中（命名为 index 属性指定值）
 *      c. 对该元素递归应用子 SqlNode（contents）
 *      d. 如果不是最后一个元素，追加 separator 指定的分隔符
 *   4. 将整个循环结果用 open 和 close 包裹
 *
 * 示例：<foreach collection="ids" item="id" separator="," open="(" close=")">
 *         #{id}
 *       </foreach>
 *
 * 遍历过程：
 *   集合 [1, 2, 3]
 *   → open + iteration_1 + separator + iteration_2 + separator + iteration_3 + close
 *   → "(#{id}[item=1],#{id}[item=2],#{id}[item=3])"
 */
public class ForEachSqlNode implements SqlNode {

    private final ExpressionEvaluator evaluator;
    private final String collectionExpression;  // collection 属性值
    private final SqlNode contents;             // 循环体内的 SqlNode
    private final String open;                  // open 属性
    private final String close;                 // close 属性
    private final String separator;             // separator 属性
    private final String item;                  // item 属性（元素变量名）
    private final String index;                 // index 属性（索引变量名）

    @Override
    public boolean apply(DynamicContext context) {
        // 获取参数对象
        Map<String, Object> bindings = context.getBindings();
        final Iterable<?> iterable = evaluator.evaluateIterable(
            collectionExpression, bindings);
        if (!iterable.iterator().hasNext()) {
            return true;  // 空集合，不生成任何内容
        }

        boolean first = true;
        applyOpen(context);  // 追加 open
        int i = 0;
        for (Object o : iterable) {
            DynamicContext oldContext = context;
            if (first || separator == null) {
                context = new PrefixedContext(context, "");
            } else {
                context = new PrefixedContext(context, separator);
            }
            int uniqueNumber = context.getUniqueNumber();
            // 将当前元素和索引放入 bindings
            bindings.put(item, o);
            if (index != null) {
                bindings.put(index, i);
            }
            // 递归执行子节点
            contents.apply(new FilteredDynamicContext(configuration, context, index, item, uniqueNumber));
            if (first) {
                first = !((PrefixedContext) context).isPrefixApplied();
            }
            context = oldContext;
            i++;
        }
        applyClose(context);  // 追加 close
        return true;
    }
}
```

## 5.5 ChooseSqlNode — 多条件选择

```java
/**
 * ChooseSqlNode 对应 <choose><when><otherwise> 标签
 * 源码：org.apache.ibatis.scripting.xmltags.ChooseSqlNode
 *
 * 核心逻辑：
 *   1. 遍历所有 <when> 子节点（whenSqlNodes）
 *   2. 找到第一个条件为 true 的 <when> → 执行其内容
 *   3. 如果所有 <when> 条件都为 false
 *      → 执行 <otherwise> 的内容（defaultSqlNode）
 *      → 如果没有 <otherwise>，什么也不执行
 *
 * 这实现了 if-else if-else 逻辑
 */
public class ChooseSqlNode implements SqlNode {
    private final List<SqlNode> whenSqlNodes;   // <when> 节点列表
    private final SqlNode defaultSqlNode;       // <otherwise> 节点

    @Override
    public boolean apply(DynamicContext context) {
        for (SqlNode sqlNode : whenSqlNodes) {
            if (sqlNode.apply(context)) {
                return true;  // 找到第一个满足条件的 when，停止遍历
            }
        }
        if (defaultSqlNode != null) {
            defaultSqlNode.apply(context);  // 都不满足，执行 otherwise
            return true;
        }
        return false;
    }
}
```

## 5.6 TrimSqlNode — 通用前缀/后缀处理

```java
/**
 * TrimSqlNode — where 和 set 的父类
 * 源码：org.apache.ibatis.scripting.xmltags.TrimSqlNode
 *
 * 核心逻辑（apply() 方法）：
 *   1. 将子节点应用到临时的 FilteredDynamicContext 中
 *   2. 获取拼接后的 SQL 片段
 *   3. 如果 SQL 片段非空：
 *      a. 去除 prefixOverrides 指定的开头字符串
 *      b. 去除 suffixOverrides 指定的结尾字符串
 *      c. 添加 prefix 前缀
 *      d. 添加 suffix 后缀
 *      e. 追加到主 context
 */
public class TrimSqlNode implements SqlNode {

    private final SqlNode contents;
    private final String prefix;            // 添加的前缀
    private final String suffix;            // 添加的后缀
    private final List<String> prefixesToOverride;  // 要去除的前缀
    private final List<String> suffixesToOverride;  // 要去除的后缀

    @Override
    public boolean apply(DynamicContext context) {
        FilteredDynamicContext filteredContext = new FilteredDynamicContext(context);
        boolean result = contents.apply(filteredContext);
        filteredContext.applyAll();
        return result;
    }
}

// FilteredDynamicContext 内部类负责实际的去除和添加逻辑
// 它重写了 appendSql() 方法，通过代理模式拦截 SQL 追加操作
```

# 6.#{} 和 ${} 的底层解析差异

```java
/**
 * #{} 和 ${} 在底层是完全不同的处理方式：

 * #{} 的处理（PreparedStatement 占位符）：
 *   1. 解析阶段：
 *      - 在 SqlSourceBuilder.parse() 中处理
 *      - 使用 GenericTokenParser + ParameterMappingTokenHandler
 *      - 将 #{xxx} 替换为 ?
 *      - 构建 ParameterMapping 对象
 *   2. 执行阶段：
 *      - ParameterHandler 根据 ParameterMapping 设置参数值
 *      - 调用 ps.setXxx(index, value)
 *      - 特性：预编译 SQL，防止 SQL 注入

 * ${} 的处理（字符串拼接）：
 *   1. 解析阶段：
 *      - 在 TextSqlNode.apply() 中处理
 *      - 使用 GenericTokenParser + BindingTokenParser
 *      - 从 bindings 中获取变量值
 *      - 直接将值拼接到 SQL 字符串中
 *   2. 执行阶段：
 *      - 值已经拼接在 SQL 中，无需参数设置
 *      - 特性：直接拼接，存在 SQL 注入风险
 */

// TextSqlNode 中 ${} 的处理：
public class TextSqlNode implements SqlNode {
    private final String text;

    public boolean isDynamic() {
        // 判断文本中是否含有 ${}
        GenericTokenParser parser = new GenericTokenParser("${", "}",
            token -> "");  // 用空串替换来检查
        return !parser.parse(text).equals(text);
    }

    @Override
    public boolean apply(DynamicContext context) {
        // 解析 ${} 占位符
        GenericTokenParser parser = new GenericTokenParser("${", "}",
            token -> {
                // 通过 OGNL 从 bindings 中获取 token 的值
                Object value = OgnlCache.getValue(token, context.getBindings());
                return value == null ? "" : String.valueOf(value);
            }
        );
        // 将替换后的文本直接追加到动态上下文中
        String sql = parser.parse(text);
        context.appendSql(sql);
        return true;
    }
}

// SqlSourceBuilder 中 #{} 的处理：
public class SqlSourceBuilder {

    public SqlSource parse(String originalSql, Class<?> parameterType,
                           Map<String, Object> additionalParameters) {
        ParameterMappingTokenHandler handler =
            new ParameterMappingTokenHandler(configuration, parameterType, additionalParameters);
        GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
        // 将 #{xxx} 替换为 ?
        String sql = parser.parse(originalSql);
        // 返回 StaticSqlSource，持有 ? 替换后的 SQL 和 ParameterMapping 列表
        return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
    }
}
```

# 7.DynamicContext — 动态 SQL 上下文

```java
/**
 * DynamicContext 是 SqlNode 遍历过程中的上下文对象
 * 源码：org.apache.ibatis.scripting.xmltags.DynamicContext
 *
 * 作用：
 *   1. 收集 SQL 片段（通过 StringBuilder 拼接）
 *   2. 存储参数绑定（bindings Map）
 *   3. 生成唯一的编号（用于 foreach 循环中的变量隔离）
 *
 * 核心字段：
 *   - sqlBuilder: StringBuilder，收集拼接的 SQL
 *   - bindings: ContextMap，存储 OGNL 表达式的参数来源
 *     其中：
 *       - "_parameter": 原始参数对象
 *       - "_databaseId": 数据库 ID
 *       - "@Param 指定的参数名": 对应的参数值
 *       - "param1", "param2", ...: 按位置命名的参数
 */
public class DynamicContext {

    // 参数上下文（OGNL 表达式的变量来源）
    private final ContextMap bindings;

    // SQL 字符串构建器
    private final StringBuilder sqlBuilder = new StringBuilder();

    // 唯一编号生成器（用于 foreach 中解决变量名冲突）
    private int uniqueNumber = 0;

    public DynamicContext(Configuration configuration, Object parameterObject) {
        if (parameterObject != null && !(parameterObject instanceof Map)) {
            // 如果参数不是 Map，包装为 MetaObject
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            bindings = new ContextMap(metaObject);
        } else {
            bindings = new ContextMap(null);
        }
        // 固定放入 _parameter 和 _databaseId
        bindings.put(PARAMETER_OBJECT_KEY, parameterObject);  // "_parameter"
        bindings.put(DATABASE_ID_KEY, configuration.getDatabaseId());  // "_databaseId"
    }
}
```

# 8.GenericTokenParser — 通用占位符解析器

```java
/**
 * GenericTokenParser 是 MyBatis 中最关键的工具类之一
 * 源码：org.apache.ibatis.parsing.GenericTokenParser
 *
 * 设计模式：策略模式
 *   - GenericTokenParser：负责查找 openToken 和 closeToken
 *   - TokenHandler：负责处理 token 中的内容
 *
 * 在 MyBatis 中的两个使用场景：
 *   1. 处理 #{} 占位符
 *      - openToken="#{", closeToken="}"
 *      - TokenHandler=ParameterMappingTokenHandler
 *      - 功能：将 #{xxx} 替换为 ?，记录 ParameterMapping
 *
 *   2. 处理 ${} 占位符
 *      - openToken="${", closeToken="}"
 *      - TokenHandler=BindingTokenParser
 *      - 功能：将 ${xxx} 替换为 OGNL 表达式求值结果
 *
 *   3. 解析 properties 中的 ${} (PropertyParser)
 *      - openToken="${", closeToken="}"
 *      - TokenHandler=VariableTokenHandler
 *      - 功能：将 ${jdbc.driver} 替换为 properties 中的值
 *
 * 算法流程（parse() 方法）：
 *   1. 查找第一个 openToken 的位置
 *   2. 如果没有找到 → 返回原字符串
 *   3. 找到后：
 *      a. 将 openToken 之前的内容追加到结果
 *      b. 查找对应的 closeToken 位置
 *      c. 提取 openToken 和 closeToken 之间的 content
 *      d. 调用 handler.handleToken(content) 获取替换内容
 *      e. 将替换内容追加到结果
 *   4. 从 closeToken 之后继续查找下一个 openToken
 *   5. 循环直到没有更多 openToken
 *   6. 追加剩余内容
 */
public class GenericTokenParser {

    private final String openToken;
    private final String closeToken;
    private final TokenHandler handler;

    public GenericTokenParser(String openToken, String closeToken, TokenHandler handler) {
        this.openToken = openToken;
        this.closeToken = closeToken;
        this.handler = handler;
    }

    // parse() 方法的核心算法伪代码：
    public String parse(String text) {
        // ... (参见手写框架章节中的完整实现)
        // 核心思路：交替追加 非token内容 和 handler处理后的内容
    }
}
```

# 9.SqlSource 的创建决策流程

```
XMLScriptBuilder 在解析 SQL 标签时，会根据是否包含动态元素决定创建哪种 SqlSource：

  parseScriptNode()
    │
    ├── 先解析出 SqlNode 树（MixedSqlNode）
    │
    ├── 判断 SqlNode 是否包含动态部分：
    │     - node.isDynamic() 检查是否含有 ${} 或动态标签
    │
    ├── 如果 isDynamic() == true：
    │     → 创建 DynamicSqlSource
    │       new DynamicSqlSource(configuration, rootSqlNode)
    │       （每次执行时都会重新遍历 SqlNode 树并解析 #{}）
    │
    └── 如果 isDynamic() == false：
          → 创建 RawSqlSource
            new RawSqlSource(configuration, rootSqlNode, parameterType)
            （构造时一次性解析 #{} → ?，存入内部 StaticSqlSource）
            RawSqlSource 构造器内部：
              → 先执行 rootSqlNode.apply(dynamicContext) 获取无 #{} 的 SQL
              → 调用 SqlSourceBuilder.parse() 解析 SQL 中的 #{}
                → 将 #{xxx} 替换为 ?
                → 创建 ParameterMapping 列表
              → 创建 StaticSqlSource(sql, parameterMappings)

  为什么需要 RawSqlSource？
    - 性能优化！
    - 对于不含动态标签的 SQL，#{} 解析只需要做一次
    - 以后每次执行只需要设置参数值，不用重复解析

  为什么需要 DynamicSqlSource？
    - 支持动态 SQL
    - 每次执行时 SQL 结构可能不同（if/where/foreach 等影响）
    - 必须每次重新构建
```

# 10.BoundSql — 最终可执行的 SQL

```java
/**
 * BoundSql 持有最终可执行 SQL 的完整信息
 * 源码：org.apache.ibatis.mapping.BoundSql
 *
 * 这是 MyBatis SQL 执行的直接数据来源。
 * 当 Executor 执行 SQL 时，使用的是 BoundSql 中的数据。
 *
 * 核心字段：
 *   - sql: String — 最终的可执行 SQL（#{} 已替换为 ?）
 *   - parameterMappings: List<ParameterMapping> — 参数映射列表
 *     （每个 ParameterMapping 对应一个 ?）
 *   - parameterObject: Object — 原始参数对象
 *   - additionalParameters: Map<String, Object> — 额外参数
 *     （用于 foreach 场景下的动态参数，如 #{item}）
 */
public class BoundSql {

    private final String sql;
    private final List<ParameterMapping> parameterMappings;
    private final Object parameterObject;
    private final Map<String, Object> additionalParameters = new HashMap<>();
    private final MetaObject metaParameters;

    public BoundSql(Configuration configuration, String sql,
                    List<ParameterMapping> parameterMappings, Object parameterObject) {
        this.sql = sql;
        this.parameterMappings = parameterMappings;
        this.parameterObject = parameterObject;
        this.metaParameters = configuration.newMetaObject(additionalParameters);
    }

    // 判断是否有额外的参数（用于 foreach 场景）
    public boolean hasAdditionalParameter(String name) {
        return additionalParameters.containsKey(name);
    }

    public void setAdditionalParameter(String name, Object value) {
        additionalParameters.put(name, value);
    }

    // getter 方法...
}
```

# 11.实例：一条动态 SQL 的完整解析过程

```
以以下 SQL 为例，追踪从 XML 到 BoundSql 的完整流程：

XML 源码：
  <select id="selectByCondition" resultType="Car">
      select * from t_car
      <where>
          <if test="brand != null and brand != ''">
              and brand like concat('%',#{brand},'%')
          </if>
          <if test="guidePrice != null">
              and guide_price > #{guidePrice}
          </if>
          <if test="carType != null and carType != ''">
              and car_type = #{carType}
          </if>
      </where>
      order by id desc
  </select>

  当参数为: {brand: "凯美瑞", guidePrice: null, carType: "燃油车"} 时：

Step 1: XMLScriptBuilder 构建 SqlNode 树
  MixedSqlNode
    ├── StaticTextSqlNode: "select * from t_car "
    ├── WhereSqlNode
    │     └── MixedSqlNode
    │           ├── IfSqlNode(test="brand != null and brand != ''")
    │           │     └── MixedSqlNode
    │           │           └── TextSqlNode("and brand like concat('%',#{brand},'%')")
    │           ├── IfSqlNode(test="guidePrice != null")
    │           │     └── MixedSqlNode
    │           │           └── TextSqlNode("and guide_price > #{guidePrice}")
    │           └── IfSqlNode(test="carType != null and carType != ''")
    │                 └── MixedSqlNode
    │                       └── TextSqlNode("and car_type = #{carType}")
    └── StaticTextSqlNode: " order by id desc"

Step 2: 由于包含动态标签 → 创建 DynamicSqlSource

Step 3: 执行时，DynamicSqlSource.getBoundSql(parameterObject)
  a. 创建 DynamicContext(context, parameterObject)
  b. 遍历 SqlNode 树：
     - "select * from t_car " → appendSql
     - WhereSqlNode:
       - 创建临时的 FilteredDynamicContext
       - IfSqlNode(brand): brand = "凯美瑞" → true
         → appendSql("and brand like concat('%',#{brand},'%')")
       - IfSqlNode(guidePrice): guidePrice = null → false → 跳过
       - IfSqlNode(carType): carType = "燃油车" → true
         → appendSql("and car_type = #{carType}")
       - 临时 SQL: "and brand like concat('%',#{brand},'%') and car_type = #{carType}"
       - 前缀处理：添加 "WHERE"，去除开头的 "and "
       - → "WHERE brand like concat('%',#{brand},'%') and car_type = #{carType}"
     - " order by id desc" → appendSql
  c. 最终拼接的 SQL：
     "select * from t_car WHERE brand like concat('%',#{brand},'%') and car_type = #{carType} order by id desc"

Step 4: SqlSourceBuilder 解析 #{}
  - 找到 #{brand} → 替换为 ?
    参数名: "brand"
  - 找到 #{carType} → 替换为 ?
    参数名: "carType"
  - 最终 SQL: "select * from t_car WHERE brand like concat('%',?, '%') and car_type = ? order by id desc"
  - parameterMappings: [
      ParameterMapping{property='brand'},
      ParameterMapping{property='carType'}
    ]

Step 5: 返回 BoundSql(sql="select * from t_car WHERE brand like concat('%',?, '%') and car_type = ? order by id desc", parameterMappings=[...])
```

# 12.缓存：MapperMethod 中的 SqlCommand

```
在 MapperMethod 中，SqlCommand 的构建中也涉及到 MappedStatement 的查找：

MappedStatement ms = configuration.getMappedStatement(statementId);

当 MappedStatement 不存在时（可能是注解方式），会：
  1. 尝试从接口中查找对应的注解方法
  2. 解析 @Select / @Insert 等注解中的 SQL
  3. 通过 LanguageDriver 创建 SqlSource
  4. 构建新的 MappedStatement 并注册到 Configuration

LanguageDriver 体系：
  LanguageDriver (接口)
    └── XMLLanguageDriver (默认实现，处理 XML 中的 SQL)
          └── RawLanguageDriver (额外限制：不支持动态 SQL 的 XMLLanguageDriver)

  LanguageDriver.createSqlSource() 方法：
    对于 XML 方式（XMLLanguageDriver）：
      → 调用 XMLScriptBuilder.parseScriptNode() → SqlNode 树 → SqlSource
    对于注解方式（XMLLanguageDriver）：
      → 同样调用 XMLScriptBuilder 处理注解中的字符串
      → 支持 <script> 标签来写动态 SQL
```
