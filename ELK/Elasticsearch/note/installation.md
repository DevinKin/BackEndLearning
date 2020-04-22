# 安装Elasticsearch

本实验环境为：Ubuntu16.04，docker安装elasticsearch。

## 安装docker

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



## 使用curl命令与Elasticsearch交互

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

## 使用Docker安装Elasticsearch

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



## 配置Elasticsearch

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

### 设置JVM参数

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

### 安全设定

某些设置是敏感的，仅依靠文件系统权限来保护其值是不够的。对于这种情况，Elasticsearch提供了密钥库和[elasticsearch-keystore](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/elasticsearch-keystore.html)工具来管理密钥库中的设置。

密钥库没有验证来阻止不支持的设置。将不支持的设置添加到密钥库中会导致Elasticsearch无法启动。

只有重新启动Elasticsearch后，对密钥库的所有修改才会生效。

Elasticsearch密钥库当前仅提供混淆。

所有安全设置都是特定于节点的设置，每个节点上的值必须相同。

#### 可重新加载的安全设置

某些安全设置被标记为可重新加载，可以重新读取此类设置并将其应用到正在运行的节点上。

所有安全设置的值（可重新加载或不可重新加载）在所有群集节点上必须相同

所有安全设置（可重新加载或不可重新加载）的值在所有群集节点上必须相同。进行所需的安全设置更改后，使用bin / elasticsearch-keystore add命令，调用：

```
POST _nodes/reload_secure_settings
```

### 日志配置

Elasticsearch使用`Log4j 2`进行日志记录。可以使用`log4j2.properties`文件进行。

Elasticsearch公开了三个属性，可以在配置文件中引用以确定日志文件的位置

- `${sys:es.logs.base_path}`：解析到日志目录
- `${sys:es.logs.cluster_name}`：将解析为群集名称（在默认配置中用作日志文件名的前缀）
- `${sys:es.logs.node_name}`：将解析为节点名称（如果显式设置了节点名称）

#### 配置日志输出级别

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

#### 弃用日志

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

#### Json日志格式

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

