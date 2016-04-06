---
layout: post
title: Spark学习(1)之Mac安装Spark
date: 2015-11-03 00:29:30
categories: 大数据
tags: Spark
---
#Spark学习(1)之Mac安装Spark

##简介
本系列文章将记录我学习以及工作中使用Spark遇到的问题以及学习心得。本文是该系列的第一篇, 主要介绍在自己的Mac上安装Spark以及在intelij idea运行demo遇到的问题.

##安装

单机伪分布集群就是在一台机器上同时运行一个worker和一个master, 我在Mac上安装单机伪分布的Spark集群时候遇到了以下几个问题:

1. ssh连接失败

现象: 使用sh start-all.sh 启动worker和master时候出现ssh connection refused的报错信息。
解决方法:

{% highlight bash linenos %}
sudo launchctl list | grep ssh  #查看ssh启动没, 若启动会打印 -	0	com.openssh.sshd
sudo launchctl load -w /System/Library/LaunchDaemons/ssh.plist #启动ssh
sudo launchctl list | grep ssh
ssh localhost  #测试链接
{% endhighlight bash %}

2. 登入localhost:8080后，发现alive worker为0, woker不能链接master

查看woker日志:
{% highlight bash linenos %}
15/11/02 22:30:40 WARN Worker: Failed to connect to master admindeMacBook-Pro-5.local:7077
akka.actor.ActorNotFound: Actor not found for: ActorSelection[Anchor(akka.tcp://sparkMaster@admindeMacBook-Pro-5.local:7077/), Path(/user/Master)]
	at akka.actor.ActorSelection$$anonfun$resolveOne$1.apply(ActorSelection.scala:65)
	at akka.actor.ActorSelection$$anonfun$resolveOne$1.apply(ActorSelection.scala:63)
	at scala.concurrent.impl.CallbackRunnable.run(Promise.scala:32)
	at akka.dispatch.BatchingExecutor$AbstractBatch.processBatch(BatchingExecutor.scala:55)
	at akka.dispatch.BatchingExecutor$Batch.run(BatchingExecutor.scala:73)
	at akka.dispatch.ExecutionContexts$sameThreadExecutionContext$.unbatchedExecute(Future.scala:74)
	at akka.dispatch.BatchingExecutor$class.execute(BatchingExecutor.scala:120)
	at akka.dispatch.ExecutionContexts$sameThreadExecutionContext$.execute(Future.scala:73)
	at scala.concurrent.impl.CallbackRunnable.executeWithValue(Promise.scala:40)
	at scala.concurrent.impl.Promise$DefaultPromise.tryComplete(Promise.scala:248)
	at akka.pattern.PromiseActorRef.$bang(AskSupport.scala:266)
	at akka.actor.EmptyLocalActorRef.specialHandle(ActorRef.scala:533)
	at akka.actor.DeadLetterActorRef.specialHandle(ActorRef.scala:569)
	at akka.actor.DeadLetterActorRef.$bang(ActorRef.scala:559)
	at akka.remote.RemoteActorRefProvider$RemoteDeadLetterActorRef.$bang(RemoteActorRefProvider.scala:87)
	at akka.remote.EndpointWriter.postStop(Endpoint.scala:557)
	at akka.actor.Actor$class.aroundPostStop(Actor.scala:477)
	at akka.remote.EndpointActor.aroundPostStop(Endpoint.scala:411)
	at akka.actor.dungeon.FaultHandling$class.akka$actor$dungeon$FaultHandling$$finishTerminate(FaultHandling.scala:210)
	at akka.actor.dungeon.FaultHandling$class.terminate(FaultHandling.scala:172)
	at akka.actor.ActorCell.terminate(ActorCell.scala:369)
	at akka.actor.ActorCell.invokeAll$1(ActorCell.scala:462)
	at akka.actor.ActorCell.systemInvoke(ActorCell.scala:478)
	at akka.dispatch.Mailbox.processAllSystemMessages(Mailbox.scala:263)
	at akka.dispatch.Mailbox.run(Mailbox.scala:219)
	at akka.dispatch.ForkJoinExecutorConfigurator$AkkaForkJoinTask.exec(AbstractDispatcher.scala:397)
	at scala.concurrent.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at scala.concurrent.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at scala.concurrent.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at scala.concurrent.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)
15/11/02 22:31:33 ERROR Worker: All masters are unresponsive! Giving up.
{% endhighlight bash %}

解决方法, 在排除了防火墙打开这个原因后，发现日志中worker和master打印的hostname不一致。于是修改hostname
sudo scutil --set HostName rcf

## Intellij 运行demo

本节主要介绍Intellij运行demo碰到的问题。

1. 首先在Intellij中新建scala工程(需要安装scala插件)，复制以下计算pi的代码到工程中。
{% highlight java linenos %}
import scala.math.random

import org.apache.spark._

/** Computes an approximation to pi */
object SparkPi {
    def main(args: Array[String]) {
        val conf = new SparkConf().setAppName("Spark Pi")
        val spark = new SparkContext(conf)
        val slices = if (args.length > 0) args(0).toInt else 2
        val n = math.min(100000L * slices, Int.MaxValue).toInt // avoid overflow
        val count = spark.parallelize(1 until n, slices).map { i =>
                val x = random * 2 - 1
                val y = random * 2 - 1
                if (x*x + y*y < 1) 1 else 0
            }.reduce(_ + _)
        println("Pi is roughly " + 4.0 * count / n)
        spark.stop()
    }
}
// scalastyle:on println
{% endhighlight java %}

2. 打开[Run... －> Editor Configuration -> Application SparkTest] 在 VM Options内输出-Dspark.master=local, 表示该demo是以本地单线程模式运行, 在program argument内输入10.
3. 运行[Run]

{% highlight java linenos %}
Exception in thread "main" java.lang.NoSuchMethodError: scala.collection.immutable.HashSet$.empty()Lscala/collection/immutable/HashSet;
	at akka.actor.ActorCell$.<init>(ActorCell.scala:336)
	at akka.actor.ActorCell$.<clinit>(ActorCell.scala)
	at akka.actor.RootActorPath.$div(ActorPath.scala:185)
	at akka.actor.LocalActorRefProvider.<init>(ActorRefProvider.scala:465)
	at akka.remote.RemoteActorRefProvider.<init>(RemoteActorRefProvider.scala:124)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
	at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
	at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
	at java.lang.reflect.Constructor.newInstance(Constructor.java:408)
	at akka.actor.ReflectiveDynamicAccess$$anonfun$createInstanceFor$2.apply(DynamicAccess.scala:78)
	at scala.util.Try$.apply(Try.scala:192)
	at akka.actor.ReflectiveDynamicAccess.createInstanceFor(DynamicAccess.scala:73)
	at akka.actor.ReflectiveDynamicAccess$$anonfun$createInstanceFor$3.apply(DynamicAccess.scala:84)
	at akka.actor.ReflectiveDynamicAccess$$anonfun$createInstanceFor$3.apply(DynamicAccess.scala:84)
	at scala.util.Success.flatMap(Try.scala:231)
	at akka.actor.ReflectiveDynamicAccess.createInstanceFor(DynamicAccess.scala:84)
	at akka.actor.ActorSystemImpl.liftedTree1$1(ActorSystem.scala:585)
	at akka.actor.ActorSystemImpl.<init>(ActorSystem.scala:578)
	at akka.actor.ActorSystem$.apply(ActorSystem.scala:142)
	at akka.actor.ActorSystem$.apply(ActorSystem.scala:119)
	at org.apache.spark.util.AkkaUtils$.org$apache$spark$util$AkkaUtils$$doCreateActorSystem(AkkaUtils.scala:121)
	at org.apache.spark.util.AkkaUtils$$anonfun$1.apply(AkkaUtils.scala:53)
	at org.apache.spark.util.AkkaUtils$$anonfun$1.apply(AkkaUtils.scala:52)
	at org.apache.spark.util.Utils$$anonfun$startServiceOnPort$1.apply$mcVI$sp(Utils.scala:1913)
	at scala.collection.immutable.Range.foreach$mVc$sp(Range.scala:166)
	at org.apache.spark.util.Utils$.startServiceOnPort(Utils.scala:1904)
	at org.apache.spark.util.AkkaUtils$.createActorSystem(AkkaUtils.scala:55)
	at org.apache.spark.rpc.akka.AkkaRpcEnvFactory.create(AkkaRpcEnv.scala:253)
	at org.apache.spark.rpc.RpcEnv$.create(RpcEnv.scala:53)
	at org.apache.spark.SparkEnv$.create(SparkEnv.scala:252)
	at org.apache.spark.SparkEnv$.createDriverEnv(SparkEnv.scala:193)
	at org.apache.spark.SparkContext.createSparkEnv(SparkContext.scala:276)
	at org.apache.spark.SparkContext.<init>(SparkContext.scala:441)
	at SparkPi$.main(sparkPi.scala:9)
	at SparkPi.main(sparkPi.scala)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:483)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:140)
{% endhighlight java %}

这是由于spark和scala版本不兼容造成的。 当前用的spark是1.5.0, 而scala版本是2.11.7。

所以解决方法是将scala的版本降为2.10.4。

## Spark的部署方式

目前最为常用的Spark运行模式有:

1. local:本地线程方式运行，主要用于开发调试Spark应用程序. 提交时用--master local
2.  Standalone:利用Spark自带的资源管理与调度器运行Spark集群，采用Master/Slave结构，为解决单点故障，可以采用ZooKeeper实现高可靠（High Availability，HA)。 提交时 --master spark://HOST:PORT
3. Apache Mesos:运行在著名的Mesos资源管理框架基础之上，该集群运行模式将资源管理交给Mesos，Spark只负责进行任务调度和计算。提交时 --master mesos://HOST:PORT
4. Hadoop YARN: 集群运行在Yarn资源管理器上，资源管理交给Yarn，Spark只负责进行任务调度和计算。YARN分为yarn-client和yarn-cluster, 分别用 --master yarn-client 和 --master yarn-cluster。两者的区别是driver运行在本地还是cluster上。

Spark运行模式中Hadoop YARN的集群运行方式最为常用.

本文完。


* 原创文章，转载请注明： 转载自[Lamborryan](<lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/spark-mac-install
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
