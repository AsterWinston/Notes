
# 1. Analyzer 三组件

```
Analyzer (分析器)
  ├── CharFilter (字符过滤器)  → 预处理字符（HTML 标签去除、表情符号替换）
  ├── Tokenizer (分词器)      → 将文本切分成词项（Term）
  └── TokenFilter (词项过滤器) → 词项转换（小写、停用词、同义词）
```

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "iPhone 15 Pro is awesome!"
}

# 返回:
{ "tokens": [
  { "token": "iphone", "position": 0 },
  { "token": "15",     "position": 1 },
  { "token": "pro",    "position": 2 },
  { "token": "is",     "position": 3 },
  { "token": "awesome","position": 4 }
]}
```

# 2. 内置分析器

| 分析器 | 说明 | 适用场景 |
|--------|------|---------|
| `standard` | 默认，Unicode 标准分词 | 通用英文 |
| `simple` | 非字母字符切分，转小写 | — |
| `whitespace` | 空格切分 | — |
| `keyword` | 不分词，整体作为词项 | 精确匹配 |
| `stop` | standard + 停用词 | — |
| `pattern` | 正则分词 | 自定义分隔符 |
| `ik_smart` | IK 中文最简分词 | 中文搜索 |
| `ik_max_word` | IK 中文最大分词 | 中文索引 |
| `pinyin` | 拼音转换 | 拼音搜索 |

# 3. IK 中文分词

```bash
# 安装 IK 插件
elasticsearch-plugin install \
  https://github.com/infinilabs/analysis-ik/releases/download/v8.11.0/elasticsearch-analysis-ik-8.11.0.zip
```

```json
# ik_smart — 粗粒度
POST /_analyze
{
  "analyzer": "ik_smart",
  "text": "中华人民共和国国歌"
}
// ["中华人民共和国", "国歌"]

# ik_max_word — 细粒度
POST /_analyze
{
  "analyzer": "ik_max_word",
  "text": "中华人民共和国国歌"
}
// ["中华人民共和国", "中华人民", "中华", "华人", "人民共和国", "人民", "共和", "国歌"]
```

## 3.1 IK 自定义词典

```bash
# config/analysis-ik/custom.dic
iPhone 15 Pro
MacBook Pro
乔布斯

# remote ext dict
# IKAnalyzer.cfg.xml
<entry key="remote_ext_dict">http://my-server/dict.txt</entry>
```

# 4. 自定义分析器

```json
PUT /shop
{
  "settings": {
    "analysis": {
      "char_filter": {
        "my_char_filter": {
          "type": "mapping",
          "mappings": ["＆ => &", "￥ => ¥"]
        }
      },
      "tokenizer": {
        "my_tokenizer": {
          "type": "pattern",
          "pattern": "[,;\\s]+"
        }
      },
      "filter": {
        "my_stop": {
          "type": "stop",
          "stopwords": ["的", "了", "是"]
        }
      },
      "analyzer": {
        "my_analyzer": {
          "type": "custom",
          "char_filter": ["html_strip", "my_char_filter"],
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "my_stop"]
        }
      }
    }
  }
}
```

# 5. 同义词

```json
PUT /shop
{
  "settings": {
    "analysis": {
      "filter": {
        "synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "iPhone,苹果手机",
            "MacBook,苹果笔记本",
            "js, javascript"
          ],
          "synonyms_path": "analysis/synonyms.txt"
        }
      },
      "analyzer": {
        "synonym_analyzer": {
          "tokenizer": "ik_max_word",
          "filter": ["synonym_filter"]
        }
      }
    }
  }
}
```

# 6. 搜索与索引分析分离

```json
PUT /shop
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",           // 索引时细粒度
        "search_analyzer": "ik_smart"         // 搜索时粗粒度
      }
    }
  }
}
```

**效果**：索引时最大分词提高召回率，搜索时最简分词提高准确率。

# 7. 停用词

```bash
# config/analysis/stopwords.txt
的
了
是
在
和
```

```json
{
  "filter": {
    "my_stop": {
      "type": "stop",
      "stopwords_path": "analysis/stopwords.txt"
    }
  }
}
```
