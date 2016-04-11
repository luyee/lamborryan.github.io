---
layout: post
title: Hadoop系列(7)之LZO Compress配置安装
date: 2016-02-23 23:30:00
categories: 大数据
tags: Hadoop Hive Spark Hbase Sqoop
---

## 前言

随着数据量不停的增加，磁盘空间开始吃紧，这个时候就需要对磁盘上的文件进行压缩，所以花了两天的时间研究了下hadoop及其他组件的压缩特性。对压缩算法进行了初步的筛选, 由于编解码的速度以及需要split, 所以选择了lzo压缩算法，本文主要介绍lzo压缩如何在hadoop集群以及相关组件上进行配置。
本文是LZO系列的第一篇, 另外一篇为[《Hadoop系列(8)之LZO Compress深入》](http://lamborryan.github.io/hadoop-lzo-compress-index/)

## 安装LZO Native库

#### 下载lzo 2.06版本，编译64位版本，同步到集群中

``` bash
wget http://www.oberhumer.com/opensource/lzo/download/lzo-2.06.tar.gz
export CFLAGS=-m64
./configure -enable-shared -prefix=/usr/local/hadoop/lzo/
sudo make && sudo make test && sudo make install
```

#### 在/usr/local/hadoop/lzo/目录下会生成相应的LZO库

``` bash
bmw@data1 lzo]$ pwd
/usr/local/hadoop/lzo
[bmw@data1 lzo]$ ll
总用量 12
drwxr-xr-x 3 root root 4096 2月  23 13:45 include
drwxr-xr-x 2 root root 4096 2月  23 13:45 lib
drwxr-xr-x 3 root root 4096 2月  23 13:45 share
[bmw@data1 lzo]$ cd lib/
[bmw@data1 lib]$ ll
总用量 1480
-rw-r--r-- 1 root root 931572 2月  23 13:45 liblzo2.a
-rwxr-xr-x 1 root root    920 2月  23 13:45 liblzo2.la
lrwxrwxrwx 1 root root     16 2月  23 13:45 liblzo2.so -> liblzo2.so.2.0.0
lrwxrwxrwx 1 root root     16 2月  23 13:45 liblzo2.so.2 -> liblzo2.so.2.0.0
-rwxr-xr-x 1 root root 574139 2月  23 13:45 liblzo2.so.2.0.0
```

#### 复制/usr/local/hadoop/lzo/lib 下的库文件到${HADOOP_HOME}/lib/native 下，并同步到其他节点

## 安装Hadoop-LZO库

* 说明: 我使用的Hadoop的版本为2.5.2。
* Hadoop 1.x的时候我们是直接按照cloudera的文档clone https://github.com/kevinweil/hadoop-lzo.git 上编译的，它是fork  https://github.com/twitter/hadoop-lzo
* 但是kevinweil这个版本已经很久没有更新了，而且它是基于Hadoop 1.x去编译的，不能用于Hadoop 2.x。而twitter/hadoop-lzo三个月将Ant的编译方式切换为Maven，默认的dependency中Hadoop jar包就是2.x的，所以要clone twitter的hadoop-lzo，用Maven编译jar包和native library

### 下载Hadoop-LZO
``` bash
git clone https://github.com/twitter/hadoop-lzo.git​
export CFLAGS=-m64
export CXXFLAGS=-m64
export C_INCLUDE_PATH=/usr/local/hadoop/lzo/include
export LIBRARY_PATH=/usr/local/hadoop/lzo/lib
mvn clean package -Dmaven.test.skip=true
```

### 修改pom.xml
``` bash
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <hadoop.current.version>2.5.2</hadoop.current.version>
    <hadoop.old.version>1.0.4</hadoop.old.version>
</properties>
```

### mvn编译
``` bash
sudo mkdir -p /usr/local/hadoop/hadoop-2.5.2/lib/native/
tar -cBf - -C target/native/Linux-amd64-64/lib . | sudo tar -xBvf - -C ${HADOOP_HOME}/lib/native
sudo cp target/hadoop-lzo-0.4.20-SNAPSHOT.jar ${HADOOP_HOME}/share/hadoop/common/
```

* 复制hadoop-lzo-0.4.20-SNAPSHOT.jar到${HADOOP_HOME}/share/hadoop/common/下，并同步到其他节点
* 复制target/native/Linux-amd64-64/lib下的库文件到${HADOOP_HOME}/lib/native下，并同步到其他节点

``` bash
[bmw@data1 hadoop-lzo]$ ls -l
总用量 176
-rw-r--r-- 1 bmw bmw 106214 2月  23 13:52 libgplcompression.a
-rw-rw-r-- 1 bmw bmw   1128 2月  23 13:52 libgplcompression.la
lrwxrwxrwx 1 bmw bmw     26 2月  23 14:00 libgplcompression.so -> libgplcompression.so.0.0.0
lrwxrwxrwx 1 bmw bmw     26 2月  23 14:00 libgplcompression.so.0 -> libgplcompression.so.0.0.0
-rwxrwxr-x 1 bmw bmw  69363 2月  23 13:52 libgplcompression.so.0.0.0
```

## Hadoop支持压缩

### 在core-site.xml上修改

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

### 在mapred-site.xml修改

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

* 其中mapred-site中设置mapred.child.env的​LD_LIBRARY_PATH很重要，因为hadoop-lzo通过JNI调用(java.library.path)​libgplcompression.so，然后libgplcompression.so​再通过dlopen这个系统调用（其实是查找系统环境变量LD_LIBRARY_PATH​）来加载liblzo2.so​。container在启动的时候，需要设置LD_LIBRARY_PATH​环境变量，来让LzoCodec加载​native-lzo library，如果不设置的话，会在container的syslog中报下面的错误​​
经过以上配置后，HDFS和MapReduce已经可以支持lzo压缩。 lzo压缩后后缀名为lzo_deflate。

## Hive支持压缩

修改hive-site.xml, 实现默认压缩， 经过这样配置后通过hive生成的最终文件或者中间文件都会被压缩，关于读取hive会根据文件的后缀名自动识别是否进行解压。

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
</configuration>
```

只需单次job有效可以:

``` xml
SET mapreduce.output.fileoutputformat.compress.codec=com.hadoop.compression.lzo.LzoCodec;
SET hive.exec.compress.output=true;
SET mapreduce.output.fileoutputformat.compress=true;
```

## Sqoop支持压缩

SQOOP命令中加入以下参数即 --compress --compression-codec lzo


### Spark支持压缩

如果不配置会出现以下错误

``` java
java.lang.RuntimeException: Error in configuring object
    at org.apache.hadoop.util.ReflectionUtils.setJobConf(ReflectionUtils.java:109)
    at org.apache.hadoop.util.ReflectionUtils.setConf(ReflectionUtils.java:75)
    at org.apache.hadoop.util.ReflectionUtils.newInstance(ReflectionUtils.java:133)
    at org.apache.spark.rdd.HadoopRDD.getInputFormat(HadoopRDD.scala:190)
    at org.apache.spark.rdd.HadoopRDD.getPartitions(HadoopRDD.scala:203)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:219)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:217)
    at scala.Option.getOrElse(Option.scala:120)
    at org.apache.spark.rdd.RDD.partitions(RDD.scala:217)
    at org.apache.spark.rdd.MapPartitionsRDD.getPartitions(MapPartitionsRDD.scala:32)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:219)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:217)
    at scala.Option.getOrElse(Option.scala:120)
    at org.apache.spark.rdd.RDD.partitions(RDD.scala:217)
    at org.apache.spark.rdd.MapPartitionsRDD.getPartitions(MapPartitionsRDD.scala:32)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:219)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:217)
    at scala.Option.getOrElse(Option.scala:120)
    at org.apache.spark.rdd.RDD.partitions(RDD.scala:217)
    at org.apache.spark.rdd.MapPartitionsRDD.getPartitions(MapPartitionsRDD.scala:32)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:219)
    at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:217)
    at scala.Option.getOrElse(Option.scala:120)
    at org.apache.spark.rdd.RDD.partitions(RDD.scala:217)
    at org.apache.spark.SparkContext.runJob(SparkContext.scala:1781)
    at org.apache.spark.rdd.RDD$$anonfun$collect$1.apply(RDD.scala:885)
    at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:147)
    at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:108)
    at org.apache.spark.rdd.RDD.withScope(RDD.scala:286)
    at org.apache.spark.rdd.RDD.collect(RDD.scala:884)
    at org.apache.spark.sql.execution.SparkPlan.executeCollect(SparkPlan.scala:105)
    at org.apache.spark.sql.hive.HiveContext$QueryExecution.stringResult(HiveContext.scala:503)
    at org.apache.spark.sql.hive.thriftserver.AbstractSparkSQLDriver.run(AbstractSparkSQLDriver.scala:58)
    at org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver.processCmd(SparkSQLCLIDriver.scala:283)
    at org.apache.hadoop.hive.cli.CliDriver.processLine(CliDriver.java:423)
    at org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver$.main(SparkSQLCLIDriver.scala:218)
    at org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver.main(SparkSQLCLIDriver.scala)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:606)
    at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:665)
    at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:170)
    at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:193)
    at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:112)
    at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.lang.reflect.InvocationTargetException
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:606)
    at org.apache.hadoop.util.ReflectionUtils.setJobConf(ReflectionUtils.java:106)
    ... 45 more
Caused by: java.lang.IllegalArgumentException: Compression codec com.hadoop.compression.lzo.LzoCodec not found.
    at org.apache.hadoop.io.compress.CompressionCodecFactory.getCodecClasses(CompressionCodecFactory.java:135)
    at org.apache.hadoop.io.compress.CompressionCodecFactory.<init>(CompressionCodecFactory.java:175)
    at org.apache.hadoop.mapred.TextInputFormat.configure(TextInputFormat.java:45)
    ... 50 more
Caused by: java.lang.ClassNotFoundException: Class com.hadoop.compression.lzo.LzoCodec not found
    at org.apache.hadoop.conf.Configuration.getClassByName(Configuration.java:1803)
    at org.apache.hadoop.io.compress.CompressionCodecFactory.getCodecClasses(CompressionCodecFactory.java:128)
    ... 52 more
```

#### 修改spark-env.sh

```bash
export SPARK_LIBRARY_PATH=$SPARK_LIBRARY_PATH:/data/bmw/services/hadoop-2.5.2/lib/native
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/data/bmw/services/hadoop-2.5.2/lib/native
export SPARK_CLASSPATH=$SPARK_CLASSPATH:/data/bmw/services/hadoop-2.5.2/share/hadoop/common/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/common/lib/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/hdfs/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/hdfs/lib/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/yarn/lib/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/yarn/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/mapreduce/lib/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/mapreduce/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/tools/lib/*:/data/bmw/services/hadoop-2.5.2/share/hadoop/tools/*
```

## Hbase支持压缩

### 安装配置

* 复制或者软连接core-site.xml, hdfs-site.xml 到${HBASE_HOME}/conf
* 在${HBASE_HOME}/conf/hbase-env.sh中加入native库

``` bash
export HBASE_LIBRARY_PATH=$HBASE_LIBRARY_PATH:/data/bmw/services/hadoop-2.5.2/lib/native
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/data/bmw/services/hadoop-2.5.2/lib/native
```

* 复制hadoop-lzo-0.4.20-SNAPSHOT.jar到${HBASE_HOME}/lib
* 以上操作需要同步集群所有节点

### 易出现错误

``` java
java.io.IOException: Compression algorithm 'lzo' previously failed test.
	at org.apache.hadoop.hbase.util.CompressionTest.testCompression(CompressionTest.java:85)
	at org.apache.hadoop.hbase.regionserver.HRegion.checkCompressionCodecs(HRegion.java:4850)
	at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:4837)
	at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:4808)
	at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:4780)
	at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:4736)
	at org.apache.hadoop.hbase.regionserver.HRegion.openHRegion(HRegion.java:4687)
	at org.apache.hadoop.hbase.regionserver.handler.OpenRegionHandler.openRegion(OpenRegionHandler.java:489)
	at org.apache.hadoop.hbase.regionserver.handler.OpenRegionHandler.process(OpenRegionHandler.java:149)
	at org.apache.hadoop.hbase.executor.EventHandler.run(EventHandler.java:129)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
```

### 验证方法

* hbase org.apache.hadoop.hbase.util.CompressionTest hdfs://223.5.12.88:9000/user.dat lzo
* create 'test', {NAME=>'cf', COMPRESSION=>'lzo'}

### 对已有表设置压缩

#### disable相关表

``` bash
disable 'test'
```

实际产品环境中，’test’表可能很大，例如上几十T的数据，disable过程会比较缓慢，需要等待较长时间。disable过程可以通过查看hbase master log日志监控。

#### 修改表的压缩格式

``` bash
alter 'test', NAME => 'f', COMPRESSION => 'snappy'
```

NAME即column family，列族。HBase修改压缩格式，需要一个列族一个列族的修改。而且这个地方要小心，别将列族名字写错，或者大小写错误。因为这个地方任何错误，都会创建一个新的列族，且压缩格式为snappy。当然，假如你还是不小心创建了一个新列族的话，可以通过以下方式删除：

``` bash
alter 'test', {NAME=>'f', METHOD=>'delete'}
```
同样提醒，别删错列族，否则麻烦又大了~

#### 重新enable表

``` bash
enable 'test'
```

enable表后，HBase表的压缩格式并没有生效，还需要一个动作，即HBase major_compact

``` bash
major_compact 'test'
```

该动作耗时较长，会对服务有很大影响，可以选择在一个服务不忙的时间来做。
describe一下该表，可以看到HBase 表压缩格式修改完毕。

## 引用

1. https://cwiki.apache.org/confluence/display/Hive/LanguageManual+LZO
2.  http://blog.csdn.net/lalaguozhe/article/details/10912527?utm_source=tuicool&utm_medium=referral
3. http://www.iteblog.com/archives/992
4. http://www.iteblog.com/archives/996
5. http://blog.csdn.net/stark_summer/article/details/48375999
6. http://hbase.apache.org/book.html#compression

##### 本文完


* 原创文章，转载请注明： 转载自[Lamborryan](<http://lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/hadoop-lzo-compress
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
