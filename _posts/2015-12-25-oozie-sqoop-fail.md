---
layout: post
title: Oozie系列(3)之解决Sqoop Job无法运行的问题
date: 2015-12-25 10:30:35
categories: 大数据
tags: Oozie Sqoop
---

## 1. 问题描述

1. 当我在OOZIE的Workflow上运行列如import --connect jdbc:mysql://XXXXX/dt_www --username XXXX --password XXXX --table duotin_users --target-dir /user/rcf/sqooptest -m 3 command运行成功
2. 当我在OOZIE的Workflow上运行如Job --exec sub-job-users缺没成功，表现为只启动Launcher的Job，而没启动Action的Job，错误日志为:Main class [org.apache.oozie.action.hadoop.SqoopMain], exit code [1]

Workflow.xml如下：

```xml
<workflow-app xmlns="uri:oozie:workflow:0.4" name="streaming-wf">
    <start to="sqoop-node"/>
    <action name="sqoop-node">
        <sqoop  xmlns="uri:oozie:sqoop-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
                <property>
                    <name>oozie.launcher.mapred.job.queue.name</name>
                    <value>hive</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionURL</name>
                    <value>jdbc:mysql://xxxx:3306/hive_meta?createDatabaseIfNotExist=true</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionDriverName</name>
                    <value>com.mysql.jdbc.Driver</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionUserName</name>
                    <value>xxxx</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionPassword</name>
                    <value>xxxx</value>
                </property>
                <property>
                    <name>hive.metastore.warehouse.dir</name>
                    <value>/user/hive/warehouse</value>
                </property>
            </configuration>
            <command>job --exec sq-job-users</command>
            <file>hive-site.xml#hive-site.xml</file>
            <archive>mysql-connector-java-5.1.34.jar#mysql-connector-java-5.1.34.jar</archive>
        </sqoop>
        <ok to="end"/>
        <error to="fail"/>
    </action>
    <kill name="fail">
        <message>Streaming Map/Reduce failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    <end name="end"/>
</workflow-app>
```

查看MapReduce日志发现:

```java
2254 [main] ERROR org.apache.sqoop.tool.JobTool  - I/O error performing job operation: java.io.IOException: Cannot restore missing job sq-job-users
	at org.apache.sqoop.metastore.hsqldb.HsqldbJobStorage.read(HsqldbJobStorage.java:256)
	at org.apache.sqoop.tool.JobTool.execJob(JobTool.java:198)
	at org.apache.sqoop.tool.JobTool.run(JobTool.java:283)
	at org.apache.sqoop.Sqoop.run(Sqoop.java:143)
	at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
	at org.apache.sqoop.Sqoop.runSqoop(Sqoop.java:179)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:218)
	at org.apache.sqoop.Sqoop.runTool(Sqoop.java:227)
	at org.apache.sqoop.Sqoop.main(Sqoop.java:236)
	at org.apache.oozie.action.hadoop.SqoopMain.runSqoopJob(SqoopMain.java:206)
	at org.apache.oozie.action.hadoop.SqoopMain.run(SqoopMain.java:174)
	at org.apache.oozie.action.hadoop.LauncherMain.run(LauncherMain.java:39)
	at org.apache.oozie.action.hadoop.SqoopMain.main(SqoopMain.java:45)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at org.apache.oozie.action.hadoop.LauncherMapper.map(LauncherMapper.java:226)
	at org.apache.hadoop.mapred.MapRunner.run(MapRunner.java:54)
	at org.apache.hadoop.mapred.MapTask.runOldMapper(MapTask.java:450)
	at org.apache.hadoop.mapred.MapTask.run(MapTask.java:343)
	at org.apache.hadoop.mapred.YarnChild$2.run(YarnChild.java:163)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:415)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1628)
	at org.apache.hadoop.mapred.YarnChild.main(YarnChild.java:158)
Intercepting System.exit(1)
<<< Invocation of Main class completed <<<
Failing Oozie Launcher, Main class [org.apache.oozie.action.hadoop.SqoopMain], exit code [1]
```

## 2. 问题定位

从日志及现象上发现如下几个特点：

1. Launcher Job启动，而Action Job没启动，而前面讲过Action Job是由Launcher Job启动的，所以由此可见问题是出现在Launcer Job中。
2. 日志提示是没有发现sq-job-users，可是我通过Sqoop job -list 发现该job明明是存在的，那么问题可能也出现在OOZIE的机制当中，前文讲过，OOZIE client调度sqoop是用一个MapReduce程序调度，在Hadoop集群随机选择一个Node运行sqoop job，所以我们完全不知道Sqoop Job是运行在namenode上还是datanode上，Sqoop是部署在NameNode上的，因为在其他node的本地metastore里面没有相关job的信息，job的信息只有namenode才有所以, 我们在DataNode上运行sqoop job --exec sq-job-users，当然无法运行了。
3. 查看了Sqoop的资料，发现其实Sqoop是支持Client调用Server执行Sqoop相关命令，Sqoop的metastore默认是存储在$HOME/.sqoop/下的，是存储在本地的解决方案，当如果要在符合在OOZIE上运行，就必须使用

share metastore。即将元数据信息存储在hsqldb关系型数据库中的。

## 3. 解决方案

1.修改sqoop-site.xml

```xml
<property>
    <name>sqoop.metastore.server.location</name>
    <value>/tmp/sqoop-metastore/shared.db</value>
    <description>Path to the shared metastore database files.
    If this is not set, it will be placed in ~/.sqoop/.
    </description>
  </property>
  <property>
    <name>sqoop.metastore.server.port</name>
    <value>16000</value>
    <description>Port that this metastore should listen on.
    </description>
  </property>
```

2.启动share metastore service

```shell
hadoop@hadoop-master:~/sqoop$ sqoop metastore > /home/hadoop/sqoop/metastore.log 2>&1 &
15/08/06 16:06:39 INFO sqoop.Sqoop: Running Sqoop version: 1.4.5
[Server@88744e5]: [Thread[main,5,main]]: checkRunning(false) entered
[Server@88744e5]: [Thread[main,5,main]]: checkRunning(false) exited
[Server@88744e5]: [Thread[main,5,main]]: setDatabasePath(0,file:/tmp/sqoop-metastore/shared.db)
[Server@88744e5]: [Thread[main,5,main]]: checkRunning(false) entered
[Server@88744e5]: [Thread[main,5,main]]: checkRunning(false) exited
[Server@88744e5]: [Thread[main,5,main]]: setDatabaseName(0,sqoop)
[Server@88744e5]: [Thread[main,5,main]]: putPropertiesFromString(): [hsqldb.write_delay=false]
[Server@88744e5]: [Thread[main,5,main]]: checkRunning(false) entered
[Server@88744e5]: [Thread[main,5,main]]: checkRunning(false) exited
[Server@88744e5]: Initiating startup sequence...
[Server@88744e5]: Server socket opened successfully in 10 ms.
[Server@88744e5]: Database [index=0, id=0, db=file:/tmp/sqoop-metastore/shared.db, alias=sqoop] opened sucessfully in 251 ms.
[Server@88744e5]: Startup sequence completed in 263 ms.
[Server@88744e5]: 2015-08-06 16:06:39.365 HSQLDB server 1.8.0 is online
[Server@88744e5]: To close normally, connect and execute SHUTDOWN SQL
[Server@88744e5]: From command line, use [Ctrl]+[C] to abort abruptly
15/08/06 16:06:39 INFO hsqldb.HsqldbMetaStore: Server started on port 16000 with protocol HSQL
```

3.创建Job

```shell
sqoop job
--meta-connect jdbc:hsqldb:hsql://hadoop-master:16000/sqoop
--create sq-job-users
--import
--connect jdbc:mysql://113.200.251.200/dt_www
--username aaa --password bbb
--table duotin_users
--hive-table data_warehouse.duotin_users
--hive-import
--hive-overwrite
-outdir /home/hadoop/statistics/outdir
--fields-terminated-by '\001'
-m 6
```

重点还是在--meta-connect jdbc:hsqldb:hsql://hadoop-master:16000/sqoop

4.验证Job，后续的sqoop操作都需要跟上--meta-connect jdbc:hsqldb:hsql://hadoop-master:16000/sqoop，比如

```shell
hadoop@hadoop-master:~/sqoop$ sqoop job -list --meta-connect jdbc:hsqldb:hsql://hadoop-master:16000/sqoop
Warning: /home/hadoop/sqoop/../hcatalog does not exist! HCatalog jobs will fail.
Please set $HCAT_HOME to the root of your HCatalog installation.
Warning: /home/hadoop/sqoop/../accumulo does not exist! Accumulo imports will fail.
Please set $ACCUMULO_HOME to the root of your Accumulo installation.
15/08/06 16:09:57 INFO sqoop.Sqoop: Running Sqoop version: 1.4.5
Available jobs:
  sq-job-users
hadoop@hadoop-master:~/sqoop$ sqoop job -show sq-job-users --meta-connect jdbc:hsqldb:hsql://hadoop-master:16000/sqoop
Warning: /home/hadoop/sqoop/../hcatalog does not exist! HCatalog jobs will fail.
Please set $HCAT_HOME to the root of your HCatalog installation.
Warning: /home/hadoop/sqoop/../accumulo does not exist! Accumulo imports will fail.
Please set $ACCUMULO_HOME to the root of your Accumulo installation.
15/08/06 16:10:15 INFO sqoop.Sqoop: Running Sqoop version: 1.4.5
Job: sq-job-users
Tool: import
Options:
----------------------------
verbose = false
db.connect.string = jdbc:mysql://XXXX/dt_www
codegen.output.delimiters.escape = 0
codegen.output.delimiters.enclose.required = false
codegen.input.delimiters.field = 0
hbase.create.table = false
db.require.password = false
hdfs.append.dir = false
db.table = duotin_users
codegen.input.delimiters.escape = 0
import.fetch.size = null
accumulo.create.table = false
db.password = XXXX
codegen.input.delimiters.enclose.required = false
db.username = XXXX
codegen.output.delimiters.record = 10
import.max.inline.lob.size = 16777216
hbase.bulk.load.enabled = false
hcatalog.create.table = false
db.clear.staging.table = false
codegen.input.delimiters.record = 0
enable.compression = false
hive.overwrite.table = true
hive.import = true
codegen.input.delimiters.enclose = 0
hive.table.name = data_warehouse.duotin_users
accumulo.batch.size = 10240000
hive.drop.delims = false
codegen.output.delimiters.enclose = 0
hdfs.delete-target.dir = false
codegen.output.dir = /home/hadoop/statistics/outdir
codegen.auto.compile.dir = true
relaxed.isolation = false
mapreduce.num.mappers = 6
accumulo.max.latency = 5000
import.direct.split.size = 0
codegen.output.delimiters.field = 1
export.new.update = UpdateOnly
incremental.mode = None
hdfs.file.format = TextFile
codegen.compile.dir = /tmp/sqoop-hadoop/compile/947edeac967c9927366a5488c2852ad3
direct.import = false
hive.fail.table.exists = false
db.batch = false
```

5.编写workflow.xml

```xml
<workflow-app xmlns="uri:oozie:workflow:0.4" name="streaming-wf">
    <start to="sqoop-node"/>
    <action name="sqoop-node">
        <sqoop  xmlns="uri:oozie:sqoop-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
                <property>
                    <name>oozie.launcher.mapred.job.queue.name</name>
                    <value>hive</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionURL</name>
                    <value>jdbc:mysql://xxxx/hive_meta?createDatabaseIfNotExist=true</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionDriverName</name>
                    <value>com.mysql.jdbc.Driver</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionUserName</name>
                    <value>xxxx</value>
                </property>
                <property>
                    <name>javax.jdo.option.ConnectionPassword</name>
                    <value>xxxx</value>
                </property>
                <property>
                    <name>hive.metastore.warehouse.dir</name>
                    <value>/user/hive/warehouse</value>
                </property>
            </configuration>
            <command>job --exec sq-job-users --meta-connect jdbc:hsqldb:hsql://hadoop-master:16000/sqoop</command>
            <file>hive-site.xml#hive-site.xml</file>
            <archive>mysql-connector-java-5.1.34.jar#mysql-connector-java-5.1.34.jar</archive>
        </sqoop>
        <ok to="end"/>
        <error to="fail"/>
    </action>
    <kill name="fail">
        <message>Streaming Map/Reduce failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    <end name="end"/>
</workflow-app>
```

## 4. 总结

造成该问题的根本原因是Oozie启动任务是以MapReduce形式, 而Yarn会随便指定某一个节点作为MapReduce App Master, 而之前的设置情况下SQOOP只能以本地的形式启动, 所以造成比如SQOOP安装在Data1节点上, Oozie却指定Data2节点为Map App Master去启动Sqoop, 在Sqoop本地模式下当然失败了. 所以我后来将Sqoop以service形式启动, 这样Data2就相当于是client, 而实际上Sqoop任务是在Data1上启动的。

这种情况不单单会在sqoop上出现, 如果以后使用oozie 场景较多的话应该是个较常见的问题。

本文完




* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/oozie-sqoop-fail
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
