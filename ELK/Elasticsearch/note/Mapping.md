# Mapping

Mapping是定义文档及包含的字段的存储和索引方式的过程。

一个Mapping定义有

- 元数据字段：自定义如何处理文档相关的元数据。如`_index`，`_id`和`_source`等字段。
- 字段或属性：一个Mapping包含了与文档相关的的字段和属性列表。

每个字段都有数据类型：

- 简单数据类型：
  - text
  - keyword
  - date
  - long
  - double
  - boolean
  - ip
- 支持JSON的层次结构性质类型：
  - object
  - nested
- 特殊类型
  - geo_point
  - geo_shape
  - completion

为不同的目的以不同的方式对同一字段建立索引通常很有用。

在索引中定义太多字段的情况可能导致映射爆炸，从而可能导致内存不足错误和难以恢复的情况。

以下设置允许您限制可以手动或动态创建的字段映射的数量，以防止不良文档导致Mapping爆炸：

- `index.mapping.total_fields.limit`：索引中的最大字段数，默认值是1000。
- `index.mapping.depth.limit`：字段最大深度，以内部对象的数量衡量。默认是20。
- `index.mapping.nested_fields.limit`：索引中最大数量的不同`nested`mappings，默认是50。
- `inidex.mapping.field_objects.limit`：一个文档中所有嵌套类型中嵌套JSON对象的最大的数量，默认是10000。
- `index.mapping.field_name_length.limit`：字段名称的最大值。默认值为`Long.MAX_VALUE`

字段和Mapping类型不需要在使用前定义，通过动态Mapping，仅通过索引文档即可添加新的字段名称，可以自定义动态映射的规则。

使用显式映射的创建一个索引

```shell
curl -X PUT "localhost:9200/my-index?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
'
```

在现有的映射中添加一个字段

```shell
curl -X PUT "localhost:9200/my-index/_mapping?pretty" -H 'Content-Type: application/json' -d'
{
  "properties": {
    "employee-id": {
      "type": "keyword",
      "index": false
    }
  }
}
'
```

除了支持的映射参数外，您无法更改现有字段的映射或字段类型。更改现有字段可能会使已经建立索引的数据无效。

如果需要更改字段的映射，请使用正确的映射创建一个新索引，然后将数据重新索引到该索引中。

重命名字段会使在旧字段名称下已建立索引的数据无效。而是添加一个别名字段以创建备用字段名称。

查看索引的Mapping

```shell
curl -X GET "localhost:9200/my-index/_mapping?pretty"
```

查看特定字段的映射

```shell
curl -X GET "localhost:9200/my-index/_mapping/field/employee-id?pretty"
```

## 删除映射类型

在Elasticsearch 7.0.0或更高版本中创建的索引不再接受`_default_`映射。

### 什么是Mapping类型

每个文档都有一个包含类型名称的`_type`元数据字段，并且可以通过在URL中指定类型名称将搜索限制为一种或多种类型。

`_type`字段和文档的`_id`字段结合生成了`_uid`字段，因此具有相同`_id`的不同类型的文档可以存在于单个索引中。

### 为什么要移除Mapping类型

在Elasticsearch索引中，在不同映射类型中具有相同名称的字段在内部由相同的Lucene字段支持。即同名的字段在不同的Mapping类型中所存储的位置相同，并且两个字段在这两种类型中必须具有相同的映射（定义）。

### 映射类型的选择

第一种选择是为每个文档类型创建一个索引。这种方法有两个优点

- 数据更可能是密集的，因此可以从Lucene中使用的压缩技术中受益。
- 在全文搜索中用于评分的`term`统计更可能是准确的，因为同一索引中的所有文档都代表一个实体。

### 自定义类型字段

```shell
PUT twitter
{
  "mappings": {
      "properties": {
        "type": { "type": "keyword" }, 
        "name": { "type": "text" },
        "user_name": { "type": "keyword" },
        "email": { "type": "keyword" },
        "content": { "type": "text" },
        "tweeted_at": { "type": "date" }
      }
  }
}

PUT twitter/_doc/user-kimchy
{
  "type": "user", 
  "name": "Shay Banon",
  "user_name": "kimchy",
  "email": "shay@kimchy.com"
}

PUT twitter/_doc/tweet-1
{
  "type": "tweet", 
  "user_name": "kimchy",
  "tweeted_at": "2017-10-24T09:00:00Z",
  "content": "Types are going away"
}

GET twitter/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "user_name": "kimchy"
        }
      },
      "filter": {
        "match": {
          "type": "tweet" 
        }
      }
    }
  }
}
```

### 没有映射类型的父/子

父子功能将继续像以前一样起作用，只是表达文档之间关系的方式已更改为使用新的`join`字段。

### 删除映射类型的时间表

Elasticsearch5.6.0

- 在索引上设置`index.mapping.single_type:true`将启用按索引的单一类型行为，该行为将在6.0中强制执行。
- 在5.6中创建的索引上可以使用父子项的`join`字段替换。

Elasticsearch6.x

- 在5.x中创建的索引将像在5.x中一样继续在6.x中运行。
- 在6.x中创建的索引仅允许每个索引使用一种类型。该类型可以使用任何名称，但是只能有一个。首选的类型名称是`_doc`，因此索引API具有与7.0中相同的路径：`PUT {index} / _ doc / {id}`和`POST {index} / _ doc`
- `_type`名称不能再与`_id`组合以形成`_uid`字段。 `_uid`字段已成为`_id`字段的别名。
- 新索引不再支持父/子的旧样式，而应使用`join`字段。
- 在6.8中，索引创建，索引模板和映射API支持查询字符串参数（include_type_name），该参数指示请求和响应是否应包含类型名称。

Elasticsearch 7.x

- 不建议在请求中指定类型。请注意，在7.0中，`_doc`是路径的永久部分，它表示端点名称而不是文档类型。
- 索引创建，索引模板和映射API中的`include_type_name`参数默认为`false`。完全设置该参数将导致弃用警告。
- `_default_`映射类型已删除。

Elasticsearch 8.x

- 不再支持在请求中指定类型。
- `include_type_name` 参数被移除。

将多类型索引迁移到单类型

`Reindex API`可用于将多类型索引转换为单类型索引。

```shell
POST _reindex
{
  "source": {
    "index": "twitter",
    "type": "user"
  },
  "dest": {
    "index": "users",
    "type": "_doc"
  }
}

POST _reindex
{
  "source": {
    "index": "twitter",
    "type": "tweet"
  },
  "dest": {
    "index": "tweets",
    "type": "_doc"
  }
}
```

### 自定义类型字段

示例添加一个自定义类型字段并将其设置为原始`_type`的值。如果有任何不同类型的文档具有冲突的ID，它也会将类型添加到`_id`中：

```shell
PUT new_twitter
{
  "mappings": {
    "_doc": {
      "properties": {
        "type": {
          "type": "keyword"
        },
        "name": {
          "type": "text"
        },
        "user_name": {
          "type": "keyword"
        },
        "email": {
          "type": "keyword"
        },
        "content": {
          "type": "text"
        },
        "tweeted_at": {
          "type": "date"
        }
      }
    }
  }
}


POST _reindex
{
  "source": {
    "index": "twitter"
  },
  "dest": {
    "index": "new_twitter"
  },
  "script": {
    "source": """
      ctx._source.type = ctx._type;
      ctx._id = ctx._type + '-' + ctx._id;
      ctx._type = '_doc';
    """
  }
}
```

### ElasticSearch7.0无类型API

在Elasticsearch 7.0中，每个API将支持无类型的请求，并且指定类型将产生弃用警告。

#### 索引APIs

索引创建，索引模板和映射API支持新的`include_type_name`URL参数，该参数指定请求和响应中的映射定义是否应包含类型名称。该参数在6.8版本默认是`true`，在7.0版本默认是`false`，在8.0版本中被移除。

```shell
curl -X PUT "localhost:9200/index?include_type_name=false&pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": { 
      "foo": {
        "type": "keyword"
      }
    }
  }
}
'
```

#### 文档APIs

在7.0，必须使用`{index}/_doc`路径来调用索引API才能生成`_id`和具有明确ID的`{index} / _ doc / {id}`

```shell
curl -X PUT "localhost:9200/index/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "foo": "baz"
}
'

```

`get`和`delete`APIs同样使用`{index}/_doc/{id}`路径。

```shell
curl -X GET "localhost:9200/index/_doc/1?pretty"
```

在7.0中，_doc表示端点名称，而不是文档类型。 _doc组件是文档索引，`get`和`delete`API路径的永久部分，在8.0中不会删除。

对于同时包含类型和终结点名称的API路径（如`_update`），在7.0中，终结点将紧随索引名称之后。

```shell
curl -X POST "localhost:9200/index/_update/1?pretty" -H 'Content-Type: application/json' -d'
{
    "doc" : {
        "foo" : "qux"
    }
}
'
curl -X GET "localhost:9200/index/_source/1?pretty"
```

#### 搜索APIs

调用诸如`_search`，`_msearch`或`_explain`之类的搜索API时，URL中不应包含类型。另外，`_type`字段不应在查询，聚合或脚本中使用。

### 响应中的类型

文档和搜索APIs会在响应中继续返回`_type`键，以避免中断响应解析。

请注意，使用已弃用的类型化API时，索引的映射类型将照常返回，但无类型API会在响应中返回虚拟类型`_doc`。

```shell
curl -X PUT "localhost:9200/index/my_type/1?pretty" -H 'Content-Type: application/json' -d'
{
  "foo": "baz"
}
'
curl -X GET "localhost:9200/index/_doc/1?pretty"
```

### 索引模板

建议通过将`include_type_name`设置为false来重新添加索引模板，从而使其无类型。在后台，无类型的模板在创建索引时将使用伪类型`_doc`。

如果将无类型的模板与类型化的索引创建调用一起使用，或者将有类型的模板与无类型的索引创建调用一起使用，则仍将应用该模板，但是索引创建的调用会决定是否应该有类型。

```shell
curl -X PUT "localhost:9200/_template/template1?pretty" -H 'Content-Type: application/json' -d'
{
  "index_patterns":[ "index-1-*" ],
  "mappings": {
    "properties": {
      "foo": {
        "type": "keyword"
      }
    }
  }
}
'
curl -X PUT "localhost:9200/_template/template2?include_type_name=true&pretty" -H 'Content-Type: application/json' -d'
{
  "index_patterns":[ "index-2-*" ],
  "mappings": {
    "type": {
      "properties": {
        "foo": {
          "type": "keyword"
        }
      }
    }
  }
}
'
curl -X PUT "localhost:9200/index-1-01?include_type_name=true&pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "type": {
      "properties": {
        "bar": {
          "type": "long"
        }
      }
    }
  }
}
'
curl -X PUT "localhost:9200/index-2-01?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "bar": {
        "type": "long"
      }
    }
  }
}
'
```

## 字段数据类型

核心的数据类型

- 字符串： test和keyword
- 数字类型：long，integer，short，byte，double，float，half_float，scaled_float
- 日期类型：date
- 日期纳秒：date_nanos
- 布尔类型：boolean
- 二进制：binary
- 范围：integer_range，float_range，long_range，double_range，date_range，ip_range

复杂数据类型

- Object：单个JSON对象
- Nested：JSON对象数组。

地理数据类型

- Geo-point：经纬度坐标点
- Geo-shape：多边形等复杂形状

特殊数据类型

- IP：IPv和IPv6地址
- Completion数据类型：提供自动完成建议
- 