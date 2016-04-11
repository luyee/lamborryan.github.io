---
layout: post
title: Oozie系列(2)之OOZIE运行Hive、Sqoop Action死锁问题分析
date: 2015-12-18 10:30:35
categories: 大数据
tags: Oozie
---

## 1. 问题描述

1.运行Hive或者Sqoop应用出现两个Job，以Sqoop为例，查看job.properties和workflow.xml

job.properties

```shell
nameNode=hdfs://hadoop-master:9000
jobTracker=hadoop-master:8032
queueName=dailyTask
examplesRoot=examples
oozie.use.system.libpath=true
oozie.wf.application.path=${nameNode}/user/rcf/examples/sqoop
```

workflow.xml

```shell
<workflow-app xmlns="uri:oozie:workflow:0.4" name="streaming-wf">
    <start to="streaming-node"/>
    <action name="streaming-node">
        <sqoop  xmlns="uri:oozie:sqoop-action:0.2">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="/user/rcf/sqooptest"/>
            </prepare>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <command>import --connect jdbc:mysql://XXXXX/dt_www --username XXXX --password XXXX --table duotin_users --target-dir /user/rcf/sqooptest -m 3</command>
            <archive>mysql-connector-java-5.1.34.jar</archive>
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

出现的现象是，同时出现oozie-launcher和oozie-action两个job，其中launcher长时间处于running,而action长时间处于accepted。如下图所示:

![img](../image/oozie-1.png)

查看Launcher日志:

```java
=================================================================

  >>> Invoking Sqoop command line now >>>

  2213 [main] WARN  org.apache.sqoop.tool.SqoopTool  - $SQOOP_CONF_DIR has not been set in the environment. Cannot check for additional configuration.
  2236 [main] INFO  org.apache.sqoop.Sqoop  - Running Sqoop version: 1.4.5
  2249 [main] WARN  org.apache.sqoop.tool.BaseSqoopTool  - Setting your password on the command-line is insecure. Consider using -P instead.
  2261 [main] WARN  org.apache.sqoop.ConnFactory  - $SQOOP_CONF_DIR has not been set in the environment. Cannot check for additional configuration.
  2332 [main] INFO  org.apache.sqoop.manager.MySQLManager  - Preparing to use a MySQL streaming resultset.
  2332 [main] INFO  org.apache.sqoop.tool.CodeGenTool  - Beginning code generation
  2590 [main] INFO  org.apache.sqoop.manager.SqlManager  - Executing SQL statement: SELECT t.* FROM `duotin_users` AS t LIMIT 1
  2616 [main] INFO  org.apache.sqoop.manager.SqlManager  - Executing SQL statement: SELECT t.* FROM `duotin_users` AS t LIMIT 1
  2623 [main] INFO  org.apache.sqoop.orm.CompilationManager  - HADOOP_MAPRED_HOME is /home/hadoop/hadoop-2.6.0
  4557 [main] INFO  org.apache.sqoop.orm.CompilationManager  - Writing jar file: /tmp/sqoop-hadoop/compile/a91d2b7338a47e338a8beec83c033cee/duotin_users.jar
  4564 [main] WARN  org.apache.sqoop.manager.MySQLManager  - It looks like you are importing from mysql.
  4564 [main] WARN  org.apache.sqoop.manager.MySQLManager  - This transfer can be faster! Use the --direct
  4564 [main] WARN  org.apache.sqoop.manager.MySQLManager  - option to exercise a MySQL-specific fast path.
  4564 [main] INFO  org.apache.sqoop.manager.MySQLManager  - Setting zero DATETIME behavior to convertToNull (mysql)
  4571 [main] INFO  org.apache.sqoop.mapreduce.ImportJobBase  - Beginning import of duotin_users
  4594 [main] WARN  org.apache.sqoop.mapreduce.JobBase  - SQOOP_HOME is unset. May not be able to find all job dependencies.
  5195 [main] INFO  org.apache.sqoop.mapreduce.db.DBInputFormat  - Using read commited transaction isolation
  5196 [main] INFO  org.apache.sqoop.mapreduce.db.DataDrivenDBInputFormat  - BoundingValsQuery: SELECT MIN(`id`), MAX(`id`) FROM `duotin_users`
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
  Heart beat
```

![img](../image/oozie-2.png)

## 2. 问题定位

看到这个现象我主要有两个反应：

1.是不是Sqoop导MySql时候，Mysql无响应。于是直接用Sqoop命令在Server上运行发现正常，于是排除这个现象。

2.在实践的过程中，如果我kill掉这个oozie workflow, 那么虽然Launcher Job会被Kill掉，但是Action Job还是会继续运行并运行成功，由此可见Aciton Job只是因为没有得到资源，被阻塞了才未运行而已。

![img](../image/oozie-3.png)

3.看到Launcher在Running到100%时候，而Action则一直在等待。由于Launcher与Action是同一个队列的，所以是不是由于Launcher Job启动并等待Aciton Job执行完成，而因为Action和Launcher在同一Queue所以Launcher把Action给阻塞了，因此造成死锁现象。由于我采用的Hadoop Schedule是Capacity Schedule，虽然它支持多个Queue同时执行job，但是不支持相同Queue内的Job并行执行。

同时我又好奇究竟Launcher是一个什么过程，于是找了很多资料，才看到这篇文章：

https://www.altiscale.com/hadoop-blog/oozie-launcher-tips-for-tackling-its-challenges/?utm_source=tuicool

http://stackoverflow.com/questions/27653937/error-on-running-multiple-workflow-in-oozie-4-1-0/27855432#27855432

>Oozie Launcher: Tips for Tackling its Challenges

>In this blog post, we will look at Apache Oozie’s launcher job and ways to avoid some of the common pitfalls associated with it. We hope this provides our readers with a better understanding of Oozie’s execution model, including its subtleties.

>Oozie’s Execution Model: A Different Approach
Oozie’s execution model is different from the default approach users take to run Hadoop jobs. When a user invokes the Hadoop, Hive, or Pig CLI tool from a Hadoop edge node, the corresponding client executable runs on that node which is configured to contact and submit jobs to the Hadoop cluster. When the same jobs are defined and submitted via an Oozie workflow action, things work differently.

>Let’s say you are submitting a workflow job using the Oozie CLI on the edge node. The Oozie client actually submits the workflow to the Oozie server, which typically runs on a different node. Regardless of where it runs, it’s the Oozie server’s responsibility to submit and run the underlying MapReduce jobs on the Hadoop cluster. Oozie doesn’t do so by using the standard client tools installed locally on the Oozie server node. Instead, it first submits a MapReduce job called the “launcher job,” which in turn runs the Hadoop, Hive, or Pig job using the appropriate client APIs.

>The Oozie launcher is basically a map-only job running a single mapper on the Hadoop cluster. This map job knows what to do for the specific action it’s supposed to run and does the appropriate thing by using the libraries for Hadoop, Pig, etc. This will result in other MapReduce jobs being spun up as required. These Oozie jobs are called “asynchronous actions” in Oozie parlance. Oozie doesn’t run these actions in its own server, but kicks them off on the Hadoop cluster using a launcher job. The reason Oozie server “outsources” the launcher to the Hadoop cluster is to protect itself from unexpected workloads and also to isolate user code from its own services. After all, Oozie has access to an awesome distributed system in the form of a Hadoop cluster. Why run workloads on its own machine when the Hadoop cluster can do it and do it very well!

>Running Out of Heap Memory

>This architecture does work seamlessly for most Oozie workflows and the users don’t have to think about it. But, there are cases where it’s important to keep the Oozie launcher in mind. For instance, users sometimes run out of heap memory for the Hadoop, Hive, or Pig CLI tool (not the underlying MapReduce job) on the edge node because of all the local processing these clients do before or after the MapReduce job(s). This is driven by the implementation of the particular query or code and is usually resolved by setting the client-side heap in the environment using HADOOP_HEAPSIZE or PIG_HEAPSIZE as explained here in the Altiscale Documentation. As you can imagine, this doesn’t work for an Oozie workflow job running the same code as an Oozie action because it will now be invoked through the launcher mapper running in a completely different environment. Think of the Oozie launcher as your client tool running on a random Hadoop node. Luckily, Oozie provides a way to configure the Oozie launcher job to suit your needs.

>Since the Oozie launcher is just another MapReduce job, any configuration you can set for any MapReduce job is valid for the launcher. But the most relevant and useful ones are usually the memory and queue setting (mapreduce.map.memory.mb and mapreduce.job.queuename). The way to set these for the launcher in an Oozie workflow action is to prefix “oozie.launcher” to the setting. For example, oozie.launcher.mapreduce.map.memory.mb will control the memory for the launcher mapper itself as opposed to just mapreduce.map.memory.mb which will only influence the memory setting for the underlying MapReduce job that the Hadoop, Hive, or Pig action runs. So, if you have a Hive query which requires you to increase the client side heap size when you submit the query using the Hive CLI, remember to increase the launcher mapper’s memory when you define the Oozie action for it.

>Managing the Queue

>Queue management is another interesting topic when it comes to the Oozie launcher. Do remember that the Oozie launchers will also be taking up map slots on the Hadoop cluster. If you run several workflows in parallel or if your Oozie workflow has multiple parallel execution paths that spawn several launchers, there is scope for a nasty deadlock. The launchers could end up consuming all of the available map slots (in Hadoop 1) or YARN containers (in Hadoop 2) and keep waiting for open slots to schedule the actual MapReduce jobs for the actions they are supposed to run. But the slots may never open up. Since the launchers are all waiting for each other, the cluster freezes up and nothing happens. This is more common than you might imagine, especially on smaller development and test clusters.

>One possible solution to this deadlock is to confine the launchers to a separate Hadoop queue from the actual MapReduce jobs. By limiting the queue capacity and making sure the launcher specific queue doesn’t take up all the slots across the entire cluster, you are able to reserve some capacity for the actual actions. This will ensure progress of your workflows and avoid the deadlock. The relevant setting here is oozie.launcher.mapreduce.job.queuename. Work with your Hadoop administrator to both configure and pick the appropriate queue for the Oozie launchers. Hadoop schedulers and queues are very customizable.

>Handling Unexpected Hive Action Failures

>Another common side effect of the Oozie launcher is “unexpected” failures of Hive actions involving queries that seem to run fine on the edge node. The Hive CLI tends to use local directories as scratch locations for its various needs. This typically happens on the edge node when using the Hive CLI, but this has to be accommodated on the Hadoop cluster node when the Hive action is invoked via the Oozie launcher.

>Hive action in Oozie requires various configuration settings that cover all the important properties required for Hive. A convenient, albeit slightly lazy, approach that we have observed several users adopt to run the Hive action is to copy the hive-site.xml from the edge node as is and drop it in as part of the Oozie workflow. This works most of the time since some of the key settings, like hive.metastore.uris, don’t really change between the edge node and the cluster node.

>But pay close attention to the hive.exec.local.scratchdir property. This is the scratch directory that Hive uses during the execution of the query. Just because a path is valid and has the right permissions for a given user on the edge node, we can’t assume the same directory paths work for that user on the Hadoop cluster nodes. Hive will try to create the directories if it doesn’t exist, but if the user running the action doesn’t have the right permissions, it’s beyond Hive’s control and the action will fail. This failure can be hard to debug and may end up consuming a lot of your valuable time.

>The action and the workflow will be reported as “ERROR” and “KILLED” respectively on the Oozie UI, but the launcher job itself will be declared to have “SUCCEEDED” on the Hadoop ResourceManager UI. This is common with Oozie launchers. When you dig into the launcher logs, you will see an error message and stack similar to the one shown below. Be sure to change the scratch directory in your action configuration or have your Hadoop administrator fix the permissions on the cluster nodes to get past this failure.

>``` Failing Oozie Launcher, Main class [org.apache.oozie.action.hadoop.HiveMain], main() threw exception, Unable to create log directory /home/my_hive_user/scratch java.lang.RuntimeException: Unable to create log directory /home/my_hive_user/scratch at org.apache.hadoop.hive.ql.session.SessionState.createTempFile(SessionState.java:427) at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:328)
```

>These are some of the launcher related issues that we have seen recur. In general, please keep in mind that anything the client tools do locally on the edge node happens via the launcher on a Hadoop cluster node in the Oozie context. Oozie users are mostly familiar with the existence of the launcher job, but often forget its role and relevance while designing particular actions or debugging specific failures. The Oozie launcher is often our forgotten friend and we recommend you never ignore it.


该篇文章提到

Oozie Client 提交job server是通过提交一个MapReduce Job (也就是Launcher Job), 该Job只有一个Map任务。Launcher Job内部会启动N个异步的Action Job，但是他自身却是阻塞等待他们的完成，相当于主线程。因为Launcher 也是个Map-Only的MapReduce，所以必须会占有Container从而对相同Queue的后续Job产生影响。所以建议Launcher设置为单个Queue.那么怎么单独为Launcher设置Queue呢，翻遍资料都没写，直到看到这段代码才明白：


```java
Configuration setupLauncherConf(Configuration conf, Element actionXml,Path appPath, Context context) throws ActionExecutorException {
      try {
          Namespace ns = actionXml.getNamespace();
          Element e = actionXml.getChild("configuration", ns);
          if (e != null) {
                String strConf =XmlUtils.prettyPrint(e).toString();
                XConfiguration inlineConf = newXConfiguration(new StringReader(strConf));

                XConfiguration launcherConf =new XConfiguration();
                for (Map.Entry<String,String> entry : inlineConf) {
                    if(entry.getKey().startsWith("oozie.launcher.")) {
                        String name =entry.getKey().substring("oozie.launcher.".length());
                        String value =entry.getValue();
                        // setting original KEY
                      launcherConf.set(entry.getKey(), value);
                        // setting un-prefixedkey (to allow Hadoop job config
                        // for the launcher job
                        launcherConf.set(name,value);
                    }
                }
                checkForDisallowedProps(launcherConf,"inline launcher configuration");
              XConfiguration.copy(launcherConf, conf);
          }
          return conf;
      }
      catch (IOException ex) {
          throw convertException(ex);
      }
    }
```

![img](../image/oozie-4.png)

最后执行的顺序是

Laucher先启动， 再启动Action， Launcher运行到100%等待Action运行，Action运行完毕后Launcher也运行完毕。

## 3. 总结

本文再次证明了oozie是如何的笨重. 通过本文遇到的问题也理解了oozie是怎么启动任务的。

本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/oozie-action-dead-lock
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
