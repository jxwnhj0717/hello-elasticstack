# Elastic Stack



## 简介

Elastic Stack可用于集群、多应用的日志分析。

Elastic Stack包含Elasticsearch、Kibana、Logstash、Beats。

Elasticsearch负责存储、索引、查询、分析文档。一条日志可以看成一个文档。

Kibana是Elasticsearch的客户端，可以查询文档，将文档的分析结果可视化。

Logstash可以将一个数据源传输到另一个数据源，传输的过程中，可以对数据清洗、加工和整理。通过配置输入插件，可以支持不同的数据源，比如Beats、Kafka。通过配置过滤器，可以支持不同过滤形式，比如解析文本、增加字段。通过配置输出插件，可以支持不同的目标数据源，比如Elasticsearch、MongoDB。

Beats负责采集数据，采集日志的组件是FileBeats。

日志通过FileBeats采集，传输给Logstash，在生产环境中，Logstash会把消息中间件配置成事件队列，达到削峰填谷的目的。Logstash对日志加工后，交给Elasticsearch生成索引。通过Kibana可以查询和分析文档。



## Elasticsearch

### 概念

+ **全文检索**

数据检索：从一系列数据中，根据一个或多个数据特征找到特定的数据。

数据分类：结构化数据，比如Mysql的表，半结构化数据，比如JSON、XML，非结构化数据，比如文章、邮件、图片、视频。

全文检索：结构化的数据可以用关系数据库检索，而半结构化、非结构化数据的检索称为全文检索。


+ **倒排索引**

forward index：关系数据库的索引一般是主键，也是记录id，记录id指向记录。

inverted index：全文检索是从文档内容中提取关键字，对关键字建立索引，索引指向文档id。

[Elasticsearch权威指南-倒排索引](https://www.elastic.co/guide/cn/elasticsearch/guide/current/inverted-index.html)


+ **映射**

文档插入索引，需要由映射关系来定义。

一个索引可以包含多个映射类型，但是Elasticsearch7.0开始，一个索引只包含一个映射类型（_doc），也就是只包含一种文档。

[Elasticsearch权威指南-多个映射类型的问题](https://www.elastic.co/guide/cn/elasticsearch/guide/current/mapping.html#_%E9%81%BF%E5%85%8D%E7%B1%BB%E5%9E%8B%E9%99%B7%E9%98%B1)



### 查询

查询分成Query和Filter，Query查询出的文档会按相关度排序，Filter查询出的文档相关度都相同。

Elasticsearch中的数据类型和其他数据库差不多，但是字符串类型比较特别。字符串类型包括text和keyword两种类型，text类型会经过分词，分成多个词项，然后编入索引，keyword类型是整个文本编入索引，不分词。text类型用于词项搜索，keyword类型用于聚合统计。比如"我爱中国"定义成text类型，经过分词可以分成3个词项“我”、“爱”、”中国“，当搜索”我“这个词项，可以查询到”我爱中国“这个字符串。如果"我爱中国"定义成keyword类型，只有搜索"我爱中国"时，才能查到”我爱中国“这个字符串。但是定义成text类型的"我爱中国"并没有在索引中保存整个文本，索引不能用于聚合统计，而keyword类型的"我爱中国"则可以。如果一个字段既希望能基于词项搜索，有希望基于全文统计，那就需要定义多类型，自身定义成一种类型，通过子字段定义成其他类型。

+ **精确查询**

使用关键字term，必须完全匹配文本。

```
GET /my_index/_search
{
    "query" : {
    	"term" : { 
    		"title" : "我爱中国"
    	}
    }
}
```

[Elasticsearch权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_finding_exact_values.html)

+ **匹配查询**

使用关键字match，查询串会先经过分词，匹配索引中的分词文本。

```
GET /my_index/_search
{
    "query": {
        "match": {
            "title": "我"
        }
    }
}
```

[Elasticsearch权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/match-query.html)

+ **组合查询**

使用关键字bool，组合多个条件，支持与、或、非。

```
GET /my_index/_search
{
   "query" : {
       "bool" : {
           "must" : [
               { "term" : {"price" : 20}}, 
               { "term" : {"title" : "我爱中国"}} 
           ]
       }
   }
}
```

[Elasticsearch权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/combining-filters.html)



### 聚合

常用的聚合是桶型聚合和指标聚合。

对于一条SQL：select avg(age) from student group by grade，group by grade对应Elasticsearch桶型聚合，avg(age)对应Elasticsearch指标聚合。采用Elasticsearch的语法：

```
GET /student/_search
{
   "aggs": {
      "grades": {
         "terms": {
            "field": "grade"
         }
      },
      "aggs": {
         "avg_age": { 
           "avg": { "field": "price" }
         }
      }
   }
}
```

这种语法结构和SQL不同的是，可以嵌套多层聚合。桶型聚合下面嵌套指标聚合，指标聚合下面又可以嵌套桶型聚合。



## Kibana

### 功能：Management-Dev Tools

开发工具界面可用于执行elasticsearch的操作，比如在增删改查文档。

[Elasticsearch7.0增删改查文档](https://learnku.com/docs/elasticsearch73/7.3/index-some-documents/6450)

[Elasticsearch2.x增删改查文档](https://www.elastic.co/guide/cn/elasticsearch/guide/current/create-doc.html)

官方的《Elasticsearch权威指南》基于Elasticsearch2.x编写，需要注意的是7.0版本中不推荐一个索引下定义多个mapping，mapping统一为_doc。



### 功能：Analytics-Discover

文档发现功能可以用来搜索“索引模式”中的文档。

首先需要定义索引模式。索引模式是一组相关的索引，比如日志通常会按天建立索引，当我们要搜索这种日志时，就需要包含每一天的索引，通常是定义一个带通配符*的索引模式，匹配所有日期的索引。索引模式的定义在Management - Stack Management / Index patterns。

然后进入搜索界面Analytics-Discover。选择一个索引模式，输入查询条件。如果查询条件比较简单，使用KQL，如果查询条件比较复杂，可以切换成Lucene，使用Elasticsearch的搜索语法。当然在搜索之前，通常需要通过logstash对日志做预处理，解析成多字段。



### 功能：Analytics-Dashboard

一个Dashboard由多个控件和图表构成，控件控制查询条件，图表展示聚合结果。

图表在Analytics-Visualize Library中定义，图表的形式可以分为表格、二维坐标图、圆形和弧形和热度图。



## FileBeat

### 配置功能

1. 指定采集日志的目录
2. 为日志增加标识信息
3. 输出到logstash



### 配置示例

+ filebeat.yml

```
# ============================== Filebeat inputs ===============================

filebeat.inputs:
- type: log
  enabled: true
  # 读取日志的目录  
  paths:
    - D:myapp/data/logs/logger*.log
  # 日志标识  
  tags: ["myapp"]
  # 读取日志编码  
  encoding: 'GBK'  
  
  # 异常日志需要合并多行日志
  # multiline: 
  #  pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  #  negate: true
  #  match: after
    
# ============================== Filebeat modules ==============================

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false  
  
# ---------------------------- Logstash Output ----------------------------
output.logstash:
  hosts: ["localhost:5044"]  
```



### 重新收集日志

1. 在Kibana中删除生成的索引，Management - Stack Management / Index Management。
2. 清空FileBeats日志收集记录，目录为${FileBeats_HOME}/data。
3. 重新运行FileBeats。



## Logstash

### 配置功能

1. 通过json解析日志
2. 输出到elasticsearch
3. 定义索引的生命周期管理，索引滚动时的别名

### 配置示例

+ config/logstash.conf

```
input {
  beats {
    port => 5044
  }
}

# 日志的文本记录在message字段，解析后的Json存储在doc字段，删除源字段message
filter {
  json {
    source => message
    target => doc
    remove_field => ["message"]
  }
}

output {
  if "myapp" in [tags] {
    elasticsearch {
      hosts => ["http://localhost:9200"]
      ilm_enabled => "true"
      ilm_policy => "myapp_policy"
      ilm_rollover_alias => "myapp"
    }
  }
}
```

+ kibana配置索引生命周期策略  

在Management - Stack Management / Index Lifecycle Management中配置索引生命周期策略，关闭Hot phase的Rollover，增加Delete phase，7天后删除索引。[配置参考资料](https://stackoverflow.com/questions/59859306/elasticsearch-ilm-not-deleting-indices)



## 参考资料

《Elasticsearch权威指南》

《Elastic Stack应用宝典》