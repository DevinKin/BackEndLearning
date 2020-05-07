# 聚合分析

聚合框架有助于基于搜索查询提供聚合数据。它基于称为聚合的简单构建块，可以进行组合以构建复杂的数据汇总。

聚合可以看作是在一组文档上建立分析信息的工作单元。聚合执行的上下文定义了此文档是什么。（例如，顶级聚合在搜索请求的已执行查询/过滤器的上下文中执行）

有以下几种类型的聚合，每种聚合都有自己的目的和输出

- Bucketing：一系列构建存储桶的聚合，其中每个存储桶都与一个键和一个文档条件相关联。执行聚合时，将对上下文中的每个文档判断所有存储桶条件，并且当条件匹配时，该文档将被视为“落入”相关存储桶。在汇总过程结束时，我们将获得一个存储桶列表-每个存储桶都带有一组“从属”的文档。
- Metric：聚合可跟踪和计算一组文档的指标。
- Matrix：矩阵聚合，可在多个字段上操作，并根据从请求的文档字段中提取的值生成矩阵结果。该类聚合目前不支持脚本。
- Pipeline：聚合其他汇总及其相关指标的输出的聚合。

由于每个存储桶`bucket`有效定义了一个文档集（所有属于该文档的存储桶），而聚合是作用在一组文档上建立分析信息的工作单元，因此有聚合有一个强大的功能，便是可以实现嵌套聚合。

## 构建聚合

构建聚合的基本结构如下：

- `aggregations`对象可以用简写`aggs`表示。
- 聚合的逻辑名称还将用于唯一地标识响应中的聚合。
- 每个聚合都具有特定的类型，通常是指定的聚合主体内的第一个键。每种聚合类型都定义自己的主体，具体取决于聚合的性质。
- 在聚合类型定义的同一级别，可以选择定义一组其他聚合。尽管仅当您定义的聚合具有存储特性时才有意义。

```json
"aggregations" : {
    "<aggregation_name>" : {
        "<aggregation_type>" : {
            <aggregation_body>
        }
        [,"meta" : {  [<meta_data_body>] } ]?
        [,"aggregations" : { [<sub_aggregation>]+ } ]?
    }
    [,"<aggregation_name_2>" : { ... } ]*
}
```

## 值的来源

一些聚合处理从聚合文档中提取的值。通常，将从使用聚合的字段关键字设置的特定文档字段中提取值。也可以定义一个脚本来生成值（每个文档）。

当为聚合中配置了`field`和`script`时，该脚本将被视为值脚本`value script`。

虽然普通脚本是在文档级别评估的（即脚本可以访问与文档关联的所有数据），但值脚本是在值级别评估的。在这种模式下，从配置的字段中提取值，并且脚本用于对这些值进行“转换”。

## 指标聚合

此类聚合基于从要聚合的文档中以一种或另一种方式提取的值来计算指标。这些值通常从文档的字段中提取（使用字段数据），但是也可以使用脚本生成。

数值指标聚合是一种特殊类型的指标聚合，可输出数值。

一些聚合输出单数值指标（如avg），称为单值数值指标聚合（single-value numeric metrics aggregation）。

一些聚合生成多个指标（如stat），称为多值数值指标聚合（multi-value numeric metrics aggregation）。

以下采用的示例为[exams.json](./exams.json)

### 平均聚合

avg聚合是单值指标聚合，用于计算从聚合文档中提取的数值的平均值。这些数值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "avg_grade" : { "avg" : { "field" : "grade" } }
    }
}
'
```

#### 脚本

根据脚本计算分数的平均值，这会将脚本参数解释为具有简便脚本语言且没有脚本参数的嵌入式脚本。

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "avg_grade" : {
            "avg" : {
                "script" : {
                    "source" : "doc.grade.value"
                }
            }
        }
    }
}
'
```

要使用存储的脚本，请使用以下语法：

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "avg_grade" : {
            "avg" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "field": "grade"
                    }
                }
            }
        }
    }
}
'

```

#### 值脚本

该考试远远超出了学生的水平，因此需要进行成绩更正。我们可以使用值脚本来获取新的平均值：

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "avg_corrected_grade" : {
            "avg" : {
                "field" : "grade",
                "script" : {
                    "lang": "painless",
                    "source": "_value * params.correction",
                    "params" : {
                        "correction" : 1.2
                    }
                }
            }
        }
    }
}
'

```

#### 缺省值

missing参数定义应如何处理缺省值的文档。默认情况下，它们将被忽略，但也可以将它们视为具有默认值。

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "grade_avg" : {
            "avg" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
'
```

### 加权平均聚合

加权平均聚合是单值指标聚合，用于计算从聚合文档中提取的数值的加权平均值。权重和数值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

计算常规平均值时，每个数据点都具有相等的“权重”，它对最终结果值的贡献相等。

另一方面，加权平均对每个数据点的加权不同。

加权平均值的公式为：`∑(value * weight) / ∑(weight)`

可以将常规平均值视为每个平均值的隐式权重为1的加权平均值。

`weighted_avg`参数

|  参数名称  |               描述               | 必填 | 默认值 |
| :--------: | :------------------------------: | :--: | :----: |
|   value    |     提供值的字段或脚本的配置     |  是  |        |
|   weight   |     提供权重字段或脚本的配置     |  是  |        |
|   format   |          数字响应格式化          | 可选 |        |
| value_type | 有关纯脚本或未映射字段的值的提示 | 可选 |        |

`value`和`weight`有特定的字段配置

`value`参数

| 参数名称 |             描述             | 必填 | 默认值 |
| :------: | :--------------------------: | :--: | :----: |
|  field   |         提取值的字段         |  是  |        |
| missing  | 如果字段完全缺失，则使用的值 | 可选 |        |

`weight`参数

| 参数名称 |        描述        | 必填 | 默认值 |
| :------: | :----------------: | :--: | :----: |
|  field   |   提取权重的字段   |  是  |        |
| missing  | 字段缺省时使用的值 | 可选 |        |

```shell
curl -X POST "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "field": "grade"
                },
                "weight": {
                    "field": "weight"
                }
            }
        }
    }
}
'
```



虽然每个字段允许多个值，但仅允许一个权重。如果汇总遇到权重超过一个的文档（例如权重字段是多值字段），它将引发异常。

单个权重将独立应用于从值字段中提取的每个值。

```shell
curl -X POST "localhost:9200/exams/_doc?refresh&pretty" -H 'Content-Type: application/json' -d'
{
    "grade": [1, 2, 3],
    "weight": 2
}
'
curl -X POST "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "field": "grade"
                },
                "weight": {
                    "field": "weight"
                }
            }
        }
    }
}
'

```

#### 脚本

值和权重都可以从脚本而不是字段中得出。

```shell
curl -X POST "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "script": "doc.grade.value + 1"
                },
                "weight": {
                    "script": "doc.weight.value + 1"
                }
            }
        }
    }
}
'
```

#### 缺省值

缺省值参数定义了应如何处理缺省值文档。

```shell
POST /exams/_search
{
    "size": 0,
    "aggs" : {
        "weighted_grade": {
            "weighted_avg": {
                "value": {
                    "field": "grade",
                    "missing": 2
                },
                "weight": {
                    "field": "weight",
                    "missing": 3
                }
            }
        }
    }
}
```

### 基数聚合

基数聚合是单值指标聚合，用于计算不同值的近似计数。值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

导入[sales.json](./example-data/sales.json)

```shell
curl -H "Content-Type: application/json" -XPOST "localhost:9200/sales/_bulk?pretty&refresh" --data-binary "@sales.json"
```

计算与查询匹配的已售产品的唯一数量

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "type_count" : {
            "cardinality" : {
                "field" : "type"
            }
        }
    }
}
'

```

#### 精度控制

此聚合还支持`precision_threshold`选项：

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "type_count" : {
            "cardinality" : {
                "field" : "type",
                "precision_threshold": 100 
            }
        }
    }
}
'

```

#### 计数是近似值

计算精确计数需要将值加载到哈希集中并返回其大小，当处理高基数集和/或较大的值时，这不会扩展，因为所需的内存使用情况以及在节点之间传递每个分片集的需求会占用过多的群集资源。

此基数聚合基于HyperLogLog ++算法，该算法基于值的哈希值进行计数。

#### 脚本

基数指标聚合支持脚本，但是，由于需要动态计算哈希值，因此性能会受到明显影响。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "type_promoted_count" : {
            "cardinality" : {
                "script": {
                    "lang": "painless",
                    "source": "doc['type'].value + ' ' + doc['promoted'].value"
                }
            }
        }
    }
}
'
```

这会将脚本参数解释为具有简便脚本语言且没有脚本参数的嵌入式脚本。要使用存储的脚本，请使用以下语法

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "type_promoted_count" : {
            "cardinality" : {
                "script" : {
                    "id": "my_script",
                    "params": {
                        "type_field": "type",
                        "promoted_field": "promoted"
                    }
                }
            }
        }
    }
}
'
```

#### 缺省值

缺省值参数定义了应如何处理缺省值文档。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "tag_cardinality" : {
            "cardinality" : {
                "field" : "tag",
                "missing": "N/A" 
            }
        }
    }
}
'
```

### 扩展统计聚合

扩展统计聚合是一种多值指标聚合，可根据从聚合文档中提取的数值计算统计信息。统计数值可以由文档指定的数值字段或者脚本提供。

`extended_stats`聚合是`stats`聚合的扩展版本，其中添加了其他指标，例如`sum_of_squares`，`variance`，`std_deviation`和`std_deviation_bounds`。

```shell
curl -X GET "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "grades_stats" : { "extended_stats" : { "field" : "grade" } }
    }
}
'
```

#### 标准偏差范围

默认情况下，`extended_stats`指标将返回一个名为`std_deviation_bounds`的对象，该对象与平均值之间的间隔为正负两个标准差。

如果要使用其他边界，例如三个标准偏差，则可以在请求中设置`sigma`，sigma控制应该显示多少平均值的标准偏差

```shell
curl -X GET "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "field" : "grade",
                "sigma" : 3 
            }
        }
    }
}
'
```

#### 脚本

计算成绩的扩展统计

```shell
curl -X GET "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "script" : {
                    "source" : "doc['grade'].value",
                    "lang" : "painless"
                 }
             }
         }
    }
}
'
```

#### 值脚本

可以根据值脚本提供新的扩展统计

```shell
curl -X GET "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "field" : "grade",
                "script" : {
                    "lang" : "painless",
                    "source": "_value * params.correction",
                    "params" : {
                        "correction" : 1.2
                    }
                }
            }
        }
    }
}
'

```

#### 缺省值 

缺省值参数定义了应如何处理缺省值文档。

```shell
curl -X GET "localhost:9200/exams/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "grades_stats" : {
            "extended_stats" : {
                "field" : "grade",
                "missing": 0 
            }
        }
    }
}
'
```

### 地理边界聚合

地理边界聚合用于计算包含一个字段的所有`geo_point`值的边界框。

新建索引`museums`，设置其属性`location`的类型为`geo_point`。

```shell
curl -X PUT "localhost:9200/museums?pretty" -H 'Content-Type: application/json' -d'
{
    "mappings": {
        "properties": {
            "location": {
                "type": "geo_point"
            }
        }
    }
}
'
```

导入地理座标数据并刷新

```shell
curl -X POST "localhost:9200/museums/_bulk?refresh&pretty" -H 'Content-Type: application/json' -d'
{"index":{"_id":1}}
{"location": "52.374081,4.912350", "name": "NEMO Science Museum"}
{"index":{"_id":2}}
{"location": "52.369219,4.901618", "name": "Museum Het Rembrandthuis"}
{"index":{"_id":3}}
{"location": "52.371667,4.914722", "name": "Nederlands Scheepvaartmuseum"}
{"index":{"_id":4}}
{"location": "51.222900,4.405200", "name": "Letterenhuis"}
{"index":{"_id":5}}
{"location": "48.861111,2.336389", "name": "Musée du Louvre"}
{"index":{"_id":6}}
{"location": "48.860000,2.327000", "name": "Musée d'Orsay"}
'
```

使用地理座标边界聚合查询

```shell
curl -X POST "localhost:9200/museums/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : { "name" : "musée" }
    },
    "aggs" : {
        "viewport" : {
            "geo_bounds" : {
                "field" : "location", 
                "wrap_longitude" : true 
            }
        }
    }
}
'
```

`geo_bounds`聚合指定用于获取边界的字段

`wrap_longitude`是一个可选参数，用于指定是否应允许边界框与国际日期线重叠。默认值是true

### 地理质心聚合

地理质心聚合是一个指标聚合，它根据`geo_point`字段的所有座标值计算加权质心。

```shell
curl -X POST "localhost:9200/museums/_bulk?refresh&pretty" -H 'Content-Type: application/json' -d'
{"index":{"_id":1}}
{"location": "52.374081,4.912350", "city": "Amsterdam", "name": "NEMO Science Museum"}
{"index":{"_id":2}}
{"location": "52.369219,4.901618", "city": "Amsterdam", "name": "Museum Het Rembrandthuis"}
{"index":{"_id":3}}
{"location": "52.371667,4.914722", "city": "Amsterdam", "name": "Nederlands Scheepvaartmuseum"}
{"index":{"_id":4}}
{"location": "51.222900,4.405200", "city": "Antwerp", "name": "Letterenhuis"}
{"index":{"_id":5}}
{"location": "48.861111,2.336389", "city": "Paris", "name": "Musée du Louvre"}
{"index":{"_id":6}}
{"location": "48.860000,2.327000", "city": "Paris", "name": "Musée d'Orsay"}
'


curl -X POST "localhost:9200/museums/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "centroid" : {
            "geo_centroid" : {
                "field" : "location" 
            }
        }
    }
}
'

```

`geo_centroid`聚合指定用于计算质心的字段。 （注意：字段必须为Geopoint类型）

当`geo_centroid`聚合作为子聚合合并到其他存储桶聚合，下面的例子返回每个城市的质心地理座标

```shell
curl -X POST "localhost:9200/museums/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "cities" : {
            "terms" : { "field" : "city.keyword" },
            "aggs" : {
                "centroid" : {
                    "geo_centroid" : { "field" : "location" }
                }
            }
        }
    }
}
'
```

在`geohash_grid`聚合中使用`geo_centroid`作为子聚合注意事项

- `geohash_grid`聚合将文档而不是单个地理位置放置到存储桶中。
- 如果文档的`geo_point`字段包含多个值，即使该文档的一个或多个地理位置不在存储桶边界之内，也可以将其分配给多个存储桶。
- 如果还是用了地心子聚合，每个质心都是使用存储桶中的所有地理位置计算得到的，包括那些在存储桶边界外的。这可能导致质心超出存储桶边界。

### 最大值/最小值聚合

最大值聚合是单值指标聚合，可跟踪并返回从聚合文档中提取的数值中的最大值。这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

最小值聚合和最大值聚合同理，不赘述。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "max_price" : { "max" : { "field" : "price" } }
    }
}
'
```

#### 脚本

最大聚合还可以计算脚本的最大值。下面的示例计算最高价格：

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "max_price" : {
            "max" : {
                "script" : {
                    "source" : "doc.price.value"
                }
            }
        }
    }
}
'

```

#### 值脚本

我们可以使用值脚本将转化率应用于每个值，然后再进行汇总

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "max_price_in_euros" : {
            "max" : {
                "field" : "price",
                "script" : {
                    "source" : "_value * params.conversion_rate",
                    "params" : {
                        "conversion_rate" : 1.2
                    }
                }
            }
        }
    }
}
'
```

#### 缺省值

缺省值参数定义了应如何处理缺省值文档。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "grade_max" : {
            "max" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
'
```

### 百分位数聚合

百分位数聚合是一种多值度量标准聚合，可对从聚合文档中提取的数值计算一个或多个百分位数。这些值可以由提供的脚本生成，也可以从文档中的特定数字或直方图字段中提取。

百分位数通常用于查找异常值。在正态分布中，第0.13和第99.87个百分位数代表与平均值的三个标准差。任何超出三个标准偏差的数据通常被视为异常。

网站加载时间百分位数统计

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time" 
            }
        }
    }
}
'
```

默认情况下，百分位数聚合会产生一定范围的百分位数：`[1, 5, 25, 50, 75, 95, 99]`

管理员只对异常值（极端百分比）感兴趣。我们可以只指定相应的百分比（请求的百分比必须是介于0到100之间的一个值）：

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time",
                "percents" : [95, 99, 99.9] 
            }
        }
    }
}
'
```

#### 带键的响应

默认情况下，键标志设置为true，该键将唯一的字符串键与每个存储桶相关联，并以散列而不是数组的形式返回范围。

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs": {
        "load_time_outlier": {
            "percentiles": {
                "field": "load_time",
                "keyed": false
            }
        }
    }
}
'

```

#### 脚本

百分位数聚合支持脚本。以下计算网站响应百分数，以秒为单位。

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "script" : {
                    "lang": "painless",
                    "source": "doc['load_time'].value / params.timeUnit", 
                    "params" : {
                        "timeUnit" : 1000   
                    }
                }
            }
        }
    }
}
'

```

#### 压缩

百分数聚合使用近似算法必须在存储器利用率与估计精度之间取得平衡。可以使用压缩参数控制此平衡。

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time",
                "tdigest": {
                  "compression" : 200 
                }
            }
        }
    }
}
'

```

TDigest算法使用多个“节点”来近似百分位数-可用节点越多，与数据量成正比的精度（以及较大的内存占用量）就越高。压缩参数将最大节点数限制为20 *压缩。

因此，通过增加压缩值，可以以更多内存为代价来提高百分位数的准确性。较大的压缩值也会使算法变慢，因为基础树数据结构的大小会增加，从而导致操作成本更高。默认压缩值为100。

#### HDR直方图

HDR直方图（高动态范围直方图）是一种替代实现，在计算延迟测量的百分位数时会很有用，因为它可以比`tdigiest`实现更快，但需要更大的内存占用空间。

可以通过在请求中指定`method`参数来使用HDR直方图

- `hdr`对象指示应使用HDR直方图来计算百分位数，并且可以在对象内部指定此算法的特定设置。
- `number_of_significant_value_digits`指定有效位数的直方图值的分辨率

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "load_time_outlier" : {
            "percentiles" : {
                "field" : "load_time",
                "percents" : [95, 99, 99.9],
                "hdr": { 
                  "number_of_significant_value_digits" : 3 
                }
            }
        }
    }
}
'

```

HDR直方图仅支持正值，如果传递负值，则将出错。

#### 缺省值

缺省值参数定义了应如何处理缺省值文档。

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "grade_percentiles" : {
            "percentiles" : {
                "field" : "grade",
                "missing": 10 
            }
        }
    }
}
'

```

### 百分数等级聚合

百分数等级（`percentile_ranks`）聚合多值度量标准聚合，该聚合针对从聚合文档中提取的数值计算一个或多个百分数等级。这些值可以由提供的脚本生成，也可以从文档中的特定数字或直方图字段中提取。

百分数等级显示低于特定值（`values`属性）的观察值的百分比，`values`是一个数组，查出低于每个元素的对应的百分数占比。

例如有一项服务协议，其中95％的页面加载在500毫秒内完成，而99％的页面加载在600ms内完成。

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "load_time_ranks" : {
            "percentile_ranks" : {
                "field" : "load_time", 
                "values" : [500, 600]
            }
        }
    }
}
'
```

#### 带键的响应

默认情况下，键控标志设置为true，会将唯一的字符串键与每个存储桶相关联，并以散列而不是数组的形式返回范围。

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs": {
        "load_time_ranks": {
            "percentile_ranks": {
                "field": "load_time",
                "values": [500, 600],
                "keyed": false
            }
        }
    }
}
'
```

#### 脚本

百分数等级聚合支持脚本编写。

```shell
curl -X GET "localhost:9200/latency/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "load_time_ranks" : {
            "percentile_ranks" : {
                "values" : [500, 600],
                "script" : {
                    "lang": "painless",
                    "source": "doc['load_time'].value / params.timeUnit", 
                    "params" : {
                        "timeUnit" : 1000   
                    }
                }
            }
        }
    }
}
'
```

#### HDR直方图

和`percentile_ranks`百分数聚合同理，不赘述。

#### 缺省值

和`percentile_ranks`同理，不赘述。

### 脚本式指标聚合

脚本式指标聚合是使用脚本执行以提供指标输出的聚合。

- `init_script`是可选参数，所有其他脚本都是必需的

首先导入样本数据[ledger.json](../example-data/ledger.json)

然后为`type`字段进行优化，没有优化的字段es默认是禁止聚合/排序操作的。所以需要将要聚合的字段添加优化。

```shell
curl -X PUT "localhost:9200/ledger/_mapping?pretty" -H 'Content-Type: application/json' -d'
                                                {
                                                  "properties": {
                                                    "type": {
                                                      "type":     "text",
                                                      "fielddata": true
                                                    }
                                                  }
                                                }
                                                '
```

执行脚本式指标聚合

```shell
POST ledger/_search?size=0
{
    "query" : {
        "match_all" : {}
    },
    "aggs": {
        "profit": {
            "scripted_metric": {
                "init_script" : "state.transactions = []", 
                "map_script" : "state.transactions.add(doc.type.value == 'sale' ? doc.amount.value : -1 * doc.amount.value)",
                "combine_script" : "double profit = 0; for (t in state.transactions) { profit += t } return profit",
                "reduce_script" : "double profit = 0; for (a in states) { profit += a } return profit"
            }
        }
    }
}
```

也可以使用存储型脚本执行脚本式指标聚合

```shell
curl -X POST "localhost:9200/ledger/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs": {
        "profit": {
            "scripted_metric": {
                "init_script" : {
                    "id": "my_init_script"
                },
                "map_script" : {
                    "id": "my_map_script"
                },
                "combine_script" : {
                    "id": "my_combine_script"
                },
                "params": {
                    "field": "amount" 
                },
                "reduce_script" : {
                    "id": "my_reduce_script"
                }
            }
        }
    }
}
'
```

#### 允许的返回类型

虽然可以在单个脚本中使用任何有效的脚本对象，但是这些脚本必须仅返回以下类型或将其存储在状态对象中

- 基本类型（primitive types）
- 字符串（String)
- 映射（Map，仅包含此处列出的类型的键和值）
- 数组（Array，数组的元素仅包含此处列出的类型）

#### 脚本范围

脚本指标聚合使用脚本在4个阶段的执行：

- `init_script`：在收集任何文件之前先行处理。允许聚合设置任何初始状态。
- `map_script`：每个收集的文档执行一次。这是个必填的脚本。如果未指定`combine_script`，则需要将结果状态存储在`state`对象中。
- `combine_script`：文档收集完成后，对每个分片执行一次。这个是必填的脚本。允许聚合合并从每个分片返回的状态。
- `reduce_script`：所有分片均返回其结果后，在协调节点上执行一次。这是个必填的脚本。这个脚本提供了访问`states`的权限，`states`是每个分片上`combine_script`结果的数组。

#### 其他参数

`params`，可选的，一个传递到`init_script`，`map_script`，`combine_script`的参数。如果不指定，默认提供空的参数。

空存储桶

如果脚本化指标聚合的父存储桶未收集任何文档，则将从分片返回空聚合响应，并返回一个空值。

### 状态聚合

状态聚合是一个多值指标聚合，可根据从聚合文档中提取的数值计算统计信息。

这些值可以从文档中的特定数字字段中提取，也可以由提供的脚本生成。

状态聚合返回的字段的组成：

- `min`
- `max`
- `sum`
- `count`
- `avg`

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "grades_stats" : { "stats" : { "field" : "grade" } }
    }
}
'
```

#### 脚本

状态聚合可以使用脚本统计

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "grades_stats" : {
             "stats" : {
                 "script" : {
                     "lang": "painless",
                     "source": "doc['grade'].value"
                 }
             }
         }
    }
}
'

```

#### 值脚本

使用值脚本对每个值进行转换后，再使用状态指标聚合统计

```shell
curl -X POST "localhost:9200/exams/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "grades_stats" : {
            "stats" : {
                "field" : "grade",
                "script" : {
                    "lang": "painless",
                    "source": "_value * params.correction",
                    "params" : {
                        "correction" : 1.2
                    }
                }
            }
        }
    }
}
'
```

#### 缺省值

缺省值和其他聚合类似，不赘述。

### 字符串状态聚合

字符串状态聚合是一个多值指标聚合，计算从聚合文档中提取的字符串值的统计信息。这些值可以从文档中的特定`keyword`字段检索，也可以由提供的脚本生成。

字符串状态聚合返回如下结果

- `count`：计算非空字段数量
- `min_length`：最短的字符串长度
- `max_length`：最长的字符串长度
- `avg_length`：字符串的平均长度
- `entropy`：香农熵值是根据聚合收集的所有项计算得出的。香农熵可量化字段中包含的信息量。

```shell
curl -X POST "localhost:9200/twitter/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "message_stats" : { "string_stats" : { "field" : "message.keyword" } }
    }
}
'
```

#### 字符分布

香农熵值的计算是基于每个字符在聚合收集的所有字术语中出现的概率。

要查看所有字符的概率分布，我们可以添加`show_distribution`（默认值：false）参数。

```shell
curl -X POST "localhost:9200/twitter/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "message_stats" : {
            "string_stats" : {
                "field" : "message.keyword",
                "show_distribution": true  
            }
        }
    }
}
'
```

`distribution`对象显示每个字符全部出现的概率。字符按降序排列。

#### 脚本

字符串状态聚合可以基于脚本计算，也可以基于存储型脚本计算。

```shell
curl -X POST "localhost:9200/twitter/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "message_stats" : {
             "string_stats" : {
                 "script" : {
                     "lang": "painless",
                     "source": "doc['message.keyword'].value"
                 }
             }
         }
    }
}
'

```

#### 值脚本

我们可以使用值脚本来修改信息（例如，可以添加前缀）并计算新的统计信息：

```shell
curl -X POST "localhost:9200/twitter/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "message_stats" : {
            "string_stats" : {
                "field" : "message.keyword",
                "script" : {
                    "lang": "painless",
                    "source": "params.prefix + _value",
                    "params" : {
                        "prefix" : "Message: "
                    }
                }
            }
        }
    }
}
'

```

#### 缺省值

缺省值和其他聚合类似，不赘述。

### 求和聚合

求和聚合是单值的指标聚合，汇总从聚合文档中提取的数值。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "constant_score" : {
            "filter" : {
                "match" : { "type" : "hat" }
            }
        }
    },
    "aggs" : {
        "hat_prices" : { "sum" : { "field" : "price" } }
    }
}
'
```

### 热门聚合

`top_hits`指标聚合器跟踪要聚合的最相关文档。该聚合器旨在用作子聚合器，以便可以按存储分区汇总最匹配的文档。

`top_hits`聚合器可以有效地用于通过存储桶聚合器按某些字段对结果集进行分组。一个或多个存储桶聚合器确定将结果集切成哪些属性。

#### 可选项

- `from`：获取的第一个结果的偏移量。
- `size`：每个存储桶返回的最匹配匹配项的最大数量。默认情况下返回前三个匹配项。
- `sort`：匹配项的排序规则。默认情况下匹配项按`score`排序。

#### 支持每个命中的特性

`top_hits`聚合返回常规搜索命中，因为可以支持许多每次命中特性。

- `Highlighting`：您从搜索结果中的一个或多个字段中获取突出显示的摘要，以便向用户显示查询匹配的位置。
- `Explain`：解释每个匹配的得分如何计算。
- `Named filters and queries`：每个过滤器和查询都可以在其顶层定义中接受`_name`
- `Source filtering`：允许控制每次匹配时如何返回`_source`字段。
- `Stored fields`：允许有选择地为搜索命中表示的每个文档加载特定的存储字段。
- `Script fields`：允许为每个匹配返回执行脚本结果（基于不同的字段）。
- `Doc value fields`：允许返回每个匹配项的字段的doc值表示形式。
- `Include versions`：返回每个搜索命中的版本。
- `Include Sequence Numbers and Primary Terms`：返回每个搜索命中的最后修改的序列号和主要术语。

#### 例子

按类型对销售进行分组，并且按类型将最后的销售显示。对于每个销售，`source`中仅包含日期和价格字段。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs": {
        "top_tags": {
            "terms": {
                "field": "type",
                "size": 3
            },
            "aggs": {
                "top_sales_hits": {
                    "top_hits": {
                        "sort": [
                            {
                                "date": {
                                    "order": "desc"
                                }
                            }
                        ],
                        "_source": {
                            "includes": [ "date", "price" ]
                        },
                        "size" : 1
                    }
                }
            }
        }
    }
}
'

```

#### 字段折叠案例

字段折叠或结果分组是一项功能，可将结果集按逻辑分组，然后按组返回顶层文档。

组的顺序由组中第一个文档的相关性确定。

在Elasticsearch中，这可以通过将`top_hits`聚合器包装为子聚合器的存储桶聚合器来实现。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "body": "elections"
    }
  },
  "aggs": {
    "top_sites": {
      "terms": {
        "field": "domain",
        "order": {
          "top_hit": "desc"
        }
      },
      "aggs": {
        "top_tags_hits": {
          "top_hits": {}
        },
        "top_hit" : {
          "max": {
            "script": {
              "source": "_score"
            }
          }
        }
      }
    }
  }
}
'

```

#### 嵌套或反向嵌套聚合器中的top_hits支持

如果将`top_hits`聚合器包装在`nested`或`reverse_nested`聚合器中，则将返回嵌套的匹配。

```shell
# 创建映射
curl -X PUT "localhost:9200/sales?pretty" -H 'Content-Type: application/json' -d'
{
    "mappings": {
        "properties" : {
            "tags" : { "type" : "keyword" },
            "comments" : { 
                "type" : "nested",
                "properties" : {
                    "username" : { "type" : "keyword" },
                    "comment" : { "type" : "text" }
                }
            }
        }
    }
}
'

# 创建文档
curl -X PUT "localhost:9200/sales/_doc/1?refresh&pretty" -H 'Content-Type: application/json' -d'
{
    "tags": ["car", "auto"],
    "comments": [
        {"username": "baddriver007", "comment": "This car could have better brakes"},
        {"username": "dr_who", "comment": "Where's the autopilot? Can't find it"},
        {"username": "ilovemotorbikes", "comment": "This car has two extra wheels"}
    ]
}
'

# 在嵌套聚合中使用top_hits
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "term": { "tags": "car" }
    },
    "aggs": {
        "by_sale": {
            "nested" : {
                "path" : "comments"
            },
            "aggs": {
                "by_user": {
                    "terms": {
                        "field": "comments.username",
                        "size": 1
                    },
                    "aggs": {
                        "by_nested": {
                            "top_hits":{}
                        }
                    }
                }
            }
        }
    }
}
'


```

### 值计数聚合

值技术聚合是单值指标聚合，用于计算从汇总文档中提取的值的数量。

通常，值计数聚合器将与其他单值聚合一起使用。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "types_count" : { "value_count" : { "field" : "type" } }
    }
}
'
```

### 绝对中位差（MAD）聚合

绝对中位差聚合是单值聚合，这种单值聚合近似于其搜索结果的绝对中位差。

绝对中位差是变化性的量度。

以下参考维基百科的定义：

在统计学中，绝对中位数MAD是对单变量数值型数据的样本偏差的一种鲁棒性测量。同时也可以表示由样本的MAD估计得出的总体参数。

对于单变量数据集$X_1,X_2,\cdots X_n$，MAD定义为数据点到中位数的绝对偏差的中位数：
$$
MAD=median(|X_i-median(X)|)
$$
示例：考虑数据集(1, 1, 2, **2**, 4, 6, 9)，它的中位数为2。数据点到2的绝对偏差为(1, 1, 0, 0, 2, 4, 7)，该偏差列表的中位数为1（因为排序后的绝对偏差为(0, 0, 1, **1**, 2, 4, 7)）。所以该数据的绝对中位差为1。

```shell
curl -X GET "localhost:9200/reviews/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "review_average": {
      "avg": {
        "field": "rating"
      }
    },
    "review_variability": {
      "median_absolute_deviation": {
        "field": "rating" 
      }
    }
  }
}
'
```

#### 近似值

计算中值绝对偏差的原生实现将整个样本存储在内存中，因此此聚合计算出一个近似值。

在资源使用和TDigest分位数近似的准确性之间进行权衡，因此，此聚合近似中值绝对偏差的精度由`compression`参数控制。

```shell
curl -X GET "localhost:9200/reviews/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "review_variability": {
      "median_absolute_deviation": {
        "field": "rating",
        "compression": 100
      }
    }
  }
}
'
```

## 存储桶聚合

存储桶聚合不像指标聚合那样计算字段的指标，而是创建文档存储桶。

每个存储桶都与一个标准（取决于聚合类型）相关联，该标准确定当前上下文中的文档是否“落入”其中。换句话说，存储桶有效定义了文档集。

存储桶本身之外，存储桶聚合还计算并返回落入每个存储桶的文档数量。

与指标聚合相反，存储桶聚合可以保存子聚合。这些子聚合针对父聚合创建的存储桶进行聚合。

单个响应中允许的最大存储桶数量是名为`search.max_buckets`的动态集群设置的限制。默认是10000。

### 邻接矩阵聚合

该存储桶聚合返回一个临接矩阵形式。请求提供了一个命名过滤器表达式的集合，类似于`filter`聚合请求。响应中的每个存储桶代表相交过滤器矩阵中的一个非空单元。

相交桶（例如A＆C）使用两个过滤器名称并使用`&`字符分隔的组合来标记。相交桶的过滤器名称互换位置和原本的相交桶本质上是一致的。

ES对过滤器名称字符串进行排序，并始终使用一对中的最低值作为“＆”分隔符左侧的值。

可以传递一个可选的`separator`参数作为过滤器分隔符的字符串。

```shell
curl -X PUT "localhost:9200/emails/_bulk?refresh&pretty" -H 'Content-Type: application/json' -d'
{ "index" : { "_id" : 1 } }
{ "accounts" : ["hillary", "sidney"]}
{ "index" : { "_id" : 2 } }
{ "accounts" : ["hillary", "donald"]}
{ "index" : { "_id" : 3 } }
{ "accounts" : ["vladimir", "donald"]}
'
curl -X GET "localhost:9200/emails/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs" : {
    "interactions" : {
      "adjacency_matrix" : {
        "filters" : {
          "grpA" : { "terms" : { "accounts" : ["hillary", "sidney"] }},
          "grpB" : { "terms" : { "accounts" : ["donald", "mitt"] }},
          "grpC" : { "terms" : { "accounts" : ["vladimir", "nigel"] }}
        }
      }
    }
  }
}
'

```

#### 限制

对于$N$个过滤器，所产的存储桶矩阵可以为$N^2/2$，默认情况下最大值为100个过滤器。这个设置可以通过更改`index.max_adjacency_matrix_filters`索引级别来配置（8.0后废弃）。

### 自动间隔日期直方图聚合

自动间隔日期直方图聚合是与日期直方图聚合类似的多桶聚合，不同之处在于它不是提供一个间隔来用作每个桶的宽度，而是提供了目标数量的存储桶，指示所需的存储桶数量，并且自动选择存储桶的间隔以最佳地实现该目标。返回的存储桶数将始终小于或等于此目标数。

`buckets`字段是可选的，如果没有指定，默认是10个存储桶。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "sales_over_time" : {
            "auto_date_histogram" : {
                "field" : "date",
                "buckets" : 10
            }
        }
    }
}
'

```

#### 键

在内部，日期表示为一个64位数字，表示自该时间点以来的毫秒数。

这些时间戳会作为存储桶的键返回，可以指定`format`参数，将时间戳转换为格式化日期输出到`key_as_string`。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "sales_over_time" : {
            "auto_date_histogram" : {
                "field" : "date",
                "buckets" : 5,
                "format" : "yyyy-MM-dd" 
            }
        }
    }
}
'

```

#### 间隔

根据聚合收集的数据选择返回的存储桶中的间隔，因此返回的存储桶数量小于等于请求的存储桶数量，可能返回的间隔如下：

|  单位   |           间隔数            |
| :-----: | :-------------------------: |
| seconds |     1，5，10和30的倍数      |
| minutes |     1，5，10和30的倍数      |
|  hours  |       1，3，12的倍数        |
|  days   |         1和7的倍数          |
| months  |         1和3的倍数          |
|  years  | 1，5，10，20，50和100的倍数 |

在最坏的情况下，如果每天的存储桶数超出了请求的存储桶数，则返回的存储桶数将是请求的存储桶数的1/7。

#### 时区

日期时间存储以UTC的时间形式的Elasticsearch中。默认情况下，所有存储桶和取整都是以UTC时间完成。

`time_zone`参数可以用于指定存储桶使用的不同的时区。

```shell
curl -X PUT "localhost:9200/my_index/log/1?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "date": "2015-10-01T00:30:00Z"
}
'
curl -X PUT "localhost:9200/my_index/log/2?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "date": "2015-10-01T01:30:00Z"
}
'
curl -X PUT "localhost:9200/my_index/log/3?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "date": "2015-10-01T02:30:00Z"
}
'
curl -X GET "localhost:9200/my_index/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "by_day": {
      "auto_date_histogram": {
        "field":     "date",
        "buckets" : 3,
        "time_zone": "-01:00"
      }
    }
  }
}
'


```

#### 最小间隔参数

`minimum_interval`允许调用者指定使用的最小取整间隔，这可以使得收集处理过程更高效，因为聚合不会尝试以任何低于`minimum_interval`的间隔进行舍入。

`minimum_interval`可以接收的单位

- year
- month
- day
- hour
- minute
- second

### 子聚合

一个特殊的单聚合桶聚合，用于选择具有指定类型的子文档，如`join`字段中所定义。

子聚合有一个可选的参数

- `type`：应该选择的子文档类型

例如，假设我们有一个问题和答案的索引。答案类型在映射中具有以下`join`字段：

```shell
curl -X PUT "localhost:9200/child_example?pretty" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "join": {
        "type": "join",
        "relations": {
          "question": "answer"
        }
      }
    }
  }
}
'
```

`question`文档包含标签字段，`answer`文档包含所有者字段。

新建一个`question`文档

```shell
curl -X PUT "localhost:9200/child_example/_doc/1?pretty" -H 'Content-Type: application/json' -d'
{
  "join": {
    "name": "question"
  },
  "body": "<p>I have Windows 2003 server and i bought a new Windows 2008 server...",
  "title": "Whats the best way to file transfer my site from server to a newer one?",
  "tags": [
    "windows-server-2003",
    "windows-server-2008",
    "file-transfer"
  ]
}
'
```

新建两个`answer`文档

```shell
curl -X PUT "localhost:9200/child_example/_doc/2?routing=1&pretty" -H 'Content-Type: application/json' -d'
{
  "join": {
    "name": "answer",
    "parent": "1"
  },
  "owner": {
    "location": "Norfolk, United Kingdom",
    "display_name": "Sam",
    "id": 48
  },
  "body": "<p>Unfortunately you\u0027re pretty much limited to FTP...",
  "creation_date": "2009-05-04T13:45:37.030"
}
'
curl -X PUT "localhost:9200/child_example/_doc/3?routing=1&refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "join": {
    "name": "answer",
    "parent": "1"
  },
  "owner": {
    "location": "Norfolk, United Kingdom",
    "display_name": "Troll",
    "id": 49
  },
  "body": "<p>Use Linux...",
  "creation_date": "2009-05-05T13:45:37.030"
}
'

```

下面的示例返回顶部的问题标签，每个标签返回顶部的答案所有者

```shell
curl -X POST "localhost:9200/child_example/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "top-tags": {
      "terms": {
        "field": "tags.keyword",
        "size": 10
      },
      "aggs": {
        "top-answers": {
          "children": {
            "type" : "answer" 
          },
          "aggs": {
            "top-names": {
              "terms": {
                "field": "owner.display_name.keyword",
                "size": 10
              }
            }
          }
        }
      }
    }
  }
}
'

```

### 复合聚合

复合聚合是一个多存储桶聚合，可以从不同的来源创建复合存储桶。

与其他多存储桶聚合不同，复合聚合可用于从多级聚合有效分页所有桶。

复合聚合提供了一种方式以流传输特定聚合的所有存储桶，类似于滚动对文档所做的操作。

复合存储桶是根据为每个文档提取/创建的值的组合构建的，每个组合都被视为复合存储桶

#### 值来源

`sources`参数用于构建复合聚合桶的源。

定义`sources`的顺序很重要，因为它还控制键的返回顺序。

传递给`sources`的名称必须是唯一的，有以下三种不同类型的`sources`：

- `terms`：其值源等效于简单`terms`聚合，这些值是从字段或脚本中提取的，就像`terms`“聚合”一样。也可以使用脚本来为复合存储桶创建值。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "product": { "terms" : { "field": "product" } } }
                ]
            }
        }
     }
}
'

curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    {
                        "product": {
                            "terms" : {
                                "script" : {
                                    "source": "doc[\u0027product\u0027].value",
                                    "lang": "painless"
                                }
                            }
                        }
                    }
                ]
            }
        }
    }
}
'

```

#### 直方图

`histogram`值源可以应用于数字值，以在这些值上建立固定的大小间隔。

`interval`参数定义了如何转换数值。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "histo": { "histogram" : { "field": "price", "interval": 5 } } }
                ]
            }
        }
    }
}
'
```

#### 日期直方图

`date_histogram`与`histogram`值源相似，但间隔由日期/时间表达式指定，可用的表达式间隔为

- year
- quarter
- month
- week
- day
- hour
- minute
- second

`format`参数可以指定输出格式化的日期。

`time_zone`参数可用于指示存储段应使用其他时区。

`offset`参数指定的正（+）或负偏移（-）持续时间更改每个存储桶的起始值。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    {
                        "date": {
                            "date_histogram" : {
                                "field": "timestamp",
                                "calendar_interval": "1d",
                                "format": "yyyy-MM-dd" 
                            }
                        }
                    }
                ]
            }
        }
    }
}
'

```

#### 混合不同的值源

`sources`参数接收是一个值的数组，因此可以混合不同的值源来创建复合存储桶。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d" } } },
                    { "product": { "terms": {"field": "product" } } }
                ]
            }
        }
    }
}
'

```

#### 顺序

默认情况下，复合存储桶按其自然顺序排序。值按其值的升序排序。

当请求多个值源时，将对每个值源进行排序，将组合存储桶的第一个值与另一个组合存储桶的第一个值进行比较，如果它们相等，则将组合存储桶中的下一个值与另一个存储桶的下一个值继续比较。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d", "order": "desc" } } },
                    { "product": { "terms": {"field": "product", "order": "asc" } } }
                ]
            }
        }
    }
}
'
```

#### 缺失bucket

默认情况下，将忽略没有给定源值的文档。可以设置`missing_bucket`为`true`包括那些没有给定值源的文档。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "product_name": { "terms" : { "field": "product", "missing_bucket": true } } }
                ]
            }
        }
     }
}
'
```

#### 大小

`size`参数来定义应该返回多少个复合桶。

#### 分页

使用`after`参数可以检索下一页的结果集。为了获取下一页的结果集，可以传递返回值中`after_key`的内容。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "size": 2,
                 "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d", "order": "desc" } } },
                    { "product": { "terms": {"field": "product", "order": "asc" } } }
                ],
                "after": { "date": 1494288000000, "product": "mad max" } 
            }
        }
    }
}
'
```

#### 提前终止

为了获得最佳性能，应在索引上设置索引排序，以使其与组合聚合中的部分或全部源顺序匹配。

复合聚合也将尝试尽早终止未排序的索引。

先定义索引的数据结构

```shell
curl -X PUT "localhost:9200/twitter?pretty" -H 'Content-Type: application/json' -d'
{
    "settings" : {
        "index" : {
            "sort.field" : ["username", "timestamp"],   
            "sort.order" : ["asc", "desc"]              
        }
    },
    "mappings": {
        "properties": {
            "username": {
                "type": "keyword",
                "doc_values": true
            },
            "timestamp": {
                "type": "date"
            }
        }
    }
}
'

```

为了优化提早终止，建议将请求中的`track_total_hits`设置为`false`。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "track_total_hits": false,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                "sources" : [
                    { "user_name": { "terms" : { "field": "user_name" } } },
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d", "order": "desc" } } }
                ]
            }
        }
     }
}
'
```

#### 子聚合

复合聚合也可以拥有子聚合。这些子聚合可用于计算其他存储桶或有关此父聚合创建的每个复合存储桶的统计信息。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "my_buckets": {
            "composite" : {
                 "sources" : [
                    { "date": { "date_histogram": { "field": "timestamp", "calendar_interval": "1d", "order": "desc" } } },
                    { "product": { "terms": {"field": "product" } } }
                ]
            },
            "aggregations": {
                "the_avg": {
                    "avg": { "field": "price" }
                }
            }
        }
    }
}
'
```

#### 管道聚合

复合聚合当前与管道聚合不兼容，在大多数情况下也没有意义。

### 日期直方图聚合

是多桶聚合，类似于正常的直方图聚合，但只能与日期或日期范围值一起使用。

日期直方图聚合和普通直方图聚合的区别是：日期直方图聚合可以使用日期时间表达式。

配置日期直方图聚合时，可以通过两种方式指定时间间隔：

- 日历感知时间间隔：夏令时会更改特定日期的时间长度，月份会更改不同的天数，并且leap秒可用于特定年份。
- 固定时间间隔：固定间隔始终是SI单位的倍数，并且不会根据日历环境而变化。

#### 日历感知时间间隔

可以使用`calendar_interval`参数设置日历间隔。日历间隔只能以单位的“单个”数量（1d，1M等）指定。指定，不支持`2d`等倍数，并且会引发异常。支持的单位值

- minute
- hour
- day
- week
- month
- quarter
- year

#### 固定时间间隔

可以使用`fixed-interval`参数设置固定时间间隔。可以以任意多个受支持单位指定固定间隔。

- seconds
- minutes
- hours
- days

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "fixed_interval" : "30d"
            }
        }
    }
}
'
```

#### Keys

日期表示为64位数字，表示自时间段以来的时间戳（以毫秒为单位）。这些时间戳作为存储桶的键名称返回。 `key_as_string`是使用`format`参数规范转换为格式化日期字符串的相同时间戳。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "sales_over_time" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "1M",
                "format" : "yyyy-MM-dd" 
            }
        }
    }
}
'

```

#### 时区

日期时间在Elasticsearch中是以UTC形式存储的。使用`time-zone`参数表示存储桶应使用其他时区。

```shell
curl -X PUT "localhost:9200/my_index/_doc/1?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "date": "2015-10-01T00:30:00Z"
}
'
curl -X PUT "localhost:9200/my_index/_doc/2?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "date": "2015-10-01T01:30:00Z"
}
'
curl -X GET "localhost:9200/my_index/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "by_day": {
      "date_histogram": {
        "field":     "date",
        "calendar_interval":  "day",
        "time_zone": "-01:00"
      }
    }
  }
}
'


```

#### 偏移量

使用offset参数以指定的正（+）或负偏移（-）时间段更改每个存储区的起始值，例如1h表示一个小时，或1d表示一天。

```shell
curl -X PUT "localhost:9200/my_index/_doc/1?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "date": "2015-10-01T05:30:00Z"
}
'
curl -X PUT "localhost:9200/my_index/_doc/2?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "date": "2015-10-01T06:30:00Z"
}
'
curl -X GET "localhost:9200/my_index/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "by_day": {
      "date_histogram": {
        "field":     "date",
        "calendar_interval":  "day",
        "offset":    "+6h"
      }
    }
  }
}
'

```

#### 键作为响应值

将`keyed`标志设置为`true`会将唯一的字符串键与每个存储桶相关联，并以散列而不是数组的形式返回范围。

### 日期范围聚合

日期范围聚合的`from`和`to`参数都可以指定[日期数学表达式](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math)，使用`format`参数指定其输入格式。

请注意，此聚合包括起始值，但不包括每个范围的起始值。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs": {
        "range": {
            "date_range": {
                "field": "date",
                "format": "MM-yyyy",
                "ranges": [
                    { "to": "now-10M/M" }, 
                    { "from": "now-10M/M" } 
                ]
            }
        }
    }
}
'

```

#### 日期范围聚合中的时区

通过指定`time_zone`参数，可以将日期从另一个时区转换为UTC。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
   "aggs": {
       "range": {
           "date_range": {
               "field": "date",
               "time_zone": "CET",
               "ranges": [
                  { "to": "2016/02/01" }, 
                  { "from": "2016/02/01", "to" : "now/d" }, 
                  { "from": "now/d" }
              ]
          }
      }
   }
}
'
```

### 多元化的采样器聚合

和`sampler`聚合一样，这是一个过滤聚合，用于将任何子聚合的处理限制为得分最高的文档的样本。

`field`和`script`参数用于提供用于重复数据删除的值。

`max_docs_per_value`参数用于控制在一个共享相同值的任何分片上收集的最大文档数，默认值是1。

如果选择`field`或`script`为单个文档产生多个值，则聚合将引发错误。

想在StackOverflow论坛帖子上看到哪些标签与`#elasticsearch`密切相关，但忽略了一些多产用户的影响，这些用户倾向于将`#Kibana`误称为`#Cabana`。

```shell
curl -X POST "localhost:9200/stackoverflow/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "query_string": {
            "query": "tags:elasticsearch"
        }
    },
    "aggs": {
        "my_unbiased_sample": {
            "diversified_sampler": {
                "shard_size": 200,
                "field" : "author"
            },
            "aggs": {
                "keywords": {
                    "significant_terms": {
                        "field": "tags",
                        "exclude": ["elasticsearch"]
                    }
                }
            }
        }
    }
}
'
```

`shard_size`参数限制在每个分片上处理的样本中收集了多少个得分最高的文档。

可选的`execution_hint`参数设置可以影响重复数据删除值的管理。

##### 限制

- 不能嵌套在`breadth_first`聚合中使用
- 有限的重复数据删除逻辑。
- 没有专门的地理/日期字段语法

### 过滤聚合

定义当前文档集上下文中与指定过滤器匹配的所有文档的单个存储桶。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "t_shirts" : {
            "filter" : { "term": { "type": "t-shirt" } },
            "aggs" : {
                "avg_price" : { "avg" : { "field" : "price" } }
            }
        }
    }
}
'
```

### 多过滤器聚合

定义一个多存储桶聚合，其中每个存储桶都与一个过滤器相关联。每个存储桶将收集与其关联的过滤器匹配的所有文档。

```shell
curl -X PUT "localhost:9200/logs/_bulk?refresh&pretty" -H 'Content-Type: application/json' -d'
{ "index" : { "_id" : 1 } }
{ "body" : "warning: page could not be rendered" }
{ "index" : { "_id" : 2 } }
{ "body" : "authentication error" }
{ "index" : { "_id" : 3 } }
{ "body" : "warning: connection timed out" }
'
curl -X GET "localhost:9200/logs/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs" : {
    "messages" : {
      "filters" : {
        "filters" : {
          "errors" :   { "match" : { "body" : "error"   }},
          "warnings" : { "match" : { "body" : "warning" }}
        }
      }
    }
  }
}
'
```

#### 匿名过滤器

`filters`字段也可以提供过滤器数组作为参数，数组中的元素都是匿名过滤器。

#### 其他存储桶

`other_bucket`参数可以将不匹配任何给定过滤器的文档放置到一个存储桶后返回。

- `false`不计算其他不匹配任何给定过滤器的文档的存储桶。
- `true`返回不匹配任何给定过滤器的文档的存储桶。默认情况下命名为`_other_`，如果使用匿名过滤器，则返回最后一个存储桶。

`other_bucket_key`参数可以指定其他不匹配任何给定过滤器的文档的存储桶的名称。

```shell
curl -X PUT "localhost:9200/logs/_doc/4?refresh&pretty" -H 'Content-Type: application/json' -d'
{
  "body": "info: user Bob logged out"
}
'
curl -X GET "localhost:9200/logs/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs" : {
    "messages" : {
      "filters" : {
        "other_bucket_key": "other_messages",
        "filters" : {
          "errors" :   { "match" : { "body" : "error"   }},
          "warnings" : { "match" : { "body" : "warning" }}
        }
      }
    }
  }
}
'
```

### 地理距离聚合

一个多存储桶聚合，作用在地理坐标字段，概念上和`range`聚合类似。

聚合评估每个文档值到`origin`的距离，并根据范围确定它所属的存储桶（如果文档和原点之间的距离在存储桶的距离范围内，则文档属于存储桶）。

`origin`参数指定原点坐标，支持[Geo-point 类型](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-point.html) 

`distance`的单位默认是米，可以使用`unit`参数指定为

- mi（miles）英里
- in（inches）英寸
- yd（yards）码
- km（kilometers）千米
- cm（centimeters）厘米
- mm（millimeters）毫米

```shell
curl -X PUT "localhost:9200/museums?pretty" -H 'Content-Type: application/json' -d'
{
    "mappings": {
        "properties": {
            "location": {
                "type": "geo_point"
            }
        }
    }
}
'
curl -X POST "localhost:9200/museums/_bulk?refresh&pretty" -H 'Content-Type: application/json' -d'
{"index":{"_id":1}}
{"location": "52.374081,4.912350", "name": "NEMO Science Museum"}
{"index":{"_id":2}}
{"location": "52.369219,4.901618", "name": "Museum Het Rembrandthuis"}
{"index":{"_id":3}}
{"location": "52.371667,4.914722", "name": "Nederlands Scheepvaartmuseum"}
{"index":{"_id":4}}
{"location": "51.222900,4.405200", "name": "Letterenhuis"}
{"index":{"_id":5}}
{"location": "48.861111,2.336389", "name": "Musée du Louvre"}
{"index":{"_id":6}}
{"location": "48.860000,2.327000", "name": "Musée d\u0027Orsay"}
'
curl -X POST "localhost:9200/museums/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "rings_around_amsterdam" : {
            "geo_distance" : {
                "field" : "location",
                "origin" : "52.3760, 4.894",
                "ranges" : [
                    { "to" : 100000 },
                    { "from" : 100000, "to" : 300000 },
                    { "from" : 300000 }
                ]
            }
        }
    }
}
'
```

有两种距离计算方式，可以使用`distance_type`参数指定计算距离的方式：

- `arc`：默认的距离计算方式，是最准确的。
- `plane`：最快但最不准确的。

### 地理哈希网格聚合

是多桶聚合，适用于`geo_point`字段，将点分组到代表网格中单元的桶中。

生成的网格可能是稀疏的，并且仅包含具有匹配数据的像元。每个单元格均使用用户可定义精度的`geohash`进行标记。

高精度的地理哈希具有较长的字符串长度，代表仅覆盖较小区域的单元。

低精度的地理哈希具有较短的字符串长度，代表每个单元都覆盖大面积的单元格。

此聚合中使用的`geohash`可以在1到12之间选择精度，使用`precision`参数指定。

指定的字段必须为`geo_point`类型（只能在mappings中显式设置），并且还可以包含geo_point字段数组，在这种情况下，将在聚合过程中考虑所有点。

```shell
curl -X PUT "localhost:9200/museums?pretty" -H 'Content-Type: application/json' -d'
{
    "mappings": {
          "properties": {
              "location": {
                  "type": "geo_point"
              }
          }
    }
}
'
curl -X POST "localhost:9200/museums/_bulk?refresh&pretty" -H 'Content-Type: application/json' -d'
{"index":{"_id":1}}
{"location": "52.374081,4.912350", "name": "NEMO Science Museum"}
{"index":{"_id":2}}
{"location": "52.369219,4.901618", "name": "Museum Het Rembrandthuis"}
{"index":{"_id":3}}
{"location": "52.371667,4.914722", "name": "Nederlands Scheepvaartmuseum"}
{"index":{"_id":4}}
{"location": "51.222900,4.405200", "name": "Letterenhuis"}
{"index":{"_id":5}}
{"location": "48.861111,2.336389", "name": "Musée du Louvre"}
{"index":{"_id":6}}
{"location": "48.860000,2.327000", "name": "Musée d\u0027Orsay"}
'
curl -X POST "localhost:9200/museums/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggregations" : {
        "large-grid" : {
            "geohash_grid" : {
                "field" : "location",
                "precision" : 3
            }
        }
    }
}
'
```

#### 高精度请求

当请求详细的存储桶（通常用于显示“放大”的地图）时，应该使用诸如`geo_bounding_box`之类的过滤器来缩小主要区域，否则可能会创建并返回数百万个存储桶。

```shell
curl -X POST "localhost:9200/museums/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggregations" : {
        "zoomed-in" : {
            "filter" : {
                "geo_bounding_box" : {
                    "location" : {
                        "top_left" : "52.4, 4.9",
                        "bottom_right" : "52.3, 5.0"
                    }
                }
            },
            "aggregations":{
                "zoom1":{
                    "geohash_grid" : {
                        "field": "location",
                        "precision": 8
                    }
                }
            }
        }
    }
}
'

```

#### 具有附加边界框过滤的请求

`geohash_grid`聚合支持可选的`bounds`参数，该参数将考虑的点限制为所提供的边界内的点。

`bounds`参数以与地理边界框查询中指定的边界相同的所有可[接受格式](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-geo-bounding-box-query.html#query-dsl-geo-bounding-box-query-accepted-formats)来接受边界框。

### 地理平铺网格聚合

一个多存储桶聚合，作用在地理坐标字段，并将点分组为代表网格中单元格的桶。

生成的网格可能是稀疏的，并且仅包含具有匹配数据的像元。

每个单元格对应许多在线地图站点使用的地图图块。每个单元格均使用`{zoom} / {x} / {y}`格式标记，其中`zoom`等于用户指定的精度。使用`precision`参数指定精度，精度的范围为`0-29`

- 高精度键的x和y范围较大，并且代表仅覆盖较小区域的图块。
- 低精度键的x和y范围较小，并且代表各自覆盖较大面积的图块。

低精度请求

```shell
curl -X POST "localhost:9200/museums/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggregations" : {
        "large-grid" : {
            "geotile_grid" : {
                "field" : "location",
                "precision" : 8
            }
        }
    }
}
'

```

高精度请求和GetHash聚合类似，建议限定边界后使用聚合。

### 全局聚合

在搜索执行上下文中定义**所有文档**的单存储桶聚合，不管其他聚合后搜索出什么结果，都会使用搜索执行上下文中的所有文档。

全局聚合器只能作为顶级聚合器放置，因为将全局聚合器嵌入另一个存储桶聚合器中没有意义。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : { "type" : "t-shirt" }
    },
    "aggs" : {
        "all_products" : {
            "global" : {}, 
            "aggs" : { 
                "avg_price" : { "avg" : { "field" : "price" } }
            }
        },
        "t_shirts": { "avg" : { "field" : "price" } }
    }
}
'
```

### 直方图聚合

基于多存储桶值源的聚合，可以应用于从文档中提取的数值或数值范围值。

文档对应的值落入存储桶的计算公式如下：`bucket_key=Math.floor((value - offset) / interval) * interval + offset`

对于范围值，文档可以分为多个存储桶。

`interval`参数必须是一个正数值，`offset`参数必须是一个在`[0,interval)`的数值。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50
            }
        }
    }
}
'
```

默认情况下，响应将用空桶填充直方图中的空白。由于`min_doc_count`，可以更改该请求并请求具有更高最小计数的存储桶。

通过`extended_bounds`设置，你可以“强制”直方图聚合构建存储桶的范围，使用`min`和`max`参数指定存储桶构建的最小值和最大值，如果小于`min`还有对应范围的存储桶，则会显示该范围的存储桶，`max`同理。

将直方图聚合嵌套在过滤器聚合中使用

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "constant_score" : { "filter": { "range" : { "price" : { "to" : "500" } } } }
    },
    "aggs" : {
        "prices" : {
            "histogram" : {
                "field" : "price",
                "interval" : 50,
                "extended_bounds" : {
                    "min" : 0,
                    "max" : 500
                }
            }
        }
    }
}
'
```

#### 偏移量

默认情况下，存储桶键从0开始，然后以`interval`相等的间隔继续。

存储桶的边界可以使用`offset`选项进行移动。

### IP范围聚合

和日期范围聚合类似，IP范围聚合作用在`IP`类型字段。

可以使用`key`参数自定义输出结果的键。

```shell
curl -X GET "localhost:9200/ip_addresses/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs": {
        "ip_ranges": {
            "ip_range": {
                "field": "ip",
                "ranges": [
                    { "key": "infinity", "to" : "10.0.0.5" },
                    { "key": "and-beyond", "from" : "10.0.0.5" }
                ],
                "keyed": true
            }
        }
    }
}
'

```

### 缺省聚合

基于字段数据的单个存储桶聚合，可在当前文档集上下文中创建所有缺少字段值（实际上是缺少字段或配置了NULL值）的所有文档的存储桶。

```shell
curl -X POST "localhost:9200/sales/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "products_without_a_price" : {
            "missing" : { "field" : "price" }
        }
    }
}
'
```

### 嵌套聚合

一种特殊的单存储桶聚合，可以聚合嵌套文档。

定义数据格式

```shell
curl -X PUT "localhost:9200/products/_doc/0?pretty" -H 'Content-Type: application/json' -d'
{
  "name": "LED TV", 
  "resellers": [
    {
      "reseller": "companyA",
      "price": 350
    },
    {
      "reseller": "companyB",
      "price": 500
    }
  ]
}
'

```

插入一条记录

```shell
curl -X PUT "localhost:9200/products/_doc/0?pretty" -H 'Content-Type: application/json' -d'
{
  "name": "LED TV", 
  "resellers": [
    {
      "reseller": "companyA",
      "price": 350
    },
    {
      "reseller": "companyB",
      "price": 500
    }
  ]
}
'

```

使用嵌套聚合，取出经销商中出售价格最低的。

```shell
curl -X GET "localhost:9200/products/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : { "name" : "led tv" }
    },
    "aggs" : {
        "resellers" : {
            "nested" : {
                "path" : "resellers"
            },
            "aggs" : {
                "min_price" : { "min" : { "field" : "resellers.price" } }
            }
        }
    }
}
'
```

嵌套聚合要求嵌套文档在最顶层文档的`path`。

### 父聚合

一种特殊的单存储桶聚合，用于选择具有指定类型（在连接字段中定义）的父级文档。

父聚合有一个单独的选项

- `type`应该选中的子类型

示例和子聚合的实例类似，父聚合使用如下

```shell
curl -X POST "localhost:9200/parent_example/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "top-names": {
      "terms": {
        "field": "owner.display_name.keyword",
        "size": 10
      },
      "aggs": {
        "to-questions": {
          "parent": {
            "type" : "answer" 
          },
          "aggs": {
            "top-tags": {
              "terms": {
                "field": "tags.keyword",
                "size": 10
              }
            }
          }
        }
      }
    }
  }
}
'

```

### 范围聚合

基于多存储桶值源的聚合，使用户能够定义一组范围，每个范围代表一个桶。

请注意，此聚合包括起始值，但不包括每个范围的起始值。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "aggs": {
        "price_range": {
            "range": {
                "field": "price",
                "ranges": [
                    {
                        "to": 1000
                    },
                    {
                        "from": 2000,
                        "to": 2500
                    },
                    {
                        "from": 3000
                    }
                ]
            },
            "aggs": {
            	"price_stats": {
            		"stats": {
            			"field": "price"
            		}
            	}
            }
        }
    }
}
'
```

### 稀有项聚合

基于多桶值源的聚合，可找到“稀有”项-分布较长的项，并且不频繁。

从概念上讲，这类似于`terms`聚合，按`_count`升序排序。

`rare_terms`参数

|   参数名称    |                     描述                     | 是否必填 | 默认值 |
| :-----------: | :------------------------------------------: | :------: | :----: |
|     field     |             用于稀有项聚合的字段             |    是    |        |
| max_doc_count |             项应出现的最大文档数             |   可选   |   1    |
|   percision   | 精度，精度越高占用内存越大，不能小于0.000001 |   可选   |  0.01  |
|    include    |                 聚合包含的项                 |   可选   |        |
|    exclude    |                聚合不包含的项                |   可选   |        |
|    missing    |                    缺省值                    |   可选   |        |

### 反向嵌套聚合

一种特殊的单存储桶聚合，可以从嵌套文档中对父文档进行聚合。

`reverse_nested`聚合必须在`nested`聚合内定义。

`path`参数定义应嵌套的嵌套对象字段。默认是空。

```shell
# 定义数据结构
curl -X PUT "localhost:9200/issues?pretty" -H 'Content-Type: application/json' -d'
{
    "mappings": {
         "properties" : {
             "tags" : { "type" : "keyword" },
             "comments" : { 
                 "type" : "nested",
                 "properties" : {
                     "username" : { "type" : "keyword" },
                     "comment" : { "type" : "text" }
                 }
             }
         }
    }
}
'

# 查询数据
curl -X GET "localhost:9200/issues/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match_all": {}
  },
  "aggs": {
    "comments": {
      "nested": {
        "path": "comments"
      },
      "aggs": {
        "top_usernames": {
          "terms": {
            "field": "comments.username"
          },
          "aggs": {
            "comment_to_issue": {
              "reverse_nested": {}, 
              "aggs": {
                "top_tags_per_comment": {
                  "terms": {
                    "field": "tags"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
'

```

### 采样器聚合

一种筛选聚合，用于将任何子聚合的处理限制为得分最高的文档样本。

使用场景

- 将分析重点放在高相关性匹配上，而不是低质量匹配的可能很长的尾部上。
- 减少仅使用样本即可产生有用结果的聚合的运行成本。

在StackOverflow数据上查询流行术语javascript或较罕见的术语kibana将匹配许多文档-大多数文档都缺少单词Kibana。使用采集器聚合收集比较精确的样本进行`significant_terms`聚合查询。

```shell
curl -X POST "localhost:9200/stackoverflow/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "query_string": {
            "query": "tags:kibana OR tags:javascript"
        }
    },
    "aggs": {
        "sample": {
            "sampler": {
                "shard_size": 200
            },
            "aggs": {
                "keywords": {
                    "significant_terms": {
                        "field": "tags",
                        "exclude": ["kibana", "javascript"]
                    }
                }
            }
        }
    }
}
'
```

### 重要项汇总

重要项聚合返回集合中项出现感兴趣或异常的集合。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "terms" : {"force" : [ "British Transport Police" ]}
    },
    "aggregations" : {
        "significant_crime_types" : {
            "significant_terms" : { "field" : "crime_type" }
        }
    }
}
'
```

### 重要文本聚合

重要文本聚合返回集中出现有趣或异常的自由文本术语。

```shell
curl -X GET "localhost:9200/news/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {"content" : "Bird flu"}
    },
    "aggregations" : {
        "my_sample" : {
            "sampler" : {
                "shard_size" : 100
            },
            "aggregations": {
                "keywords" : {
                    "significant_text" : { "field" : "content" }
                }
            }
        }
    }
}
'
```

### 项聚合

基于多存储桶值源的聚合，其中动态构建桶，每个唯一项一个存储桶。

```shell
curl -X GET "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "aggs" : {
        "genres" : {
            "terms" : { "field" : "genre" } 
        }
    }
}
'
```

### 范围字段分桶的区别

文件会针对它们进入的每个存储桶进行计数，范围字段可能落入多个存储桶，可能计数值会有偏差。

查询范围不是聚合过滤器。下列查询的日期直方图聚合不会根据`query`的范围进行查询。

```shell
curl -X POST "localhost:9200/range_index/_search?size=0&pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
      "range": {
        "time_frame": {
          "gte": "2019-11-01",
          "format": "yyyy-MM-dd"
        }
      }
    },
    "aggs" : {
        "november_data" : {
            "date_histogram" : {
                "field" : "time_frame",
                "calendar_interval" : "day",
                "format": "yyyy-MM-dd"
              }
        }
    }
}
'
```

## 管道聚合

管道聚合工作于其他聚合的产生的输出，添加信息到输出树中。

管道聚合可以分为两类

- 父管道聚合：它能够计算新的存储桶或新的聚合以添加到现有存储桶中。
- 相邻管道聚合：它能够计算与该同级聚合处于同一级别的新聚合。

管道聚合可以通过使用`buckets_path`参数指示所需指标的路径来应用执行计算所需的聚合。

`buckets_path`参数语法

```properties
AGG_SEPARATOR       =  `>` ;
METRIC_SEPARATOR    =  `.` ;
AGG_NAME            =  <the name of the aggregation> ;
METRIC              =  <the name of the metric (in case of multi-value metrics aggregation)> ;
MULTIBUCKET_KEY     =  `[<KEY_NAME>]`
PATH                =  <AGG_NAME><MULTIBUCKET_KEY>? (<AGG_SEPARATOR>, <AGG_NAME> )* ( <METRIC_SEPARATOR>, <METRIC> ) ;
```

例如，路径`my_bucket> my_stats.avg`将指向`my_stats`指标中的avg值，该值包含在`my_bucket`存储桶聚合中。

管道聚合有两种方式处理数据中的空白，使用`gap_policy`参数指定处理方式

- `skip`：选项将缺失数据视为存储桶不存在。
- `insert_zeros`：选项将缺失数据处理为0。

### 平均存储桶聚合

平均存储桶聚合是相邻管道聚合。计算相邻聚合中指定指标的平均值。

#### 语法

`avg_bucket`聚合的参数

|   参数名称   |        描述        | 是否必须 | 默认值 |
| :----------: | :----------------: | :------: | :----: |
| buckets_path |     存储桶路径     |    是    |        |
|  gap_policy  | 处理空白数据的方式 |    否    |  skip  |
|    format    |   聚合输出的格式   |    否    |  null  |

```shell
curl -X POST "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "sales_per_month": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "month"
      },
      "aggs": {
        "sales": {
          "sum": {
            "field": "price"
          }
        }
      }
    },
    "avg_monthly_sales": {
      "avg_bucket": {
        "buckets_path": "sales_per_month>sales" 
      }
    }
  }
}
'
```

### 导数聚合

导数聚合是父管道聚合。用于计算父聚合（或直方图）聚合特定指标的导数。

特定指标必须是数字，并且封闭的直方图必须将min_doc_count设置为0（直方图聚合的默认设置）。

#### 一阶导数

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_deriv": {
                    "derivative": {
                        "buckets_path": "sales" 
                    }
                }
            }
        }
    }
}
'
```

#### 二阶导数

以通过将导数管线聚合链接到另一个导数管线聚合的结果来计算二阶导数。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_deriv": {
                    "derivative": {
                        "buckets_path": "sales"
                    }
                },
                "sales_2nd_deriv": {
                    "derivative": {
                        "buckets_path": "sales_deriv" 
                    }
                }
            }
        }
    }
}
'
```

#### 单位

导数聚合允许导数值指定单位。会返回`normalized_value`值报告x坐标单位的导数值。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_deriv": {
                    "derivative": {
                        "buckets_path": "sales",
                        "unit": "day" 
                    }
                }
            }
        }
    }
}
'
```

### 最大存储桶聚合

最大存储桶聚合是相邻管道聚合。在相邻聚合中使用指定指标的最大值标识存储桶，并输出存储桶的键和值。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "max_monthly_sales": {
            "max_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
'
```

### 最小存储桶聚合

最小存储桶聚合是相邻管道聚合。在相邻聚合中使用指定指标的最小值标识存储桶，并输出存储桶的键和值。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "min_monthly_sales": {
            "min_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
'
```

### 总和存储桶聚合

总和存储同聚合是相邻管道聚合。用于计算同级聚合中指定指标的所有存储区之间的总和。指定的指标必须是数字，并且同级聚合必须是多桶聚合。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "sum_monthly_sales": {
            "sum_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
'
```

### 统计存储桶聚合

相邻管道聚合。可以在相邻聚合中跨指定指标的所有存储桶计算各种统计信息。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "stats_monthly_sales": {
            "stats_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
'
```

### 扩展统计存储桶聚合

相邻管道聚合。可以在相邻聚合中跨指定指标的所有存储桶计算各种统计信息。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "stats_monthly_sales": {
            "extended_stats_bucket": {
                "buckets_path": "sales_per_month>sales" 
            }
        }
    }
}
'
```

### 百分位存储桶聚合

相邻管道聚合。用于计算同级聚合中指定指标的所有存储桶的百分比。

`percentiles_bucket`聚合的参数

|   参数名称   |                   描述                   | 是否必须 |            默认值            |
| :----------: | :--------------------------------------: | :------: | :--------------------------: |
| buckets_path |                存储桶路径                |    是    |                              |
|  gap_policy  |            处理空白数据的方式            |    否    |             skip             |
|    format    |              聚合输出的格式              |    否    |             null             |
|   percents   |           要计算的百分位数列表           |    否    | [ 1, 5, 25, 50, 75, 95, 99 ] |
|    keyed     | 该标志将范围作为哈希而不是键值对数组返回 |    否    |             true             |

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                }
            }
        },
        "percentiles_monthly_sales": {
            "percentiles_bucket": {
                "buckets_path": "sales_per_month>sales", 
                "percents": [ 25.0, 50.0, 75.0 ] 
            }
        }
    }
}
'
```

### 移动平均聚合

给定一系列有序的数据，移动平均聚合将在数据上滑动一个窗口并发出该窗口的平均值。

`moving_avg`聚合的参数如下

|   参数名称   |                       描述                       | 是否必须 |      默认值       |
| :----------: | :----------------------------------------------: | :------: | :---------------: |
| buckets_path |                    存储桶路径                    |    是    |                   |
|  gap_policy  |                处理空白数据的方式                |    否    |       skip        |
|    format    |                  聚合输出的格式                  |    否    |       null        |
|    model     |          我们希望使用的移动平均权重模型          |    否    |      simple       |
|    window    |          要在直方图中“滑动”的窗口的大小          |    否    |         5         |
|   minimize   |             如果模型应在算法上最小化             |    否    | 大多数模型为false |
|   settings   | 特定于模型的设置，其内容根据指定的模型而有所不同 |    否    |                   |

`moving_avg`聚合必须嵌套在`histogram`或者是`date_histogram`聚合中

### 移动函数聚合

给定一系列有序数据，移动功能聚合将在数据上滑动一个窗口，并允许用户指定在每个数据窗口上执行的自定义脚本。

`moving_fn`聚合的参数

|   参数名称   |              描述              | 是否必须 | 默认值 |
| :----------: | :----------------------------: | :------: | :----: |
| buckets_path |           存储桶路径           |    是    |        |
|    window    | 要在直方图中“滑动”的窗口的大小 |    否    |   5    |
|    script    |    在每个窗口需要执行的脚本    |    是    |        |
|    shift     |          窗口位置偏移          |    否    |   0    |

`moving_fn`聚合必须嵌套在`histogram`或者是`date_histogram`聚合中

```shell
curl -X POST "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs": {
        "my_date_histo":{                
            "date_histogram":{
                "field":"date",
                "calendar_interval":"1M"
            },
            "aggs":{
                "the_sum":{
                    "sum":{ "field": "price" } 
                },
                "the_movfn": {
                    "moving_fn": {
                        "buckets_path": "the_sum", 
                        "window": 10,
                        "script": "MovingFunctions.unweightedAvg(values)"
                    }
                }
            }
        }
    }
}
'
```

#### 用户自定义脚本

```shell
curl -X POST "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs": {
        "my_date_histo":{
            "date_histogram":{
                "field":"date",
                "calendar_interval":"1M"
            },
            "aggs":{
                "the_sum":{
                    "sum":{ "field": "price" }
                },
                "the_movavg": {
                    "moving_fn": {
                        "buckets_path": "the_sum",
                        "window": 10,
                        "script": "return values.length > 0 ? values[0] : Double.NaN"
                    }
                }
            }
        }
    }
}
'
```

#### 预定义函数

`moving_fn`脚本上下文于定义了许多函数

- `max()`
- `min()`
- `sum()`
- `stdDev()`
- `unweightedAvg()`
- `linearWeightedAvg()`
- `ewma()`
- `holt()`
- `holtWinters()`

### 累积求和聚合

累积求和聚合是一个父管道聚合。用于计算父直方图（或日期直方图）聚合中指定指标的累积总和。

指定的指标必须是数字，并且包围的直方图必须将`min_doc_count`设置为0。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "cumulative_sales": {
                    "cumulative_sum": {
                        "buckets_path": "sales" 
                    }
                }
            }
        }
    }
}
'
```

### 累积基数聚合

累积基数聚合是一个父管道聚合。用于计算父直方图（或日期直方图）聚合中指定指标的累积总和。

指定的指标必须是基数聚合，并且封闭的直方图必须将`min_doc_count`设置为0（直方图聚合的默认设置）。

```shell
curl -X GET "localhost:9200/user_hits/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "users_per_day" : {
            "date_histogram" : {
                "field" : "timestamp",
                "calendar_interval" : "day"
            },
            "aggs": {
                "distinct_users": {
                    "cardinality": {
                        "field": "user_id"
                    }
                },
                "total_new_users": {
                    "cumulative_cardinality": {
                        "buckets_path": "distinct_users" 
                    }
                }
            }
        }
    }
}
'
```

### 存储桶脚本聚合

存储桶脚本聚合是父管道聚合执行脚本，该脚本可以在父多桶聚合中的指定指标上按桶进行计算。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "total_sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "t-shirts": {
                  "filter": {
                    "term": {
                      "type": "t-shirt"
                    }
                  },
                  "aggs": {
                    "sales": {
                      "sum": {
                        "field": "price"
                      }
                    }
                  }
                },
                "t-shirt-percentage": {
                    "bucket_script": {
                        "buckets_path": {
                          "tShirtSales": "t-shirts>sales",
                          "totalSales": "total_sales"
                        },
                        "script": "params.tShirtSales / params.totalSales * 100"
                    }
                }
            }
        }
    }
}
'
```

### 存储桶选择器聚合

存储桶选择器聚合是执行脚本的父管道聚合。该脚本确定当前存储桶是否将保留在父多存储桶聚合中。指定的指标必须为数字，并且脚本必须返回布尔值。

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "total_sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_bucket_filter": {
                    "bucket_selector": {
                        "buckets_path": {
                          "totalSales": "total_sales"
                        },
                        "script": "params.totalSales > 200"
                    }
                }
            }
        }
    }
}
'
```

### 存储桶排序聚合

存储桶排序聚合是父管道聚合。对其父多存储桶聚合进行排序。可以指定零个或多个排序字段以及相应的排序顺序。

每个存储桶的排序基于`_key`，`_count`或其子聚合排序。

`bucket_sort`聚合的参数

|  参数名称  |                        描述                        | 是否必须 | 默认值 |
| :--------: | :------------------------------------------------: | :------: | :----: |
|    sort    |                   需要排序的字段                   |    否    |        |
|    from    | 设置值之前的位置的存储桶将被截断，即从from开始排序 |    否    |   0    |
|    size    |  存储桶返回的数量，默认是返回父聚合中的所有存储桶  |    否    |        |
| gap_policy |                 处理空白数据的方式                 |    否    |  skip  |

```shell
curl -X POST "localhost:9200/sales/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "size": 0,
    "aggs" : {
        "sales_per_month" : {
            "date_histogram" : {
                "field" : "date",
                "calendar_interval" : "month"
            },
            "aggs": {
                "total_sales": {
                    "sum": {
                        "field": "price"
                    }
                },
                "sales_bucket_sort": {
                    "bucket_sort": {
                        "sort": [
                          {"total_sales": {"order": "desc"}}
                        ],
                        "size": 3
                    }
                }
            }
        }
    }
}
'
```

### 串行差分聚合

串行差分是一种在不同的时滞或周期从其自身中减去时间序列中的值的技术。例如数据点$f(x)=f(x_t)-f(x_{t-n})$，其中$n$是周期。

`serial_diff`聚合的参数

|   参数名称   |                   描述                   | 是否必须 | 默认值 |
| :----------: | :--------------------------------------: | :------: | :----: |
| buckets_path |                存储桶路径                |    是    |        |
|  gap_policy  |            处理空白数据的方式            |    否    |  skip  |
|    format    |              聚合输出的格式              |    否    |  null  |
|     lag      | 从当前值中减去的历史存储区，必须是正整数 |    否    |   1    |

`serial_diff`聚合必须嵌套在`histogram`聚合或者是`date_histogram`聚合中使用

```shell
curl -X POST "localhost:9200/_search?pretty" -H 'Content-Type: application/json' -d'
{
   "size": 0,
   "aggs": {
      "my_date_histo": {                  
         "date_histogram": {
            "field": "timestamp",
            "calendar_interval": "day"
         },
         "aggs": {
            "the_sum": {
               "sum": {
                  "field": "lemmings"     
               }
            },
            "thirtieth_difference": {
               "serial_diff": {                
                  "buckets_path": "the_sum",
                  "lag" : 30
               }
            }
         }
      }
   }
}
'
```

## 只返回聚合结果

在许多情况下，需要聚合结果但不需要搜索匹配。对于这些情况，可以通过设置`size = 0`忽略匹配。使得请求更高效。

## 聚合元数据

您可以在请求时将一段元数据与单个聚合相关联，这些元数据将在响应时返回。

```shell
curl -X GET "localhost:9200/twitter/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "titles": {
      "terms": {
        "field": "title"
      },
      "meta": {
        "color": "blue"
      }
    }
  }
}
'
```

## 返回聚合的类型

有时候需要知道聚合的类型，可以使用`typed_keys`参数可用于更改响应中的聚合名称，以便以其内部类型作为前缀，如`stats_bucket#`前缀。

```shell
curl -X GET "localhost:9200/twitter/_search?typed_keys&pretty" -H 'Content-Type: application/json' -d'
{
  "aggregations": {
    "tweets_over_time": {
      "date_histogram": {
        "field": "date",
        "calendar_interval": "year"
      },
      "aggregations": {
        "top_users": {
            "top_hits": {
                "size": 1
            }
        }
      }
    }
  }
}
'
```



