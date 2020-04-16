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

以下采用的示例为[exams.json]()

#### Avg聚合

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