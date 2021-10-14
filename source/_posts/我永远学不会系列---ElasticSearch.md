---
layout: posts
title: 我永远学不会系列---ElasticSearch
date: 2019-01-10 00:38:36
categories: 搜索
tags: [ElasticSearch]
top: false
---
确实这个ElasticSearch是目前来说一个公用的标准，前几年的时候用的比较多的搜索引擎比如Solr，这几年用的多的就是这个ES。现在哪个系统不涉及到搜索，一般情况下，按条件搜索就是拼装SQL语句，然后去数据库里边去查询记录，但是这样效率实在是太低了，一个电商网站上边的搜索框，如果都用这样的方式去搜索，在高并发的场景下一旦请求量大了，很可能数据库就崩了。所以说现在做搜索引擎，特别是电商领域，这个ES实在是太牛逼了。

<!--more--> 

#### 一、关于Lucene

要想会ES，不知道Lucene是干啥的就有点说不过去了呀。

Lucene是一个开放源代码的全文检索引擎工具包，**但它不是一个完整的全文检索引擎**，而是一个全文检索引擎的架构，提供了完整的查询引擎和索引引擎，部分文本分析引擎（英文与德文两种西方语言）。Lucene提供了一个简单却强大的应用程式接口，能够做到全文索引和搜寻。在Java开发环境里Lucene是一个成熟的免费开源工具。就其本身而言，Lucene是当前以及最近几年最受欢迎的免费Java信息检索程序库。人们经常提到信息检索程序库，虽然与搜索引擎有关，但不应该将信息检索程序库与搜索引擎相混淆。

##### 什么是全文搜索

我们生活中的数据总体分为两种：结构化数据和非结构化数据。

　　　　（1）**结构化数据**：指具有固定格式或有限长度的数据，如数据库，元数据等。

　　　　（2）**非结构化数据**：指不定长或无固定格式的数据，如邮件，word文档等磁盘上的文件

非结构化数据查询方法：

**（1）顺序扫描法(Serial Scanning)**

　　所谓顺序扫描，比如要找内容包含某一个字符串的文件，就是一个文档一个文档的看，对于每一个文档，从头看到尾，如果此文档包含此字符串，则此文档为我们要找的文件，接着看下一个文件，直到扫描完所有的文件。如利用windows的搜索也可以搜索文件内容，只是**相当的慢**。

**（2）全文检索(Full-text Search)**

　　将非结构化数据中的一部分信息提取出来，重新组织，使其变得有一定结构，然后对此有一定结构的数据进行搜索，从而达到搜索相对较快的目的。这部分从非结构化数据中提取出的然后重新组织的信息，我们称之**索引**。

例如：字典。字典的拼音表和部首检字表就相当于字典的索引，对每一个字的解释是非结构化的，如果字典没有音节表和部首检字表，在茫茫辞海中找一个字只能顺序扫描。然而字的某些信息可以提取出来进行结构化处理，比如读音，就比较结构化，分声母和韵母，分别只有几种可以一一列举，于是将读音拿出来按一定的顺序排列，每一项读音都指向此字的详细解释的页数。我们搜索时按结构化的拼音搜到读音，然后按其指向的页数，便可找到我们的非结构化数据——也即对字的解释。

　　**这种先建立索引，再对索引进行搜索的过程就叫全文检索(Full-text Search)。**

　　虽然创建索引的过程也是非常耗时的，但是索引一旦创建就可以多次使用，全文检索主要处理的是查询，所以耗时间创建索引是值得的。

全文检索的应用场景：

　　对于数据量大、数据结构不固定的数据可采用全文检索方式搜索，比如百度、Google等搜索引擎、论坛站内搜索、电商网站站内搜索等。

#### 二、Lucene实现全文检索的流程

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx029GACu4vAAE5LCKAGCQ915.png)

1、绿色表示索引过程，对要搜索的原始内容进行索引构建一个索引库，索引过程包括：

　　　　确定原始内容即要搜索的内容→采集文档→创建文档→分析文档→索引文档

2、红色表示搜索过程，从索引库中搜索内容，搜索过程包括：

　　　　用户通过搜索界面→创建查询→执行搜索，从索引库搜索→渲染搜索结果

##### A.创建索引

对文档索引的过程，将用户要搜索的文档内容进行索引，索引存储在索引库（index）中。

这里我们要搜索的文档是磁盘上的文本文件，根据案例描述：凡是文件名或文件内容包括关键字的文件都要找出来，这里要对文件名和文件内容创建索引。

`原始文档` 要索引和搜索的内容。原始内容包括互联网上的网页、数据库中的数据、磁盘上的文件等。

`创建文档对象` 获取原始内容的目的是为了索引，在索引前需要将原始内容创建成文档（Document），文档中包括一个一个的域（Field），域中存储内容。

举个列子：我们可以将磁盘上的一个文件当成一个Document，Document中包括一些Field（file_name文件名称、file_path文件路径、file_size文件大小、file_content文件内容）

> （1）每个Document可以有多个Field
>
> （2）不同的Document可以有不同的Field
>
> （3）同一个Document可以有相同的Field（域名和域值都相同）
>
> （4）每个文档都有一个唯一的编号，就是文档id。

`分析文档` 将原始内容创建为包含域（Field）的文档（document），需要再对域中的内容进行分析，分析的过程是经过对原始文档提取单词、将字母转为小写、去除标点符号、去除停用词等过程生成最终的语汇单元，可以将语汇单元理解为一个一个的单词。

比如下边的文档经过分析如下：

　　原文档内容：

　　Lucene is a Java full-text search engine.

　　分析后得到的**语汇单元**：

　　lucene、java、full、search、engine

每个单词叫做一个**Term**，不同的域中拆分出来的相同的单词是不同的term。term中包含两部分一部分是文档的域名，另一部分是单词的内容。

　　例如：文件名中包含apache和文件内容中包含的apache是不同的term。

`创建索引` 对所有文档分析得出的语汇单元进行索引，索引的目的是为了搜索，最终要实现只搜索被索引的语汇单元从而找到Document（文档）。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx03NyAI4_-AADYoWJWvdU967.png)

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx03P-AcKPGAAG87qEuEig803.png)

（1）创建索引是对语汇单元索引，通过词语找文档，这种索引的结构叫**倒排索引结构**。

（2）传统方法是根据文件找到该文件的内容，在文件内容中匹配搜索关键字，这种方法是顺序扫描方法，数据量大、搜索慢。

（3）倒排索引结构是根据内容（词语）找文档，如下图

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx03VyAHUjMAAP6YKyaiKA394.png)

**倒排索引结构也叫反向索引结构，包括索引和文档两部分，索引即词汇表，它的规模较小，而文档集合较大。**

##### B.一个代码实例(创建索引)

##### C.Field域的属性概述

**是否分析**：是否对域的内容进行分词处理。前提是我们要对域的内容进行查询。

**是否索引**：将Field分析后的词或整个Field值进行索引，只有索引方可搜索到。

比如：商品名称、商品简介分析后进行索引，订单号、身份证号不用分析但也要索引，这些将来都要作为查询条件。

**是否存储**：将Field值存储在文档中，存储在文档中的Field才可以从Document中获取

比如：商品名称、订单号，凡是将来要从Document中获取的Field都要存储。

**是否存储的标准：是否要将内容展示给用户**

| Field类                                                      | 数据类型               | Analyzed(是否分析) | Indexed(是否索引) | Stored(是否存储) | 说明                                                         |
| ------------------------------------------------------------ | ---------------------- | ------------------ | ----------------- | ---------------- | ------------------------------------------------------------ |
| StringField(FieldName, FieldValue,Store.YES))                | 字符串                 | N                  | Y                 | Y或N             | 这个Field用来构建一个字符串Field，但是不会进行分析，会将整个串存储在索引中，比如(订单号,姓名等),是否存储在文档中用Store.YES或Store.NO决定 |
| LongField(FieldName, FieldValue,Store.YES)                   | Long型                 | Y                  | Y                 | Y或N             | 这个Field用来构建一个Long数字型Field，进行分析和索引，比如(价格),是否存储在文档中用Store.YES或Store.NO决定 |
| StoredField(FieldName, FieldValue)                           | 重载方法，支持多种类型 | N                  | N                 | Y                | 这个Field用来构建不同类型Field,不分析，不索引，但要Field存储在文档中 |
| TextField(FieldName, FieldValue, Store.NO)或TextField(FieldName, reader) | 字符串或流             | Y                  | Y                 | Y或N             | 如果是一个Reader, lucene猜测内容比较多,会采用Unstored的策略. |

##### D.查询索引

查询索引也是搜索的过程。搜索就是用户输入关键字，从索引（index）中进行搜索的过程。根据关键字搜索索引，根据索引找到对应的文档，从而找到要搜索的内容（这里指磁盘上的文件）。

对要搜索的信息创建Query查询对象，Lucene会根据Query查询对象生成最终的查询语法，类似关系数据库Sql语法一样Lucene也有自己的查询语法，比如：“name:lucene”表示查询Field的name为“lucene”的文档信息。

搜索索引过程：

　　根据查询语法在倒排索引词典表中分别找出对应搜索词的索引，从而找到索引所链接的文档链表。

　　比如搜索语法为“fileName:lucene”表示搜索出fileName域中包含Lucene的文档。

　　搜索过程就是在索引上查找域为fileName，并且关键字为Lucene的term，并根据term找到文档id列表。

可通过两种方法创建查询对象：

 1）使用Lucene提供Query子类

 Query是一个抽象类，lucene提供了很多查询对象，比如TermQuery项精确查询，NumericRangeQuery数字范围查询等。

 如下代码：

```java
Query query = new TermQuery(new Term("name", "lucene"));
```

 2）使用QueryParse解析查询表达式

 QueryParse会将用户输入的查询表达式解析成Query对象实例。

 如下代码：

```java
QueryParser queryParser = new QueryParser("name", **new** IKAnalyzer());

Query query = queryParser.parse("name:lucene");
```

##### E.一个代码实例(全文检索)

https://www.cnblogs.com/xiaobai1226/p/7652093.html可以参考这个博客内容。

#### 三、有关ES

Elasticsearch是一个开源的高扩展的分布式全文检索引擎，它可以近乎实时的存储、检索数据；本身扩展性很好，可以扩展到上百台服务器，处理PB级别的数据。 Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。

##### 1.ES主要解决问题

```javascript
`检索相关数据` `返回统计结果` `速度要快`
```

##### 2.ES核心概念

1）Cluster：集群

ES可以作为一个独立的单个搜索服务器。不过，为了处理大型数据集，实现容错和高可用性，ES可以运行在许多互相合作的服务器上。这些服务器的集合称为集群。

2）Node：节点

形成集群的每个服务器称为节点。

3）Shard：分片

当有大量的文档时，由于内存的限制、磁盘处理能力不足、无法足够快的响应客户端的请求等，一个节点可能不够。这种情况下，数据可以分为较小的分片。每个分片放到不同的服务器上。
当你查询的索引分布在多个分片上时，ES会把查询发送给每个相关的分片，并将结果组合在一起，而应用程序并不知道分片的存在。即：这个过程对用户来说是透明的。

4）Replia：副本

为提高查询吞吐量或实现高可用性，可以使用分片副本。
副本是一个分片的精确复制，每个分片可以有零个或多个副本。ES中可以有许多相同的分片，其中之一被选择更改索引操作，这种特殊的分片称为主分片。
当主分片丢失时，如：该分片所在的数据不可用时，集群将副本提升为新的主分片。

5）全文检索

全文检索就是对一篇文章进行索引，可以根据关键字搜索，类似于mysql里的like语句。
全文索引就是把内容根据词的意义进行分词，然后分别创建索引，例如”你们的激情是因为什么事情来的” 可能会被分词成：“你们“，”激情“，“什么事情“，”来“ 等token，这样当你搜索“你们” 或者 “激情” 都会把这句搜出来。

##### 3.ES数据架构的主要概念(对比MySQL)

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx032GAabQUAADdDeKhOR0899.png)

- 关系型数据库中的数据库（DataBase），等价于ES中的索引（Index）
- 一个数据库下面有N张表（Table），等价于1个索引Index下面有N多类型（Type）
- 一个数据库表（Table）下的数据由多行（ROW）多列（column，属性）组成，等价于1个Type由多个文档（Document）和多Field组成。
- 在一个关系型数据库里面，schema定义了表、每个表的字段，还有表和字段之间的关系。 与之对应的，在ES中：Mapping定义索引下的Type的字段处理规则，即索引如何建立、索引类型、是否保存原始索引JSON文档、是否压缩原始JSON文档、是否需要分词处理、如何进行分词处理等。
- 在数据库中的增insert、删delete、改update、查search操作等价于ES中的增PUT/POST、删Delete、改_update、查GET.

##### 4.要知道ELK是什么？

ELK=`elasticsearch`+`Logstash`+`kibana`
elasticsearch：后台分布式存储以及全文检索
logstash: 日志加工、“搬运工”
kibana：数据可视化展示。
ELK架构为数据分布式存储、可视化查询和日志解析创建了一个功能强大的管理链。 三者相互配合，取长补短，共同完成分布式大数据处理工作。

##### 5.ES的一个Demo

开启ES之后，访问http://localhost:9200 会返回一个JSON串，包含当前节点、集群、版本等信息

```json
{
    "name": "kETAJ4V",
    "cluster_name": "elasticsearch",
    "cluster_uuid": "4CJW1jhoRCuTiLXaXvmXSw",
    "version": {
        "number": "6.6.0",
        "build_flavor": "default",
        "build_type": "zip",
        "build_hash": "a9861f4",
        "build_date": "2019-01-24T11:27:09.439740Z",
        "build_snapshot": false,
        "lucene_version": "7.6.0",
        "minimum_wire_compatibility_version": "5.6.0",
        "minimum_index_compatibility_version": "5.0.0"
    },
    "tagline": "You Know, for Search"
}
```

默认情况下，ES 只允许本机访问，如果需要远程访问，可以修改 Elastic 安装目录的`config/elasticsearch.yml`文件，去掉`network.host`的注释，将它的值改成`0.0.0.0`，然后重新启动 Elastic。

```
network.host: 0.0.0.0
```

**1)新建和删除 Index**

新建 Index，可以直接向 Elastic 服务器发出 PUT 请求。新建一个名叫`weather`的 Index。

```
localhost:9200/weather
```

返回一个JSON串：

```json
{
    "acknowledged": true,
    "shards_acknowledged": true,
    "index": "weather"
}
```

里面的`acknowledged`字段表示操作成功。

删除Index，可以直接向 Elastic 服务器发出 DELETE请求

```
localhost:9200/weather
```

返回一个JSON串：

```json
{
    "acknowledged": true
}
```

2)**中文分词设置**

```json
$ curl -X PUT 'localhost:9200/accounts' -d '
{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}'
```

上面代码中，首先新建一个名称为`accounts`的 Index，里面有一个名称为`person`的 Type。`person`有三个字段。`user` `title` `desc`

这三个字段都是中文，而且类型都是文本（text），所以需要指定中文分词器，不能使用默认的英文分词器。

Elastic 的分词器称为 analyzer。对每个字段指定分词器。

```json
"user": {
  "type": "text",
  "analyzer": "ik_max_word",
  "search_analyzer": "ik_max_word"
}
```

上面代码中，`analyzer`是字段文本的分词器，`search_analyzer`是搜索词的分词器。`ik_max_word`分词器是插件`ik`提供的，可以对文本进行最大数量的分词。

#### 四、**数据操作**

##### 新增记录

向指定的 /Index/Type 发送 PUT 请求，就可以在 Index 里面新增一条记录。比如，向`/accounts/person`发送请求，就可以新增一条人员记录。

```json
#指定ID
$ curl -X PUT 'localhost:9200/accounts/person/1' -d '
{
  "user": "张三",
  "title": "工程师",
  "desc": "数据库管理"
}'
```

服务器返回的 JSON 对象，会给出 Index、Type、Id、Version 等信息

```json
{
  "_index":"accounts",
  "_type":"person",
  "_id":"1",
  "_version":1,
  "result":"created",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":true
}
```

请求路径`/accounts/person/1`，最后的`1`是该条记录的 Id。它不一定是数字，任意字符串（比如`abc`）都可以。

新增记录的时候，也可以不指定 Id，这时要改成 POST 请求。

```json
#不指定ID
$ curl -X POST 'localhost:9200/accounts/person' -d '
{
  "user": "李四",
  "title": "工程师",
  "desc": "系统管理"
}'
```

上面代码中，向`/accounts/person`发出一个 POST 请求，添加一个记录。这时，服务器返回的 JSON 对象里面，`_id`字段就是一个随机字符串。

```json
{
  "_index":"accounts",
  "_type":"person",
  "_id":"AV3qGfrC6jMbsbXb6k1p",
  "_version":1,
  "result":"created",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":true
}
```

> 注意，如果没有先创建 Index（这个例子是`accounts`），直接执行上面的命令，ES也不会报错，而是直接生成指定的 Index。所以，打字的时候要小心，不要写错 Index 的名称。

##### 查看记录

向`/Index/Type/Id`发出 GET 请求，就可以查看这条记录。

```
$ curl 'localhost:9200/accounts/person/1?pretty=true'
```

上面代码请求查看`/accounts/person/1`这条记录，URL 的参数`pretty=true`表示以易读的格式返回。

返回的数据中，`found`字段表示查询成功，`_source`字段返回原始记录。

```json
{
  "_index" : "accounts",
  "_type" : "person",
  "_id" : "1",
  "_version" : 1,
  "found" : true,
  "_source" : {
    "user" : "张三",
    "title" : "工程师",
    "desc" : "数据库管理"
  }
}
```

##### 删除记录

删除记录就是发出 DELETE 请求

```
$ curl -X DELETE 'localhost:9200/accounts/person/1'
```

##### 更新记录

更新记录就是使用 PUT 请求，重新发送一次数据。

```json
$ curl -X PUT 'localhost:9200/accounts/person/1' -d '
{
    "user" : "张三",
    "title" : "工程师",
    "desc" : "数据库管理，软件开发"
}' 

{
  "_index":"accounts",
  "_type":"person",
  "_id":"1",
  "_version":2,
  "result":"updated",
  "_shards":{"total":2,"successful":1,"failed":0},
  "created":false
}
```

##### 查询记录

使用 GET 方法，直接请求`/Index/Type/_search`，就会返回所有记录。

```json
$ curl 'localhost:9200/accounts/person/_search'

{
  "took":2,
  "timed_out":false,
  "_shards":{"total":5,"successful":5,"failed":0},
  "hits":{
    "total":2,
    "max_score":1.0,
    "hits":[
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"AV3qGfrC6jMbsbXb6k1p",
        "_score":1.0,
        "_source": {
          "user": "李四",
          "title": "工程师",
          "desc": "系统管理"
        }
      },
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"1",
        "_score":1.0,
        "_source": {
          "user" : "张三",
          "title" : "工程师",
          "desc" : "数据库管理，软件开发"
        }
      }
    ]
  }
}
```

- `total`：返回记录数，本例是2条。
- `max_score`：最高的匹配程度，本例是`1.0`。
- `hits`：返回的记录组成的数组

##### 全文搜索

Elastic 的查询非常特别，使用自己的查询语法，要求 GET 请求带有数据体。

```
$ curl 'localhost:9200/accounts/person/_search'  -d '
{
  "query" : { "match" : { "desc" : "软件" }}
}'
```

上面代码使用 Match 查询，指定的匹配条件是`desc`字段里面包含”软件”这个词。返回结果如下。

```json
{
  "took":3,
  "timed_out":false,
  "_shards":{"total":5,"successful":5,"failed":0},
  "hits":{
    "total":1,
    "max_score":0.28582606,
    "hits":[
      {
        "_index":"accounts",
        "_type":"person",
        "_id":"1",
        "_score":0.28582606,
        "_source": {
          "user" : "张三",
          "title" : "工程师",
          "desc" : "数据库管理，软件开发"
        }
      }
    ]
  }
}
```

Elastic 默认一次返回10条结果，可以通过`size`字段改变这个设置。

```
$ curl 'localhost:9200/accounts/person/_search'  -d '
{
  "query" : { "match" : { "desc" : "管理" }},
  "size": 1
}'
```

还可以通过`from`字段，指定位移。

```
# 从位置1开始（默认是从位置0开始），只返回一条结果
$ curl 'localhost:9200/accounts/person/_search'  -d '
{
  "query" : { "match" : { "desc" : "管理" }},
  "from": 1,
  "size": 1
}'
```

#### 五、ES的分布式架构原理

多台机器上启动多个ES进程实例，组成了一个ES集群。ES中存储数据的基本单位是索引，比如说你现在要在ES中存储一些订单数据，你就应该在es中创建一个索引，order_index，所有的订单数据就都写到这个索引里面去，一个索引差不多就是相当于是mysql里的个库，这个库里那边包含一张表type，好像从6.X开始是一个Index中包含一个Type，从7.X中彻底移除Type。

接着创建一个索引，这个索引可以拆分成多个shard（分片），每个shard存储部分数据。

这个shard的数据实际是有多个备份，就是说每个shard都有一个primary shard，负责写入数据，但是还有几个replica shard。primary shard写入数据之后，会将数据同步到其他几个replica shard上去。

> 这个primary shard和replica shard是在不同机器上的。

通过这个replica的方案，每个shard的数据都有多个备份，如果某个机器宕机了，没关系啊，还有别的数据副本在别的机器上呢。

es集群多个节点，会自动选举一个节点为master节点，这个master节点其实就是干一些管理的工作的，比如维护索引元数据拉，负责切换primary shard和replica shard身份拉，之类的。

要是master节点宕机了，那么会重新选举一个节点为master节点。

如果是非master节点宕机了，那么会由master节点，让那个宕机节点上的primary shard的身份转移到其他机器上的replica shard。急着你要是修复了那个宕机机器，重启了之后，master节点会控制将缺失的replica shard分配过去，同步后续修改的数据之类的，让集群恢复正常。

#### 六、ES的写入和查询工作流程是怎么样的？

##### 一个写流程是怎么样的呢？

一个大的流程：

- 客户端选择一个node发送请求过去，这个node就成为coordinating node（协调节点）
- coordinating node，对document进行路由，将请求转发给对应的node（有primary shard）
- 实际的node上的primary shard处理请求，然后将数据同步到replica node
- coordinating node，如果发现primary node和所有replica node都搞定之后，就返回响应结果给客户端

底层的原理：关键词：`refresh`、`flush`、`translog`、`merge`

- 先写入buffer，在buffer里的时候数据是搜索不到的；同时将数据写入translog日志文件

  - 内存buffer，并不是直接写到了磁盘文件上的

- 如果buffer快满了，或者到一定时间，就会将buffer数据

  ```
  refresh
  ```

  到一个新的

  ```
  segment file
  ```

  中，但是此时数据不是直接进入segment file的磁盘文件的，而是先进入

  ```
  os cache
  ```

  的。这个过程就是refresh

  - os cache 是操作系统级缓存
  - 每隔1秒钟，es将buffer中的数据写入一个新的segment file，每秒钟会产生一个新的磁盘文件，segment
    file，这个segment file中就存储最近1秒内buffer中写入的数据
  - 只要buffer中的数据被refresh操作，刷入os cache中，就代表这个数据就可以被搜索到了，所以说ES是`准实时`的

- 重复上面的步骤，新的数据不断进入buffer和translog，不断将buffer数据写入一个又一个新的segment file中去，每次refresh完buffer清空，translog保留。随着这个过程推进，translog会变得越来越大。当translog达到一定长度的时候，就会触发commit操作。

  - commit操作发生第一步，就是将buffer中现有数据refresh到os cache中去，清空buffer
  - 将一个commit point写入磁盘文件，里面标识着这个commit point对应的所有segment file
  - 强行将os cache中目前所有的数据都fsync到磁盘文件中去

- 将现有的translog清空，然后再次重启启用一个translog，此时commit操作完成。默认每隔30分钟会自动执行一次commit，但是如果translog过大，也会触发commit。整个commit的过程，叫做`flush`操作。我们可以手动执行flush操作，就是将所有os cache数据刷到磁盘文件中去。

- translog其实也是先写入os cache的，默认每隔5秒刷一次到磁盘中去，所以默认情况下，可能有5秒的数据会仅仅停留在buffer或者translog文件的os cache中

- 如果是删除操作，commit的时候会生成一个.del文件，里面将某个doc标识为deleted状态，那么搜索的时候根据.del文件就知道这个doc被删除了

- 如果是更新操作，就是将原来的doc标识为deleted状态，然后新写入一条数据

- buffer每次refresh一次，就会产生一个segment file，所以默认情况下是1秒钟一个segment file，segment file会越来越多，此时会定期执行`merge`

- 每次merge的时候，会将多个segment file合并成一个，同时这里会将标识为deleted的doc给物理删除掉，然后将新的segment file写入磁盘，这里会写一个commit point，标识所有新的segment file，然后打开segment file供搜索使用，同时删除旧的segment file

##### 一个搜索流程是怎样的呢？

- 客户端发送请求到一个coordinate node
- 协调节点将搜索请求转发到所有的shard对应的primary shard或replica shard也可以
- query phase：每个shard将自己的搜索结果（其实就是一些doc id），返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果
- fetch phase：接着由协调节点，根据doc id去各个节点上拉取实际的document数据，最终返回给客户端

#### 七、如何优化查询性能呢？

ES性能优化是没有什么银弹的，啥意思呢？就是不要期待着随手调一个参数，就可以万能的应对所有的性能慢的场景。

##### 方案一、filesystem cache

你往es里写的数据，实际上都写到磁盘文件里去了，磁盘文件里的数据操作系统会自动将里面的数据缓存到os cache里面去。

es的搜索引擎严重依赖于底层的filesystem cache，如果给filesystem cache更多的内存，尽量让内存可以容纳所有的indx segment file索引数据文件，那么你搜索的时候就基本都是走内存的，性能会非常高。

**最佳的情况下，就是你的机器的内存，至少可以容纳你的总数据量的一半**

对于存入的数据来说，提前规划好用哪些字段去查，仅仅放要用于搜索的几个关键字段即可，尽量写入es的数据量跟es机器的filesystem cache是差不多的就可以了；其他不用来检索的数据放hbase里，或者mysql。

##### 方案二、数据预热

就比如说，微博，你可以把一些大v，平时看的人很多的数据给提前你自己后台搞个系统，每隔一会儿，你自己的后台系统去搜索一下热数据，刷到filesystem cache里去，后面用户实际上来看这个热数据的时候，他们就是直接从内存里搜索了，很快。

对于那些你觉得比较热的，经常会有人访问的数据，最好做一个专门的缓存预热子系统，就是对热数据，每隔一段时间，你就提前访问一下，让数据进入filesystem cache里面去。这样期待下次别人访问的时候，一定性能会好一些。

##### 方案三、冷热分离

es可以做类似于mysql的水平拆分，就是说将大量的访问很少，频率很低的数据，单独写一个索引，然后将访问很频繁的热数据单独写一个索引。

最好是将冷数据写入一个索引中，然后热数据写入另外一个索引中，这样可以确保热数据在被预热之后，尽量都让他们留在filesystem os cache里，别让冷数据给冲刷掉。

##### 方案四、document模型设计

es里面的复杂的关联查询，复杂的查询语法，尽量别用，一旦用了性能一般都不太好。

很多操作，不要在搜索的时候才想去执行各种复杂的乱七八糟的操作。es能支持的操作就是那么多，不要考虑用es做一些它不好操作的事情。如果真的有那种操作，尽量在document模型设计的时候，写入的时候就完成。另外对于一些太复杂的操作，比如join，nested，parent-child搜索都要尽量避免，性能都很差的。


  