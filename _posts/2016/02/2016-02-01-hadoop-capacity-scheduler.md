---
layout: post
title: Hadoop系列(5)之容量调度器Capacity Scheduler配置
date: 2016-02-01 10:30:35
categories: 大数据
tags: Hadoop
---

## 1. 应用场景

本文只关注配置，关于调度器的算法以及核心内容将在下一篇介绍。
Capacity Scheduler是YARN中默认的资源调度器，但是在默认情况下只有root.default 一个queue。而当不同用户提交任务时，任务都会在这个队里里面按优先级先进先出，大大影响了多用户的资源使用率。现在公司的任务主要分为三种：

* 每天晚上进行的日常任务dailyTask，这些任务需要在尽可能短的时间内完成，且由于关乎业务，所以一旦启动就必须提供尽可能多的资源，因为我分配的资源也最多 70%，队列名位dailyTask。
* 白天，通过Hive来获取一些数据信息，这部分任务由其他的同事进行操作，对查询速度要求并不是很高，且断断续续的，我分配的中等20%，队列名hive。
* 最后这种我定义为是其他的类型，长时间的一些数据处理任务，分配资源少一些10%，队列名为default(这个队列一定要存在)。

需要说明的是以下几点：

* 单个队列比如dailyTask，如果有多个任务在队列里面，那么只能有一个任务在运行，其他任务在等待。
* 虽然我设置了队列的资源容量capacity,但是由于资源是共享式的，所以如果有其他queue的资源处理空闲，那么该queue就会从空闲的queue借资源，但是该queue资源最多不会超过 maximum-capacity。所以dailyTask由于在晚上时候运行，其他queue都是空闲的，所以它的实际资源利用率基本上是 maximum-capacity，也就是100%。而白天只有Hive队列和Default队列在使用，所以他们两会共享dailyTask的空余的70%资源。
* 如果在HIVE队列和Default队列都在运行时候，突然dailyTask队列add进了任务了，dailyTask就会等待借出去的资源被回收再分配。
* 目前这样的配置，刚好符合现在的需求，相比于公平调度器较为复杂的配置，目前容量调度器无疑更合适，所以先选择容量调度器。

## 2. 配置

1.首先在yarn-site.xml中配置调度器类型

```xml
<property>
	<name>yarn.resourcemanager.scheduler.class</name>
	<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

2.在Capacity Scheduler专属配置文件capacity-scheduler.xml，需要说明默认存在root的queue，其他的队列都是它的子队列。

* yarn.scheduler.capacity.root.default.maximum-capacity,队列最大可使用的资源率。
* capacity：队列的资源容量（百分比）。 当系统非常繁忙时，应保证每个队列的容量得到满足，而如果每个队列应用程序较少，可将剩余资源共享给其他队列。注意，所有队列的容量之和应小于100。    
* maximum-capacity：队列的资源使用上限（百分比）。由于存在资源共享，因此一个队列使用的资源量可能超过其容量，而最多使用资源量可通过该参数限制。
* user-limit-factor：单个用户最多可以使用的资源因子，默认情况为1，表示单个用户最多可以使用队列的容量不管集群有空闲，如果该值设为5，表示这个用户最多可以使用5*capacity的容量。实际上单个用户的使用资源为 min(user-limit-factor*capacity，maximum-capacity)。这里需要注意的是，如果队列中有多个用户的任务，那么每个用户的使用量将稀释。
* minimum-user-limit-percent：每个用户最低资源保障（百分比）。任何时刻，一个队列中每个用户可使用的资源量均有一定的限制。当一个队列中同时运行多个用户的应用程序时中，每个用户的使用资源量在一个最小值和最大值之间浮动，其中，最小值取决于正在运行的应用程序数目，而最大值则由minimum-user-limit-percent决定。比如，假设minimum-user-limit-percent为25。当两个用户向该队列提交应用程序时，每个用户可使用资源量不能超过50%，如果三个用户提交应用程序，则每个用户可使用资源量不能超多33%，如果四个或者更多用户提交应用程序，则每个用户可用资源量不能超过25%。
* maximum-applications ：集群或者队列中同时处于等待和运行状态的应用程序数目上限，这是一个强限制，一旦集群中应用程序数目超过该上限，后续提交的应用程序将被拒绝，默认值为10000。所有队列的数目上限可通过参数yarn.scheduler.capacity.maximum-applications设置（可看做默认值），而单个队列可通过参数yarn.scheduler.capacity.<queue-path>.maximum-applications设置适合自己的值。
* maximum-am-resource-percent：集群中用于运行应用程序ApplicationMaster的资源比例上限，该参数通常用于限制处于活动状态的应用程序数目。该参数类型为浮点型，默认是0.1，表示10%。所有队列的ApplicationMaster资源比例上限可通过参数yarn.scheduler.capacity. maximum-am-resource-percent设置（可看做默认值），而单个队列可通过参数yarn.scheduler.capacity.<queue-path>. maximum-am-resource-percent设置适合自己的值。
* state ：队列状态可以为STOPPED或者RUNNING，如果一个队列处于STOPPED状态，用户不可以将应用程序提交到该队列或者它的子队列中，类似的，如果ROOT队列处于STOPPED状态，用户不可以向集群中提交应用程序，但正在运行的应用程序仍可以正常运行结束，以便队列可以优雅地退出。
* acl_submit_applications：限定哪些Linux用户/用户组可向给定队列中提交应用程序。需要注意的是，该属性具有继承性，即如果一个用户可以向某个队列中提交应用程序，则它可以向它的所有子队列中提交应用程序。配置该属性时，用户之间或用户组之间用“，”分割，用户和用户组之间用空格分割，比如“user1, user2 group1,group2”。
* acl_administer_queue：为队列指定一个管理员，该管理员可控制该队列的所有应用程序，比如杀死任意一个应用程序等。同样，该属性具有继承性，如果一个用户可以向某个队列中提交应用程序，则它可以向它的所有子队列中提交应用程序。   

```xml
<property>
    <name>yarn.scheduler.capacity.maximum-applications</name>
    <value>10000</value>
    <description>
      Maximum number of applications that can be pending and running.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.maximum-am-resource-percent</name>
    <value>0.1</value>
    <description>
      Maximum percent of resources in the cluster which can be used to run
      application masters i.e. controls number of concurrent running
      applications.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.resource-calculator</name>
    <value>org.apache.hadoop.yarn.util.resource.DefaultResourceCalculator</value>
    <description>
      The ResourceCalculator implementation to be used to compare
      Resources in the scheduler.
      The default i.e. DefaultResourceCalculator only uses Memory while
      DominantResourceCalculator uses dominant-resource to compare
      multi-dimensional resources such as Memory, CPU etc.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>default,dailyTask,hive</value>
    <description>
      The queues at the this level (root is the root queue).具有三个子队列
    </description>
  </property>
  <!-- default -->
  <property>
    <name>yarn.scheduler.capacity.root.default.capacity</name>
    <value>20</value>
    <description>Default queue target capacity.</description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.default.user-limit-factor</name>
    <value>1</value>
    <description>
      Default queue user limit a percentage from 0.0 to 1.0.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.default.maximum-capacity</name>
    <value>100</value>
    <description>
      The maximum capacity of the default queue.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.default.state</name>
    <value>RUNNING</value>
    <description>
      The state of the default queue. State can be one of RUNNING or STOPPED.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.default.acl_submit_applications</name>
    <value>*</value>
    <description>
      The ACL of who can submit jobs to the default queue.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.default.acl_administer_queue</name>
    <value>*</value>
    <description>
      The ACL of who can administer jobs on the default queue.
    </description>
  </property>
  <!--dailyTask -->
  <property>
        <name>yarn.scheduler.capacity.root.dailyTask.capacity</name>
        <value>70</value>
        <description>Default queue target capacity.</description>
   </property>
   <property>
        <name>yarn.scheduler.capacity.root.dailyTask.user-limit-factor</name>
        <value>1</value>
        <description>
        Default queue user limit a percentage from 0.0 to 1.0.
        </description>
   </property>
   <property>
    <name>yarn.scheduler.capacity.root.dailyTask.maximum-capacity</name>
    <value>100</value>
    <description>
        The maximum capacity of the default queue..
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.dailyTask.state</name>
    <value>RUNNING</value>
    <description>
        The state of the default queue. State can be one of RUNNING or STOPPED.
    </description>
  </property>
    <property>
    <name>yarn.scheduler.capacity.root.dailyTask.acl_submit_applications</name>
    <value>hadoop</value>
    <description>
        The ACL of who can submit jobs to the default queue.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.root.dailyTask.acl_administer_queue</name>
    <value>hadoop</value>
    <description>
            The ACL of who can administer jobs on the default queue.
    </description>
  </property>
  <!--hive -->
    <property>
        <name>yarn.scheduler.capacity.root.hive.capacity</name>
        <value>10</value>
        <description>Default queue target capacity.</description>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.hive.user-limit-factor</name>
        <value>1</value>
        <description>
            Default queue user limit a percentage from 0.0 to 1.0.
        </description>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.hive.maximum-capacity</name>
        <value>100</value>
        <description>
            The maximum capacity of the default queue..
        </description>
    </property>
    <property>
      <name>yarn.scheduler.capacity.root.hive.state</name>
      <value>RUNNING</value>
      <description>
        The state of the default queue. State can be one of RUNNING or STOPPED.
      </description>
    </property>
    <property>
      <name>yarn.scheduler.capacity.root.hive.acl_submit_applications</name>
      <value>*</value>
      <description>
        The ACL of who can submit jobs to the default queue.
      </description>
   </property>
   <property>
     <name>yarn.scheduler.capacity.root.hive.acl_administer_queue</name>
     <value>*</value>
     <description>
       The ACL of who can administer jobs on the default queue.
     </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.node-locality-delay</name>
    <value>40</value>
    <description>
      Number of missed scheduling opportunities after which the CapacityScheduler
      attempts to schedule rack-local containers.
      Typically this should be set to number of nodes in the cluster, By default is setting
      approximately number of nodes in one rack which is 40.
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.queue-mappings</name>
    <value></value>
    <description>
      A list of mappings that will be used to assign jobs to queues
      The syntax for this list is [u|g]:[name]:[queue_name][,next mapping]*
      Typically this list will be used to map users to queues,
      for example, u:%user:%user maps all users to queues with the same name
      as the user. 进行过映射后，user只能访问对应的quue_name，后续的queue.name设置都没用了。
    </description>
  </property>
  <property>
    <name>yarn.scheduler.capacity.queue-mappings-override.enable</name>
    <value>false</value>
    <description>
      If a queue mapping is present, will it override the value specified
      by the user? This can be used by administrators to place jobs in queues
      that are different than the one specified by the user.
      The default is false.
    </description>
  </property>
```

## 3. 如何使用该队列

```shell
mapreduce:在Job的代码中，设置Job属于的队列,例如hive：
conf.setQueueName("hive");
hive:在执行hive任务时，设置hive属于的队列,例如dailyTask:
set mapred.job.queue.name=dailyTask;
```

## 4. 刷新配置

生产环境中，队列及其容量的修改在现实中是不可避免的，而每次修改，需要重启集群，这个代价很高，如果修改队列及其容量的配置不重启呢:
1.在主节点上根据具体需求，修改好mapred-site.xml和capacity-scheduler.xml
2.把配置同步到所有节点上yarn rmadmin -refreshQueues
这样就可以动态修改集群的队列及其容量配置，不需要重启了，刷新mapreduce的web管理控制台可以看到结果。
注意:如果配置没有同步到所有的节点，一些队列会无法启用。
可以查看http://hadoop-master:8088/cluster/scheduler

![img](../image/hadoop/capacity-schedule.png)

```shell
队列容量=yarn.scheduler.capacity.<queue-path>.capacity/100
队列绝对容量=父队列的 队列绝对容量*队列容量
队列最大容量=yarn.scheduler.capacity.<queue-path>.maximum-capacity/100
队列绝对最大容量=父队列的 队列绝对最大容量*队列最大容量
绝对资源使用比=使用的资源/全局资源
资源使用比=使用的资源/(全局资源 * 队列绝对容量)
最小分配量=yarn.scheduler.minimum-allocation-mb
用户上限=MAX(yarn.scheduler.capacity.<queue-path>.minimum-user-limit-percent,1/队列用户数量)
用户调整因子=yarn.scheduler.capacity.<queue-path>.user-limit-factor
最大提交应用=yarn.scheduler.capacity.<queue-path>.maximum-applications
    如果小于0 设置为(yarn.scheduler.capacity.maximum-applications*队列绝对容量)
单用户最大提交应用=最大提交应用*(用户上限/100)*用户调整因子
AM资源占比（AM可占用队列资源最大的百分比)
    =yarn.scheduler.capacity.<queue-path>.maximum-am-resource-percent
    如果为空，设置为yarn.scheduler.capacity.maximum-am-resource-percent
最大活跃应用数量=全局总资源/最小分配量*AM资源占比*队列绝对最大容量
单用户最大活跃应用数量=(全局总资源/最小分配量*AM资源占比*队列绝对容量)*用户上限*用户调整因子
本地延迟分配次数=yarn.scheduler.capacity.node-locality-delay<code>
```

## 5. 总结

本文介绍了如何配置yarn的容量调度器Capacity Scheduler配置。

本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/hadoop-capacity-scheduler
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
