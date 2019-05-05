---
layout: post
title: "Elasticsearch-Filebeat-Kibana-Analysis"
subtitle: '组内技术分享'
author: "Elvis"
header-style: text
tags:
  - Elasticsearch
  - Filebeat
  - Kibana
  - 组内技术分享
---

### Elasticsearch Pipeline 预处理

Pipeline和java中的Stream进行类比,两者从功能和概念上很类似,我们经常会对Stream中的数据进行处理,比如map操作,peek操作,reduce操作,count操作等,这些操作从行为上说,就是对数据的加工,而Pipeline也是如此,Pipeline也会对通过该Pipeline的数据(一般来说是文档)进行加工,比如,修改文档的某个字段值,修改文档某个字段的类型等等.而Elasticsearch对该加工行为进行抽象包装,并称之为Processors.Elasticsearch命名了多种类型的Processors来规范对文档的操作,比如set,append,date,join,json,kv等等.不同类型的Processors。

**Pipeline定义**

<pre name="code" class="c++"> 
PUT _ingest/pipeline/helloworld
{
  "description" : "describe pipeline set hello to world",
  "processors" : [
    {
      "set" : {
        "field": "hello",
        "value": "world"
      }
    }
  ]
}
</pre>

上面的例子,表明通过指定的URL请求"_ingest/pipeline"定义了一个ID为"my-pipeline-id"的pipeline,其中请求体中的存在两个必须要的元素:

* description 描述该pipeline是做什么的
* processors 定义了一系列的processors,这里只是简单的定义了一个赋值操作,即将字段名为"hello"的字段值都设置为"world"

**Grok Processor**

<pre name="code" class="c++"> 
  {  
        "grok": {  
          "field": "forwardedFor",  
          "patterns": [  
            "%{TOKEN:directRemoteAddress}"  
          ],  
          "pattern_definitions": {  
            "TOKEN": "[.0-9]+$"  
          },  
          "ignore_failure": true  
        }  
  }
</pre>

上面的例子，field的值是要处理的字段名称。patterns是匹配的表达式，返回的是第一个匹配的值。以上两个字段都是Grok Processor必须的。
pattern_definitions是匹配的value重新定义的值，ignore_failure此字段若为true，若field为空或者不存在则会不修改退出此document。

**Gsub Processor**

<pre name="code" class="c++"> 
  {
        "gsub": {
          "field": "forwardedFor",
          "pattern": ",",
          "replacement": ", "
        }
  }
</pre>

Gsub Processor能够解决一些字符串中才特有的问题,比如我想把字符串格式的日期格式如"yyyy-MM-dd HH:mm:ss"转换成"yyyy/MM/dd HH:mm:ss"
的格式,我们可以借助于Gsub Processor来解决,而Gsub Processor也正是利用正则来完成这一任务的。

**Date Index Name Processor**

<pre name="code" class="c++"> 
 {
        "date_index_name": {
          "field": "@timestamp",
          "index_name_prefix": "splog-access-authgate-",
          "index_name_format": "yyyy.MM.dd",
          "date_rounding": "d",
          "timezone": "Asia/Shanghai",
          "ignore_failure": true
        }
  }
</pre>

Date Index Name Processor是日期索引处理器。我们写入的文档可以根据其中的某个日期格式的字段来指定该文档将写入哪个索引中,该功能配上template,
能够实现很强大的日志收集功能,比如上面的例子就是按天来将日志写入Elasticsearch。

**Date Processor**

<pre name="code" class="c++"> 
 {
        "date": {
          "field": "datestamp",
          "target_field": "@timestamp",
          "formats": [
            "yyyy-MM-dd HH:mm:ss.SSS",
            "ISO8601"
          ],
          "timezone": "Asia/Shanghai",
          "ignore_failure": true
        }
  }
</pre>

就是将原有的字符串表示的日期格式进行格式转换(最终转换成的是ISO时间格式)。

**Remove Processor**

<pre name="code" class="c++"> 
 {
        "remove": {
          "field": "message"
        }
 }
</pre>

在经过Remove Processor处理后,message字段将不存在了。

以上只是常用的Processors，若要查看更多，可以参考[ProcessorsAPI](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest-processors.html)



### Elasticsearch 查询示例

**多字段匹配查询**

<pre name="code" class="json"> 
{
    "query": {
        "multi_match" : {
            "query" : "127.0.0.1:8080",
            "fields" : ["orgHost","host"]
        }
    }
}
</pre>
注：一次查找中查询多个字段。


**子句灵活匹配查询**
<pre name="code" class="json"> 
{
  "bool": {
    "should": [
      { "term": { "orgHost": "127.0.0.1:8080"}},
      { "term": { "orgHost": "172.16.1.220:8080"}},
      { "term": { "orgHost": "192.168.1.1:8080"}}
    ],
    "minimum_should_match": 2
  }
}
</pre>

注：minimum_should_match 还可以使用百分比 如："minimum_should_match": "75%"，但值 75% 会被截断成 2 。即三条 should 语句中至少有两条必须匹配。


**布尔查询**
<pre name="code" class="json"> 
{
    "query": {
        "bool": {
            "must": {
                "bool" : { "should": [
                      { "match": { "orgHost": "127.0.0.1:8080" }},
                      { "match": { "orgHost": "172.16.1.220:8080" }}
                ]}
            },
            "must": { "match": { "statusCode": "200" }},
            "must_not": { "match": {"statusCode": "500" }}
        }
    }
}
</pre>
注：可以创建任意复杂的或者深度嵌套的查询。



**字段名模糊匹配查询**

<pre name="code" class="json"> 
{
    "query": {
        "multi_match" : {
            "query" : "127.0.0.1:8080",
            "fields" : ["*rgHost"]
        }
    }
}
</pre>
注：模糊给出字段名，多匹配字段查找。

**通配符查询**

<pre name="code" class="json"> 
{
    "query": {
        "wildcard" : {
           "orgPathName":"/ebus/*"
        }
    }
}
</pre>
注：通配符查询允许你指定匹配的模式，？ 匹配任何字符，*匹配零个或多个字符


**正则查询**

<pre name="code" class="json"> 
{
    "query": {
        "regexp" : {
           "email":"^[\w.\-]+@(?:[a-z0-9]+(?:-[a-z0-9]+)*\.)+[a-z]{2,3}$"
        }
    }
}
</pre>

**范围查询**

<pre name="code" class="json"> 
{
  "query": {
    "range": {
      "second": {
        "gte": "2019-05-01 19:53:41",
        "lte": "2019-05-01 19:54:41"
      }
    }
  }
}
</pre>


**过滤查询**

<pre name="code" class="json"> 
{
  "query": {
    "filtered": {
        "filter": {
            "must": { "match": { "statusCode": "200" }},
            "must_not": { "match": {"statusCode": "500" }}
        }
    }
  }
}
</pre>

以上只是常用的Elasticsearch查询，若要查看更多，可以参考[Elasticsearch深入搜素](https://www.elastic.co/guide/cn/elasticsearch/guide/current/search-in-depth.html)