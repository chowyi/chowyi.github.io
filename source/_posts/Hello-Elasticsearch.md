---
title: Hello Elasticsearch
permalink: hello-elasticsearch
date: 2017-4-28 15:48:59
tags:
- Elasticsearch
categories:
- DevOps
---


ELK在运维开发领域提及的频率很高，在我的工作中也有少量涉及。带着好奇与兴趣，找点时间，先来学习一下ELK中的E——Elasticsearch。
<!--more-->

## What is Elasticsearch?
看看 [Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/master/getting-started.html)的介绍:
> Elasticsearch is a real-time distributed search and analytics engine. It allows you to explore your data at a speed and at a scale never before possible. It is used for full-text search, structured search, analytics, and all three in combination

关键词：实时，分布式，全文检索，结构化搜索，分析

> Elasticsearch is your new best friend.
既然如此，赶快学起来！

## 安装

1. 因为Elasticsearch依赖Java环境，所以先安装JDK。
    下载JDK解压配置环境变量即可。
2. 安装Elasticsearch
    在[Elasticsearch官网](https://www.elastic.co/downloads/elasticsearch)下载解压即可。
    我安装的是Elasticsearch-5.3.0版本，以下所有步骤皆在此版本下完成。
3. 启动
    执行`bin/elasticsearch`即可。
    可能遇到的报错：
    ```
    Java HotSpot(TM) 64-Bit Server VM warning: INFO: os::commit_memory(0x000000008a660000, 1973026816, 0) failed; error='Cannot allocate memory' (errno=12)
    #
    # There is insufficient memory for the Java Runtime Environment to continue.
    # Native memory allocation (mmap) failed to map 1973026816 bytes for committing reserved memory.
    # An error report file with more information is saved as:
    # /home/vagrant/download/elasticsearch-5.3.0/hs_err_pid1573.log
    ```
    解决方案：
    修改配置文件`config/jvm.options`：
    ```
    # Xms represents the initial size of total heap space
    # Xmx represents the maximum size of total heap space

    -Xms2g
    -Xmx2g
    ```

    把这两行的配置值改小即可：
    ```
    # Xms represents the initial size of total heap space
    # Xmx represents the maximum size of total heap space

    -Xms256M
    -Xmx256M
    ```
4. 完成
    Elasticsearch默认启动在`9200`端口，执行`curl http://localhost:9200/`测试，可以才看到相关信息：
    ```
    {
        "name" : "DZkUNFW",
        "cluster_name" : "elasticsearch",
        "cluster_uuid" : "YEdrTublSi2T-JDns8D5pg",
        "version" : {
            "number" : "5.3.0",
            "build_hash" : "3adb13b",
            "build_date" : "2017-03-23T03:31:50.652Z",
            "build_snapshot" : false,
            "lucene_version" : "6.4.1"
        },
        "tagline" : "You Know, for Search"
    }
    ```
    要修改启动端口，可以在`config/elasticsearch.yml`总找到如下配置项修改：
    ```
    # Set a custom port for HTTP:
    #
    #http.port: 9200
    ```

## 几个重要概念

- 索引（index）[名词]
    > As explained previously, an index is like a database in a traditional relational database. It is the place to store related documents. The plural of index is indices or indexes. 
    索引[名词]类似于关系型数据库中的一个数据库，是存储相互关联的文档的地方。
- 索引（index）[动词]
    > **To index a document** is to store a document in an **index (noun)** so that it can be retrieved and queried. It is much like the `INSERT` keyword in SQL except that, if the document already exists, the new document would replace the old. 
    **索引一个文档**就是把一个文档存储在一个**索引[名词]**中b一遍检索和查询。除了文档已存在时是更新，它很像SQL中的`INSERT`语句。
- 类型（type）
    暂时没有找到官方的定义。个人理解，类型和关系型数据库中的表（table）很像，用来存储一类文档。
- 文档（document）
    暂时没有找到官方的定义。个人理解，文档和关系型数据库中的一条记录类似，不同的是文档没有严格的列定义，而是采用json格式储存。

## 试一试

我跟着官网上[Elasticsearch: The Definitive Guide [2.x]](https://www.elastic.co/guide/en/elasticsearch/guide/current/_indexing_employee_documents.html)中的示例练习了一遍，使用curl调用RESTful接口。现在我再用Python客户端`pyelasticsearch`试试。

### 安装`pyelasticsearch`
```
pip install pyelasticsearch
```

### 初始化pyelasticsearch实例
```
from pyelasticsearch import ElasticSearch

es = ElasticSearch('http://localhost:9200')
```
更多参数和详细介绍可以看[pyelasticsearch的文档](http://pyelasticsearch.readthedocs.io/en/latest/api/)。

### Example 1：[Indexing Employee Documents](https://www.elastic.co/guide/en/elasticsearch/guide/current/_indexing_employee_documents.html)
任务：索引一条雇员信息到文档中，每个文档都是`employee`类型,该类型位于索引`megacorp`中。
使用`curl`这么做：
```
curl -XPUT 'localhost:9200/megacorp/employee/1?pretty' -H 'Content-Type: application/json' -d'
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
'
```
使用`pyelasticsearch`：
```
employee = {
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}

es.index('megacorp', 'employee', employee, 1)
```

再加入两条文档，可以使用同上面一样的方法，也可以试试`pyelasticsearch`中的批量操作：
```
es.bulk(
    [
        es.index_op(
            {
                "first_name" :  "Jane",
                "last_name" :   "Smith",
                "age" :         32,
                "about" :       "I like to collect rock albums",
                "interests":  [ "music" ]
            },
            id=2
        ),
        es.index_op(
            {
                "first_name" :  "Douglas",
                "last_name" :   "Fir",
                "age" :         35,
                "about":        "I like to build cabinets",
                "interests":  [ "forestry" ]
            },
            id=3
        )
    ],
    doc_type='employee',
    index='megacorp'
)
```

### Example 2：[Retrieving a Document](https://www.elastic.co/guide/en/elasticsearch/guide/current/_retrieving_a_document.html)
任务：检索单个雇员的数据。
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/1?pretty'
```
使用`pyelasticsearch`：
```
es.get('megacorp', 'employee', 1)
```

### Example 3：[Search Lite](https://www.elastic.co/guide/en/elasticsearch/guide/current/_search_lite.html)
任务：搜索所有雇员
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?pretty'
```
使用`pyelasticsearch`：
```
es.search({}, index='megacorp', doc_type='employee')
```

任务：搜索姓氏为Smith的雇员
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?q=last_name:Smith&pretty'
```
使用`pyelasticsearch`：
```
es.search('last_name:Smith', index='megacorp', doc_type='employee')
```

### Example 4：[Search with Query DSL](https://www.elastic.co/guide/en/elasticsearch/guide/current/_search_with_query_dsl.html#_search_with_query_dsl)
任务：使用查询表达式搜索姓氏为Smith的雇员
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
'
```
使用`pyelasticsearch`：
```
es.search({
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}, index='megacorp', doc_type='employee')
```

### Example 5：[More-Complicated Searches](https://www.elastic.co/guide/en/elasticsearch/guide/current/_more_complicated_searches.html#_more_complicated_searches)
任务：搜索年龄大于30岁并且姓氏为Smith的雇员
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
'
```
使用`pyelasticsearch`：
```
es.search({
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}, index='megacorp', doc_type='employee')
```

### Example 6：[Full-Text Search](https://www.elastic.co/guide/en/elasticsearch/guide/current/_full_text_search.html#_full_text_search)
任务：搜索所有喜欢攀岩（rock climbing）的雇员
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
'
```
使用`pyelasticsearch`：
```
es.search({
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}, index='megacorp', doc_type='employee')
```
这里的查询结果很有意思：
```
{
    "took": 5,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 0.53484553,
        "hits": [
            {
                "_index": "megacorp",
                "_type": "employee",
                "_id": "1",
                "_score": 0.53484553,
                "_source": {
                    "interests": [
                        "sports",
                        "music"
                    ],
                    "about": "I love to go rock climbing",
                    "first_name": "John",
                    "last_name": "Smith",
                    "age": 25
                }
            },
            {
                "_index": "megacorp",
                "_type": "employee",
                "_id": "2",
                "_score": 0.26742277,
                "_source": {
                    "interests": [
                        "music"
                    ],
                    "about": "I like to collect rock albums",
                    "first_name": "Jane",
                    "last_name": "Smith",
                    "age": 32
                }
            }
        ]
    }
}
```
返回的结果中包含两项文档, 因为 Jane Smith 的 about 中包含了 rock，但因为没有 climbing，所以相关性得分低于 John Smith。

> This is a good example of how Elasticsearch can search within full-text fields and return the most relevant results first. This concept of relevance is important to Elasticsearch, and is a concept that is completely foreign to traditional relational databases, in which a record either matches or it doesn’t.

*Elasticsearch: The Definitive Guide* 中指出，在Elasticsearch中相关性是一个重要的概念，特别是区别于传统的关系型数据库中要么匹配要么不匹配的概念。

### Example 6：[Phrase Search](https://www.elastic.co/guide/en/elasticsearch/guide/current/_phrase_search.html#_phrase_search)
任务：对上面的搜索改进，精确的匹配短语`rock climbing`
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
'
```
使用`pyelasticsearch`：
```
es.search({
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}, index='megacorp', doc_type='employee')
```

### Example 7：[Highlighting Our Searches](https://www.elastic.co/guide/en/elasticsearch/guide/current/highlighting-intro.html#highlighting-intro)
任务：高亮搜索结果
只要再加一个参数就可以了。
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?pretty' -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
'
```
使用`pyelasticsearch`：
```
es.search({
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}, index='megacorp', doc_type='employee')
```

### Example 8：[Analytics](https://www.elastic.co/guide/en/elasticsearch/guide/current/_analytics.html#_analytics)
任务：统计雇员的喜好
这里要用到Elasticsearch中的聚合(aggregations)功能。
使用`curl`这么做：
```
curl -XGET 'localhost:9200/megacorp/employee/_search?pretty' -H 'Content-Type: application/json' -d'
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests.keyword" }
    }
  }
}
'
```
使用`pyelasticsearch`：
```
es.search({
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests.keyword" }
    }
  }
}, index='megacorp', doc_type='employee')
```
查询结果如下：
```
{
  "took" : 17,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {...},
  "aggregations" : {
    "all_interests" : {
      "doc_count_error_upper_bound" : 0,
      "sum_other_doc_count" : 0,
      "buckets" : [
        {
          "key" : "music",
          "doc_count" : 2
        },
        {
          "key" : "forestry",
          "doc_count" : 1
        },
        {
          "key" : "sports",
          "doc_count" : 1
        }
      ]
    }
  }
}
```
**注意：** 在这一步中[Elasticsearch: The Definitive Guide [2.x]](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)中给出的查询表达式为：
```
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
```
运行结果报错：
```
Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index. Note that this can however use significant memory.
```
可能与我使用Elasticsearch版本为5.3.0有关，根据错误信息查询到的[资料](https://www.elastic.co/guide/en/elasticsearch/reference/5.1/fielddata.html)，具体原因待学习后补充。

TODO：学习Fielddata相关知识。

## 总结
整个学习过程我先了解到了Elasticsearch是一个实时的分布式的全文搜索引擎，它具有高性能，易扩展，易使用等特点。它可以以文档的形式存储数据，并且可以索引，搜索，分析数据。
接下来我尝试在自己的Linux虚拟机上安装了Elasticsearch。只要有Java环境，下载解压即可运行。
最后，我通过调用RESTful接口和使用`pyelasticsearch`客户端两种方式完成了[Elasticsearch: The Definitive Guide [2.x]](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)的入门练习。学习并实际使用到了索引（index）[名词]，类型（type），文档（document），索引（index）[动词]几个概念。
目前为止我对Elasticsearch有了一个简单直观的认识，学习新的知识，很开心！

