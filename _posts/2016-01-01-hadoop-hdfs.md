---
layout: post
title: Hadoop系列(1)之HDFS框架
date: 2016-01-01 10:30:35
categories: 大数据
tags: Hadoop
---

## 1.什么是HDFS

1.首先HDFS是一个文件系统，所以他具有文件系统该有的路径目录等概念，也具有文件系统的块的概念，以及相应的文件系统的操作。

hadoop fs 命令

```shell
hadoop fs -ls /
hadoop fs -lsr
hadoop fs -mkdir /user/hadoop
hadoop fs -put a.txt /user/hadoop/
hadoop fs -get /user/hadoop/a.txt /
hadoop fs -cp src dst
hadoop fs -mv src dst
hadoop fs -cat /user/hadoop/a.txt
hadoop fs -rm /user/hadoop/a.txt
hadoop fs -rmr /user/hadoop/a.txt
hadoop fs -text /user/hadoop/a.txt
hadoop fs -copyFromLocal localsrc dst 与hadoop fs -put功能类似。
hadoop fs -moveFromLocal localsrc dst 将本地文件上传到hdfs，同时删除本地文件。
```

2.其次HDFS是一个分布式文件系统，所以他有其他文件系统没有的一些特性。

* 处理超大文件。超大是多大呢？目前已有PB级大小的文件的案例。那么它是靠什么实现处理超大文件的能力呢，我认为主要靠两点：
    * 较大的块，HDFS上默认的块的大小是64M，也就是说文件系统的最小的单位就是64M，而且这个值是可以增加，我们可以想象下，1G大小的文件只要20个块单元都不用,这样大大降低了寻址的难度，总之增加块的大小有不少好处。HDFS会将数据块按键值对存放在HDFS上，并映射到内存中，所以当有大量小文件产生时候就会对内存造成影响，这个时候就要用一些特殊的手段进行优化。
    * 分布式存储，你一台服务器再怎么大，总存不下PB级的文件吧，所以N多台服务器分布式存储就提供了条件。
* 流式的访问数据。HDFS的一个基本思想是一次写入，多次读取的模式是效率最高的，而Hadoop是以批处理为目标设计的，所以他牺牲了实时性换来了高吞吐能力。
* 商用硬件。构建HDFS的服务器只需要一般的服务器就行，他的观点是“人多力量大”。但是一旦使用商用硬件，那么硬件的故障概率就会特别高，所以HDFS的容错能力就需要很高，在HDFS存在数据的备份，默认为3分，这个跟Raid有点相似。

## 2.HDFS框架

### 2.1 框架图

![img](../image/hadoop/hdfs-1.png)

### 2.2 HDFS的组成

#### 2.2.1 NameNode

NameNode是Master节点，他管理整个文件系统的命名空间，维护文件系统的树以及树内的文件和索引目录，所以如果你要知道某个目录下有啥文件，这个文件有啥属性，你只要去问他就行了。以上这些信息是存放在Master这台服务器的本地磁盘中，命名空间镜像fsimage和编辑日志edits。除了命名空间外，NameNode还存放了每个文件的每个块存放的DataNode节点，这个信息是每次系统启动时候由NameNode重建的，因此并不是永久存放的。

![img](../image/hadoop/hdfs-2.png)

#### 2.2.2 SecondaryNameNode

咋一看以为SecondaryNameNode是NameNode的一个备胎，即同步NameNode，当NameNode挂的时候当NameNode用。当然这个是他的一个功能，还有一个功能是周期性的将NameNode的命名空间镜像文件和修改日志合并，防止日志文件过大。

#### 2.2.3 DataNode

终于轮到DataNode，前面可知NameNode是大哥，那么DataNode就是小弟了，他就负责干活了，包括文件的存储，以及文件块的读写。

### 2.3 HDFS的文件存储与复制

1.HDFS被设计成一个在大集群中跨机器可靠的存储超大文件。它将每个文件存储成一系列的数据块，除最后一个外，其他的块大小相等。为了容错，文件的所有块都会有备份，默认情况下为3备份。NameNode周期性的从集群中的每个DateNode接受心跳信号，和块状态报告。接受心跳信号意味着DataNode运行正常，块状态报告包含了一个该DataNode所有数据的列表。

2.HDFS中DataNode的文件是存放在本地文件系统上。

```shell
${dfs.data.dir}/current/VERSION
                       /blk_<id_1>
                       /blk_<id_1>.meta
                       /blk_<id_1>
                       /blk_<id_1>.meta
                       /...
                       /blk_<id_64>
                       /blk_<id_64>.meta
                       /subdir0/
                       /subdir1/
                       /...
                       /subdir63/
               /previous/
               /detach/
               /tmp/
               /in_use.lock
               /storage
```

3.副本的存放策略。HDFS的存放策略是将一个副本存放在本地机架的节点上，一个副本存放在同一机架的不同节点上，另一个副本存放在不同机架的节点上。由于不同机架同时出错的概率还是很低的，所以这种策略安全系数较高。

4.副本选择，为了降低整体的带宽和读写延时，尽量读取程序最近的副本。

5.心跳检测和重新复制。如果NameNode没有接受到某个DataNode的心跳信号，则认为该DataNode宕机，那么就由可能引起一些文件数据块的备份数减少，NameNode检测到这些需要被复制的数据块后就会启动复制。

6.集群均衡。HDFS支持集群均衡，当某个Datanode的节点上的空闲空间低于特定的临界点时候，HDFS就会按照均衡策略自动从数据这个DataNode移动其他空闲的DataNode上。最好的一个例子就是前几天我对Hadoop集群进行扩容。

> 关于进行balance，需要设置dfs.balance.bandwidthPerSec  默认设置：1048576（1 M/S），参数含义：设置balance工具在运行中所能占用的带宽。
hadoop balance -threshold 1
-threshold 默认设置：10，参数取值范围：0-100，参数含义：判断集群是否平衡的目标参数，每一个 datanode 存储使用率和集群总存储使用率的差值都应该小于这个阀值 ，理论上，该参数设置的越小，整个集群就越平衡，但是在线上环境中，hadoop集群在进行balance时，还在并发的进行数据的写入和删除，所以有可能无法到达设定的平衡参数值。

7.数据完整性。HDFS客户端软件实现了对HDFS文件内容的校验和(checksum)检查。当客户端创建一个新的HDFS文件，会计算这个文件每个数据块的校验和，并将校验和作为一个单独的隐藏文件保存在同一个HDFS名字空间下。当客户端获取文件内容后，它会检验从Datanode获取的数据跟相应的校验和文件中的校验和是否匹配，如果不匹配，客户端可以选择从其他Datanode获取该数据块的副本。

8.staging。客户端创建文件的请求其实并没有立即发送给Namenode，事实上，在刚开始阶段HDFS客户端会先将文件数据缓存到本地的一个临时文件。应用程序的写操作被透明地重定向到这个临时文件。当这个临时文件累积的数据量超过一个数据块的大小，客户端才会联系Namenode。Namenode将文件名插入文件系统的层次结构中，并且分配一个数据块给它。然后返回Datanode的标识符和目标数据块给客户端。接着客户端将这块数据从本地临时文件上传到指定的Datanode上。当文件关闭时，在临时文件中剩余的没有上传的数据也会传输到指定的Datanode上。然后客户端告诉Namenode文件已经关闭。此时Namenode才将文件创建操作提交到日志里进行存储。如果Namenode在文件关闭前宕机了，则该文件将丢失。上述方法是对在HDFS上运行的目标应用进行认真考虑后得到的结果。这些应用需要进行文件的流式写入。如果不采用客户端缓存，由于网络速度和网络堵塞会对吞估量造成比较大的影响。这种方法并不是没有先例的，早期的文件系统，比如AFS，就用客户端缓存来提高性能。为了达到更高的数据上传效率，已经放松了POSIX标准的要求。

### 2.4 HDFS的存储空间及回收

* 文件的删除和恢复。当用户或应用程序删除某个文件时，这个文件并没有立刻从HDFS中删除。实际上，HDFS会将这个文件重命名转移到/trash目录。只要文件还在/trash目录中，该文件就可以被迅速地恢复。文件在/trash中保存的时间是可配置的，当超过这个时间时，Namenode就会将该文件从名字空间中删除。删除文件会使得该文件相关的数据块被释放。注意，从用户删除文件到HDFS空闲空间的增加之间会有一定时间的延迟。只要被删除的文件还在/trash目录中，用户就可以恢复这个文件。如果用户想恢复被删除的文件，他/她可以浏览/trash目录找回该文件。/trash目录仅仅保存被删除文件的最后副本。/trash目录与其他的目录没有什么区别，除了一点：在该目录上HDFS会应用一个特殊策略来自动删除文件。目前的默认策略是删除/trash中保留时间超过6小时的文件。将来，这个策略可以通过一个被良好定义的接口配置。
* 减少副本系数。当一个文件的副本系数被减小后，Namenode会选择过剩的副本删除。下次心跳检测时会将该信息传递给Datanode。Datanode遂即移除相应的数据块，集群中的空闲空间加大。同样，在调用setReplication API结束和集群中空闲空间增加间会有一定的延迟。

## 3. HDFS读写过程

### 3.1 HDFS读过程

![img](../image/hadoop/hdfs-3.png)

1. HDFS Client向远程的NameNode发起RPC请求；
2. NameNode会视情况返回文件的部分或者全部Block列表，对于每个Block，NameNode都会返回该Block的DataNode地址，比如Block1：host2，host3，host5；Block2：host1，host4，host3.
3. Client会选取离Client最接近的DataNode来读取Block；如果Client本身就是DataNode，那么将从本地直接读取数据。
4. Client读完当前Block的数据后，关闭与当前的DataNode链接，并为读取下一个Block寻找最佳的DataNode；
5. 当读完列表的Block后，且文件读取还没结束，Client会继续向NameNode获取下一批的Block列表。
6. 读取时会对一个Block都会进行CheckSum验证，如果读取DataNode时出现错误，Client会通知NameNode，然后再从下一个拥有该Block备份数据的DataNode继续读。

### 3.2 HDFS写过程

![img](../image/hadoop/hdfs-4.png)

1. HDFS Client 向NameNode发送RPC请求，请求写数据；
2. NameNode会检查创建的文件是否已经存在，创建者是否有权限进行操作，成功则会为文件创建一个记录，否则会让客户端抛出异常；

    > 【注】客户端创建文件的请求其实并没有立即发送给Namenode，事实上，在刚开始阶段HDFS客户端会先将文件数据缓存到本地的一个临时文件。应用程序的写操作被透明地重定向到这个临时文件。当这个临时文件累积的数据量超过一个数据块的大小，客户端才会联系Namenode。Namenode将文件名插入文件系统的层次结构中，并且分配一个数据块给它。然后返回Datanode的标识符和目标数据块给客户端。接着客户端将这块数据从本地临时文件上传到指定的Datanode上。当文件关闭时，在临时文件中剩余的没有上传的数据也会传输到指定的Datanode上。然后客户端告诉Namenode文件已经关闭。此时Namenode才将文件创建操作提交到日志里进行存储。如果Namenode在文件关闭前宕机了，则该文件将丢失。

3. Client开始写文件时候，先会将文件按64k进行切分成多个Packets，并在内部以数据队列“data queue”的形式管理这些Packets，并向NameNode申请新的Blocks，NameNode创建并记录Block信息后，会返回可用的DataNode，如Block1：host2，host1，host3；Block2：host7,host8，host4；DataNode的host配置策略跟读过程一样
4. Client以PipeLine(管道)的形式将Packet写入所有的Replicas中，Client把Packet以流的方式写入第一个DataNode，该DataNode把该Packet存储后，再传递给PipeLine的下一个DataNode，直到最后一个DataNode，这种写数据的方式呈流水线的形式。
5. 最后一个DataNode成功存储之后会返回一个Ack Packet，通过PipeLine传递至Client，在Client维护的Ack Queue在成功收到DataNode返回的AckPacket后会从Ack queue移除相应的Packet。
6. 如果传输过程中，有某个DataNode出现故障，那么当前的PipeLine会关闭，出现故障的DataNode会从PipeLine中移除，剩余的Block会继续剩下的DataNode中继续以PipleLine形式传输，同时NameNode会分配一个新的DataNode，保持Replicas设定的数量。

## 4. 总结

本文主要介绍分布式文件系统HDFS的基本框架以及读写原理, 希望通过本文能让你对hdfs有个基本的印象。


本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/hadoop-hdfs
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
