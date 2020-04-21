# Elasticsearch官方文档学习

- 版本：7.6
- 环境：Ubuntu16.04
- 安装方式：docker

## Elasticsearch介绍

### 数据输入：文档和索引

Elasticsearch是一个分布式文档存储框架，Elasticsearch存储已序列化为JSON文档的复杂数据结构。

当集群中有多个Elasticsearch节点时，存储的文档将分布在集群中，并且可以从任何节点立即访问。

Elasticsearch使用称为倒排索引的数据结构，该结构支持非常快速的全文本搜索。

索引可以认为是文档的优化集合，每个文档都是字段的集合，这些字段是包含您的数据的键值对。

Elasticsearch还具有无模式的能力，这意味着可以为文档建立索引，而无需明确指定如何处理文档中可能出现的每个不同字段。

可以定义规则来控制动态映射，也可以显式定义映射以完全控制字段的存储和索引方式，进一步控制数据的使用方式。

定义自己的映射能够：

- 区分全文字符串字段和精确值字符串字段
- 执行特定于语言的文本分析
- 优化字段以进行部分匹配
- 使用自定义日期格式
- 使用无法自动检测到的数据类型，例如geo_point和geo_shape

### 信息输出：搜索和分析

Elasticsearch提供了一个简单，一致的REST API，用于管理您的集群以及建立索引和搜索数据。

#### 查询数据

Elasticsearch REST API支持结构化查询，全文查询和结合了两者的复杂查询。

- 结构化查询：结构化查询类似于您可以在SQL中构造的查询类型。
- 全文查询：查找与查询字符串匹配的所有文档，并按相关性排序返回它们（它们与您的搜索字词的匹配程度如何）。

Elasticsearch在支持高性能地理和数字查询的优化数据结构中索引非文本数据。

Elasticsearch查询方式：

- Query DSL：使用Elasticsearch全面的JSON样式的查询语言（Query DSL）访问所有这些搜索功能。

- SQL：构造SQL风格的查询以在Elasticsearch内部本地搜索和聚合数据

#### 分析数据

Elasticsearch聚合使您能够构建数据的复杂摘要，并深入了解关键指标，模式和趋势。

聚合操作与搜索操作请求可以一起运行。

### 可伸缩性和弹性：集群，节点和分片

Elasticsearch知道如何平衡多节点集群以提供规模和高可用性。

Elasticsearch索引实际上只是一个或多个物理碎片的逻辑分组，其中每个分片实际上是一个独立的索引。

分片有两种类型：

- 主分片
- 副本分片

索引中的每个文档都属于一个主分片。副本分片是主分片的副本。副本可提供数据的冗余副本，以防止硬件故障并提高处理读取请求（如搜索或检索文档）的能力。

创建索引时，索引中主分片的数量是固定的，但是副本分片的数量可以随时更改，而不会中断索引或查询操作。

#### 容灾处理

Elasticsearch提供了 Cross-cluster replication (CCR，跨集群复制)机制，可以自动将索引从主群集同步到可以用作热备份的辅助远程群集。

跨集群复制是主动-被动的。主群集上的索引是活动的领导者索引，并处理所有写请求。复制到辅助群集的索引是只读随从者。

## 安装Elasticsearch

本实验环境为：Ubuntu16.04，docker安装elasticsearch。

### 安装docker

首先配置国内源，编辑`/etc/apt/source.list`，追加下面的内容：

```txt
# 配置阿里源：
deb http://mirrors.aliyun.com/ubuntu/ xenial main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main

deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main

deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe

deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe

```

安装docker

```shell

sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce

```

修改docker daemon配置，[使用阿里云容器镜像服务加速](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)：

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://tcwnwmtq.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

安装docker-compose：

```shell
sudo apt-get install docker-compose
```



### 使用curl命令与Elasticsearch交互

对Elasticsearch的请求包含与任何HTTP请求相同的部分：

- `<VERB>`：HTTP请求方式，如：`GET`，`POST`，`PUT`，`HEAD`或者`DELETE`
- `<PROTOCOL>`：有`http`和`https`。
- `<HOST>`：Elasticsearch集群中任何节点的主机名
- `<PORT>`：运行Elasticsearch HTTP服务的端口，默认为9200
- `<PATH>`：API端点，可以包含多个组件，例如_cluster / stats或_nodes / stats / jvm。
- `<QUERY_STRING>`：任何可选的查询字符串参数。例如，？pretty将漂亮地打印JSON响应以使其更易于阅读

```shell
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```

### 使用Docker安装Elasticsearch

- 获取es7.6镜像：`docker pull docker.elastic.co/elasticsearch/elasticsearch:7.6.2`

使用Docker启动单节点集群：

```shell
docker run -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:7.6.2

# 查看单节点集群运行状态
curl -X GET "localhost:9200/_cat/health?v"
```



使用Docker Compose启动多节点集群：

```yaml
version: '2.2'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic
  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    networks:
      - elastic
  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/usr/share/elasticsearch/data
    networks:
      - elastic

volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

networks:
  elastic:
    driver: bridge
```

使用`docker-compse`启动集群：

```shell
docker-compose up
```

运行集群时可能会出现内存不足的情况，报错如下：

```
es03    | [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

解决方案如下：

```shell
# 查看虚拟内存参数
sudo sysctl -a|grep vm.max_map_count
# 设置虚拟内存参数值为报错值多一点，暂时生效
sudo sysctl -w vm.max_map_count=263000
# 想要设置虚拟内存参数永久生效，编辑/etc/sysctl.conf文件
sudo vim /etc/sysctl.conf
# 追加如下内容
vm.max_map_count=263000
```

请求`_cat/nodes`健康检查api，校验节点是否正在运行：

```shell
curl -X GET "localhost:9200/_cat/nodes?v&pretty"
```



### 配置Elasticsearch

Elasticsearch有三个配置文件：

- `elasticsearch.yml` 用于配置 Elasticsearch
- `jvm.options` 用于配置Elasticsearch相关JVM参数
- `log4j2.properties` 用于配置Elasticsearch日志方式

ES的配置默认在`$ES_HOME/config`文件夹中，可以通过设置`ES_PATH_CONF`环境变量改变ES配置文件默认路径。

在ES配置文件中可以使用`${...}`表示所引用的环境变量。

```yaml
node.name:    ${HOSTNAME}
network.host: ${ES_NETWORK_HOST}
```

#### 设置JVM参数

Elasticsearch可以使用两种方式设置JVM参数

- `jvm.options`文件中指定（普通环境首选方式）
- 设置环境变量`ES_JAVA_OPTS`（docker环境首选方式）

`jvm.options`该文件包含遵循特殊语法的以行分隔的JVM参数列表

- 仅由空格组成的行将被忽略

- 以＃开头的行被视为注释，并被忽略

- 以-开头的行被视为独立于JVM版本而应用的JVM选项

- 以数字开头，以：后跟-的行被视为仅在JVM版本与该数字匹配时才适用的JVM选项。

- 以数字开头，以-开头，以：开头的行被视为JVM选项，仅在JVM版本大于或等于该数字时才适用

- 以数字开头，以-开头，以数字开头，以：开头的行被视为JVM选项，仅当JVM版本在两个数字范围内时才适用

- 其他所有行均被拒绝

  ```shell
  -Xmx2g
  8:-Xmx2g
  8-:-Xmx2g
  8-9:-Xmx2g
  ```

#### 安全设定

某些设置是敏感的，仅依靠文件系统权限来保护其值是不够的。对于这种情况，Elasticsearch提供了密钥库和[elasticsearch-keystore](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/elasticsearch-keystore.html)工具来管理密钥库中的设置。

密钥库没有验证来阻止不支持的设置。将不支持的设置添加到密钥库中会导致Elasticsearch无法启动。

只有重新启动Elasticsearch后，对密钥库的所有修改才会生效。

Elasticsearch密钥库当前仅提供混淆。

所有安全设置都是特定于节点的设置，每个节点上的值必须相同。

##### 可重新加载的安全设置

某些安全设置被标记为可重新加载，可以重新读取此类设置并将其应用到正在运行的节点上。

所有安全设置的值（可重新加载或不可重新加载）在所有群集节点上必须相同

所有安全设置（可重新加载或不可重新加载）的值在所有群集节点上必须相同。进行所需的安全设置更改后，使用bin / elasticsearch-keystore add命令，调用：

```
POST _nodes/reload_secure_settings
```

#### 日志配置

Elasticsearch使用`Log4j 2`进行日志记录。可以使用`log4j2.properties`文件进行。

Elasticsearch公开了三个属性，可以在配置文件中引用以确定日志文件的位置

- `${sys:es.logs.base_path}`：解析到日志目录
- `${sys:es.logs.cluster_name}`：将解析为群集名称（在默认配置中用作日志文件名的前缀）
- `${sys:es.logs.node_name}`：将解析为节点名称（如果显式设置了节点名称）

##### 配置日志输出级别

有四种方式配置日志输出级别

- 使用命令行参数`-E <name of logging hierarchy>=<level>`，如(`-E logger.org.elasticsearch.transport=trace`)，当在单个节点上临时调试问题时，这是最合适的。

- 在`elasticsearch.yml`中配置：`<name of logging hierarchy>: <level>`，如(`logger.org.elasticsearch.transport: trace`)，当您临时调试问题但未通过命令行（例如通过服务）启动Elasticsearch或希望更永久地调整日志记录级别时，这是最合适的.

- 通过集群设置日志输出级别：当您需要动态地调整正在运行的群集上的日志记录级别时，这是最合适的。

  ```
  PUT /_cluster/settings
  {
    "transient": {
      "<name of logging hierarchy>": "<level>"
    }
  }
  ```

- 通过`log4j2.properties`文件设置日志输出级别：当您需要对记录器进行细粒度控制时，这是最合适的。

  ```properties
  logger.<unique_identifier>.name = <name of logging hierarchy>
  logger.<unique_identifier>.level = <level>
  ```

##### 弃用日志

默认情况下，在WARN级别启用弃用日志记录，该级别将发出所有弃用日志消息。

```properties
logger.deprecation.level = warn
```

默认的日志记录配置已将弃用日志的滚动策略设置为在1 GB之后滚动和压缩，并最多保留五个日志文件（四个滚动日志和活动日志）。

可以在`config / log4j2.properties`文件中禁用弃用日志功能，方法是将弃用日志级别设置为以下错误：

```properties
logger.deprecation.name = org.elasticsearch.deprecation
logger.deprecation.level = error
```

##### Json日志格式

为了简化对Elasticsearch日志的解析，现在以JSON格式打印日志。这是通过Log4J布局属性`appender.rolling.layout.type = ESJsonLayout`进行配置的。此布局需要设置`type_name`属性，该属性用于在解析时区分日志流。

```properties
appender.rolling.layout.type = ESJsonLayout
appender.rolling.layout.type_name = server
```



您仍然可以使用自己的自定义布局。为此，请使用其他布局替换`appender.rolling.layout.type`行。请参阅下面的示例

```properties
appender.rolling.type = RollingFile
appender.rolling.name = rolling
appender.rolling.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}_server.log
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = [%d{ISO8601}][%-5p][%-25c{1.}] [%node_name]%marker %.-10000m%n
appender.rolling.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}-%i.log.gz
```



##  Elasticsearch入门

### 启动并运行Elasticsearch集群

使用cat health API验证三节点集群是否正在运行:

```
GET /_cat/health?v
```

### 索引一些文档

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

#### 批量索引文档

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

### 开始查询

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

### 使用汇总分析结果

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

## 聚合分析

聚合框架有助于基于搜索查询提供聚合数据。它基于称为聚合的简单构建块，可以进行组合以构建复杂的数据汇总。

聚合可以看作是在一组文档上建立分析信息的工作单元。聚合执行的上下文定义了此文档是什么。（例如，顶级聚合在搜索请求的已执行查询/过滤器的上下文中执行）

有以下几种类型的聚合，每种聚合都有自己的目的和输出

- Bucketing：一系列构建存储桶的聚合，其中每个存储桶都与一个键和一个文档条件相关联。执行聚合时，将对上下文中的每个文档判断所有存储桶条件，并且当条件匹配时，该文档将被视为“落入”相关存储桶。在汇总过程结束时，我们将获得一个存储桶列表-每个存储桶都带有一组“从属”的文档。
- Metric：聚合可跟踪和计算一组文档的指标。
- Matrix：矩阵聚合，可在多个字段上操作，并根据从请求的文档字段中提取的值生成矩阵结果。该类聚合目前不支持脚本。
- Pipeline：聚合其他汇总及其相关指标的输出的聚合。

由于每个存储桶`bucket`有效定义了一个文档集（所有属于该文档的存储桶），而聚合是作用在一组文档上建立分析信息的工作单元，因此有聚合有一个强大的功能，便是可以实现嵌套聚合。

#### 构建聚合

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

#### 值的来源

一些聚合处理从聚合文档中提取的值。通常，将从使用聚合的字段关键字设置的特定文档字段中提取值。也可以定义一个脚本来生成值（每个文档）。

当为聚合中配置了`field`和`script`时，该脚本将被视为值脚本`value script`。

虽然普通脚本是在文档级别评估的（即脚本可以访问与文档关联的所有数据），但值脚本是在值级别评估的。在这种模式下，从配置的字段中提取值，并且脚本用于对这些值进行“转换”。

### 指标聚合

此族中的聚合基于从要聚合的文档中以一种或另一种方式提取的值来计算指标。这些值通常从文档的字段中提取（使用字段数据），但是也可以使用脚本生成。

数值指标聚合是一种特殊类型的指标聚合，可输出数值。

一些聚合输出单数值指标（如avg），称为单值数值指标聚合（single-value numeric metrics aggregation）。

一些聚合生成多个指标（如stat），称为多值数值指标聚合（multi-value numeric metrics aggregation）。

以下采用的示例为[exams.json](./exams.json)

#### 平均聚合

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

##### 脚本

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

##### 值脚本

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

##### 缺省值

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

#### 加权平均聚合

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

##### 脚本

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

##### 缺省值

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

#### 基数聚合

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

##### 精度控制

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

##### 计数是近似值

计算精确计数需要将值加载到哈希集中并返回其大小，当处理高基数集和/或较大的值时，这不会扩展，因为所需的内存使用情况以及在节点之间传递每个分片集的需求会占用过多的群集资源。

此基数聚合基于HyperLogLog ++算法，该算法基于值的哈希值进行计数。

##### 脚本

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

##### 缺省值

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

#### 扩展统计聚合

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

##### 标准偏差范围

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

##### 脚本

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

##### 值脚本

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

##### 缺省值

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

#### 地理边界聚合

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

#### 地理质心聚合

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

#### 最大值/最小值聚合

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

##### 脚本

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

##### 值脚本

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

##### 缺省值

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

#### 百分位数聚合

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

##### 带键的响应

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

##### 脚本

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

##### 压缩

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

##### HDR直方图

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

##### 缺省值

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

#### 百分数等级聚合

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

##### 带键的响应

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

##### 脚本

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

##### HDR直方图

和`percentile_ranks`百分数聚合同理，不赘述。

##### 缺省值

和`percentile_ranks`同理，不赘述。

#### 脚本式指标聚合

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

##### 允许的返回类型

虽然可以在单个脚本中使用任何有效的脚本对象，但是这些脚本必须仅返回以下类型或将其存储在状态对象中

- 基本类型（primitive types）
- 字符串（String)
- 映射（Map，仅包含此处列出的类型的键和值）
- 数组（Array，数组的元素仅包含此处列出的类型）

##### 脚本范围

脚本指标聚合使用脚本在4个阶段的执行：

- `init_script`：在收集任何文件之前先行处理。允许聚合设置任何初始状态。
- `map_script`：每个收集的文档执行一次。这是个必填的脚本。如果未指定`combine_script`，则需要将结果状态存储在`state`对象中。
- `combine_script`：文档收集完成后，对每个分片执行一次。这个是必填的脚本。允许聚合合并从每个分片返回的状态。
- `reduce_script`：所有分片均返回其结果后，在协调节点上执行一次。这是个必填的脚本。这个脚本提供了访问`states`的权限，`states`是每个分片上`combine_script`结果的数组。

##### 其他参数

`params`，可选的，一个传递到`init_script`，`map_script`，`combine_script`的参数。如果不指定，默认提供空的参数。

空存储桶

如果脚本化指标聚合的父存储桶未收集任何文档，则将从分片返回空聚合响应，并返回一个空值。

#### 状态聚合

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

##### 脚本

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

##### 值脚本

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

##### 缺省值

缺省值和其他聚合类似，不赘述。

#### 字符串状态聚合

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

##### 字符分布

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

##### 脚本

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

##### 值脚本

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

##### 缺省值

缺省值和其他聚合类似，不赘述。

#### 求和聚合

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

#### 热门聚合

`top_hits`指标聚合器跟踪要聚合的最相关文档。该聚合器旨在用作子聚合器，以便可以按存储分区汇总最匹配的文档。

`top_hits`聚合器可以有效地用于通过存储桶聚合器按某些字段对结果集进行分组。一个或多个存储桶聚合器确定将结果集切成哪些属性。

##### 可选项

- `from`：获取的第一个结果的偏移量。
- `size`：每个存储桶返回的最匹配匹配项的最大数量。默认情况下返回前三个匹配项。
- `sort`：匹配项的排序规则。默认情况下匹配项按`score`排序。

##### 支持每个命中的特性

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

##### 例子

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

##### 字段折叠案例

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

##### 嵌套或反向嵌套聚合器中的top_hits支持

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

#### 值计数聚合

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

#### 绝对中位差（MAD）聚合

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

##### 近似值

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



## DSL查询

## 跨集群搜索

## 使用脚本

## 映射

## 文本分析

## ES相关模块

## 索引相关模块

## 预处理节点

## 管理索引的生命周期

## SQL查询

## 监控ES集群

## 冻结索引

## 汇总或转换数据

## 搭建高可用ES集群

## 快照和还原

## 集群保护

## 集群和索引事件警报

## 命令行工具

## 测试

## 专业术语

## 其余API