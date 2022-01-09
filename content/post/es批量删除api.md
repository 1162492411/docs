---
title: "Es批量删除api"
date: 2022-01-09T09:38:36+08:00
draft: false
toc: TRUE
categories:
  - 实战
tags:
  - Java
  - ES
  - ES基础Api
---

ES中通过该API实现数据的批量删除,官方对该API的定义为 : 根据特定的查询条件对ES相关索引中某些特定的文档进行批量删除

# API介绍

DSL 如下

```json
POST indexName1/_delete_by_query
{
  "query": { //这里传递查询条件
    "match": {
      "message": "some message"
    }
  }
}
```

响应如下

```json
{
  "took" : 147,// 从整个操作的开始到结束的毫秒数
  "timed_out": false,// 
  "deleted": 119,// 成功删除的文档数
  "batches": 1,// 通过查询删除的滚动响应数量
  "version_conflicts": 0,// 根据查询删除时，版本冲突的数量
  "noops": 0,// 
  "retries": {// 根据查询删除的重试次数是响应于完整队列
    "bulk": 0,
    "search": 0
  },
  "throttled_millis": 0,// 请求休眠的毫秒数，与`requests_per_second`一致
  "requests_per_second": -1.0,// 
  "throttled_until_millis": 0,// 
  "total": 119,// 
  "failures" : [ ]// 失败的索引数组
}
```

# 原理

> 在`_delete_by_query`执行期间，依次执行多个搜索请求，以便找到要删除的所有匹配文档。每次发现一批文档时，执行相应的批量请求以删除所有这些文档。如果搜索或批量请求被拒绝，`_delete_by_query`依赖于默认策略来重试拒绝的请求（最多10次，以指数返回）。达到最大重试次数限制会导致`_delete_by_query`中止，并在响应失败中返回所有故障。已经执行的删除仍然保持。换句话说，进程没有回滚，只会中止。当第一个故障导致中止时，失败批量请求返回的所有故障都会返回到故障元素中;因此，有可能会有不少失败的实体。注意，该API并不是真正将数据删除，而是将其标记为删除状态，并不会真正删除数据及释放磁盘空间

# 参数

| 参数                   | 默认值 | 值域                   | 作用                                                         |
| ---------------------- | ------ | ---------------------- | ------------------------------------------------------------ |
| refresh                |        | [true,false,wait_for]  | 发送`refresh`将在一旦根据查询删除完成之后， 刷新所有涉及到的分片；如果设置为wait_for,表示使用集群自动刷新机制,具体等待的时间优先取索引级配置,默认1s |
| wait_for_completion    |        | [true,false，任意正数] | 值为false时es将会创建一个任务来执行删除，可以与TASK API结合使用，ES会增加一条.tasks/task/${taskId}数据用来记录删除进度 |
| wait_for_active_shards |        |                        | 控制在继续请求之前必须有多少个分片必须处于活动状态           |
| timeout                |        |                        | 控制每个写入请求等待不可用分片变成可用的时间                 |
| requests_per_second    | -1     | 任意正数,-1            | -1时表示禁用该项，任意正数时，在多个批量批次间将会按该值进行等待 |
| conflicts              |        | proceed                | 计算删除数据时的版本冲突数量,使用该参数后将不会导致存在版本冲突时整个删除进度终止 |

# 异步

   当我们在删除请求中添加wait_for_completion=false时ES会返回一个taskId来异步进行删除数据。可以通过API查询删除进度

```json
-- 查询任务的进度
GET /_tasks/1
-- 响应
{
  "completed" : false,
  "task" : {
    "node" : "fFDobwtSQvS1ishWYtWQcg",
    "id" : 142113449,
    "type" : "transport",
    "action" : "indices:data/write/delete/byquery",
    "status" : {
      "total" : 10918603,
      "updated" : 0,
      "created" : 0,
      "deleted" : 0,
      "batches" : 243,
      "version_conflicts" : 242000,
      "noops" : 0,
      "retries" : {
        "bulk" : 0,
        "search" : 0
      },
      "throttled_millis" : 0,
      "requests_per_second" : -1.0,
      "throttled_until_millis" : 0
    },
    "description" : "delete-by-query [ads_lading_trade_brief_es_02]",
    "start_time_in_millis" : 1604650445876,
    "running_time_in_nanos" : 159708214419,
    "cancellable" : true,
    "headers" : {
     }
  }
}
-- 取消任务
POST _tasks/task_id:1/_cancel 
```



# 切片

   为了在尽量短的时间内删除数据，可以将数据通过scroll划分为不同的切片，然后并行删除多个切片。

   当然，该API还支持自动切片。

# 分段

​    在调用该API后，数据仅仅是被标记为删除状态，实际上并未物理删除，需要依赖于Lucene对分段进行合并才能真正完成物理删除数据的效果。

​    ES写入数据的过程实际上被拆分为了以下几步 : 

1. 先写到内存中，此时不可搜索

2. 默认经过 1s 之后会通过refresh被写入 lucene 的底层文件 segment 中 ，此时可以搜索到

3. 多个segment通过merge被合并为一个大的segment

4. 多个大的segment通过flush被刷新到磁盘

   https://img-blog.csdnimg.cn/20200823133011451.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zNzY5MjQ5Mw==,size_16,color_FFFFFF,t_70#pic_center

## refresh

​     在lucene 中，为了实现高索引速度，使用了segment 分段架构存储。一批写入数据保存在一个段中，其中每个段是磁盘中的单个文件，每个段会消耗文件句柄、内存和cpu运行周期。

  在 index ，Update , Delete , Bulk 等操作中，可以设置 refresh 的值。取值如下：

* refresh=true : 更新数据之后，立刻对相关的分片(包括副本) 刷新，这个刷新操作保证了数据更新的结果可以立刻被搜索到。

* refresh=wait_for : 这个参数表示，刷新不会立刻进行，而是等待一段时间才刷新 ( index.refresh_interval)，默认时间是 1 秒。刷新时间间隔可以通过index 的配置动态修改。或者直接手动刷新 POST /twitter/_refresh

* refresh=false : refresh 的默认值，更新数据之后不立刻刷新，在返回结果之后的某个时间点会自动刷新，也就是随机的，看es服务器的运行情况。

  ## merge

​    当lucene中的段的数量过多时，搜索性能会降低，同时搜索时占用的内存也会更多，因此有必要控制段的数量。ES在后台进行段合并，将小段合并为大段，将大段合并为更大的段，同时，在段合并期间，被标记为删除状态的数据不会被合并到新段中。段合并需要消耗IO资源和CPU时间。

​    es对于merge比较保守，可以通过参数限制每次merge的小段的数量(max_num_segments,默认1)，限制每次merge的带宽(max_bytes_per_sec,默认20MB/s) 

# 实战

## 场景一 : 根据批量的商品码删除ES数据

```java
import org.elasticsearch.ElasticsearchStatusException;
import org.elasticsearch.action.ActionListener;
import org.elasticsearch.action.admin.indices.create.CreateIndexRequest;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.get.GetRequest;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.Response;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.common.xcontent.XContentBuilder;
import org.elasticsearch.common.xcontent.XContentFactory;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.*;
import org.elasticsearch.index.reindex.BulkByScrollResponse;
import org.elasticsearch.index.reindex.DeleteByQueryRequest;

@Autowired
private RestHighLevelClient restHighLevelClient;

    @Override
    public void deleteBatchForArchive(List<String> goodsCodeList) {
        DeleteByQueryRequest request = new DeleteByQueryRequest(priceItemIndex);
        request.setDocTypes(priceItemType);
        request.setQuery(installSearchRequestForArchive(goodsCodeList));
        request.setConflicts("proceed");
        ActionListener<BulkByScrollResponse> listener = new ActionListener<BulkByScrollResponse>() {
            @Override
            public void onResponse(BulkByScrollResponse bulkByScrollResponse) {
                if(Objects.isNull(bulkByScrollResponse)){
                    log.info("返回空数据");
                    return;
                }
                if(goodsCodeList.size() != bulkByScrollResponse.getDeleted()){
                    log.error("删除了{}条商品码,其中{}条数据存在版本冲突,{}条数据删除失败,详情为:{}",bulkByScrollResponse.getDeleted(),
                            bulkByScrollResponse.getVersionConflicts(), CollectionUtil.getListSize(bulkByScrollResponse.getBulkFailures()),
                            new Gson().toJson(bulkByScrollResponse.getBulkFailures()));
                }
            }
            @Override
            public void onFailure(Exception e) {
                log.error("归档ES数据发送请求时异常,商品码为:{},",new Gson().toJson(goodsCodeList),e);
            }
        };
        // 调用异步方法,当ES侧完成请求时回调上面自定义的listener
        restHighLevelClient.deleteByQueryAsync(request,RequestOptions.DEFAULT,listener);
    }

    /**
     * 根据码列表+失效状态进行查询
     * {
     * 	"query": {
     * 		"bool": {
     * 			"must": [{
     * 				"term": {"effective": {"value": 0,"boost": 1.0}}},
     * 				{"terms": {"code": ["214016384187744256", "214015610148470784", "214060765785659392"],"boost": 1.0}
     *            }],
     * 			"adjust_pure_negative": true,
     * 			"boost": 1.0* 		}
     *    }
     * }
     * @param goodsCodeList
     * @return
     */
    private QueryBuilder installSearchRequestForArchive(List<String> goodsCodeList){
        //组装Query对象
        BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery();
        TermQueryBuilder statusTermQuery = QueryBuilders.termQuery("effective", 0);
        TermsQueryBuilder goodsCodeTermsQuery = QueryBuilders.termsQuery("code", goodsCodeList);
        queryBuilder.must(statusTermQuery);
        queryBuilder.must(goodsCodeTermsQuery);
        return queryBuilder;
    }
```

