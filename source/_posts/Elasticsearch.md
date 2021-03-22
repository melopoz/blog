

ELK

elasticsearch + Logstash + Kibana

在加上kafka更好



Elasticsearch基于Lucene

下载elasticsearch并安装 port9200

下载head (Node.js) port9100

config中配置跨域

```yaml
http.cors.enable: true
http.cors.allow-origin: "*"
```





Kibana下载 要和Elasticsearch的版本一致  可以汉化



使用了Lucene倒排索引

分片就是倒排索引



下载ik分词器

在kibana中使用rest请求

```json
GET _analyze
{
  "analyzer": "ik_smart",
  "text": "java大数据行业"
}

GET _analyze
{
  "analyzer": "ik_max_word",
  "text": "java大数据行业"
}

GET _analyze
{
  "analyzer": "standard",
  "text": "↑这个是标准的分词器"
}

GET _analyze
{
  "analyzer": "keyword",
  "text": "这个不会拆分text"
}
```

字典中没有的词需要自己加入到字典中， 在ik中增加自己的配置，新增.dic文件



字段如果是keyword类型，就不会被分词器拆分，如果是text才可以



version 0-6 一个索引下有可以有多个type

version 7     一个索引下只有一个type

version 8      索引下直接使用documentId，没有type



- 查看所有索引 GET /_cat/indices
- 查看集群是否健康 GET /_cat/health
- 查看集群节点 GET /_cat/nodes

### CRUD

1. 创建一个库  创建规则

   ```json
   PUT /索引名
   {
     "mappings": {
       "properties": {
         "name": {
           "type": "text"
         },
         "height": {
           "type": "double"
         },
         "birthday": {
           "type": "date"
         }
       }
     }
   }
   ```

2. 查看索引下所有数据

   ```json
   GET /team/_search?pretty
   // _doc
   // _search
   // _mapping...
   结果
   {
     "took" : 0,
     "timed_out" : false,// 超时
     "_shards" : {//分区信息
       "total" : 1,
       "successful" : 1,
       "skipped" : 0,
       "failed" : 0
     },
     "hits" : {
       "total" : {
         "value" : 3,
         "relation" : "eq"
       },
       "max_score" : 1.0,
       "hits" : [ // 全部数据
         {
           "_index" : "team",//索引
           "_type" : "_doc",
           "_id" : "1",
           "_score" : 1.0,//分值
           "_source" : {//source数据
             "name" : "Thunders",
             "area" : "north-west",
             "address" : "Oklahoma City",
             "Coach" : "Mitchell Donovan"
           }
         },
         {
           "_index" : "team",
           "_type" : "_doc",
           "_id" : "2",
           "_score" : 1.0,
           "_source" : {
             "name" : "Warriors",
             "area" : "north-west",
             "address" : "Golden State",
             "Coach" : "Steve Kerr"
           }
         },
         {
           "_index" : "team",
           "_type" : "_doc",
           "_id" : "3",
           "_score" : 1.0,
           "_source" : {
             "name" : "Clippers",
             "area" : "north-west",
             "address" : "Los Angeles",
             "Coach" : "Doc Rivers"
           }
         }
       ]
     }
   }
   ```

   

3. 增加一条索引

   ```json
   PUT /索引名/type名/id
   {
     "name": "melopoz",
     "height": "190"
   }
   ```

4. 修改一条索引  每次version会+1

   ```json
   //如果缺少字段，更新之后就会丢失
   PUT /索引/type/id
   {
     "doc": {
       "feildName": "newValue"   
     }
   }
   //不会丢失字段 在id后加update。
   POST /索引/type/id/_update
   {
     "doc": {
       "feildName": "newValue"
     }
   }
   ```

5. 删除  DELETE

### 复杂查询

- match 使用分词器解析

```json
GET /../../..
{
  "query": {
    "match": {
      "字段名": "值",
      ...
    }
  },
  //字段过滤
  "_source": ["name", "height"], 
  //排序 多种写法
  "sort": [
    {
      "height": "asc"
    },{}
  ]
  //分页
  "from": 0,
  "size": 20
}
```

- boolean查询

```json
GET /../../..
{
  "query": {
    "bool": {
      "must": [//should(or) must(and) must_not(!=)
        {
          "match": {
            "name": "melopoz"
          }
        },{
          "match": {
            "height": 190
          }
        }
      ],
      "filter": {
        "range": {
          "height": {
            "gt": 185 //gt gte lt lte
          }
        }
      }
    }
  }
}
```

- term 直接精确查询 若果存储的value首字母大写了，查询时的参数首字母也要小写。因为分词器分析出的都是小写。 standard

```json
GET /../../..
{
  "query": {
    "term": {
      "name": "melopoz" //如果name字段的type为keyword，有name=melopoz2的数据，也不会被查询到
    }
  }
}
```

- 高亮查询

```
GET /../../..
{
  "query": {
    "match": {
      "name": "melopoz"
    }
  },
  "highlight": {
    "pre_tags": "<hl class="highlight_word">"
    "post_tags": "</h1>"
    "name":{}
  }
}
```

### 聚合索引

```
//查询20-30,0-40,40-50这三个年龄段分别有多少人
GET /..
{
  "size": 0, // size=0则不显示数据只显示聚合结果，每个阶段的人数
  "user": {
    "age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "from": 20,
            "to": 30
          },{
            "from": 30,
            "to": 40
          },{
            "from": 40,
            "to": 50
          }
        ]
      }
    }
  }
}
```

### tokenizer分词器

- keyword分词器

```json
GET /kibana_sample_data_ecommerce/_analyze
{
  "text": ["Happy Birthday"],
  "tokenizer": "keyword"
}
结果：
{
  "tokens" : [
    {
      "token" : "Happy Birthday",
      "start_offset" : 0,
      "end_offset" : 14,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

- standard标准分词器

```
GET /kibana_sample_data_ecommerce/_analyze
{
  "text": ["Happy Birthday"],
  "tokenizer": "standard", //使用standard  标准分词器
  "filter": ["lowercase"] //转换为小写
}
结果：
{
  "tokens" : [
    {
      "token" : "happy",
      "start_offset" : 0,
      "end_offset" : 5,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "birthday",
      "start_offset" : 6,
      "end_offset" : 14,
      "type" : "<ALPHANUM>",
      "position" : 1
    }
  ]
}

```



### Spring Data Elasticsearch

​	可以使用RestClient(RestHighLevelClient)、Jest、SpringDataElasticsearch



#### RestHighLevelClient

_My Project.spring-elasticsearch



#### Jest

引入依赖 （就不需要再引入elasticsearch的依赖了）

```xml
<!-- https://mvnrepository.com/artifact/io.searchbox/jest -->
<dependency>
    <groupId>io.searchbox</groupId>
    <artifactId>jest</artifactId>
    <version>6.3.1</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

直接注入bean，不需要自己配置bean

```java
@Autowired
JestClient jestClient;
```

添加索引

```java
import io.searchbox.core.Index;
```

```java
Player player = new Player(...);
Index index = new Index.Builder(player).index("索引").type("类型").build();
try{
	jestClient.execute(index);
}catch (IOException e){
	e.printStackTrace();
}
```

...

#### SpringDataElasticsearch

依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--需要使用json-->
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.8.5</version>
</dependency>
```

实体类

```java
@Document(indexName="nba", type="player")
public class Player{
...
}
```

repository接口

```java
public interface PlayerRepo extends ElasticsearchRepository<Player, Integer>{
}
```

注入bean即可使用

```java
@Autowired
PlayerRepo playerRepo;
```

添加索引

```java
playerRepo.index(new Player(...));
```

使用接口的方法即可