---
layout: post
title: Hive数据仓库(10)之Hive与Elasticsearch的结合
date: 2016-04-07 13:30:00
categories: 大数据 搜索引擎
tags: Hive Elasticsearch
---

## 1. 简介

Elasticsearch跟Solr并列为开源界最流行的两大搜索引擎, 相比solr, Elasticsearch又如其名一样以其灵活性和可扩展性闻名与世。Hive 与 Elasticsearch的结合就是一个鲜明的例子, 通过Hive与Elasticsearch结合, 我们不仅可以通过MapReduce来快速的为Elasticsearch的批量进行索引, 也可以通过Hive SQL以及直接MapReduce来进行更复杂的批量数据处理, 从而大大扩展了Elasticsearch。

本文主要介绍如何配置Hive和Elasticsearch的结合, 并介绍了我配置过程中遇到的坑.

## 2. 配置与安装

现说明下我使用的环境:

* Hive 1.2.1
* Elasticsearch 2.3.0
* Elasticsearch-Hadoop 2.2.0

### 2.1 依赖

首先当然得去下载[Elasticsearch-Hadoop](<http://download.elastic.co/hadoop/elasticsearch-hadoop-2.2.0.zip>), 下载后进行解压, 由于本文只需要用到Hive, 其他的Hadoop,Pig,Spark等不在考虑范围内, 所以只需要提取```elasticsearch-hadoop-hive-2.2.0.jar```。

如果你嫌麻烦你可以把jar包扔到$HIVE_HOME/lib下, 我把它放在$HIVE_HOME/auxpath(你可以随便放)。

启动hive时候把它加进去就行了。

```shell
add jar $HIVE_HOME/auxpath/elasticsearch-hadoop-hive-2.2.0.jar
```

### 2.2 Read配置

既然进行Read配置, 那么Elasticsearch的索引上当然得有数据了。

在本文中,
index 为 xml_stat,
type 为 cashflow

```sql
CREATE EXTERNAL TABLE `elasticsearch`.`cashflow_read`(
  `id` bigint,
  `deal_id` bigint,
  `user_id` bigint,
  `flow_type` string,
  `amount` bigint,
  `create_time` string,
  `status` bigint ,
  `status_time` string,
  `ref_id` bigint,
  `channel_type` bigint,
  `operation_type` bigint,
  `sub_operation_type` bigint)
STORED BY
  'org.elasticsearch.hadoop.hive.EsStorageHandler'
TBLPROPERTIES (
  'es.mapping.names'='id:_metadata._id',
  'es.nodes'='data4:9200',
  'es.resource'='xml_stat/cashflow',
  'es.mapping.date.rich' = 'false',
  'es.read.metadata' = 'true');
```

>注意：因为在ES中，xml_stat/cashflow的_id为id，要想把_id映射到Hive表字段中，必须使用这种方式：
> 'es.read.metadata' = 'true',
> 'es.mapping.names' = 'id:_metadata._id'

当启动MapReduce时候map个数就是分片个数。

可以通过在Hive外部表中指定search条件，只查询过滤后的数据。比如，下面的建表语句会从ES中搜索_id=98E5D2DE059F1D563D8565的记录：

只需要在TABLPROPERTIES中加入'es.query'='?q=_id:98E5D2DE059F1D563D8565'

如果数据量不大，可以使用Hive的Local模式来执行，这样不必提交到Hadoop集群：

```shell
set hive.exec.mode.local.auto.inputbytes.max=134217728;
set hive.exec.mode.local.auto.tasks.max=10;
set hive.exec.mode.local.auto=true;
```

### 2.3 Writer配置

相比于Read, Writer 略有不同.

```sql
CREATE EXTERNAL TABLE `elasticsearch`.`cashflow`(
  `id` bigint,
  `deal_id` bigint,
  `user_id` bigint,
  `flow_type` string,
  `amount` bigint,
  `create_time` string,
  `status` bigint ,
  `status_time` string,
  `ref_id` bigint,
  `channel_type` bigint,
  `operation_type` bigint,
  `sub_operation_type` bigint)
STORED BY
  'org.elasticsearch.hadoop.hive.EsStorageHandler'
TBLPROPERTIES (
  'es.index.auto.create'='true',
  'es.mapping.id'='id',
  'es.mapping.names'='deal_id:deal_id,user_id:user_id,flow_type:flow_type,amount:amount,create_time:create_time,status:status,status_time:status_time,ref_id:ref_id,channel_type:channel_type,operation_type:operation_type,sub_operation_type:sub_operation_type',
  'es.nodes'='data4:9200',
  'es.nodes.discovery'='false',
  'es.resource'='xml_stat/cashflow',
  'es.mapping.date.rich' = 'false');
```

> 这里要注意下：如果是往_id中插入数据，需要设置'es.mapping.id'='id'参数，表示Hive中的id字段对应到ES中的_id，而es.mapping.names中不需要再映射，这点和读取时候的配置不一样。如此配置完后查询也是可以的。

创建好外部表后, 我们从hive的另外表导入到该表中, 也就是导入到es中。

```sql
insert overwrite table elasticsearch.cashflow
select
`id`,
`deal_id`,
`user_id`,
`flow_type`,
`amount`,
from_unixtime(unix_timestamp(`create_time`,"yyyy-MM-dd HH:mm:ss"),"yyyy-MM-dd HH:mm:ss") as create_time,
`status`,
from_unixtime(unix_timestamp(`status_time`,"yyyy-MM-dd HH:mm:ss"),"yyyy-MM-dd HH:mm:ss") as status_time,
`ref_id`,
`channel_type`,
`operation_type`,
`sub_operation_type`
from xml_stat.cashflow;
```

当删除elasticsearch.cashflow时候ES的xml_stat/cashflow也会删除, 这点与mongo-hadoop类似。

当进行写入时候注意控制Map个数, 否则很容易出现问题。

关于配置项详见[Elasticsearch文档](<https://www.elastic.co/guide/en/elasticsearch/hadoop/current/configuration.html>)

更多的Hive的操作也可以查看[ES-Hive文档](<https://www.elastic.co/guide/en/elasticsearch/hadoop/current/hive.html>)

## 3. 踩的坑

在使用过程中踩了几个坑, 主要分为安装和类型转换

### 3.1 host

我使用的服务器有两个网卡, 内网ip_a和外网ip_2.

为了外网能访问, 我在Elasticsearch中配置network.host: 0:0:0:0, 随后设置es.nodes＝ip_a:9200, 这样环境下在运行hive时候出现如下错误

```java
2932 2016-04-06 16:48:56,667 FATAL [main] ExecReducer: org.apache.hadoop.hive.ql.metadata.HiveException: Hive Runtime Error while processing row (tag=0) {"key":{},"value":{"_col0":412302,"_col1":136515366968180733,"_col2":6158970,"_col3":     "ALIPAY","_col4":-1930,"_col5":"2015-05-01 00:14:26.0","_col6":1,"_col7":"2015-05-01 00:14:26.0","_col8":86944,"_col9":2,"_col10":1,"_col11":1}}
2933         at org.apache.hadoop.hive.ql.exec.mr.ExecReducer.reduce(ExecReducer.java:244)
2934         at org.apache.hadoop.mapred.ReduceTask.runOldReducer(ReduceTask.java:444)
2935         at org.apache.hadoop.mapred.ReduceTask.run(ReduceTask.java:392)
2936         at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:168)
2937         at java.security.AccessController.doPrivileged(Native Method)
2938         at javax.security.auth.Subject.doAs(Subject.java:422)
2939         at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1614)
2940         at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:163)
2941 Caused by: org.elasticsearch.hadoop.rest.EsHadoopNoNodesLeftException: Connection error (check network and/or proxy settings)- all nodes failed; tried [[ip_b:9200]]
2942         at org.elasticsearch.hadoop.rest.NetworkClient.execute(NetworkClient.java:142)
2943         at org.elasticsearch.hadoop.rest.RestClient.execute(RestClient.java:423)
2944         at org.elasticsearch.hadoop.rest.RestClient.executeNotFoundAllowed(RestClient.java:431)
2945         at org.elasticsearch.hadoop.rest.RestClient.exists(RestClient.java:507)
2946         at org.elasticsearch.hadoop.rest.RestClient.touch(RestClient.java:513)
2947         at org.elasticsearch.hadoop.rest.RestRepository.touch(RestRepository.java:491)
2948         at org.elasticsearch.hadoop.rest.RestService.initSingleIndex(RestService.java:412)
2949         at org.elasticsearch.hadoop.rest.RestService.createWriter(RestService.java:400)
2950         at org.elasticsearch.hadoop.mr.EsOutputFormat$EsRecordWriter.init(EsOutputFormat.java:173)
2951         at org.elasticsearch.hadoop.hive.EsHiveOutputFormat$EsHiveRecordWriter.write(EsHiveOutputFormat.java:58)
2952         at org.apache.hadoop.hive.ql.exec.FileSinkOperator.process(FileSinkOperator.java:753)
2953         at org.apache.hadoop.hive.ql.exec.Operator.forward(Operator.java:837)
2954         at org.apache.hadoop.hive.ql.exec.LimitOperator.process(LimitOperator.java:54)
2955         at org.apache.hadoop.hive.ql.exec.Operator.forward(Operator.java:837)
2956         at org.apache.hadoop.hive.ql.exec.SelectOperator.process(SelectOperator.java:88)
2957         at org.apache.hadoop.hive.ql.exec.mr.ExecReducer.reduce(ExecReducer.java:235)
```

由于MapReduce链接了外网ip_b导致不能链接ES, 查看了ES启动日志。

```shell
publish_address {ip_b:9300}, bound_addresses {0.0.0.0:9300}
```

所以只能把network.host＝ip_a

### 3.2 Long与Int

如果我在hive设置字段类型为int, 则在进行select时候会出现以下错误:

```shell
Failed with exception java.io.IOException:org.apache.hadoop.hive.ql.metadata.HiveException:
java.lang.ClassCastException: org.apache.hadoop.io.LongWritable cannot be cast to
org.apache.hadoop.io.IntWritable
```

这是由于hive表中field类型为int时，映射到es中变成long，所以会报此错误。将hive表中int改为bigint即可。

### 3.3 DateTime映射

比如我在hive中有create_time字段类型为string, "2015-09-05 17:48:26". 那么我要怎样才能转成ES的datetime类型呢。

即提前为索引建好mapping, 关闭es.index.auto.create。

创建cashflow.json, 由于这里只涉及到type, 所以其他的properties都没设置:

```json
{
    "mappings":{
        "cashflow" : {
            "properties" : {
                "amount" : {
                    "type" : "long"
                },
                "channel_type" : {
                    "type" : "long"
                },
                "create_time" : {
                    "type" : "date",
		    "format": "yyyy-MM-dd HH:mm:ss"
                },
                "deal_id" : {
                    "type" : "long"
                },
                "flow_type" : {
                    "type" : "string"
                },
                "id" : {
                    "type" : "long"
                },
                "operation_type" : {
                    "type" : "long"
                },
                "ref_id" : {
                    "type" : "long"
                },
                "status" : {
                    "type" : "long"
                },
                "status_time" : {
                    "type" : "date",
		    "format": "yyyy-MM-dd HH:mm:ss"
                },
                "sub_operation_type" : {
                    "type" : "long"
                },
                "user_id" : {
                    "type" : "long"
                }
            }
        }
    }
}
```

创建mapping

```shell
curl -XPUT data4:9200/xml_stat -d @../js/cashflow.js
```

如此就建好mapping了

```shell
[bmw@data1 auxpath]$ curl -XGET 'http://data4:9200/xml_stat/cashflow/_mapping?pretty'
{
  "xml_stat" : {
    "mappings" : {
      "cashflow" : {
        "properties" : {
          "amount" : {
            "type" : "long"
          },
          "channel_type" : {
            "type" : "long"
          },
          "create_time" : {
            "type" : "date",
            "format" : "yyyy-MM-dd HH:mm:ss"
          },
          "deal_id" : {
            "type" : "long"
          },
          "flow_type" : {
            "type" : "string"
          },
          "id" : {
            "type" : "long"
          },
          "operation_type" : {
            "type" : "long"
          },
          "ref_id" : {
            "type" : "long"
          },
          "status" : {
            "type" : "long"
          },
          "status_time" : {
            "type" : "date",
            "format" : "yyyy-MM-dd HH:mm:ss"
          },
          "sub_operation_type" : {
            "type" : "long"
          },
          "user_id" : {
            "type" : "long"
          }
        }
      }
    }
  }
}
```

建完以后只能用hive write，还不能用hive read ES.

需要将'es.mapping.date.rich'设置为'false'

> es.mapping.date.rich (default true)
> Whether to create a rich Date like object for Date fields in Elasticsearch or returned them as primitives (String or long). By default this is true. The actual object type is based on the library used; noteable exception being Map/Reduce which provides no built-in Date object and as such LongWritable and Text are returned regardless of this setting.

## 4. 总结

本文主要简单介绍了如何用HIVE和Elasticsearch结合进行数据分析, 对于Elasticsearch-Hadoop的理解还较浅, 本文只是入门篇。

## 引用

1. [Elasticsearch Formart](<https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html>)
2. [使用Hive读写ElasticSearch中的数据](<http://lxw1234.com/archives/2015/12/585.htm>)

* 原创文章，转载请注明： 转载自[Lamborryan](<http://lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/hive-elasticsearch
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
