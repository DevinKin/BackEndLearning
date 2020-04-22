# Elasticsearch入门

## 启动并运行Elasticsearch集群

使用cat health API验证三节点集群是否正在运行:

```
GET /_cat/health?v
```

## 索引一些文档

使用`PUT`请求直接执行此操作，该请求指定要添加文档的索引。

```shell
curl -X PUT "localhost:9200/customer/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "name": "John Doe"
}
'
```

如果该文档和索引尚不存在，此请求将自动创建该客户索引，添加ID为1的新文档，并存储名称字段并为其建立索引。

查看刚刚创建完成的文档。

```shell
curl -X GET "localhost:9200/customer/_doc/1?pretty"
```

### 批量索引文档

使用批量处理批处理文档操作比单独提交请求要快得多，因为它可以最大程度地减少网络往返次数。

- 下载`Elasticsearch`的[account.json](https://raw.githubusercontent.com/elastic/elasticsearch/master/docs/src/test/resources/accounts.json)样例数据进行分析。

- 使用以下`_bulk`请求将`account.json`数据导入`bank`索引

  ```shell
  curl -H "Content-Type: application/json" -XPOST "localhost:9200/bank/_bulk?pretty&refresh" --data-binary "@accounts.json"
  curl "localhost:9200/_cat/indices?v"
  ```

- 响应表明成功索引了1000个文档。

  ```
  health status index    uuid                   pri rep docs.count docs.deleted store.size pri.store.size
  green  open   bank     01er1xe4T1a9stXnL-dYyA   1   1       1000            0      1.7mb        869.6kb
  green  open   customer Fc-9g3HORpOfDdh7gQhT8g   1   1          1            0      6.9kb          3.4kb
  
  ```

## 开始查询

将account.json数据导入Elasticsearch之后，可以通过将请求发送到`_search`来检索。

检索bank索引中按帐号排序的所有文档，使用`_search`

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ]
}
'
```

上面的查询请求得到的响应还包含了下列以下信息：

- `took`：Elasticsearch 运行查询所需的时间，单位是毫秒。
- `time_out`：搜索请求是否超时。
- `_shards`：搜索了多少个分片，以及成功，失败或跳过了多少个分片。
- `max_score`：找多最相关文档的分数。
- `hits.total.value`：找到了多少个匹配的文档。
- `hits.sort`：文档排序位置（不按相关性得分排序时）。
- `hits._score`：文档的相关性得分（使用match_all时不适用）

每个搜索请求都是独立的：Elasticsearch在请求中不维护任何状态信息。要翻阅搜索结果，请在请求中指定`from`和`size`参数。

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_all": {} },
  "sort": [
    { "account_number": "asc" }
  ],
  "from": 10,
  "size": 10
}
'
```

要在字段中搜索特定字词，可以使用`match`查询，下面的查询获取`address`包含`mill`或者`lane`：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": { "match_phrase": { "address": "mill lane" } }
}
'
```

可以使用`bool`来组合多个查询条件，构造更复杂的查询。

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "age": "40" } }
      ],
      "must_not": [
        { "match": { "state": "ID" } }
      ]
    }
  }
}
'
```

布尔查询中的每个`must`，`should`和`must_not`元素都称为查询子句。文档满足每个必须或应该条款中的条件的程度会提高文档的相关性得分。默认情况下，Elasticsearch返回按这些相关性得分排名的文档。

下面的请求使用范围过滤器将结果限制为余额在20,000美元到30,000美元（含）之间的帐户：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": { "match_all": {} },
      "filter": {
        "range": {
          "balance": {
            "gte": 20000,
            "lte": 30000
          }
        }
      }
    }
  }
}
'
```

## 使用汇总分析结果

使用`terms`聚合将`bank`索引中的所有账户按状态分组，并按降序返回账户数量最多的十个州。

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      }
    }
  }
}
'
```

响应中的`bucket`是`state`(州)字段的值。`doc_count`表明各个`state`的帐号数量。因为请求中设置`size=0`，所以响应仅包含聚合结果。

可以组合聚合构建更复杂的数据汇总。下面的请求嵌套了`avg`聚合在上一次`group_by_state`聚合，来计算每个州银行平均帐号余额。

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword"
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'

```

可以通过在`terms`聚合中指定顺序来使用嵌套聚合的结果进行排序，而不必按计数对结果进行排序：

```shell
curl -X GET "localhost:9200/bank/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "group_by_state": {
      "terms": {
        "field": "state.keyword",
        "order": {
          "average_balance": "desc"
        }
      },
      "aggs": {
        "average_balance": {
          "avg": {
            "field": "balance"
          }
        }
      }
    }
  }
}
'

```

## 