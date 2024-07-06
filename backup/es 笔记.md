ES（Elasticsearch）是一个基于Lucene构建的开源搜索引擎，主要用于处理搜索、分析和存储大量数据。
elasticsearch之所以有如此高性能的搜索表现，正是得益于底层的倒排索引技术。
正向索引：先找到文档（一条数据），在匹配内容
倒排索引：通过词条来找文档（一条数据）
总结：
正向索引：
- 优点： 
  - 可以给多个字段创建索引
  - 根据索引字段搜索、排序速度非常快
- 缺点： 
  - 根据非索引字段，或者索引字段中的部分词条查找时，只能全表扫描。

倒排索引：
- 优点： 
  - 根据词条搜索、模糊搜索时，速度非常快
- 缺点： 
  - 只能给词条创建索引，而不是字段
  - 无法根据字段做排序
 

倒排索引中有两个非常重要的概念：
- 文档（Document）：用来搜索的数据，其中的每一条数据就是一个文档。例如一个网页、一个商品信息
- 词条（Term）：对文档数据或用户搜索数据，利用某种算法分词，得到的具备含义的词语就是词条。例如：我是中国人，就可以分为：我、是、中国人、中国、国人这样的几个词条

使用 es  来进行关键字搜索流程：
1.  对 es 中 text 类型的数据进行分词，保存到一个词库中，词库中保存词条和文档的对应关系
2.  对用户输入的内容进行分词
3.  根据分好词的词条去词库中匹配，获取匹配成功的文档id
4.  根据获取到的文档id来收集好数据，进行返回
![download_image(4)](https://github.com/2004GIF/chenzian-shrimp.github.io/assets/126451952/50430132-32fa-4e61-adae-9fcb0ca63cbe)

索引库操作
Index就类似数据库表，Mapping映射就类似表的结构。我们要向es中存储数据，必须先创建Index和Mapping

Mapping映射属性
Mapping是对索引库中文档的约束，常见的Mapping属性包括：
- type：字段数据类型，常见的简单类型有： 
  - 字符串：text（可分词的文本）、keyword（精确值，例如：品牌、国家、ip地址）
  - 数值：long、integer、short、byte、double、float、
  - 布尔：boolean
  - 日期：date
  - 对象：object
- index：是否创建索引，默认为true
- analyzer：使用哪种分词器
- properties：该字段的子字段

示例：
```json
# PUT /heima
{
  "mappings": {
    "properties": {
      "info":{
        "type": "text",
        "analyzer": "ik_smart"
      },
      "email":{
        "type": "keyword",
        "index": "false"
      },
      "name":{
        "properties": {
          "firstName": {
            "type": "keyword"
          }
        }
      }
    }
  }
}
```



### 索引库操作
#### 新增索引
控制台：
````json
PUT /items   //  Restful API  接口
{
  "mappings": {   //  映射，其实就是对字段的约束
    "properties": {
       "id":{   // 字段名
         "type": "keyword" // 使用的类型 keyword 
       },
       "info":{
         "type": "text",
         "analyzer": "ik_smart"
       }
    }
  }
}

````
java API：
````java
  @Test
    void testCreateIndex() throws IOException {
        // 1. 准备request
        CreateIndexRequest request = new CreateIndexRequest("items");
        // 2. 准备请求参数
        request.source(MAPPING_TEMPLATE, XContentType.JSON);
        // 3. 发送请求 client.indices() 获取操作索引库的客户端对象
        client.indices().create(request, RequestOptions.DEFAULT);
    }

private static final String MAPPING_TEMPLATE = "{\n" +
            "  \"mappings\": {\n" +
            "    \"properties\": {\n" +
            "      \"id\": {\n" +
            "        \"type\": \"keyword\"\n" +
            "      },\n" +
            "      \"name\":{\n" +
            "        \"type\": \"text\",\n" +
            "        \"analyzer\": \"ik_max_word\"\n" +
            "      },\n" +
            "      \"price\":{\n" +
            "        \"type\": \"integer\"\n" +
            "      },\n" +
            "      \"stock\":{\n" +
            "        \"type\": \"integer\"\n" +
            "      },\n" +
            "      \"image\":{\n" +
            "        \"type\": \"keyword\",\n" +
            "        \"index\": false\n" +
            "      },\n" +
            "      \"category\":{\n" +
            "        \"type\": \"keyword\"\n" +
            "      },\n" +
            "      \"brand\":{\n" +
            "        \"type\": \"keyword\"\n" +
            "      },\n" +
            "      \"sold\":{\n" +
            "        \"type\": \"integer\"\n" +
            "      },\n" +
            "      \"commentCount\":{\n" +
            "        \"type\": \"integer\"\n" +
            "      },\n" +
            "      \"isAD\":{\n" +
            "        \"type\": \"boolean\"\n" +
            "      },\n" +
            "      \"updateTime\":{\n" +
            "        \"type\": \"date\"\n" +
            "      }\n" +
            "    }\n" +
            "  }\n" +
            "}";


````

#### 删除索引库
控制台：
````json
DELETE  /items
````
Java :
````java
  @Test
    void testDeleteIndex() throws IOException {
        DeleteIndexRequest request = new DeleteIndexRequest("items");
        client.indices().delete(request, RequestOptions.DEFAULT);
    }
````
#### 获取索引库信息
控制台
````json
GET /item
````
Java：
````java
 @Test
    void testGetIndex() throws IOException {
        GetIndexRequest request = new GetIndexRequest("items");
       // 这里是在判断的索引库是否存在，如果直接获取索引库信息还需要自己进行解析
        boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);  
        System.out.println("exists = " + exists); 
    }
````

#### 修改索引库（只能给索引库新增字段，不能修改原来的字段）
控制台：
````json
PUT /my_user/_mapping
{
  "properties":{
    "username":{
      "type": "text"
    }
  }
}

````
java : 略

### 操作文档
#### 新增文档
控制台：
````json
POST /my_user/_doc/1   // /索引库名/_doc/文档的id （不写自动生成）
{
  "age": 12 ,
  "info": "是一个好人",
  "name":{
    "firstName":"赵",
    "lastName": "云"
  }
}

````
Java：
````java
  @Test
    void testCreateDoc() throws IOException {
        // 0. 准备 json 数据
        Item item = itemService.getById(317578L);
        ItemDoc itemDoc = BeanUtil.copyProperties(item, ItemDoc.class);
        itemDoc.setCommentCount(0);
        // 1. 准备 request
        IndexRequest request = new IndexRequest("items").id(itemDoc.getId());

        // 2. 准备请求参数
        request.source(JSONUtil.toJsonStr(itemDoc), XContentType.JSON);

        // 3. 发送请求
        IndexResponse response = client.index(request, RequestOptions.DEFAULT);
        System.out.println("response = " + response);

    }
```

#### 获取文档
````json
GET /items/_doc/1   // 获取items索引库中id为1的文档数据
````
Java：
````java
    @Test
    void testGetDoc() throws IOException {
        // 1. 准备 request
        GetRequest request = new GetRequest("items", "317578");
        // 2. 发送请求
        GetResponse response = client.get(request, RequestOptions.DEFAULT);
        // 3. 解析结果
        String json = response.getSourceAsString();
        ItemDoc itemDoc = JSONUtil.toBean(json, ItemDoc.class);
        System.out.println("itemDoc = " + itemDoc);
    }
````

#### 删除文档
控制台：
````json
DELETE /items/_doc/1   // 删除id为1的文档
````
Java ： 
````java
 @Test
    void testDelDoc() throws IOException {
        DeleteRequest request = new DeleteRequest("items", "317578");

        client.delete(request, RequestOptions.DEFAULT);
    }

````
#### 修改文档
##### 全量修改 （覆盖原来的文档 , 新增或者修改，主要看id是否存在）
控制台：
````json
# 全量修改（全局修改 ， 如果id存在修改文档，如果id不存在新增文档）
PUT /my_user/_doc/2
{
  "age": 19,
  "info": "是一个好人",
  "name": {
    "firstName": "赵",
    "lastName": "拔"
  }
}
````
Java：
````java
  @Test
    void testCreateDoc() throws IOException {
        // 0. 准备 json 数据
        Item item = itemService.getById(317578L);
        ItemDoc itemDoc = BeanUtil.copyProperties(item, ItemDoc.class);
        itemDoc.setCommentCount(0);
        // 1. 准备 request
        IndexRequest request = new IndexRequest("items").id(itemDoc.getId());

        // 2. 准备请求参数
        request.source(JSONUtil.toJsonStr(itemDoc), XContentType.JSON);

        // 3. 发送请求
        IndexResponse response = client.index(request, RequestOptions.DEFAULT);
        System.out.println("response = " + response);

    }
````
##### 定量修改  （修改文档中某个值）
控制台：
````json
# 定量修改（局部修改）
POST /my_user/_update/1
{
  "doc": {
    "age": 100
  }
}

````
Java：
````java
@Test
    void testUpdateDoc() throws IOException {
        UpdateRequest request = new UpdateRequest("items", "317578");
        Map<String, Object> map = new HashMap<>();
        map.put("price", 30000);
        map.put("commentCount", 10);
        request.doc(map);
        client.update(request, RequestOptions.DEFAULT);
    }
````

### 对文档的批量操作
#### 新增
控制台：
````
# 批量新增
POST /_bulk
{"index": {"_index": "my_user" , "_id": 3}}
{"age": 22 , "info":"是一个好人" , "name": { "firstName": "李" ,"lastName": "四" }}
{"index": {"_index": "my_user" , "_id": 5}}
{"age": 22 , "info":"是一个好人" , "name": { "firstName": "王" ,"lastName": "五" }}

````
Java：
````java
 @Test
    void testBulkAddDoc() throws IOException {
        Page<Item> page = Page.of(1, 5);
        itemService.page(page);
        List<Item> itemList = page.getRecords();
        List<ItemDoc> itemDocs = BeanUtil.copyToList(itemList, ItemDoc.class);
   
        BulkRequest request = new BulkRequest();
        // 在 BulkReust 对象中添加其他操作的对象就可以完成批量操作了，必须可以信息 DeleteRequest 、UpdateRequest、IndexRequest 
        // 其他其他的request 对象
        for (ItemDoc itemDoc : itemDocs) {
            IndexRequest indexRequest = new IndexRequest("items").id(itemDoc.getId()).source(JSONUtil.toJsonStr(itemDoc), XContentType.JSON);
            request.add(indexRequest);
        }

        client.bulk(request, RequestOptions.DEFAULT);
    }
````

## 使用DSL 语句来查询
Elasticsearch的查询可以分为两大类：
- 叶子查询（Leaf query clauses）：一般是在特定的字段里查询特定值，属于简单查询，很少单独使用。
- 复合查询（Compound query clauses）：以逻辑方式组合多个叶子查询或者更改叶子查询的行为方式。

叶子查询：
-  全文检索查询（Full Text Queries）：利用分词器对用户输入搜索条件先分词，得到词条，然后再利用倒排索引搜索词条。例如：
      - match：
      - multi_match
- 精确查询（Term-level queries）：不对用户输入搜索条件分词，根据字段内容精确值匹配。但只能查找keyword、数值、日期、boolean类型的字段。例如：
     - ids
     - term
     - range
- 地理坐标查询：用于搜索地理位置，搜索方式很多，例如：
     - geo_bounding_box：按矩形搜索
     - geo_distance：按点和半径搜索

 
###  全文检索查询
利用分词器对用户输入搜索条件先分词，得到词条，然后再利用倒排索引搜索词条。比如：match 、    multi_match
#### match 查询 ---> 针对可分词的字段
控制台：
````
GET /items/_search
{
  "query": {
    "match": {
      "brand": "华为"
    }
  }
}
````

Java：
````java
  @Test
    void testSearchMatch() throws IOException {
        // 1. 准备 request 对象
        SearchRequest request = new SearchRequest("items");
        // 2. 准备请求参数 QueryBuilders.matchQuery("name" , "华为") 一个字段模糊查询 参数一：字段名 参数二：关键字
        request.source().query(QueryBuilders.matchQuery("name" , "华为"));
        // QueryBuilders.multiMatchQuery("华为", "name" , "brand") ， 参数一：关键字 参数二：多个字段
//        request.source().query(QueryBuilders.multiMatchQuery("华为", "name", "brand"));
        // 3. 发送请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);

        // 4.解析结果
        SearchHits hits = response.getHits();
        long total = hits.getTotalHits().value;
        System.out.println("total = " + total);
        SearchHit[] searchHits = hits.getHits();
        for (SearchHit searchHit : searchHits) {
            String json = searchHit.getSourceAsString();
            System.out.println("每一条数据： " + json);
        }
    }

````
查询结果
 
![download_image](https://github.com/2004GIF/chenzian-shrimp.github.io/assets/126451952/244e5a06-70e1-4705-88b1-db16ecff0fde)

Java：解析结果函数
````java
List<ItemDoc> handlerResponse(SearchResponse response) {
        List<ItemDoc> result = new ArrayList<>();
        SearchHits responseHits = response.getHits();
        long total = responseHits.getTotalHits().value;
        System.out.println("total = " + total);
        SearchHit[] hits = responseHits.getHits();
        for (SearchHit hit : hits) {
            String json = hit.getSourceAsString();
            ItemDoc itemDoc = JSONUtil.toBean(json, ItemDoc.class);

            Map<String, HighlightField> hfs = hit.getHighlightFields();

            if (CollUtil.isNotEmpty(hfs)) {
                HighlightField hf = hfs.get("name");
                StringBuilder sb = new StringBuilder();
                // 注意高亮的返回的结果是一个数组，如果设置高亮的内容过长 es 在底层会对结果进行分片
                Text[] fragments = hf.getFragments();
                for (Text fragment : fragments) {
                    sb.append(fragment.string());
                }
                itemDoc.setName(sb.toString());
            }

            result.add(itemDoc);
        }
        return result;
    }
````

####   multi_match查询 ---> 针对可分词的字段
控制台
````
GET /items/_search
{
  "query": {
    "multi_match": {
      "query": "华为",
      "fields": [ "name" ,"brand" ]
    }
  }
}
````
Java：
````java
@Test
    void testSearchMatch() throws IOException {
        // 1. 准备 request 对象
        SearchRequest request = new SearchRequest("items");
        // 2. 准备请求参数 QueryBuilders.matchQuery("name" , "华为") 一个字段模糊查询 参数一：字段名 参数二：关键字
//        request.source().query(QueryBuilders.matchQuery("name" , "华为"));
//         QueryBuilders.multiMatchQuery("华为", "name" , "brand") ， 参数一：关键字 参数二：多个字段
        request.source().query(QueryBuilders.multiMatchQuery("华为", "name", "brand"));
        // 3. 发送请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);

        // 4.解析结果
        SearchHits hits = response.getHits();
        long total = hits.getTotalHits().value;
        System.out.println("total = " + total);
        SearchHit[] searchHits = hits.getHits();
        for (SearchHit searchHit : searchHits) {
            String json = searchHit.getSourceAsString();
            System.out.println("每一条数据： " + json);
        }
    }
````

### 精确查询
不对用户输入搜索条件分词，根据字段内容精确值匹配。但只能查找keyword、数值、日期、boolean类型的字段。例如：  ids 、  term、   range

#### ids
控制台：
````
GET /items/_search
{
  "query": {
    "ids": {
      "values": ["5935273" , "40116238853"]
    }
  }
}

````

Java ：
````java
  @Test
    void testSearchIds() throws IOException {
        SearchRequest request = new SearchRequest("items");
        // 构建请求参数
        request.source().query(QueryBuilders.idsQuery().addIds("5935273","40116238853"));
        
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        List<ItemDoc> itemDocs = handlerResponse(response);
        itemDocs.forEach(System.out::println);
    }
```

#### term  就是 eq
控制台：
````

````
Java ：
````java


````



#### range 就是 between
查询价格大于 200 块的商品
控制台：
````
GET /items/_search
{
  "query": {
    "range": {
      "price": {
        "lte": 20000
      }
    }
  }
}
````
Java ：
````java
   @Test
    void testSearchRange() throws IOException {
        SearchRequest request = new SearchRequest("items");
        // 构建请求参数
        request.source().query(QueryBuilders.rangeQuery("price").lte(20000));

        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        List<ItemDoc> itemDocs = handlerResponse(response);
        itemDocs.forEach(System.out::println);
    }

````


### 1.3.复合查询
复合查询大致可以分为两类：
- 第一类：基于逻辑运算组合叶子查询，实现组合条件，例如
  - bool
- 第二类：基于某种算法修改查询时的文档相关性算分，从而改变文档排名。例如：
  - function_score
  - dis_max
#### bool查询
即布尔查询。就是利用逻辑运算来组合一个或多个查询子句的组合。bool查询支持的逻辑运算有：
- must：必须匹配每个子查询，类似“与”
- should：选择性匹配子查询，类似“或”
- must_not：必须不匹配，不参与算分，类似“非”
- filter：必须匹配，不参与算分
注意：出于性能考虑，与搜索关键字无关的查询尽量采用must_not或filter逻辑运算，避免参与相关性算分。
查询  甘蒂牧场 中价格大于 200 的脱脂牛奶
控制台
````
GET /items/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "脱脂牛奶"
          }
        }
      ],
      "filter": [
        {
          "term": {
            "brand": "甘蒂牧场"
          },
          "range": {
            "price": {
              "lte": 20000
            }
          }
        }
      ]
    }
  }
}
````

java ：
````java
    @Test
    void testSearchDom() throws IOException {
        int pageNo = 2, pageSize = 5;
        // 准备 request
        SearchRequest request = new SearchRequest("items");

        // 2.1 构建查询条件
        BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
        boolQueryBuilder.must(QueryBuilders.matchQuery("name", "脱脂牛奶"));
        boolQueryBuilder.filter(QueryBuilders.termQuery("brand", "甘蒂牧场"));
        boolQueryBuilder.filter(QueryBuilders.rangeQuery("price").lt(20000));
        request.source()
                .query(boolQueryBuilder);

    
        // 发送请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 解析结果
        List<ItemDoc> itemDocs = handlerResponse(response);
        System.out.println("itemDocs = " + itemDocs);
    }

```
注意：只有字段是 text 类型是才使用全文检索


### 排序
elasticsearch默认是根据相关度算分（_score）来排序，但是也支持自定义方式对搜索结果排序。不过分词字段无法排序，能参与排序字段类型有：keyword类型、数值类型、地理坐标类型、日期类型等 ，对 text 类型进行排序没有意义。
对价格进行排序
控制台
````
GET /items/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": {
        "order": "desc"
      }
    }
  ]
}
````
java
````java
 @Test
    void testSearchSort() throws IOException {
        SearchRequest request = new SearchRequest("items");
        // 构建请求参数
        request.source().query(QueryBuilders.matchAllQuery());
        request.source().sort(SortBuilders.fieldSort("price"));

        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        List<ItemDoc> itemDocs = handlerResponse(response);
        itemDocs.forEach(System.out::println);
    }

````
### 分页
elasticsearch 默认情况下只返回top10的数据。而如果要查询更多数据就需要修改分页参数了。
#### 基础分页
elasticsearch中通过修改from、size参数来控制要返回的分页结果：
- from：从第几个文档开始
- size：总共查询几个文档
类似于mysql中的limit ?, ?
查询第2页五条数据
控制台
````
GET /items/_search
{
  "query": {
    "match_all": {}
  },
  "from": 5,
  "size": 5
}

````
Java 
````java
    @Test
    void testSearchPage() throws IOException {
        SearchRequest request = new SearchRequest("items");
        // 构建请求参数
        request.source().query(QueryBuilders.matchAllQuery());
        request.source().from(5).size(5);

        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        List<ItemDoc> itemDocs = handlerResponse(response);
        itemDocs.forEach(System.out::println);
    }
````

### 高亮
高亮原理：进行给查询的数据中匹配的字条使用标签包裹
给 name 中包含脱脂牛奶的关键字添加高亮效果
````
GET /items/_search
{
  "query": {
    "match": {
      "name": "脱脂牛奶"
    }
  },
  "highlight": {
    "fields": {
      "name": {}
    }
  }
}
````

````java
    @Test
    void testHighlight() throws IOException {
        // 准备 request
        SearchRequest request = new SearchRequest("items");
        // 构建查询条件
        request.source().query(QueryBuilders.matchQuery("name", "脱脂牛奶"));
        // 构建高亮条件
//        request.source().highlighter(new HighlightBuilder().field("name"));
        request.source().highlighter(SearchSourceBuilder.highlight().field("name"));
        // 发送请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);

        List<ItemDoc> itemDocs = handlerResponse(response);

        itemDocs.forEach(System.out::println);

    }
````
注意点：使用高亮必须要查询的关键字


### 数据聚合
聚合（aggregations）可以让我们极其方便的实现对数据的统计、分析、运算。
聚合常见的有三类：
-  桶（Bucket）聚合：用来对文档做分组 
  - TermAggregation：按照文档字段值分组，例如按照品牌值分组、按照国家分组
  - Date Histogram：按照日期阶梯分组，例如一周为一组，或者一月为一组
-  度量（Metric）聚合：用以计算一些值，比如：最大值、最小值、平均值等 
  - Avg：求平均值
  - Max：求最大值
  - Min：求最小值
  - Stats：同时求max、min、avg、sum等
-  管道（pipeline）聚合：其它聚合的结果为基础做进一步运算

#### 桶中 TermAggregation 查询
统计在商品索引库中有多少个商品分类
控制台
````
GET /items/_search
{
  "size": 0, 
  "aggs": {
    "category_agg": {
      "terms": {
        "field": "category",
        "size": 20
      }
    }
  }
}
````
语法说明：
- size：设置size为0，就是每页查0条，则结果中就不包含文档，只包含聚合
- aggs：定义聚合
  - category_agg：聚合名称，自定义，但不能重复
    - terms：聚合的类型，按分类聚合，所以用term
      - field：参与聚合的字段名称
      - size：希望返回的聚合结果的最大数量
Java
```java
  @Test
    void testAgg1() throws IOException {
        // 准备 request
        SearchRequest request = new SearchRequest("items");
        // 构建查询条件
        request.source().query(QueryBuilders.matchAllQuery());
        // 构建高亮条件 
        // AggregationBuilders.terms("cate_agg") 设置聚合的类型( terms )  和 聚合的名称(cate_agg)
        // field("category“)  根据哪个字段
        request.source().aggregation(AggregationBuilders.terms("cate_agg").field("category").size(20));
        // 发送请求
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);

        List<ItemDoc> itemDocs = handlerResponse(response);

        itemDocs.forEach(System.out::println);
    }
```
### 总结
aggs代表聚合，与query同级，此时query的作用是？
- 限定聚合的的文档范围
聚合必须的三要素：
- 聚合名称
- 聚合类型
- 聚合字段
聚合可配置属性有：
- size：指定聚合结果数量
- order：指定聚合结果排序方式
- field：指定聚合字段
