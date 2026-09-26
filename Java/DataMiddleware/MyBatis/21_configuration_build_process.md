# 1.配置构建全过程概览

```
MyBatis 启动时最重要的工作就是解析配置文件，构建 Configuration 对象。

整个构建过程分为两大部分：
  1. 解析 mybatis-config.xml（核心配置文件）  → XMLConfigBuilder
  2. 解析每个 XxxMapper.xml（SQL 映射文件）   → XMLMapperBuilder

调用入口：
  SqlSessionFactoryBuilder.build(InputStream)
    → XMLConfigBuilder.parse()
      → 解析 properties、settings、typeAliases、plugins、
        environments、mappers 等一系列配置
      → 对于 mappers 中的每个 XML 文件：
        → XMLMapperBuilder.parse()
          → 解析 namespace、cache、resultMap、select/insert/update/delete
          → 构建 MappedStatement
          → 注册到 Configuration
    → 返回 Configuration
```

# 2.XMLConfigBuilder.parse() 完整解析流程

```java
/**
 * XMLConfigBuilder 是 mybatis-config.xml 的解析器
 * 源码：org.apache.ibatis.builder.xml.XMLConfigBuilder
 *
 * parse() 方法是整个配置解析的入口
 */
public class XMLConfigBuilder extends BaseBuilder {

    private boolean parsed;
    private final XPathParser parser;
    private String environment;
    private final ReflectorFactory localReflectorFactory = new DefaultReflectorFactory();

    /**
     * 解析 mybatis-config.xml
     *
     * @return 完整的 Configuration 对象
     */
    public Configuration parse() {
        if (parsed) {
            throw new BuilderException("Each XMLConfigBuilder can only be used once.");
        }
        parsed = true;

        // ★ 根节点是 <configuration>
        parseConfiguration(parser.evalNode("/configuration"));
        return configuration;
    }

    /**
     * 核心方法：逐个解析 configuration 下的子节点
     *
     * 注意：解析顺序是有严格要求的！
     */
    private void parseConfiguration(XNode root) {
        try {
            // 1. properties（最先解析，因为后面的节点可能引用 properties 中的变量）
            propertiesElement(root.evalNode("properties"));

            // 2. settings（修改 Configuration 中的各种 flag 和属性）
            Properties settings = settingsAsProperties(root.evalNode("settings"));
            loadCustomVfs(settings);
            loadCustomLogImpl(settings);
            settingsElement(settings);

            // 3. typeAliases（注册类型别名）
            typeAliasesElement(root.evalNode("typeAliases"));

            // 4. plugins（注册拦截器/插件）
            pluginElement(root.evalNode("plugins"));

            // 5. objectFactory（自定义对象工厂）
            objectFactoryElement(root.evalNode("objectFactory"));

            // 6. objectWrapperFactory（自定义对象包装工厂）
            objectWrapperElement(root.evalNode("objectWrapperFactory"));

            // 7. reflectorFactory（自定义反射工厂）
            reflectorFactoryElement(root.evalNode("reflectorFactory"));

            // 8. settings 二次回填（处理 settings 中尚未生效的配置）
            settingsElement(settings);

            // 9. environments（环境配置：事务管理器 + 数据源）
            environmentsElement(root.evalNode("environments"));

            // 10. databaseIdProvider（数据库厂商标识支持）
            databaseIdProviderElement(root.evalNode("databaseIdProvider"));

            // 11. typeHandlers（类型处理器注册）
            typeHandlerElement(root.evalNode("typeHandlers"));

            // 12. mappers（最后解析，因为需要前面注册的别名、typeHandler 等）
            mapperElement(root.evalNode("mappers"));

        } catch (Exception e) {
            throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
        }
    }
}
```

# 3.各节点的解析细节

## 3.1 properties 解析

```java
/**
 * 解析 <properties> 节点
 *
 * 属性来源优先级（从高到低）：
 *   1. 通过方法参数传入的 Properties（SqlSessionFactoryBuilder.build() 的第二个参数）
 *   2. <properties resource=""/> 或 <properties url=""/> 加载的外部文件
 *   3. <properties> 标签内直接写的 <property name="" value=""/>
 */
private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
        // 1. 先读取标签体内直接写的 <property>
        Properties defaults = context.getChildrenAsProperties();

        // 2. 再读取 resource 或 url 属性指定的外部文件
        String resource = context.getStringAttribute("resource");
        String url = context.getStringAttribute("url");
        if (resource != null && url != null) {
            throw new BuilderException("The properties element cannot specify both a resource and a url.");
        }
        if (resource != null) {
            defaults.putAll(Resources.getResourceAsProperties(resource));
        } else if (url != null) {
            defaults.putAll(Resources.getUrlAsProperties(url));
        }

        // 3. 合并到 Configuration 的 variables 中
        Properties vars = configuration.getVariables();
        if (vars != null) {
            defaults.putAll(vars);
        }
        // 4. 将合并后的 Properties 存入 Configuration
        parser.setVariables(defaults);
        configuration.setVariables(defaults);
    }
}
```

## 3.2 settings 解析

```java
/**
 * 解析 <settings> 节点，将设置应用到 Configuration
 *
 * MyBatis 支持的所有 settings（部分列举）：
 *   - cacheEnabled                    二级缓存开关（默认 true）
 *   - lazyLoadingEnabled              延迟加载开关（默认 false）
 *   - aggressiveLazyLoading           激进延迟加载（默认 false，3.4.1+ 改为 false）
 *   - multipleResultSetsEnabled        允许多结果集（默认 true）
 *   - useColumnLabel                  使用列标签代替列名（默认 true）
 *   - useGeneratedKeys                 允许 JDBC 自动生成主键（默认 false）
 *   - autoMappingBehavior             自动映射行为（默认 PARTIAL: NONE/PARTIAL/FULL）
 *   - autoMappingUnknownColumnBehavior 遇到未知列行为（默认 NONE: NONE/WARNING/FAILING）
 *   - defaultExecutorType             默认执行器类型（默认 SIMPLE: SIMPLE/REUSE/BATCH）
 *   - defaultStatementTimeout          默认超时秒数（默认 0，不限制）
 *   - defaultFetchSize                默认 fetchSize（未设置）
 *   - defaultResultSetType            默认 ResultSet 类型（未设置）
 *   - safeRowBoundsEnabled             安全 RowBounds（默认 false）
 *   - safeResultHandlerEnabled         安全 ResultHandler（默认 true）
 *   - mapUnderscoreToCamelCase         驼峰命名自动映射（默认 false）
 *   - localCacheScope                  一级缓存范围（默认 SESSION: SESSION/STATEMENT）
 *   - jdbcTypeForNull                  空值的 JDBC 类型（默认 OTHER）
 *   - lazyLoadTriggerMethods           触发延迟加载的方法（默认 equals,clone,hashCode,toString）
 *   - defaultScriptingLanguage         默认脚本语言（默认 XMLLanguageDriver）
 *   - defaultEnumTypeHandler           默认枚举处理器（默认 EnumTypeHandler）
 *   - returnInstanceForEmptyRow        空行返回空实例（默认 false）
 *   - logPrefix                        日志前缀
 *   - logImpl                          日志实现
 *   - proxyFactory                    代理工厂（默认 JAVASSIST: JAVASSIST/CGLIB）
 *   - vfsImpl                          VFS 实现
 *   - useActualParamName               使用实际参数名（默认 true，Java 8+）
 *   - configurationFactory             配置工厂
 *   - shrinkWhitespacesInSql           压缩 SQL 空白（默认 false）
 *   - defaultSqlProviderType           默认的 SqlProvider 类型
 *   - nullableOnForEach                foreach 中 null 集合的处理（默认 true）
 */
private void settingsElement(Properties props) {
    // 通过元编程方式设置 Configuration 的属性
    // 例如：props 中有 "cacheEnabled=true"
    //   → MetaObject.forObject(configuration).setValue("cacheEnabled", true)

    configuration.setAutoMappingBehavior(
        AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(
        AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
    configuration.setProxyFactory(
        (ProxyFactory) createInstance(props.getProperty("proxyFactory")));
    configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
    configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
    configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
    configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
    configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
    configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
    configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
    configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
    configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
    configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
    configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
    configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
    configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
    configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
    configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
    configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    // ... 更多设置
}
```

## 3.3 typeAliases 解析

```java
/**
 * 解析 <typeAliases> 节点
 *
 * 两种方式：
 *   1. <typeAlias type="com.xxx.pojo.Car" alias="car"/>
 *   2. <package name="com.xxx.pojo"/>  → 批量注册，别名默认为类名首字母小写
 *
 * MyBatis 内置的类型别名（在 Configuration 的构造函数中注册）：
 *   string    → String
 *   byte      → Byte
 *   long      → Long
 *   short     → Short
 *   int       → Integer
 *   integer   → Integer
 *   double    → Double
 *   float     → Float
 *   boolean   → Boolean
 *   date      → Date
 *   decimal   → BigDecimal
 *   bigdecimal→ BigDecimal
 *   object    → Object
 *   map       → Map
 *   hashmap   → HashMap
 *   list      → List
 *   arraylist → ArrayList
 *   collection→ Collection
 *   iterator  → Iterator
 *   cursor    → Cursor
 *   （以及各种基本类型和数组类型）
 */
private void typeAliasesElement(XNode parent) {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                // 方式2：批量扫描
                String packageName = child.getStringAttribute("name");
                configuration.getTypeAliasRegistry().registerAliases(packageName);
            } else {
                // 方式1：逐个注册
                String alias = child.getStringAttribute("alias");
                String type = child.getStringAttribute("type");
                try {
                    Class<?> clazz = Resources.classForName(type);
                    if (alias == null) {
                        // 如果没有指定 alias，使用 @Alias 注解的值或类名
                        typeAliasRegistry.registerAlias(clazz);
                    } else {
                        typeAliasRegistry.registerAlias(alias, clazz);
                    }
                } catch (ClassNotFoundException e) {
                    throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
                }
            }
        }
    }
}
```

## 3.4 mappers 解析

```java
/**
 * 解析 <mappers> 节点
 *
 * 四种指定方式：
 *   1. <mapper resource=""/>   → 从类路径加载 XML 文件
 *   2. <mapper url=""/>        → 从 URL 加载 XML 文件
 *   3. <mapper class=""/>      → 加载 Mapper 接口（XML 和 注解都解析）
 *   4. <package name=""/>      → 批量扫描包下的 Mapper 接口
 */
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            if ("package".equals(child.getName())) {
                // 方式4：批量扫描
                String mapperPackage = child.getStringAttribute("name");
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");

                // 方式1：resource
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
                        XMLMapperBuilder mapperParser = new XMLMapperBuilder(
                            inputStream, configuration, resource,
                            configuration.getSqlFragments());
                        mapperParser.parse();
                    }
                }
                // 方式2：url
                else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    try (InputStream inputStream = Resources.getUrlAsStream(url)) {
                        XMLMapperBuilder mapperParser = new XMLMapperBuilder(
                            inputStream, configuration, url,
                            configuration.getSqlFragments());
                        mapperParser.parse();
                    }
                }
                // 方式3：class
                else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                }
                else {
                    throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```

# 4.XMLMapperBuilder — SQL 映射文件解析

```java
/**
 * XMLMapperBuilder 负责解析 XxxMapper.xml 文件
 * 源码：org.apache.ibatis.builder.xml.XMLMapperBuilder
 *
 * parse() 解析流程：
 *   1. 检查是否已解析过（防止重复解析）
 *   2. 解析 <cache> — 二级缓存配置
 *   3. 解析 <cache-ref> — 缓存引用
 *   4. 解析 <resultMap> — 结果映射
 *   5. 解析 <sql> — SQL 片段
 *   6. 解析 <select|insert|update|delete> — 构建 MappedStatement
 *   7. 解析完成的 MappedStatement 注册到 Configuration
 */
public class XMLMapperBuilder extends BaseBuilder {

    private final XPathParser parser;
    private final MapperBuilderAssistant builderAssistant;

    /** 存放 <sql> 标签定义的 SQL 片段 */
    private final Map<String, XNode> sqlFragments;

    private final String resource;

    public void parse() {
        // ★ 1. 检查是否已解析 ★
        if (!configuration.isResourceLoaded(resource)) {
            // ★ 2. 解析 <mapper> 根标签 ★
            configurationElement(parser.evalNode("/mapper"));
            // ★ 3. 标记已加载，防止重复 ★
            configuration.addLoadedResource(resource);
            // ★ 4. 注册 Mapper 接口到 MapperRegistry ★
            bindMapperForNamespace();
        }

        // ★ 5. 解析尚未解析的 ResultMap（处理延迟解析的情况）★
        parsePendingResultMaps();
        parsePendingCacheRefs();
        parsePendingStatements();
    }

    private void configurationElement(XNode context) {
        try {
            // 1. 获取 namespace
            String namespace = context.getStringAttribute("namespace");
            if (namespace == null || namespace.isEmpty()) {
                throw new BuilderException("Mapper's namespace cannot be empty");
            }
            builderAssistant.setCurrentNamespace(namespace);

            // 2. 解析 <cache-ref>
            cacheRefElement(context.evalNode("cache-ref"));

            // 3. 解析 <cache>
            cacheElement(context.evalNode("cache"));

            // 4. 解析 <parameterMap>（已废弃，保留兼容）
            parameterMapElement(context.evalNodes("/mapper/parameterMap"));

            // 5. 解析 <resultMap>
            resultMapElements(context.evalNodes("/mapper/resultMap"));

            // 6. 解析 <sql>（SQL 片段）
            sqlElement(context.evalNodes("/mapper/sql"));

            // 7. 解析 <select|insert|update|delete> — 构建 MappedStatement
            buildStatementFromContext(context.evalNodes("select|insert|update|delete"));

        } catch (Exception e) {
            throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
        }
    }
}
```

# 5.XMLStatementBuilder — 单条 SQL 语句的解析

```java
/**
 * XMLStatementBuilder 负责解析 XML 中的单个 select/insert/update/delete 标签
 * 源码：org.apache.ibatis.builder.xml.XMLStatementBuilder
 *
 * 核心流程：
 *   1. 解析 id、parameterType、resultType、resultMap 等属性
 *   2. 解析子节点构建 SqlNode 树（处理动态 SQL 标签）
 *   3. 通过 LanguageDriver 创建 SqlSource
 *   4. 通过 MapperBuilderAssistant 构建 MappedStatement
 *   5. 注册到 Configuration
 */
public class XMLStatementBuilder extends BaseBuilder {

    private final MapperBuilderAssistant builderAssistant;
    private final XNode context;
    private final String requiredDatabaseId;

    public void parseStatementNode() {
        // 1. 获取基础属性
        String id = context.getStringAttribute("id");
        String databaseId = context.getStringAttribute("databaseId");

        // 2. 检查 databaseId 是否匹配
        if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
            return;
        }

        // 3. 获取各种属性
        String nodeName = context.getNode().getNodeName();
        SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
        boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
        boolean useCache = context.getBooleanAttribute("useCache", isSelect);
        boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

        // 解析 <include>（SQL 片段引用）
        XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
        includeParser.applyIncludes(context.getNode());

        // 解析参数和结果类型
        String parameterType = context.getStringAttribute("parameterType");
        Class<?> parameterTypeClass = resolveClass(parameterType);
        String lang = context.getStringAttribute("lang");
        LanguageDriver langDriver = getLanguageDriver(lang);

        // 解析 <selectKey>
        processSelectKeyNodes(id, parameterTypeClass, langDriver);

        // ★ 4. 核心：解析 SQL → SqlSource ★
        KeyGenerator keyGenerator;
        String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
        keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
        if (configuration.hasKeyGenerator(keyStatementId)) {
            keyGenerator = configuration.getKeyGenerator(keyStatementId);
        } else {
            keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
                configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
                ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
        }

        // ★ 使用 LanguageDriver 创建 SqlSource ★
        SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);

        // 5. 设置 StatementType
        StatementType statementType = StatementType.valueOf(
            context.getStringAttribute("statementType", StatementType.PREPARED.toString()));

        // 6. 设置 fetchSize / timeout
        Integer fetchSize = context.getIntAttribute("fetchSize");
        Integer timeout = context.getIntAttribute("timeout");
        String parameterMap = context.getStringAttribute("parameterMap");
        String resultType = context.getStringAttribute("resultType");
        Class<?> resultTypeClass = resolveClass(resultType);
        String resultMap = context.getStringAttribute("resultMap");
        if (resultTypeClass == null && resultMap == null) {
            // 对于 insert/update/delete，resultType 不是必须的，给一个默认值
            resultTypeClass = MapperBuilderAssistant.getDefaultResultType();
        }
        String resultSetType = context.getStringAttribute("resultSetType");
        ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);

        // 7. 如果是 SELECT，且没有 resultMap，尝试自动生成 ResultMap
        String keyProperty = context.getStringAttribute("keyProperty");
        String keyColumn = context.getStringAttribute("keyColumn");

        // ★ 8. 通过 MapperBuilderAssistant 构建 MappedStatement 并注册 ★
        builderAssistant.addMappedStatement(
            id, sqlSource, statementType, sqlCommandType,
            fetchSize, timeout, parameterMap,
            parameterTypeClass, resultMap, resultTypeClass,
            resultSetTypeEnum, flushCache, useCache, resultOrdered,
            keyGenerator, keyProperty, keyColumn,
            databaseId, langDriver, null);
    }
}
```

# 6.MapperBuilderAssistant — 构建助手

```java
/**
 * MapperBuilderAssistant 是 XMLMapperBuilder 的辅助类
 * 源码：org.apache.ibatis.builder.MapperBuilderAssistant
 *
 * 负责：
 *   1. 管理 namespace 前缀
 *   2. 构建和注册 MappedStatement
 *   3. 构建和注册 ResultMap
 *   4. 构建和注册 Cache
 *   5. 参数类型转换
 */
public class MapperBuilderAssistant extends BaseBuilder {

    private String currentNamespace;
    private String resource;
    private Cache currentCache;
    private boolean unresolvedCacheRef;

    /**
     * 构建并注册 MappedStatement 到 Configuration
     *
     * 注意：MappedStatement 是不可变对象，通过 Builder 模式构建
     */
    public MappedStatement addMappedStatement(
            String id, SqlSource sqlSource,
            StatementType statementType, SqlCommandType sqlCommandType,
            Integer fetchSize, Integer timeout, String parameterMap,
            Class<?> parameterType, String resultMap, Class<?> resultType,
            ResultSetType resultSetType, boolean flushCache, boolean useCache,
            boolean resultOrdered, KeyGenerator keyGenerator, String keyProperty,
            String keyColumn, String databaseId,
            LanguageDriver lang, String resultSets) {

        // 1. 构建完整的 statementId = namespace + "." + id
        id = applyCurrentNamespace(id, false);

        // 2. 复用 ResultMap（如果 namespace 不同但 ResultMap 结构相同）
        if (resultMap != null) {
            resultMap = applyCurrentNamespace(resultMap, true);
        }

        MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
            .resource(resource)
            .fetchSize(fetchSize)
            .timeout(timeout)
            .statementType(statementType)
            .keyGenerator(keyGenerator)
            .keyProperty(keyProperty)
            .keyColumn(keyColumn)
            .databaseId(databaseId)
            .lang(lang)
            .resultOrdered(resultOrdered)
            .resultSets(resultSets)
            .resultMaps(getStatementResultMaps(resultMap, resultType, id))
            .resultSetType(resultSetType)
            .flushCacheRequired(flushCache)
            .useCache(useCache)
            .cache(currentCache);

        // 3. 设置 ParameterMap
        ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
        if (statementParameterMap != null) {
            statementBuilder.parameterMap(statementParameterMap);
        }

        // 4. 构建 MappedStatement
        MappedStatement statement = statementBuilder.build();

        // 5. ★ 注册到 Configuration ★
        configuration.addMappedStatement(statement);

        return statement;
    }
}
```

# 7.Configuration 对象结构

```java
/**
 * Configuration 是 MyBatis 的"心脏"
 * 源码：org.apache.ibatis.session.Configuration
 *
 * 它包含了 MyBatis 运行时所需的所有信息
 */
public class Configuration {

    // ============ 环境相关 ============
    protected Environment environment;              // 环境（含数据源和事务工厂）

    // ============ SQL 映射相关 ============
    protected final Map<String, MappedStatement> mappedStatements = new StrictMap<>("Mapped Statements collection");
    protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
    protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
    protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");

    // ============ Mapper 注册 ============
    protected final MapperRegistry mapperRegistry = new MapperRegistry(this);

    // ============ 拦截器 ============
    protected final InterceptorChain interceptorChain = new InterceptorChain();

    // ============ 类型系统 ============
    protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry(this);
    protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();

    // ============ 其他工厂和组件 ============
    protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();
    protected ObjectFactory objectFactory = new DefaultObjectFactory();
    protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();
    protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
    protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // 或 CglibProxyFactory

    // ============ 设置项 ============
    protected boolean cacheEnabled = true;
    protected boolean lazyLoadingEnabled = false;
    protected boolean aggressiveLazyLoading = false;       // 3.4.1+ 默认 false
    protected boolean multipleResultSetsEnabled = true;
    protected boolean useGeneratedKeys = false;
    protected boolean useColumnLabel = true;
    protected boolean useActualParamName = true;           // 3.4.1+
    protected boolean mapUnderscoreToCamelCase = false;
    protected boolean safeRowBoundsEnabled = false;
    protected boolean safeResultHandlerEnabled = true;
    protected boolean callSettersOnNulls = false;
    protected boolean returnInstanceForEmptyRow = false;
    protected boolean shrinkWhitespacesInSql = false;      // 3.5.5+
    protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;
    protected AutoMappingUnknownColumnBehavior autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;
    protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
    protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
    protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
    protected Integer defaultStatementTimeout;
    protected Integer defaultFetchSize;
    protected ResultSetType defaultResultSetType;
    protected Class<? extends LanguageDriver> defaultScriptingLanguage;
    protected Class<? extends TypeHandler> defaultEnumTypeHandler;
    protected Set<String> lazyLoadTriggerMethods = new HashSet<>(Arrays.asList("equals", "clone", "hashCode", "toString"));
    protected String logPrefix;
    protected Class<? extends Log> logImpl;
    protected Class<? extends VFS> vfsImpl;
    protected Class<?> configurationFactory;

    // ============ 已加载资源跟踪 ============
    protected final Set<String> loadedResources = new HashSet<>();
    protected String databaseId;

    // ============ Properties ============
    protected Properties variables = new Properties();

    // ============ 核心方法 ============

    /** 添加 MappedStatement */
    public void addMappedStatement(MappedStatement ms) {
        mappedStatements.put(ms.getId(), ms);
    }

    /** 获取 MappedStatement */
    public MappedStatement getMappedStatement(String id) {
        return this.getMappedStatement(id, true);
    }

    /** 创建 Executor */
    public Executor newExecutor(Transaction transaction) {
        return newExecutor(transaction, defaultExecutorType);
    }

    public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
        executorType = executorType == null ? defaultExecutorType : executorType;
        Executor executor;
        if (ExecutorType.BATCH == executorType) {
            executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
            executor = new ReuseExecutor(this, transaction);
        } else {
            executor = new SimpleExecutor(this, transaction);
        }
        if (cacheEnabled) {
            executor = new CachingExecutor(executor);
        }
        executor = (Executor) interceptorChain.pluginAll(executor);
        return executor;
    }
}
```

# 8.构建过程时序总结

```
┌─────────────┐    ┌─────────────────┐    ┌─────────────────────┐    ┌──────────────┐
│  Spring/用户  │    │SqlSessionFactory │    │   XMLConfigBuilder    │    │Configuration │
│              │    │    Builder        │    │                       │    │              │
└──────┬───────┘    └────────┬────────┘    └──────────┬────────────┘    └──────┬───────┘
       │                     │                        │                       │
       │ build(is)           │                        │                       │
       │────────────────────>│                        │                       │
       │                     │                        │                       │
       │                     │ new XMLConfigBuilder() │                       │
       │                     │───────────────────────>│                       │
       │                     │                        │                       │
       │                     │      parse()           │                       │
       │                     │───────────────────────>│                       │
       │                     │                        │                       │
       │                     │                        │ parseConfiguration()  │
       │                     │                        │──────────┐            │
       │                     │                        │<─────────┘            │
       │                     │                        │                       │
       │                     │                        │  parse properties      │
       │                     │                        │  parse settings        │
       │                     │                        │  parse typeAliases     │
       │                     │                        │  parse plugins         │
       │                     │                        │  parse environments    │  setEnvironment()
       │                     │                        │────────────────────────────────────────>
       │                     │                        │                       │  setDataSource()
       │                     │                        │                       │
       │                     │                        │  parse mappers        │
       │                     │                        │──────────┐            │
       │                     │                        │<─────────┘            │
       │                     │                        │                       │
       │                     │                        │  for each mapper:     │
       │                     │                        │   new XMLMapperBuilder│
       │                     │                        │   mapperParser.parse()│
       │                     │                        │     ──> 解析 cache     │
       │                     │                        │     ──> 解析 resultMap │
       │                     │                        │     ──> 解析 sql片段   │
       │                     │                        │     ──> 解析每条 SQL   │
       │                     │                        │       ├─> 构建 SqlNode  │
       │                     │                        │       ├─> 创建 SqlSource│
       │                     │                        │       └─> 构建 MpStmt  │  addMappedStatement()
       │                     │                        │────────────────────────────────────────>
       │                     │                        │                       │
       │                     │    return Configuration│                       │
       │                     │<───────────────────────│                       │
       │                     │                        │                       │
       │                     │ build(config)          │                       │
       │                     │───────────────────────────────────────────────>
       │                     │                        │    新的                │
       │                     │                        │  DefaultSqlSessionFactory│
       │                     │<───────────────────────────────────────────────
       │                     │                        │                       │
       │ return factory      │                        │                       │
       │<────────────────────│                        │                       │
       │                     │                        │                       │
```
