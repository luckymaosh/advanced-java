## 深度分页机
今天在使用ElacticSearch做分页查询的时候，遇到一个奇怪的问题，分页获取前9999条数据的时候都是正常的，但每次获取第10000条数据的时候就无法获取到结果。检查自己代码中的分页逻辑也未发现什么问题，于是进行单步调试，当单步获取第10000条数据的时候捕捉到了下面的异常：


```
Failed to execute phase [query_fetch], all shards failed; shardFailures {[1w_m0BF0Sbir4I0hRWAmDA][fuxi_user_feature-2018.01.09][0]: RemoteTransportException[[10.1.113.169][10.1.113.169:9300][indices:data/read/search[phase/query+fetch]]]; nested: QueryPhaseExecutionException[Result window is too large, from + size must be less than or equal to: [10000] but was [10100]. See the scroll api for a more efficient way to request large data sets. This limit can be set by changing the [index.max_result_window] index level parameter.]; }

```

## 解决办法
最后通过查阅了解到出现这个问题是由于ElasticSearch的默认 深度翻页 机制的限制造成的。ES默认的分页机制一个不足的地方是，比如有5010条数据，当你仅想取第5000到5010条数据的时候，ES也会将前5000条数据加载到内存当中，所以ES为了避免用户的过大分页请求造成ES服务所在机器内存溢出，默认对深度分页的条数进行了限制，默认的最大条数是10000条，这是正是问题描述中当获取第10000条数据的时候报Result window is too large异常的原因。

要解决这个问题，可以使用下面的方式来改变ES默认深度分页的index.max_result_window 最大窗口值,调整这个参数的大小


```
curl -XPUT http://127.0.0.1:9200/my_index/_settings -d '{ "index" : { "max_result_window" : 500000}}'
```

虽然这样可以暂时解决这个问题，但是这样子做带来的坏处是，消耗更多的内存和cpu，如果数据更多，那是不是再把max_result_window调大呢？ 资源永远是有限的，事物变化可能是无限的。那有没有更好的解决方案呢？ 答案是有的，ES早已经想到这个问题，所以已经为我们准备了scroll api（游标）。

## scroll 滚动API

scroll api是一个游标，查询返回这个游标，客户端再遍历这个游标即可。
为了满足深度分页的场景，es 提供了 scroll 的方式进行分页读取。原理上是对某次查询生成一个游标 scroll_id ， 后续的查询只需要根据这个游标去取数据，直到结果集中返回的 hits 字段为空，就表示遍历结束。scroll_id 的生成可以理解为建立了一个临时的历史快照，在此之后的增删改查等操作不会影响到这个快照的结果。


- 查询步骤1，获取scrollId，查询参数包括：index、type、scroll，scroll 字段指定了scroll_id 的有效生存期，以分钟为单位，过期之后会被es 自动清理

```
[root@dnsserver ~]# curl -XGET 127.0.0.1:9200/my_index/type/_search?pretty&scroll=2m -d 
'{"query":{"match_all":{}}, "sort": ["_doc"]}'

# 返回结果
{
  "_scroll_id": "cXVlcnlBbmRGZXRjaDsxOzg3OTA4NDpTQzRmWWkwQ1Q1bUlwMjc0WmdIX2ZnOzA7",
  "took": 1,
  "timed_out": false,
  "_shards": {
  "total": 1,
  "successful": 1,
  "failed": 0
  },
  "hits":{...}
}
```

- 第二步，根据scrollID遍历数据。此时查询不需要index和type参数


```
[root@dnsserver ~]# curl -XGET '127.0.0.1:9200/_search/scroll?scroll=1m&scroll_id=cXVlcnlBbmRGZXRjaDsxOzg4NDg2OTpTQzRmWWkwQ1Q1bUlwMjc0WmdIX2ZnOzA7'

#返回结果
{
    "_scroll_id": "cXVlcnlBbmRGZXRjaDsxOzk1ODg3NDpTQzRmWWkwQ1Q1bUlwMjc0WmdIX2ZnOzA7",
    "took": 106,
    "_shards": {
        "total": 1,
        "successful": 1,
        "failed": 0
    },
    "hits": {
        "total": 22424,
        "max_score": 1.0,
        "hits": [{
                "_index": "product",
                "_type": "info",
                "_id": "did-519392_pdid-2010",
                "_score": 1.0,
                "_routing": "519392",
                "_source": {
                    ....
                }
            }
        ]
    }
}
```
## 后记

但是ES作为一个搜索引擎，更适合的搜素场景，而不是大范围的遍历展示。大部分情况，没有必要范围超过10000的数据。如果的确需要大量数据的遍历展示，考虑是否可以用其他更合适的存储。或者根据业务场景看能否用ElasticSearch的 滚动API (类似于迭代器，但有时间窗口概念)来替代。