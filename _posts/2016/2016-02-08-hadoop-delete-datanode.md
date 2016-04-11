---
layout: post
title: Hadoop系列(6)之如何动态的删除DataNode
date: 2016-02-08 10:30:35
categories: 大数据
tags: Hadoop
---

## 1. 应用场景

今早运营跟我说昨天的数据没有分析出来，我第一反应就是MapReduce又跑挂了。结果登上服务器发现线上所有的DataNode全部dead了，而NodeManager还全部在线。
线上的服务器的HADOOP是2.6.0
部署情况：

* hadoop1    NameNode
* hadoop2    SecondNamenode
* hadoop3    ResourceManege     JobHistoryServer
* hadoop4    NodeManager        DataNode
* hadoop5 ~ hadoop9  DataNode  

看到DataNode全部挂掉，第一反应就是在NameNode运行 ```./start-all.sh```

运行的结果是除了hadoop4 其他的DataNode启动正常，暂时不去考虑为啥所有的DataNode会全部dead。我们先来看下为啥hadoop4的DataNode dead，查看日志发现：

```java
************************************************************/
2015-07-10 11:52:35,108 INFO org.apache.hadoop.hdfs.server.datanode.DataNode: registered UNIX signal handlers for [TERM, HUP, INT]
2015-07-10 11:52:35,752 WARN org.apache.hadoop.hdfs.server.datanode.DataNode: Invalid dfs.datanode.data.dir /home/hadoop/data/hadoop :
org.apache.hadoop.util.DiskChecker$DiskErrorException: Directory is not writable: /home/hadoop/data/hadoop
	at org.apache.hadoop.util.DiskChecker.checkAccessByFileMethods(DiskChecker.java:193)
	at org.apache.hadoop.util.DiskChecker.checkDirAccess(DiskChecker.java:174)
	at org.apache.hadoop.util.DiskChecker.checkDir(DiskChecker.java:157)
	at org.apache.hadoop.hdfs.server.datanode.DataNode$DataNodeDiskChecker.checkDir(DataNode.java:2239)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.checkStorageLocations(DataNode.java:2281)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.makeInstance(DataNode.java:2263)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.instantiateDataNode(DataNode.java:2155)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.createDataNode(DataNode.java:2202)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.secureMain(DataNode.java:2378)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.main(DataNode.java:2402)
2015-07-10 11:52:35,754 FATAL org.apache.hadoop.hdfs.server.datanode.DataNode: Exception in secureMain
java.io.IOException: All directories in dfs.datanode.data.dir are invalid: "/home/hadoop/data/hadoop/"
	at org.apache.hadoop.hdfs.server.datanode.DataNode.checkStorageLocations(DataNode.java:2290)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.makeInstance(DataNode.java:2263)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.instantiateDataNode(DataNode.java:2155)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.createDataNode(DataNode.java:2202)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.secureMain(DataNode.java:2378)
	at org.apache.hadoop.hdfs.server.datanode.DataNode.main(DataNode.java:2402)
2015-07-10 11:52:35,755 INFO org.apache.hadoop.util.ExitUtil: Exiting with status 1
2015-07-10 11:52:35,758 INFO org.apache.hadoop.hdfs.server.datanode.DataNode: SHUTDOWN_MSG:
```

/home/hadoop/data/hadoop存放了datanode的元数据，如果该目录的数据丢失，那么HDFS在该节点上的数据都丢失了。

```shell
hadoop@dt104:~/hadoop/logs$ ls /home/hadoop/data/hadoop
ls: reading directory /home/hadoop/data/hadoop: Input/output error
```

从上述的错误信息，可以看出基本上是挂载在该文件路径的硬盘坏了，那么我只能把Hadoop4这个节点从线上的集群下架。

## 2. 动态删除DataNode和NodeManager

主要分为以下几步:

1.在${HADOOP_HOME}/etc/hadoop目录下新建文件exclude，并在exclude中写入需要删除的节点IP ```hadoop4```，可以支持多个，一行一个。这里需要说明一下，如果删除hadoop4节点后，剩余的dataNode的数量小于备份数，需要减少备份数。

```xml
<property>
    <name>dfs.replication</name>
    <value>2</value>
    <description>该值需要小于DataNode的个数</description>
 </property>
```

2.在${HADOOP_HOME}/etc/hadoop的hdfs-site.xml修改配置,该配置是告诉NameNode这个DataNode不需要再去关注了.

```xml
<property>
    <name>dfs.hosts.exclude</name>
    <value>/home/hadoop/hadoop/etc/hadoop/exclude</value>
</property>
```

3.在${HADOOP_HOME}/etc/hadoop的yarn-site.xml修改配置,该配置是告诉ResourceManager这个NodeManager不需要再去关注了

```xml
<property>
    <name>yarn.resourcemanager.nodes.exclude-path</name>
   <value>/home/hadoop/hadoop/etc/hadoop/exclude</value>
</property>
```

4.由于NameNode和ResourceManager不是部署在一起的，所以需要将这些配置文件更新到其他的slave上。

5.动态更新这些配置文件

```shell
#更新HDFS的配置文件hdfs-site.xml
hdfs dfsadmin -refreshNodes  
#更新YARN的配置文件yarn-site.xml
yarn rmadmin -refreshNodes  
```

6.运行hdfs dfsadmin -report查看Hadoop4的状态，该状态说明HADOOP4正在进行关闭，且正在进行数据的转移，这样的过程对保护数据很有效果。不过由于数据量大时，Decommission过程会运行较久。

```shell
Dead datanodes (1):
Name: 192.168.2.104:50010 (hadoop4)
Hostname: hadoop4
Decommission Status : Decommission in progress
Configured Capacity: 0 (0 B)
DFS Used: 0 (0 B)
Non DFS Used: 0 (0 B)
DFS Remaining: 0 (0 B)
DFS Used%: 100.00%
DFS Remaining%: 0.00%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 0
Last contact: Mon Jun 29 20:21:58 CST 2015
Decommissioning datanodes (1):
Name: 192.168.2.104:50010 (hadoop4)
Hostname: hadoop4
Decommission Status : Decommission in progress
Configured Capacity: 0 (0 B)
DFS Used: 0 (0 B)
Non DFS Used: 0 (0 B)
DFS Remaining: 0 (0 B)
DFS Used%: 100.00%
DFS Remaining%: 0.00%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 0
Last contact: Mon Jun 29 20:21:58 CST 2015
```

7.当DataNode的状态变为Decommissioned了说明该DataNode已被删除。

8.运行以下命令进行数据进行数据均衡，数据量大时候均衡也较旧，dfs.balance.bandwidthPerSec我设了20M/s,如果write比较频繁，均衡的效果不会太明显。

> 关于进行balance，需要设置dfs.balance.bandwidthPerSec  默认设置：1048576（1 M/S），参数含义：设置balance工具在运行中所能占用的带宽。
./start-balancer.sh -threshold 1
-threshold 默认设置：10，参数取值范围：0-100，参数含义：判断集群是否平衡的目标参数，每一个 datanode 存储使用率和集群总存储使用率的差值都应该小于这个阀值 ，理论上，该参数设置的越小，整个集群就越平衡，但是在线上环境中，hadoop集群在进行balance时，还在并发的进行数据的写入和删除，所以有可能无法到达设定的平衡参数值。

9.至此动态删除DataNode算结束了，集群也运行正常，当hadoop4的服务器硬盘修好且上线了，我再来写如何动态增加DataNode。

## 3. 动态添加DataNode和NodeManager

添加就是删除的逆过程了。

1.在${HADOOP_HOME}/etc/hadoop目录下清空exclude，但是exclude文件得保留的，因为yarn-site.xml和hdfs-site.xml里面还有dfs.hosts.exclude和yarn.resourcemanager.nodes.exclude-path，除非删除这两个。

2.启动需要添加的DataNode和NodeManager的进程

```shell
./hadoop-daemon.sh start datanode
./yarn-daemon.sh start nodemanager
```

3.动态更新配置文件

```shell
#更新HDFS的配置文件hdfs-site.xml
hdfs dfsadmin -refreshNodes  
#更新YARN的配置文件yarn-site.xml
yarn rmadmin -refreshNodes
```

4.发现hadoop4的容量没用，因此需要进行均衡

```shell
Name: 192.168.2.104:50010 (hadoop4)
Hostname: hadoop4
Decommission Status : Normal
Configured Capacity: 126567288832 (117.87 GB)
DFS Used: 131607446 (125.51 MB)
Non DFS Used: 10050729066 (9.36 GB)
DFS Remaining: 116384952320 (108.39 GB)
DFS Used%: 0.10%
DFS Remaining%: 91.96%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Mon Jul 13 10:21:08 CST 2015
```

5.进行负载均衡 ```./start-balancer.sh -threshold 1``` 同2.8

## 4. 总结

本文介绍了如何动态的删除和添加hadoop 节点。

本文完




* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/hadoop-delete-datanode
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
