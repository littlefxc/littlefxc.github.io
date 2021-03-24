---
title: spring boot 2.3.7 + es7 集成
date: 2021-01-14 21:04:41
categories: elasticsearch
tags: es, springboot
---

[springboot2.3.3中关于spring-data-elasticsearch-4.0.3的使用_阿良~的博客-CSDN博客](https://blog.csdn.net/qq_20280007/article/details/108388459?utm_medium=distribute.pc_relevant.none-task-blog-OPENSEARCH-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-OPENSEARCH-1.control)

# 依赖

```xml
<!-- spring boot 2.3.7 以上 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.7.RELEASE</version>
    <relativePath/>
</parent>

<!-- 2.3.7.RELEASE 支持 ES 7.X -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    <version>2.3.7.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

# 配置文件

```yaml
spring:
  elasticsearch:
    rest:
      uris: <http://localhost:9200>
```

# **对象映射**

- 首先要清楚注解的使用。

  - @Document：类注解，以指示该类是映射到数据库的候选对象。最重要的属性是：
    - indexName：用于存储此实体的索引的名称
    - type：映射类型。如果未设置，则使用小写的类的简单名称。（当前版本开始不推荐使用）
    - shards：索引的分片数。（经测试，分片数无法设置，应该是个BUG，推荐先创建索引设置好分片数和副本数）
    - replicas：索引的副本数。（经测试，分片数无法设置，应该是个BUG）
    - refreshIntervall：索引的刷新间隔。用于索引创建。默认值为“ 1s”。
    - indexStoreType：索引的索引存储类型。用于索引创建。默认值为“ fs”。
    - createIndex：配置是否在存储库引导中创建索引。默认值为true。
    - versionType：版本管理的配置。默认值为EXTERNAL。
  - @Id：字段注解，以标记用于标识目的的字段。
  - @Field：在字段级别应用并定义字段的属性，大多数属性映射到各自的Elasticsearch映射定义（以下列表不完整，请查看注释Javadoc以获取完整的参考）：
    - name：字段名称，它将在Elasticsearch文档中表示，如果未设置，则使用Java字段名称。
    - type：字段类型，可以是文本，关键字，长整数，短整数，字节，双精度，浮点型，半浮点数，标度浮点数，日期，布尔值，二进制，整数等。（属性（FieldType）类型不灵活）
    - format和日期类型的pattern定义。必须为日期类型定义。
    - store：标记是否将原始字段值存储在Elasticsearch中，默认值为false。
    - analyzer，searchAnalyzer，normalizer用于指定自定义分析和正规化。
  - @GeoPoint：将字段标记为geo_point数据类型。如果字段是GeoPoint类的实例，则可以省略。

- 示例

  ```java
  import org.springframework.data.annotation.Id;
  import org.springframework.data.elasticsearch.annotations.Document;
  import org.springframework.data.elasticsearch.annotations.Field;
  import org.springframework.data.elasticsearch.annotations.FieldType;
  
  /**
   * 经测试，更改 shards，replicas的值无法更改
   */
  @Document(indexName = "stu", shards = 3, replicas = 2, createIndex = false)
  public class Stu {
  
      @Id
      private Long stuId;
  
      @Field(store = true)
      private String name;
  
      @Field(store = true)
      private Integer age;
  
      @Field(store = true)
      private Float money;
  
      @Field(store = true, type = FieldType.Keyword)
      private String sign;
  
      @Field(store = true)
      private String description;
  
      // 省略 getter、setter
  
      @Override
      public String toString() {
          return "Stu{" +
                  "stuId=" + stuId +
                  ", name='" + name + '\\'' +
                  ", age=" + age +
                  ", money=" + money +
                  ", sign='" + sign + '\\'' +
                  ", description='" + description + '\\'' +
                  '}';
      }
  }
  ```

# ElasticsearchRestTemplate 用法

```java
import com.fengxuechao.FoodieSearchApp;
import com.fengxuechao.es.pojo.Stu;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.fetch.subphase.highlight.HighlightBuilder;
import org.elasticsearch.search.sort.FieldSortBuilder;
import org.elasticsearch.search.sort.SortBuilder;
import org.elasticsearch.search.sort.SortOrder;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.core.ElasticsearchRestTemplate;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.document.Document;
import org.springframework.data.elasticsearch.core.mapping.IndexCoordinates;
import org.springframework.data.elasticsearch.core.query.*;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = FoodieSearchApp.class)
public class ESTest {

    @Autowired
    private ElasticsearchRestTemplate esTemplate;

    /**
     * 不建议使用 ElasticsearchRestTemplate 对索引进行管理（创建索引，更新映射，删除索引）
     * 索引就像是数据库或者数据库中的表，我们平时是不会是通过java代码频繁的去创建修改删除数据库或者表的
     * 我们只会针对数据做CRUD的操作
     * 在es中也是同理，我们尽量使用 ElasticsearchRestTemplate 对文档数据做CRUD的操作
     * 1. 属性（FieldType）类型不灵活
     * 2. 主分片与副本分片数无法设置
     */

    @Test
    public void createIndexStu() {

        Stu stu = new Stu();
        stu.setStuId(1007L);
        stu.setName("iron man");
        stu.setAge(22);
        stu.setMoney(1000.8f);
        stu.setSign("I am iron man");
        stu.setDescription("I have a spider man");

        IndexQuery indexQuery = new IndexQueryBuilder().withObject(stu).build();
        esTemplate.index(indexQuery, esTemplate.getIndexCoordinatesFor(stu.getClass()));
    }

    @Test
    public void deleteIndexStu() {
        esTemplate.indexOps(Stu.class).delete();
    }

//    ------------------------- 我是分割线 --------------------------------

    @Test
    public void updateStuDoc() {

        Map<String, Object> sourceMap = new HashMap<>();
        sourceMap.put("name", "spider man");
        sourceMap.put("money", 99.8f);
        sourceMap.put("age", 33);

        IndexRequest indexRequest = new IndexRequest();
        indexRequest.source(sourceMap);

        UpdateQuery updateQuery = UpdateQuery.builder("1004")
                .withDocument(Document.from(sourceMap))
                .build();

//        update stu set sign='abc',age=33,money=88.6 where docId='1002'

        esTemplate.update(updateQuery, IndexCoordinates.of("stu"));
    }

    @Test
    public void getStuDoc() {
        Stu stu = esTemplate.get("1004", Stu.class);

        System.out.println(stu);
    }

    @Test
    public void deleteStuDoc() {
        esTemplate.delete("1002", IndexCoordinates.of("stu"));
    }

//    ------------------------- 我是分割线 --------------------------------

    @Test
    public void searchStuDoc() {

        Pageable pageable = PageRequest.of(0, 2);

        NativeSearchQuery query = new NativeSearchQueryBuilder()
                .withQuery(QueryBuilders.matchQuery("description", "spider man"))
                .withPageable(pageable)
                .build();
        SearchHits<Stu> hits = esTemplate.search(query, Stu.class, esTemplate.getIndexCoordinatesFor(Stu.class));
        System.out.println("检索后的总分页数目为：" + hits.getTotalHits());
        for (SearchHit<Stu> hit : hits) {
            System.out.println(hit.getContent());
        }

    }

    @Test
    public void highlightStuDoc() {

        String preTag = "<font color='red'>";
        String postTag = "</font>";

        Pageable pageable = PageRequest.of(0, 10);

        SortBuilder sortBuilder = new FieldSortBuilder("money")
                .order(SortOrder.DESC);
        SortBuilder sortBuilderAge = new FieldSortBuilder("age")
                .order(SortOrder.ASC);

        NativeSearchQuery query = new NativeSearchQueryBuilder()
                .withQuery(QueryBuilders.matchQuery("description", "spider man"))
                .withHighlightFields(new HighlightBuilder.Field("description")
                        .preTags(preTag)
                        .postTags(postTag))
                .withSort(sortBuilder)
                .withSort(sortBuilderAge)
                .withPageable(pageable)
                .build();

        SearchHits<Stu> hits = esTemplate.search(query, Stu.class);

        System.out.println("检索后的总分页数目为：" + hits.getTotalHits());
        // 将
        List<SearchHit<Stu>> list = hits.get()
                .peek(stuSearchHit -> stuSearchHit.getContent().setDescription(stuSearchHit.getHighlightField("description").get(0))).collect(Collectors.toList());
        for (SearchHit<Stu> hit : list) {
            System.out.println(hit.getContent());
        }

    }
}
```