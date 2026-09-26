
# 1. Bulk 优化

```bash
# 最佳实践
# 1. 每批 5-15MB
# 2. 文档数 500-1000
# 3. 多线程并行提交
# 4. 检查 response errors

# Java RestHighLevelClient
BulkRequest request = new BulkRequest();
for (Order order : orders) {
    request.add(new IndexRequest("orders")
        .id(order.getId())
        .source(json, XContentType.JSON));
}
BulkResponse response = client.bulk(request, RequestOptions.DEFAULT);
if (response.hasFailures()) {
    // 处理失败
}
```

# 2. Refresh Interval

```json
# 批量导入时关闭 refresh
PUT /orders/_settings
{ "refresh_interval": "-1" }

# 导入完成后恢复
PUT /orders/_settings
{ "refresh_interval": "30s" }
```

**效果**：关闭 refresh 后索引吞吐量提升 2-5 倍。

# 3. Translog 优化

```json
# 批量导入时：异步 + 增大阈值
PUT /orders/_settings
{
  "translog": {
    "durability": "async",        // 异步刷盘
    "sync_interval": "30s",      // 30s 刷一次
    "flush_threshold_size": "1gb" // 1GB 阈值
  }
}

# 导入完成后恢复
PUT /orders/_settings
{
  "translog": {
    "durability": "request"       // 每次请求刷盘（安全）
  }
}
```

# 4. 副本策略

```json
# 批量导入前：关闭副本
PUT /orders/_settings
{ "number_of_replicas": 0 }

# 导入完成：添加副本（自动同步）
PUT /orders/_settings
{ "number_of_replicas": 1 }

# 等待副本就绪
GET /_cluster/health/orders?wait_for_status=green
```

**效果**：不写副本，写入性能提升 1 倍。

# 5. 段合并优化

```json
# 导入完成后强制合并
POST /orders/_forcemerge?max_num_segments=1

# 动态调整合并策略
PUT /orders/_settings
{
  "index.merge.policy.max_merged_segment": "5gb",
  "index.merge.scheduler.max_thread_count": 1
}
```

# 6. Indexing Buffer

```json
# 增大索引缓冲区
PUT /orders/_settings
{
  "index.memory.index_buffer_size": "30%"  // 堆的 30%
}
```

```yaml
# 或 elasticsearch.yml 全局
indices.memory.index_buffer_size: 30%
```

# 7. Mapping 优化

```json
# 1. 关闭不需要的索引选项
{
  "title": {
    "type": "text",
    "norms": false             // 不需要评分时关闭
  },
  "status": {
    "type": "keyword",
    "doc_values": false        // 不需要排序/聚合时关闭
  },
  "large_field": {
    "type": "text",
    "index": false             // 不可搜索时关闭索引
  }
}

# 2. 禁用 _source（不推荐，失去原始文档）
"_source": { "enabled": false }

# 3. 禁用 _all（7.x 已移除）
```

# 8. 批量导入 Checklist

```
1. □ refresh_interval = -1
2. □ number_of_replicas = 0
3. □ translog durability = async + 1GB threshold
4. □ index_buffer_size = 30%
5. □ Bulk 5-15MB / 批
6. □ 导入后恢复 refresh + replicas
7. □ forcemerge max_num_segments=1
8. □ 监控 _cat/thread_pool/write
```

# 9. 写入性能基准

```
测试：3 节点，primary_shards=3, replica=1

| 优化前         | 优化后          | 提升  |
|---------------|----------------|------|
| 5000 docs/s   | 25000 docs/s   | 5x   |

优化项：refresh=-1, replica=0, translog async, bulk 1000/batch
```
