---
layout: post
title: Hive数据仓库之mongo-hadoop的使用
date: 2015-11-17 12:29:30
categories: 大数据
tags: Hive Mongo-Hadoop
---
# Hive数据仓库之mongo-hadoop的使用

## 前言
公司使用mongo存储了部分业务数据,当然mongo有很多优点:

1. 作为nosql没有固定的schema, 模式自由
2. 面向集合存储，易存储对象类型的数据, 一条记录就是一个json语句
3. 支持查询, 且查询方式比较丰富, 想想同为nosql的hbase在这方面的无奈
4. 其他稳定和性能方面的优点就不说了。

易使用和模式自由使得开发人员爱不释手, 但是对于数据人员就比较头疼了, schema的不固定和异于sql的查询方式对于数据同步,清洗都会造成或多或少的困扰。

对于Mysql这类关系数据库, 我们可以通过sqooq进行数据同步, 但是sqoop不支持mongodb。 不过, 幸亏monogo自带了mongo-hadoop工具包, 我们可以在mongodb上面进行mapreduce/spark和基于mapreduce/spark的工具(Hive,Pig)的使用开发.

本文主要介绍了Hive如何通过mongo-hadoop来对mongodb上的数据进行清洗分析。每次当Hive进行查询时Hive都会对mongodb数据重新进行读取, 因此保证了每次mongo数据是最新的, 所以省略了数据同步过程。

## Mongo-Hadoop的使用
到写文章时候Mongo-Hadoop的版本是1.4.2, 且1.5.0正在开发, 从1.5.0的源码来看它的特性更是我需要的, 所以我就从master分支上拉了最新的代码进行编译, 所以后续的版本号都是1.5.0-SNAPSHOT. 编译mongo-hadoop需要gradlew, 运行./gradlew jar就行

### Mongo-Hadoop的安装
要使得Hive能成功访问Mongo需要以下几个包:

* bson-3.0.4.jar
* mongodb-driver-3.0.4.jar
* mongodb-driver-core-3.0.4.jar
* mongo-hadoop-core-1.5.0-SNAPSHOT.jar
* mongo-hadoop-hive-1.5.0-SNAPSHOT.jar

如果缺其中1个就会报错:ClassNotFoundException/NoClassDefFoundError

### Hive table
创建Hive表使加入对mongo的url:

``` sql
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
```

上述是Hive的建表语句:

1. 使用EXTERNAL外部表, 在warehouse里面的product目录下面是没有数据的, 保障每次查询都是从mongo进行现查.
2. Mongodb的ObjectId在hive里需要用 STRUCT<oid:STRING,bsontype:INT> 进行转换, 查询时候使用product_id.oid来获取.
3. Hive的STRUCT, MAP, ARRAY 对应mongo的STRUCT, MAP, ARRAY.
4. 需要在STORED BY 注明使用的是mongo-hadoop-hive-1.5.0-SNAPSHOT.jar的com.mongodb.hadoop.hive.MongoStorageHandler进行解析.
5. SERDEPROPERTIES存放的是mongo表的列与hive表的列的映射，需要注意的是如果两个列的列名一样时候也需要写出"a"="a",否则会出现null
6. mongo.columns.mapping的属性是指存放mongo.uir配置文件的路径, 1.5.0区别与之前版本的一个原因是, 1.5.0不仅支持在TBLPROPERTIES设置mongo.uri, 还支持mongo.properties.path.这样使得安全性更高。如果在TBLPROPERTIES直接写mongo.uri, 用户就可以通过查看建表语句show create table Product 查看这个mongo.uri的信息了。
7. mongo.uri的格式如下 mongo.uri=mongodb://user:passwd@localhost:1201/default.Product
8. 一个表对应一个mongo.properties.path, 所以当表很多的时候需要写很多个配置文件，这个可以后续进行优化
9. mongo.properties.path的路径是hdfs的文件路径

### Spark SQL

Spark 已经支持兼容了Hive, 虽然并没有完全兼容。

要使用Spark SQL查询需要加上面几个包放到SPARK_CLASSPATH里面，可以在spark-env.sh进行修改。

接着启动spark thrift即可(不用启动hive server2)用beeline访问 sh sbin/start-thriftserver.sh。

需要注意的是:

1. Hive存放properties是在hdfs上, 而spark则是在本地文件系统中。
2. 之前使用STRUCT存放ObjectId, 会在spark上打warn日志

15/11/19 17:23:15 WARN BSONSerDe: SWEET ------ structName is oid
15/11/19 17:23:15 WARN BSONSerDe: SWEET ------ structName is bsontype

### Hive write to mongo

首先你得有mongo的write的权限.

其次, 不要建external表

``` sql
CREATE TABLE IF NOT EXISTS UserCollect(
    userId BIGINT,
    items ARRAY<STRUCT<templateId:STRING,skuId:STRING,count:INT,buyTime:BIGINT>>)
STORED BY 'com.mongodb.hadoop.hive.MongoStorageHandler'
WITH SERDEPROPERTIES('mongo.columns.mapping'='{"userId":"userId","items.templateId":"items.templateId","items.skuId":"items.skuId","items.count":"items.count","items.buyTime":"items.buyTime"}')
TBLPROPERTIES("mongo.properties.path"="${hivevar:properties_root}/usercollect.properties");
```

insert into table  和 insert into overwrite table 在这里效果是一样的

``` sql
INSERT INTO TABLE xml_meta_mongo.UserCollect
SELECT
customId AS userId,
collect_list(
    NAMED_STRUCT (
        "templateId", templateId,
        "skuId", skuId,
        "count", count,
        "buyTime", buyTime
    )
) AS items
FROM st
```

如果需要_id, 则需要在mongodb创建表是设置好

``` sql
db.createCollection("UserCollect", {autoIndexId:true})
```


### 注意事项

#### 权限问题

mongo-hadoop之所以能运行mapreduce或者spark的应用, 是因为mongo提供了一个split功能，正是由于这个功能的存在才能进行任务的划分。但是mongo的split功能的权限非常高,
如果我用只读的账户去进行之前的建表行为就会出现以下错误:

``` java
14/05/02 13:17:01 ERROR util.MongoTool: Exception while executing job...
java.io.IOException: com.mongodb.hadoop.splitter.SplitFailedException: Unable to calculate input splits: need to login
at com.mongodb.hadoop.MongoInputFormat.getSplits(MongoInputFormat.java:53)
at org.apache.hadoop.mapreduce.JobSubmitter.writeNewSplits(JobSubmitter.java:493)
at org.apache.hadoop.mapreduce.JobSubmitter.writeSplits(JobSubmitter.java:510)
at org.apache.hadoop.mapreduce.JobSubmitter.submitJobInternal(JobSubmitter.java:394)
at org.apache.hadoop.mapreduce.Job$10.run(Job.java:1295)
```

* This is because the connector needs to run the MongoDB-internal splitVector DB command, under the covers, to work out how to split the MongoDB data up into sections ready to distribute across the Hadoop Cluster. However, by default, you are unlikely to have given sufficient privileges to the user, used by the connector, to allow this DB command to be run. This issue can be simulated easily by opening a mongo shell against the database, authenticating with your username and password and then running the splitVector command manually.

* To address this issue, you first need to use the mongo shell, authenticated as your administration user, and run the updateUser command to give the connector user the clusterManager role, to enable the connector to run the DB commands it requires.

从上面两段话可以看出需要clusterManager role 才能进行split

文章来自

http://pauldone.blogspot.com/2014/05/mongodb-connector-authentication.html

https://github.com/mongodb/mongo-hadoop/wiki/Usage-with-Authentication

#### 删除drop table

当有足够的权限时候，如果没有使用external建表, 那么删除该表的时候也会删除mongo的表, 所以需要注意一定要用external。

* If the table created is EXTERNAL, when the table is dropped only its metadata is deleted; the underlying MongoDB collection remains intact. On the other hand, if the table is not EXTERNAL, dropping the table deletes both the metadata associated with the table and the underlying MongoDB collection.

来自:

https://github.com/mongodb/mongo-hadoop/wiki/Hive-Usage

## 总结

本文主要介绍了如何使用mongo-hadoop 与 hive结合, 在mongo上执行hsql查询。 同时介绍了mongo-hadoop的几个安全注意事项。

本文完


* 原创文章，转载请注明： 转载自[Lamborryan](<http://lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/mongo-hadoop-hive
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
