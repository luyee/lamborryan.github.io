---
layout: post
title: Hive数据仓库之mongo-hadoop的split
date: 2015-11-23 21:30:45
categories: 大数据
tags: Hive, Mongo-Hadoop
---
# Hive数据仓库之mongo-hadoop的split

## 前言

前文简单介绍了mongo-hadoop的使用情况, 如果是非生产环境下, 那么按照前文的方法使用mongo-hadoop基本就没啥问题了,但是在生产环境上还是会有不少问题的,本文将在前文基础上更深入的聊一下mongo-hadoop在hive上的使用。

## 问题描述

在生产环境上, 创建以下table:
{% highlight bash sql %}
CREATE EXTERNAL TABLE IF NOT EXISTS Product(
    product_id STRUCT<oid:STRING,bsontype:INT>,
    shopId STRUCT<oid:STRING,bsontype:INT>,
    ptId STRUCT<oid:STRING,bsontype:INT>,
    categoryIds ARRAY<STRUCT<oid:STRING,bsontype:INT>> ,
    fprice INT ,
    pprice INT ,
    dprice INT ,
    status STRING,
    created STRING,
    updated STRING,
    isTop BOOLEAN,
    topTime STRING,
    soldCount STRING,
    ltype STRING,
    lcount STRING,
    productLimitCount STRING,
    statusChangeTime STRING,
    offlineReason STRING,
    seller STRING,
    sellerId STRUCT<oid:STRING,bsontype:INT>)
STORED BY 'com.mongodb.hadoop.hive.MongoStorageHandler'
WITH SERDEPROPERTIES('mongo.columns.mapping'='{"product_id":"_id","shopId":"shopId","ptId":"ptId","categoryIds":"categoryIds",
    "fprice":"fprice","pprice":"pprice","dprice":"dprice","status":"status","created":"created","updated":"updated",
    "isTop":"isTop","topTime":"topTime","soldCount":"soldCount","ltype":"ltype","lcount":"lcount",
    "productLimitCount":"productLimitCount","statusChangeTime":"statusChangeTime","offlineReason":"offlineReason",
    "seller":"seller","sellerId":"sellerId"}')
TBLPROPERTIES("mongo.properties.path"="${hivevar:properties_root}/product.properties");
{% endhighlight sql %}

创建表的过程没有啥问题, 但是当你去select查询时候会出现以下问题:

{% highlight bash java %}
Error: java.io.IOException: com.mongodb.MongoNotPrimaryException: The server is not the primary and did not execute the operation (state=,code=0)
{% endhighlight java %}

## 问题原因

* 我所用的生产环境的mongodb是单shard多节点模式, 即数据都存放在一个shard里同时存在一个主节点(primary), 多个从节点(secondry)。secondry节点负责数据读取, primary负责数据写入, 这样的读写分离大大降低了primary节点的负载, 因此公司在权限管理上对primary特别看重。
* mongo-hadoop进行split时候会根据具体的mongo部署选择split方案, 由于是单机shard多节点模式, 所以mongodb-hadoop就会使用自带的splitVector来进行split, 前文讲到splitVector需要较高权限的clusterManager role, 现在又发现运行splitVector必须允许在primary上。

## 解决方法

* However, even once the collection stats are retrieved, the next step in calculating the splits is to use the splitVector command, which can only be executed on a primary. If you're running unsharded, you'll need to do one of the following:
* Point the connector at a primary
Disable calculating splits by setting mongo.input.split.create_input_splits to false
* Write your own Splitter, and set mongo.splitter.class to your implementation.
If you have a sharded cluster, you can use your read preference to target hidden secondaries from each shard. Just explicitly define what Splitter class you want to use, i.e. set mongo.splitter.class to com.mongodb.hadoop.splitter.ShardMongoSplitter or com.mongodb.hadoop.splitter.ShardChunkMongoSplitter.

在mongodb-hadoop jira上给出了具体的方法 https://jira.mongodb.org/browse/HADOOP-220
同时再细看mongodb-hadoop的文档, 不难发现其实他在上面也有说明, https://github.com/mongodb/mongo-hadoop/wiki/Configuration-Reference
所以这也给我的教训蛮深,看文档要仔细。

最后我采用的解决方法是不让mongo-hadoop自己选择split方案, 而是采用SingleMongoSplitter, 即只会起一个map, 但它大大影响了性能

mongo.splitter.class=com.mongodb.hadoop.splitter.SingleMongoSplitter

## 知识点

* mongodb-hadoop支持多种split方案, 包括:
    com.mongodb.hadoop.splitter.StandaloneMongoSplitter
    com.mongodb.hadoop.splitter.ShardMongoSplitter
    com.mongodb.hadoop.splitter.ShardChunkMongoSplitter
    com.mongodb.hadoop.splitter.MultiMongoCollectionSplitter
  分别对应单shard,多shard等情况, 可以通过mongo.splitter.class进行设置， 更多的配置详见:https://github.com/mongodb/mongo-hadoop/wiki/Configuration-Reference
* 单个shard时候, com.mongodb.hadoop.splitter.StandaloneMongoSplitter使用splitVector命令进行split, 因此需要在primary节点上运行,
* 当是多个shards时候, mongodb-hadoop会从config.shards读取分片信息从而进行split
* 当单shard但是运行了mongos时候, mongodb-hadoop会运行collStats命令

## 总结

以后看文档需要更加仔细.

本文完
