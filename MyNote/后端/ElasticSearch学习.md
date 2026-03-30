# ElasticSearch学习

### 一、安装与部署

ElasticSearch是Java实现的，所以此处的yaml很像jar的application.yaml 我们可以指定一些Java_opts之类的参数。

为了方便，在我的测试环境将es和kibana的部署脚本写在一起。

es和kibana的docker-compose.yaml

```
version: "3.8"

services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:9.1.6
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-single-node
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - /root/opt/data/es_kibana/es/data:/usr/share/elasticsearch/data
      - /root/opt/data/es_kibana/es/logs:/usr/share/elasticsearch/logs
      - /root/opt/data/es_kibana/es/plugins:/usr/share/elasticsearch/plugins
    ports:
      - "9200:9200"
      - "9300:9300"
    healthcheck:
      test: ["CMD-SHELL", "curl -s http://localhost:9200 >/dev/null || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 50

  kibana01:
    image: docker.elastic.co/kibana/kibana:9.1.6
    container_name: kibana01
    environment:
      - ELASTICSEARCH_HOSTS=http://es01:9200
    ports:
      - "5601:5601"
    volumes:
      - /root/opt/data/es_kibana/kibana/data:/usr/share/kibana/data
      - /root/opt/data/es_kibana/kibana/logs:/usr/share/kibana/logs
    depends_on:
      - es01

```

### 二、基础理论知识

#### 1、es的组成

**节点 (Node)**

- **Node**：集群中的一台服务器/实例
- 每个 Node 都可以存储数据并参与索引和查询
- Node 类型：
  1. **Master Node**：管理集群状态、索引元数据、分片分配
  2. **Data Node**：存储数据、处理搜索和聚合请求
  3. **Coordinating Node**（客户端节点）：只负责接收请求、路由查询、聚合结果

> 在实际操作中，你启动一个 ES 实例，它默认就是一个 Data + Master 节点。

**集群 (Cluster)**

- **Cluster**：由一个或多个 Node 组成的 Elasticsearch 系统
- **作用**：
  - 提供统一的 REST API 接口
  - 自动管理分片和副本分配
  - 容错：节点挂掉不影响服务

> 举例：你本地跑一个 ES + 公司服务器跑多个 ES，多个节点就可以组成一个集群。

**索引 (Index)**

- **Index**：逻辑上的“数据库”，存储文档集合
- 每个 Index 有：
  - **主分片 (Primary Shard)**：存储实际数据
  - **副本分片 (Replica Shard)**：副本用于高可用和查询负载分担
- Index 对应你之前创建的 `maindata_new_test`、`ikun` 等

**文档 (Document)**

- 文档是 ES 中的最小单位，存储 JSON 格式的数据
- 对应数据库里的“行”
- 例如：

```json
{
  "name": "hekun",
  "age": 26,
  "email": "hekun@example.com"
}
```

**字段 (Field)**

- 文档里的属性
- 有 **类型**，影响查询方式：
  - `text` → 全文检索，分词
  - `keyword` → 精确匹配，不分词
  - `date` → 时间类型
  - `integer/long/double` → 数字类型

总的来说 一个简单的例子：

Cluster (集群)
 ├── Node 1 (Master + Data)
 │     ├── Index "maindata_new_test"
 │     │     ├── Primary Shard 0
 │     │     └── Replica Shard 0
 │     └── Index "ikun"
 │           ├── Primary Shard 0
 │           └── Replica Shard 0
 └── Node 2 (Data)
       └── 存储副本分片

#### 2、倒排索引

#### 5、其他

## **ES vs ClickHouse 对比**

| 特性       | Elasticsearch (ES)                 | ClickHouse (CH)                         |
| ---------- | ---------------------------------- | --------------------------------------- |
| 类型       | 分布式全文搜索 + 分析引擎          | 列式数据库 OLAP 引擎                    |
| 数据结构   | 文档（JSON）                       | 表（行/列）                             |
| 存储方式   | 倒排索引 + 列式存储（部分字段）    | 列式存储                                |
| 核心优势   | 快速全文搜索、模糊匹配、聚合查询   | 大规模分析、聚合查询、指标统计          |
| 查询类型   | matchQuery / termQuery / boolQuery | SQL 类语法（SELECT + WHERE + GROUP BY） |
| 写入特点   | 实时写入 + 分片副本                | 批量写入更快，实时插入也支持            |
| 查询速度   | 对文本、全文检索高效               | 对大规模结构化数据分析高效              |
| 适合场景   | 日志搜索、文本分析、推荐系统       | 数据仓库、指标分析、用户行为分析        |
| 分布式能力 | 支持主/副本分片，自动分布          | 分布式表 + 分区 + 排序键                |
| 数据量规模 | 中等到大，亿级文档可行             | 大规模，亿级/十亿级行级别               |
| 数据一致性 | 最终一致性（分片复制延迟）         | 支持强一致性和分布式事务                |

### 三、集成Spring及使用

#### 1、相关依赖：

```yaml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

#### 2、ElasticSearchConfig配置：

```java
import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.elasticsearch.client.ClientConfiguration;
import org.springframework.data.elasticsearch.client.RestClients;

@Configuration
public class ElasticSearchConfig {

    @Value("${spring.elasticsearch.rest.uris}")
    private String esUris;

    @Value("${spring.elasticsearch.rest.username}")
    private String esUsername;

    @Value("${spring.elasticsearch.rest.password}")
    private String esPassword;

    @Bean
    public RestHighLevelClient restHighLevelClient() {
        ClientConfiguration clientConfiguration = ClientConfiguration.builder()
                .connectedTo(esUris)
                .withBasicAuth(esUsername, esPassword)
                .withConnectTimeout(5000)
                .withSocketTimeout(60000)
                .build();

        return RestClients.create(clientConfiguration).rest();
    }
}
```

#### 3、yml里的配置以及参数详解

```yaml
spring:
  elasticsearch:
    rest:
      # ---------------------------
      # 基础连接
      # ---------------------------
      uris: http://10.1.224.5:9200,http://10.1.224.6:9200,http://10.1.224.7:9200  # 集群节点地址，多节点用逗号分隔
      username: elastic                       # 登录用户名
      password: xxxxx                         # 登录密码
      path-prefix: /es                        # 如果通过 Nginx 或有上下文路径
      connection-timeout: 5s                  # 建立连接超时时间
      read-timeout: 60s                       # 请求响应超时

      # ---------------------------
      # 连接池配置（高并发必备）
      # ---------------------------
      max-connect-per-route: 20               # 每个节点最大连接数
      max-connect-total: 100                  # 总连接数
      keep-alive: true                         # 是否启用长连接

      # ---------------------------
      # 集群嗅探（自动发现节点）
      # ---------------------------
      enable-sniffer: true                     # 开启嗅探功能
      sniff-interval: 5m                       # 每 5 分钟嗅探一次集群节点
      sniff-on-failure: true                    # 请求失败后自动嗅探更新节点

      # ---------------------------
      # 安全配置
      # ---------------------------
      ssl:
        enabled: true                           # 启用 TLS/SSL
        trust-store: classpath:es-truststore.jks # 信任证书
        trust-store-password: xxxxx
        key-store: classpath:es-keystore.jks     # 客户端证书（如果需要双向认证）
        key-store-password: xxxxx

      # ---------------------------
      # 高级/可选配置
      # ---------------------------
      default-headers:
        X-My-Header: MyValue                    # 给每个请求添加默认 HTTP Header
      retry-on-failure: true                     # 请求失败自动重试
      socket-timeout: 60s                        # 底层套接字超时，精细控制

```

#### 4、数据的增删改查

##### 4.0 es详细信息的获取：

```json
// 当我们在rest界面什么参数都不填写的时候 直接执行查询就可以得到es的详细信息
{
  "name": "node-1",                   // 节点名称，每个 ES 节点在集群中唯一标识
  "cluster_name": "elasticsearch",    // 集群名称，所有节点通过 cluster_name 归属同一个集群
  "cluster_uuid": "j1y_6TFhSmWv9nQDxSm8Cg", // 集群的唯一 ID，用于集群识别（全局唯一）
  "version": {
    "number": "7.16.3",               // Elasticsearch 版本号
    "build_flavor": "default",        // 构建类型，可能是 default / oss / basic / platinum 等
    "build_type": "tar",              // 安装包类型，如 tar / deb / rpm / docker
    "build_hash": "4e6e4eab2297e949ec994e688dad46290d018022", // 构建的 commit hash，用于追踪源码
    "build_date": "2022-01-06T23:43:02.825887787Z",           // 构建时间（UTC）
    "build_snapshot": false,          // 是否为 snapshot 版本（开发/测试版本）
    "lucene_version": "8.10.1",       // 内部使用的 Lucene 版本，ES 的索引搜索底层引擎
    "minimum_wire_compatibility_version": "6.8.0",  // 与集群中其他节点通信的最低协议版本
    "minimum_index_compatibility_version": "6.0.0-beta1" // 可以打开的最低版本索引兼容性
  },
  "tagline": "You Know, for Search"   // ES 默认返回的提示语
}
```

##### 4.1 字段类型的获取：

首先，我们在构建查询逻辑之前应当先知道我们的目标字段的属性，我们可以通过可视化界面写查询,例如对于一个测试索引：maindata_new_test

```json
maindata_new_test/_mapping GET:
{
  "maindata_new_test": {
    "mappings": {
      "properties": {
        "author": { 
          "type": "text"  // 长文本字段，分词，适合模糊匹配，如文章标题、摘要
        },
        "accountType": { 
          "type": "keyword"  // 短文本 / 精确值字段，不分词，适合状态码、类别、标签
        },
        "createTime": { 
          "type": "date"  // 日期字段，适合 rangeQuery（时间范围查询）在es里es会把时间给我们存成时间戳，不过我们还是可以使用时间格式来匹配 yyyy-MM-dd HH:mm:ss
        },
        "autoTaskId": { 
          "type": "integer"  // 整数字段，适合精确匹配或范围查询
        },
        "dataCenterIdNew": { 
          "type": "long"  // 长整数字段，适合大 ID 或数量级较大的计数
        },
        "latitude": { 
          "type": "double"  // 浮点数字段，适合经纬度、分数等精度要求高的数值
        }
      }
    }
  }
}

```

##### 4.3 关于字段的详解：

###### keyword：

**特点**：不分词，原样存储

**查询方式**：可以用 **精确匹配**

- 单值：`termQuery("status", "1")` → 相当于 SQL 的 `=`
- 多值：`termsQuery("status", ["1","2"])` → 相当于 SQL 的 `IN`

**适用场景**：

- 状态码、ID、类别、标签、标记、布尔值等

例如：

```java
boolQuery.must(QueryBuilders.termQuery("accountType", "admin"));
```

###### text字段：

**特点**：会被分词（分析器），存成词的列表

**查询方式**：

- `matchQuery("title", "安全事件")` → 分词后匹配（模糊匹配）
- `matchPhraseQuery("title", "安全事件")` → 精确短语匹配

**注意**：

- 不能用 `termQuery` 精确匹配原始字符串，否则通常查不到
- 如果想要精确匹配 text 字段，需要 mapping 中增加 `keyword` 子字段

```java
// 分词匹配
boolQuery.must(QueryBuilders.matchQuery("title", "安全事件"));

// 精确匹配，假设 mapping 有 keyword 子字段
boolQuery.must(QueryBuilders.termQuery("title.keyword", "系统安全事件"));
```

###### 日期类型(date)：

**存储**：内部以毫秒时间戳存储

**查询方式**：

- `rangeQuery("field").gte("2025-11-01").lte("2025-11-14")`
- 可以用 ISO 字符串或毫秒时间戳

**注意**：

- 显示格式可自定义（mapping 中 `format`）
- 查询范围建议用 rangeQuery

```java
boolQuery.filter(QueryBuilders.rangeQuery("releaseTime")
                .gte("2025-11-01 00:00:00")
                .lte("2025-11-14 23:59:59"));
```

###### 数字类型(integer / long / float / double / half_float / scaled_float):

**存储**：按数值存储，可做精确匹配或范围查询

**查询方式**：

- 精确匹配：`termQuery("autoTaskId", 123)`
- 多值匹配：`termsQuery("status", [1,2,3])`
- 范围查询：`rangeQuery("score").gte(80).lte(100)`

```java
// 匹配单个值
boolQuery.must(QueryBuilders.termQuery("autoTaskId", 1001));

// 匹配范围
boolQuery.filter(QueryBuilders.rangeQuery("score").gte(60).lte(100));
```

###### 更多的字段类型以及使用场景：

| 类型           | 说明          | 常见用途           | 查询方式                               |
| -------------- | ------------- | ------------------ | -------------------------------------- |
| text           | 分词字符串    | 文章内容、标题     | matchQuery / matchPhraseQuery          |
| keyword        | 不分词字符串  | 状态码、标签、ID   | termQuery / termsQuery / aggs          |
| date           | 日期          | 发布时间、更新时间 | rangeQuery / termQuery（精确时间戳）   |
| integer        | 整数          | ID、状态码         | termQuery / termsQuery / rangeQuery    |
| long           | 长整数        | 大 ID、计数        | termQuery / rangeQuery                 |
| float / double | 浮点          | 分数、经纬度       | termQuery / rangeQuery                 |
| scaled_float   | 定点数        | 金额、精确小数     | termQuery / rangeQuery                 |
| boolean        | 布尔          | true/false 状态    | termQuery                              |
| binary         | 二进制        | 文件或序列化对象   | -                                      |
| geo_point      | 地理坐标      | 经纬度查询         | geoDistanceQuery / geoBoundingBoxQuery |
| geo_shape      | 多边形 / 路径 | 地理区域搜索       | geoShapeQuery                          |
| nested         | 嵌套对象      | JSON 数组对象      | nestedQuery                            |
| object         | 内嵌对象      | JSON 对象          | matchQuery / termQuery                 |
| ip             | IP 地址       | 服务器 IP          | termQuery / rangeQuery                 |

###### boolQuery.must与boolQuery.filter

| 特性            | must                             | filter                                     |
| --------------- | -------------------------------- | ------------------------------------------ |
| 是否必须匹配    | ✅                                | ✅                                          |
| 是否影响 _score | ✅                                | ❌                                          |
| 是否可缓存      | ❌（一般不缓存）                  | ✅（ES 会自动缓存，提高性能）               |
| 使用场景        | 模糊匹配 / 全文检索 / 影响相关性 | 精确匹配 / 范围查询 / 标签 / 日期 / 状态码 |

###### 一个查询示例

```java
		// 创建 BoolQueryBuilder
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        // 动态构建查询条件：检查每个字段是否为空，如果不为空则加入查询
        if (criteria.getAuthor() != null && !criteria.getAuthor().isEmpty()) {
            boolQuery.must(QueryBuilders.matchQuery("author", criteria.getAuthor()));  // 作者匹配
        }
        if (criteria.getStartDate() != null) {
            boolQuery.filter(QueryBuilders.rangeQuery("releaseTime").gte(criteria.getStartDate()));  // 发布时间范围-开始时间
        }
        // 判断并处理 mediaLink 字段
        if (criteria.getMediaLink() != null && !criteria.getMediaLink().isEmpty()) {
            // tonalState 是逗号分隔的字符串，例如 "0,1,2"
            String[] mediaLinks = criteria.getMediaLink().split(",");
            boolQuery.must(QueryBuilders.termsQuery("mediaLink", (Object[]) mediaLinks));  // 使用 termsQuery 匹配多个值
        }

        // 构建查询
        NativeSearchQuery query = new NativeSearchQueryBuilder()
                .withQuery(boolQuery)  // 使用构建好的布尔查询
                .withPageable(PageRequest.of(criteria.getPageNum(), criteria.getPageSize()))  // 分页设置
                .build();

        // 执行查询
        SearchHits<JSONObject> searchHits = elasticsearchOperations.search(query, JSONObject.class, IndexCoordinates.of("maindata_new_test"));

        // 获取查询结果
        List<JSONObject> results = searchHits.stream()
                .map(hit -> hit.getContent())
                .collect(Collectors.toList());

        // 构建分页结果
        PageResult pageResult = new PageResult();
        pageResult.setData(results);  // 当前页数据
        pageResult.setCount(searchHits.getTotalHits());  // 总数

        return pageResult;
```

##### 4.4 删除索引

简单的逻辑

```java
@Test
void deleteIndex(){
    IndexOperations indexOps = elasticsearchOperations.indexOps(IndexCoordinates.of("ikun"));
    if (indexOps.exists()) {
        boolean deleted = indexOps.delete();
        System.out.println("索引删除结果：" + deleted);
    }
}
```

##### 4.5 新增索引结构

```java
@Test
void addIndex(){
    IndexOperations indexOps = elasticsearchOperations.indexOps(IndexCoordinates.of("ikun"));
    if (!indexOps.exists()){
        Map<String, Object> settings = new HashMap<>();
        //主分片数量
        settings.put("index.number_of_shards", 3);
        //副本数量
        settings.put("index.number_of_replicas", 1);
        indexOps.create(settings);
        indexOps.putMapping(indexOps.createMapping(TestIndex.class));
        //当然一般情况下来说不会这么玩儿
    }
}

@Document(indexName = "ikun")
class TestIndex{
    @Id
    private String id; // ES 文档 ID

    @Field(type = FieldType.Text)
    private String name;

    @Field(type = FieldType.Integer)
    private Integer age;

    @Field(type = FieldType.Boolean)
    private boolean flame;

    @Field(type = FieldType.Text)
    private String address;

    @Field(type = FieldType.Keyword)
    private String email;
}

```

##### 4.6 新增数据

5、可视化界面操作：

```
_cat/indices/indexName?format=json  GET：可以查看索引的部分情况，比如数据量、健康程度等，示例返回：
[ - 
  { - 
    "health": "green",
    "status": "open",
    "index": "ikun",
    "uuid": "p5PoKsPYTW-wrMVYRdmj2Q",
    "pri": "3",
    "rep": "0",
    "docs.count": "1",
    "docs.deleted": "0",
    "store.size": "5.1kb",
    "pri.store.size": "5.1kb"
  }
]
indexName/_mapping    GET： 可以查看字段的数据结构，例如：
indexName/_search   GET： 可以查询默认条数的数据
{
  "query": {
    "term": {
      "data_flag": 9
    }
  },
  "sort": [
    {
      "updateTime": {
        "order": "desc"
      }
    }
  ]
}


indexName/_settings  GET ：查看分片、副本、分析器配置
/_aliases   POST: 给索引取别名、删除别名 需要携带传参：
{
  "actions": [
    {
      "add": {
        "index": "ikun",
        "alias": "ikun_alias"
      }
    },
    {
      "remove": {
        "index": "old_index",
        "alias": "ikun_alias"
      }
    }
  ]
}
indexName/_doc/id   GET： 根据id精准获取数据
_cluster/health   GET：获取集群的健康状态


indexName/_delete_by_query 根据条件删除
{
  "query": {
    "term": {
      "dataFlag": 9
    }
  }
}
```



### 四、集群与高可用



```

```

