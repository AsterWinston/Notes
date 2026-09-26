
# 1. ES Adapter 概述

ES Adapter 将 MySQL 的数据变更实时同步到 Elasticsearch，常用于构建准实时的搜索引擎、日志分析平台等。

```
MySQL (源) ──> Canal Server ──> Adapter ES ──> Elasticsearch
                                                    │
                                              ┌─────▼─────┐
                                              │  Kibana   │
                                              │  搜索服务  │
                                              └───────────┘
```

# 2. 配置详解

## 2.1 application.yml 配置

```yaml
server:
  port: 8081

canal.conf:
  mode: tcp
  canalServerHost: 127.0.0.1:11111
  
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://127.0.0.1:3306/source_db?useUnicode=true
      username: canal
      password: canal_password
  
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: es                  # ES 适配器
              hosts: 127.0.0.1:9200     # ES 集群地址
              properties:
                # ES 连接属性
                cluster.name: elasticsearch
                mode: rest              # rest / transport
                # transport 模式（旧版）：
                # cluster.name: my-cluster
                # client.transport.sniff: true
                
                # 安全认证
                security.auth: admin:admin123
                
                # 写入配置
                es.batch.size: 1000     # 批量写入大小
                es.batch.timeout: 5000  # 批量写入超时（ms）
```

## 2.2 ES 表映射配置 (es/xxx.yml)

```yaml
# conf/es/product.yml

# 基础配置
dataSourceKey: defaultDS
destination: example
groupId: g1
esMapping:
  # 索引配置
  _index: product_index                         # ES 索引名
  _id: _id                                      # ES 文档 ID（默认用源表主键）
  
  # SQL 配置（用于全量同步时的多表关联查询）
  sql: "select p.id, p.name, p.price, p.status, 
                c.name as category_name
         from product p
         left join category c on p.category_id = c.id"
  
  # 源表更新条件（增量同步时用到）
  # etlCondition: "where status != 'deleted'"
  
  # 字段映射
  _type: _doc                                   # ES type（ES 7.x+ 已废弃）
  upsert: true                                  # 不存在时执行 upsert（即 INSERT）
  
  # 关系型数据中的一对多处理
  # 如果 sql 是多表 join，需要在下面的父子关系中声明
  
  # 字段映射关系
  mappings:
    - column: id                 # SQL 查询结果的列名
      field: product_id          # ES 中的字段名
    - column: name
      field: product_name
    - column: price
      field: unit_price
      type: double               # ES 字段类型
    - column: status
      field: product_status
    - column: category_name
      field: category
      
  # 提交批次
  commitBatch: 3000
  
  # ETL 全量同步条件
  # etlCondition: "where update_time > '2024-01-01'"
```

## 2.3 简单映射配置（单表）

```yaml
# conf/es/user_simple.yml
dataSourceKey: defaultDS
destination: example
groupId: g1
esMapping:
  _index: user_index
  _id: id
  sql: "select id, name, email, status, created_at from users"
  upsert: true
  commitBatch: 3000
```

# 3. 同步策略

## 3.1 INSERT 同步 → ES 索引

```java
public class ESSyncService {
    
    /**
     * 同步 INSERT 到 ES
     */
    public void syncInsert(ESSyncConfig config, Dml dml, 
                            CanalEntry.Entry entry) {
        
        if (dml.getData() == null || dml.getData().isEmpty()) {
            return;
        }
        
        // 1. 构建批量索引操作
        BulkRequest bulkRequest = new BulkRequest();
        
        for (Map<String, String> row : dml.getData()) {
            String docId = getDocId(config, row);  // 获取文档 ID
            
            // 构建 ES 文档
            Map<String, Object> esDoc = convertToESDoc(config, row);
            
            // 构建 Index 请求（upsert）
            IndexRequest indexRequest = new IndexRequest(config.getIndex())
                .id(docId)
                .source(esDoc, XContentType.JSON);
            
            bulkRequest.add(indexRequest);
        }
        
        // 2. 批量执行
        if (bulkRequest.numberOfActions() > 0) {
            BulkResponse response = restHighLevelClient.bulk(
                bulkRequest, RequestOptions.DEFAULT);
            
            if (response.hasFailures()) {
                logger.error("es insert has failures: {}", response.buildFailureMessage());
                throw new ESSyncException(response.buildFailureMessage());
            }
            
            logger.debug("es insert success: {} docs", bulkRequest.numberOfActions());
        }
    }
    
    /**
     * 将 DML 行数据转换为 ES 文档
     */
    private Map<String, Object> convertToESDoc(ESSyncConfig config, 
                                                Map<String, String> row) {
        Map<String, Object> doc = new HashMap<>();
        
        for (ESSyncConfig.FieldMapping mapping : config.getMappings()) {
            String column = mapping.getColumn();
            String field = mapping.getField();
            String value = row.get(column);
            
            // 类型转换
            Object convertedValue = convertType(mapping, value);
            doc.put(field, convertedValue);
        }
        
        return doc;
    }
    
    /**
     * 类型转换
     */
    private Object convertType(ESSyncConfig.FieldMapping mapping, String value) {
        if (value == null) return null;
        
        switch (mapping.getType()) {
            case "integer":
                return Integer.parseInt(value);
            case "long":
                return Long.parseLong(value);
            case "double":
            case "float":
                return Double.parseDouble(value);
            case "boolean":
                return Boolean.parseBoolean(value) || "1".equals(value);
            case "date":
                return value;  // ES 接受 ISO 格式的日期字符串
            case "nested":
                return parseNested(mapping, value);
            case "object":
                return value;  // JSON 字符串
            default:
                return value;  // string
        }
    }
}
```

## 3.2 UPDATE 同步 → ES 更新

```java
public void syncUpdate(ESSyncConfig config, Dml dml, CanalEntry.Entry entry) {
    
    if (dml.getData() == null || dml.getData().isEmpty()) {
        return;
    }
    
    BulkRequest bulkRequest = new BulkRequest();
    
    for (int i = 0; i < dml.getData().size(); i++) {
        Map<String, String> newRow = dml.getData().get(i);
        Map<String, String> oldRow = dml.getOld().get(i);
        
        String docId = getDocId(config, newRow);
        
        // 构建 ES 文档
        Map<String, Object> esDoc = convertToESDoc(config, newRow);
        
        if (config.isUpsert()) {
            // 使用 IndexRequest（upsert 模式）
            IndexRequest indexRequest = new IndexRequest(config.getIndex())
                .id(docId)
                .source(esDoc, XContentType.JSON);
            bulkRequest.add(indexRequest);
        } else {
            // 使用 UpdateRequest（部分更新）
            UpdateRequest updateRequest = new UpdateRequest(
                config.getIndex(), docId)
                .doc(esDoc, XContentType.JSON)
                .docAsUpsert(false);
            bulkRequest.add(updateRequest);
        }
    }
    
    if (bulkRequest.numberOfActions() > 0) {
        BulkResponse response = restHighLevelClient.bulk(
            bulkRequest, RequestOptions.DEFAULT);
        
        if (response.hasFailures()) {
            logger.error("es update has failures: {}", response.buildFailureMessage());
            throw new ESSyncException(response.buildFailureMessage());
        }
    }
}
```

## 3.3 DELETE 同步 → ES 删除

```java
public void syncDelete(ESSyncConfig config, Dml dml, CanalEntry.Entry entry) {
    
    if (dml.getData() == null || dml.getData().isEmpty()) {
        return;
    }
    
    BulkRequest bulkRequest = new BulkRequest();
    
    for (Map<String, String> row : dml.getData()) {
        String docId = getDocId(config, row);
        
        DeleteRequest deleteRequest = new DeleteRequest(config.getIndex(), docId);
        bulkRequest.add(deleteRequest);
    }
    
    if (bulkRequest.numberOfActions() > 0) {
        BulkResponse response = restHighLevelClient.bulk(
            bulkRequest, RequestOptions.DEFAULT);
        
        if (response.hasFailures()) {
            logger.error("es delete has failures: {}", response.buildFailureMessage());
            throw new ESSyncException(response.buildFailureMessage());
        }
        
        logger.debug("es delete success: {} docs", bulkRequest.numberOfActions());
    }
}
```

# 4. 多表关联同步

## 4.1 1:1 关联

```yaml
# 用户表和用户扩展表（1:1）合并到同一 ES 文档
esMapping:
  _index: user_full_index
  _id: u.id
  sql: >
    SELECT u.id, u.name, u.email,
           e.phone, e.address, e.avatar
    FROM users u
    LEFT JOIN user_extend e ON u.id = e.user_id
```

## 4.2 1:N 关联（嵌套文档）

```yaml
# 用户和订单（1:N）使用 ES 的 nested 类型
esMapping:
  _index: user_order_index
  _id: u.id
  # 主表 SQL
  sql: "SELECT u.id, u.name, u.email FROM users u"
  
  # 子表配置
  etlCondition: "where u.id > 0"
  commitBatch: 3000
  upsert: true
  
  # 嵌套关系
  relations:
    - name: orders              # 嵌套字段名
      type: nested             # nested 类型
      parent: id               # 父文档标识列
      sql: "SELECT o.id, o.order_no, o.amount, o.user_id FROM orders o"
      mappings:
        - column: id
          field: order_id
        - column: order_no
          field: order_no
        - column: amount
          field: amount
```

# 5. 自定义索引模板

## 5.1 预定义 Mapping 和 Setting

```yaml
# conf/es/article.yml
dataSourceKey: defaultDS
destination: example
groupId: g1
esMapping:
  _index: article_index
  _id: id
  sql: "select id, title, content, author, publish_time, tags from articles"
  upsert: true
  
  # 索引设置（创建索引时使用）
  _settings:
    number_of_shards: 3
    number_of_replicas: 1
    refresh_interval: "5s"
    analysis:
      analyzer:
        ik_max_word_analyzer:
          type: custom
          tokenizer: ik_max_word
        ik_smart_analyzer:
          type: custom
          tokenizer: ik_smart
  
  # 字段 Mapping
  _mapping:
    properties:
      id:
        type: long
      title:
        type: text
        analyzer: ik_max_word
      content:
        type: text
        analyzer: ik_max_word
      author:
        type: keyword
      publish_time:
        type: date
        format: "yyyy-MM-dd HH:mm:ss"
      tags:
        type: keyword
```

## 5.2 模板代码实现

```java
public class ESTemplateManager {
    
    /**
     * 创建或更新索引模板
     */
    public void applyTemplate(ESSyncConfig config) throws IOException {
        
        String indexName = config.getIndex();
        
        // 检查索引是否存在
        GetIndexRequest getRequest = new GetIndexRequest(indexName);
        boolean exists = restHighLevelClient.indices()
            .exists(getRequest, RequestOptions.DEFAULT);
        
        if (!exists) {
            CreateIndexRequest createRequest = new CreateIndexRequest(indexName);
            
            // 设置 Settings
            if (config.getSettings() != null) {
                Settings settings = Settings.builder()
                    .loadFromMap(config.getSettings())
                    .build();
                createRequest.settings(settings);
            }
            
            // 设置 Mapping
            if (config.getMapping() != null) {
                String mappingJson = JsonUtils.toJson(config.getMapping());
                createRequest.source(mappingJson, XContentType.JSON);
            }
            
            restHighLevelClient.indices()
                .create(createRequest, RequestOptions.DEFAULT);
            
            logger.info("created index: {}", indexName);
        }
    }
}
```

# 6. 同步性能优化

## 6.1 ES 批量写入优化

```java
public class ESBatchOptimizer {
    
    private static final int MAX_BULK_SIZE = 1000;     // 单次批量最大条数
    private static final int MAX_BULK_BYTES = 5 * 1024 * 1024;  // 5MB
    
    /**
     * 优化后的批量写入
     */
    public void optimizedBulk(List<IndexRequest> requests) {
        
        int currentSize = 0;
        List<IndexRequest> currentBatch = new ArrayList<>(MAX_BULK_SIZE);
        
        for (IndexRequest request : requests) {
            int docSize = estimateDocSize(request);
            
            if (currentBatch.size() >= MAX_BULK_SIZE
                    || currentSize + docSize > MAX_BULK_BYTES) {
                // 提交当前批次
                executeBulk(currentBatch);
                currentBatch.clear();
                currentSize = 0;
            }
            
            currentBatch.add(request);
            currentSize += docSize;
        }
        
        // 提交最后一个批次
        if (!currentBatch.isEmpty()) {
            executeBulk(currentBatch);
        }
    }
    
    private void executeBulk(List<IndexRequest> batch) {
        BulkRequest bulkRequest = new BulkRequest();
        batch.forEach(bulkRequest::add);
        
        // 设置等待活跃分片数（性能优化：不需要等待所有副本）
        bulkRequest.waitForActiveShards(1);
        
        BulkResponse response = restHighLevelClient.bulk(
            bulkRequest, RequestOptions.DEFAULT);
        
        // 处理部分失败
        if (response.hasFailures()) {
            handlePartialFailure(response);
        }
    }
}
```

## 6.2 ES 连接池配置

```yaml
# application.yml
canal.conf:
  canalAdapters:
    - instance: example
      groups:
        - groupId: g1
          outerAdapters:
            - name: es
              hosts: 192.168.1.50:9200,192.168.1.51:9200,192.168.1.52:9200
              properties:
                rest.host: 192.168.1.50,192.168.1.51,192.168.1.52
                rest.port: 9200
                rest.scheme: http
                rest.connect.timeout: 5000
                rest.socket.timeout: 60000
                rest.connection.request.timeout: 10000
                rest.max.conn.total: 100
                rest.max.conn.per.route: 50
```

# 7. 常见问题

## 7.1 数据一致性保障

```java
// ES 写入是近实时的，需要保证数据能被及时检索到
public void ensureVisibility(BulkResponse response) {
    // 执行 refresh 操作（生产环境不建议高频 refresh）
    // restHighLevelClient.indices().refresh(
    //     new RefreshRequest(indexName), RequestOptions.DEFAULT);
    
    // 更好的方案：使用 ES 默认的 refresh_interval (1s)
    // 或者将 refresh_interval 设置为 -1（手动控制）
}
```

## 7.2 幂等性处理

```java
// ES 的 IndexRequest 天然支持幂等性：
// - 相同 doc ID 的 Index 操作会覆盖旧版本
// - 利用 _version 或 _seq_no + _primary_term 实现乐观锁
public void idempotentUpdate(ESSyncConfig config, Map<String, String> row) {
    
    IndexRequest request = new IndexRequest(config.getIndex())
        .id(getDocId(config, row))
        .source(convertToESDoc(config, row), XContentType.JSON);
    
    // 设置版本号（乐观锁）
    // if (row.containsKey("_version")) {
    //     request.version(Long.parseLong(row.get("_version")));
    //     request.versionType(VersionType.EXTERNAL);
    // }
    
    bulkRequest.add(request);
}
```

## 7.3 大文档处理

```java
// 对于超大的文本字段（如文章正文），建议：
// 1. 使用存储到对象存储（OSS/MinIO），ES 只存 URL
// 2. 开启 ES 的压缩
// 3. 排除不需要搜索的大字段
```

```yaml
# 示例：排除大字段（只存 URL）
esMapping:
  _index: article_index
  _id: id
  sql: "select id, title, summary, author, publish_time, content_url from articles"
  mappings:
    - column: id
      field: article_id
    - column: title
      field: title
    - column: summary
      field: summary
    - column: content_url
      field: content_url   # 内容正文存在 OSS，ES 只存储 URL
```

# 8. ETL 全量同步

```bash
# 触发 ES 全量同步
curl http://localhost:8081/etl/es/product.yml -X POST

# 返回
# {
#   "succeeded": true,
#   "resultMessage": "导入 ES 数据：500000 条"
# }
```

```java
@PostMapping("/etl/es/{task}")
public EtlResult etlES(@PathVariable String task) {
    
    ESSyncConfig config = configLoader.loadESConfig("es/" + task);
    
    // 从源库查询数据
    String sql = config.getSql();
    if (StringUtils.isNotEmpty(config.getEtlCondition())) {
        sql += " " + config.getEtlCondition();
    }
    
    List<Map<String, Object>> rows = sourceJdbcTemplate.queryForList(sql);
    
    // 分页批量写入 ES
    int batchSize = config.getCommitBatch();
    int total = rows.size();
    int success = 0;
    
    for (int i = 0; i < total; i += batchSize) {
        int end = Math.min(i + batchSize, total);
        
        BulkRequest bulkRequest = new BulkRequest();
        for (int j = i; j < end; j++) {
            Map<String, Object> row = rows.get(j);
            String docId = String.valueOf(row.get(config.getIdColumn()));
            
            bulkRequest.add(new IndexRequest(config.getIndex())
                .id(docId)
                .source(convertRowToDoc(config, row), XContentType.JSON));
        }
        
        BulkResponse response = restClient.bulk(bulkRequest);
        if (!response.hasFailures()) {
            success += (end - i);
        }
    }
    
    return EtlResult.success(total, success);
}
```
