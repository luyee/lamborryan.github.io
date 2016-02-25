---
layout: post
title: Hadoop之LZO Compress深入
date: 2016-02-23 23:30:00
categories: 大数据
tags: Hadoop Hive Spark Hbase Sqoop
---
# Hadoop之LZO Compress深入
## 1.前言
[<Hadoop之LZO Compress配置安装>](http://lamborryan.github.io/hadoop-lzo-compress/)介绍了Hadoop及其他组件增加LZO压缩功能的配置安装, 本文将进一步介绍使用中遇到的坑以及如何实现LZO文件的split。

## 2.LZOP和LZO的区别

在前文配置中我默认情况下使用**com.hadoop.compression.lzo.LzoCodec**来做为默认的输出压缩格式，而LzoCodec输出的文件格式是.lzo_deflate。这时就会让人疑惑为啥LZO压缩输出的文件格式是这样的。
当我要使用以下命令对压缩文件建立索引时又发现索引建立失败。

``` bash
hadoop jar  \  
/data/bmw/services/hadoop/share/hadoop/common/hadoop-lzo-0.4.20-SNAPSHOT.jar \  
com.hadoop.compression.lzo.DistributedLzoIndexer \
/user/hive/warehouse/xml_stat.db/fact_coupon/000000_0.lzo_deflate \
```

* lzo压缩默认的是不支持切分的，也就是说，如果直接把lzo文件当作Mapreduce任务的输入，那么Mapreduce只会用一个Map来处理这个输入文件，这显然不是我们想要的。其实我们只需要对lzo文件建立索引，这样这个lzo文件就会支持切分，也就可以用多个Map来处理lzo文件。
* 以上/user/hive/warehouse/xml_stat.db/fact_coupon/000000_0.lzo_deflate文件路径是在hdfs上, 生成出来的索引文件后缀为.index，并存放在lzo同一目录下.在本例中产生的索引文件是存放在/user/hive/warehouse/xml_stat.db/fact_coupon目录下，名称为000000_0.lzo.index。
* 也可以使用以下命令对lzo文件来建立索引, 这个方法和上面方法产生出来的索引文件是一样的；但是上面的方法是通过启用Mapreduce任务来执行的，而这里的方法只在一台客户机上运行，效率很慢！

``` bash
hadoop jar \
/data/bmw/services/hadoop/share/hadoop/common/hadoop-lzo-0.4.20-SNAPSHOT.jar \
com.hadoop.compression.lzo.LzoIndexer  \
/user/hive/warehouse/xml_stat.db/fact_coupon/000000_0.lzo_deflate \
```

言归正传, 为啥lzo_defate不支持索引。

* LzopCodec为了兼容LZOP程序添加了如 bytes signature, header等信息
* 如果使用 LzoCodec作为Reduce输出，则输出文件扩展名为".lzo_deflate"，它无法被lzop读取；如果使用LzopCodec作为Reduce输出，则扩展名为".lzo"，它可以被lzop读取生成lzo index job的”DistributedLzoIndexer“无法为 LzoCodec，即 ".lzo_deflate"扩展名的文件创建index
* lzo_deflate文件无法作为MapReduce输入，”.LZO"文件则可以。
* 综上所述得出最佳实践：map输出的中间数据使用 LzoCodec，reduce输出使用 LzopCodec

所以接下来又需要修改配置文件。

## 3.配置

#### core-site.xml

``` xml
<property>
    <name>io.compression.codecs</name>
    <value>org.apache.hadoop.io.compress.GzipCodec,org.apache.hadoop.io.compress.DefaultCodec,com.hadoop.compression.lzo.LzoCodec,com.hadoop.compression.lzo.LzopCodec</value>
</property>
<property>
    <name>io.compression.codec.lzo.class</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
```

#### mapred-site.xml

``` xml
<property>
    <name>mapreduce.map.output.compress</name>
    <value>true</value>
</property>
<property>
    <name>mapred.map.output.compression.codec</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
<property>
    <name>mapred.child.env</name>
    <value>LD_LIBRARY_PATH=/data/bmw/services/hadoop-2.5.2/lib/native</value>
</property>
```

#### hive-site.xml

``` xml
<property>
  <name>mapreduce.output.fileoutputformat.compress.codec</name>
  <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
<property>
  <name>hive.exec.compress.output</name>
  <value>true</value>
</property>
<property>
  <name>mapreduce.output.fileoutputformat.compress</name>
  <value>true</value>
</property>
<property>
  <name>hive.exec.compress.intermediate</name>
  <value>true</value>
  <description> This controls whether intermediate files produced by Hive betweenmultiple map-reduce jobs are compressed. The compression codec and other optionsare determined from hadoop config variables mapred.output.compress* </description>
</property>
<property>
    <name>hive.intermediate.compression.codec</name>
    <value>com.hadoop.compression.lzo.LzoCodec</value>
</property>
</configuration>
```

> 注意: 这里增加了hive.intermediate.compression.codec为com.hadoop.compression.lzo.LzoCodec

这是因为hive job是有多个mapreduce job组成，每个mapreduce job称为hive stage, 如果不设置该属性，每个stage输出都将会是lzop压缩即lzo_defate，而lzo_defate不能做为lzop读取，所以会造成hive job任务失败, 即使你配置了inputformat为com.hadoop.mapred.DeprecatedLzoTextInputFormat也没用。 因为inputformat只会适应hive的输入即第一个stage, 后续的stage都是中间结果。错误信息如下：

``` java
caused by: java.io.EOFException: Premature EOF from inputStream
	at com.hadoop.compression.lzo.LzopInputStream.readFully(LzopInputStream.java:75)
	at com.hadoop.compression.lzo.LzopInputStream.readHeader(LzopInputStream.java:114)
	at com.hadoop.compression.lzo.LzopInputStream.<init>(LzopInputStream.java:54)
	at com.hadoop.compression.lzo.LzopCodec.createInputStream(LzopCodec.java:83)
	at org.apache.hadoop.hive.ql.io.RCFile$ValueBuffer.<init>(RCFile.java:667)
	at org.apache.hadoop.hive.ql.io.RCFile$Reader.<init>(RCFile.java:1431)
	at org.apache.hadoop.hive.ql.io.RCFile$Reader.<init>(RCFile.java:1342)
	at org.apache.hadoop.hive.ql.io.rcfile.merge.RCFileBlockMergeRecordReader.<init>(RCFileBlockMergeRecordReader.java:46)
	at org.apache.hadoop.hive.ql.io.rcfile.merge.RCFileBlockMergeInputFormat.getRecordReader(RCFileBlockMergeInputFormat.java:38)
	at org.apache.hadoop.hive.ql.io.CombineHiveRecordReader.<init>(CombineHiveRecordReader.java:65)
	... 16 more
```

#### Sqoop

指定压缩方式:--compress  --compression-codec lzop

## 4.索引

如何建立索引在前面部分已经讲过, 本节讲下建了索引会带来什么问题以及如何解决。
建立索引后会在相同目录下生成.index, 如果不做过滤那么hive或者mapreduce都会把这些文件做为输入源, 从而产生噪声.

#### Sqoop

Sqoop默认会建立hive表，如果不做任何处理, 那么hive的inputformat就是TextInputFormat, 他会读取index文件， 所以我需要手动为sqoop导入的表建好。

``` sql
CREATE TABLE IF NOT EXISTS xml_stat.`cashflow`(
  `id` bigint,
  `deal_id` bigint,
  `user_id` bigint,
  `flow_type` string,
  `amount` int,
  `create_time` string,
  `status` int,
  `status_time` string,
  `ref_id` int,
  `channel_type` int,
  `operation_type` int,
  `sub_operation_type` int)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\u0001'
LINES TERMINATED BY '\n'
STORED AS INPUTFORMAT  'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat';
```

``` bash
$SQOOP_HOME/bin/sqoop import \
--connect jdbc:mysql://localhost:3306/xmlife_deal  \
--username aaa \
--password bbb \
--table cashflow \
--hive-table xml_stat.cashflow \
--hive-import  \
--split-by id \
--hive-overwrite \
--hive-drop-import-delims \
--compress \
--compression-codec lzop \
--outdir ${outdir} \
-m 5
```

sqoop import默认会对导入的表cashflow建索引, 如此cashflow不但已完成索引可以进行split, 同时也输出正常

#### hive

hive 要过滤index文件, 需要设置intputformat:

``` sql
CREATE TABLE a
STORED AS INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
AS SELECT * FROM b
```

或者

``` sql
create table lzo(
id int,
name string)
STORED AS INPUTFORMAT 'com.hadoop.mapred.DeprecatedLzoTextInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat';
```

这里普及下InputFormat河SerDe的区别：
* 调用InputFormat，将文件切成不同的文档。每篇文档即一行(Row)。
* 调用SerDe的Deserializer，将一行(Row)，切分为各个字段。

#### mapreduce

对应Streaming作业还需要注意的是，使用DeprecatedLzoTextInputFormat输入格式，会把文本的行号当作key传送到reduce的，所以我们需要将行号去掉

``` bash
$ bin/hadoop jar \
$HADOOP_HOMOE/share/hadoop/tools/lib/hadoop-streaming-2.2.0.jar \
-inputformat com.hadoop.mapred.DeprecatedLzoTextInputFormat \
-input /home/wyp/input \
-D stream.map.input.ignoreKey=true \
-output /home/wyp/results \
-mapper /bin/cat \
-reducer wc \
```

至此可以对单个LZO压缩文件进行split并进行多个map并行访问。

## 总结
本文主要介绍了hadoop及其组件在使用LZO压缩中碰到的坑, 以及如何对LZO文件建立索引以及保证数据的准确性。

本文完
