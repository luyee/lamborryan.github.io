---
layout: post
title: Gobblin系列三之Azkaban Schedule
date: 2016-03-15 13:30:00
categories: 大数据
tags: Gobblin
---
# Gobblin系列三之Azkaban Schedule

## 前言

Gobblin支持三种Schedule即Quartz, Azkaban, Oozie, 默认是采用Quartz. 由于项目的工作流schedule已经采用Azkaban, 所以要将Gobblin Task配置到Azkaban. 但是没相到以为分分钟就能搞定的结果花了我整整一天的时间, 主要问题还是因为Gobblin的资料的匮乏, 在这实现过程中我查阅了Gobblin，Azkaban 和Azkaban-jobtype plugin的源码, 可见繁琐程度. 本文将描述怎么配置Gobblin-Azkaban，并结合主要的代码流程。之所以采用azkaban, 主要因为azkaban使用简单，能有效的进行工作流依赖管理。

## 配置过程

### 注意事项

首先需要注意以下两点:

* 使用bin/gobblin-standalone.sh启动gobblin-standalone模式采用的是Quartz模式. job.schedule就是按照Quartz的规则来实现定时任务。所以要实现Azkaban Schedule第一步就需要关闭gobblin-standalone进程, 删掉job.schedule配置。
* 实现Azkaban Schedule的另一个前提条件是配置Azkaban的jobtype plugin。这是因为Gobblin-Azkaban需要通过hadoopJava或者java jobtype来实现的.

在Gobblin的文档上关于配置Azkaban Schedule只有以下这段话:

> Gobblin can be launched via Azkaban, and open-source Workflow Manager for scheduling and launching Hadoop jobs. Gobblin's AzkabanJobLauncher can be used to launch a Gobblin job through Azkaban.

> One has to follow the typical setup to create a zip file that can be uploaded to Azkaban (it should include all dependent jars, which can be found in gobblin-dist.tar.gz). The .job file for the Azkaban Job should contain all configuration properties that would be put in a .pull file (for example, the Wikipedia Example .pull file). All Gobblin system dependent properties (e.g. conf/gobblin-mapreduce.properties or conf/gobblin-standalone.properties) should also be in the zip file.

> In the Azkaban .job file, the type parameter should be set to hadoopJava (see here for more information about the hadoopJava Job Type). The job.class parameter should be set to gobblin.azkaban.AzkabanJobLauncher.

### 安装Azkaban jobtype plugin

1.下载[azkaban-jobtype-2.5.0.tar.gz](https://s3.amazonaws.com/azkaban2/azkaban-plugins/2.5.0/azkaban-jobtype-2.5.0.tar.gz) ,由于在amazonaws上所以下载速度很慢。

2.将azkaban-jobtype-2.5.0.tar.gz 解压到${AZKABAN_HOME}/plugins目录下, 并建立软连接

``` bash
ln -s azkaban-jobtype-2.5.0 jobtype
```

3.修改${AZKABAN_HOME}/conf/azkaban.properties, 增加配置

```bash
azkaban.jobtype.plugin.dir=plugins/jobtype
```

4.配置jobtype插件, 修改${AZKABAN_HOME}/plugins/jobtype/common.properties

```bash
hadoop.home=/opt/hadoop
#hive.home=/opt/hive
jobtype.global.classpath=${hadoop.home}/etc/hadoop/*,${hadoop.home}/lib/native/*,${hadoop.home}/share/common/*,${hadoop.home}/share/hdfs/*,${hadoop.home}/    share/yarn/*
```

5.重启Azkaban

```bash
bin/azkaban-solo-start.sh
```

需要注意的是这里的配置使用的是相对路径, 所以需要在${AZKABAN_HOME}目录下运行bin/azkaban-solo-start.sh命令。

### Gobblin Job

#### 创建Gobblin的azkaban Job

新建job文件gobblin_test.job

``` bash
type=java
job.class=gobblin.azkaban.AzkabanJobLauncher
classpath=/data/bmw/services/gobblin/gobblin/lib/*
ENV.JOB_PROP_FILE=hdfs_standalone_txt_test.pull
method.run=run
method.cancel=cancel
```
> 虽然官方文档中说JobType要设置为hadoopJava, 但是在实际应用中如果设置为hadoopJava, 那么会存在gobblin job会一直运行不会结束的bug, 当我把JobType设置为java后运行正常. 所以本文后续全部使用java jobtype。 bug类似于[这里](https://groups.google.com/forum/#!searchin/gobblin-users/azkaban/gobblin-users/wxegYW_FGbI/1dA-KgpQCQAJ)

从上面的例子可以看出几个要素。

1.配置gobblin的azkaban Job需要设置Job Type为java，所以Azkaban会使用jobtype plugin的JavaJobRunnerMain来启动Gobblin Job。

``` java
public JavaJobRunnerMain() throws Exception {
    // 代码省略
    * * *

    // 根据job.class配置项实例化gobblin.azkaban.AzkabanJobLauncher类
    _javaObject = getObject(_jobName, className, prop, _logger);
    if (_javaObject == null) {
        _logger.info("Could not create java object to run job: " + className);
        throw new Exception("Could not create running object");
    }

    // 根据method.cancel配置项映射到gobblin.azkaban.AzkabanJobLauncher的method, 默认method为cancel
    _cancelMethod = prop.getProperty(CANCEL_METHOD_PARAM, DEFAULT_CANCEL_METHOD);

    // 根据method.run配置项映射到gobblin.azkaban.AzkabanJobLauncher的method, 默认method为run
    final String runMethod = prop.getProperty(RUN_METHOD_PARAM, DEFAULT_RUN_METHOD);
    _logger.info("Invoking method " + runMethod);

    if (shouldProxy(prop)) {
        _logger.info("Proxying enabled.");
        runMethodAsUser(prop, _javaObject, runMethod, proxyUser);
    } else {
        _logger.info("Proxy check failed, not proxying run.");
        // 运行gobblin.azkaban.AzkabanJobLauncher的run method.
        runMethod(_javaObject, runMethod);
    }
    // 代码省略
    * * *
}

private void runMethod(Object obj, String runMethod) throws IllegalAccessException, InvocationTargetException,
        NoSuchMethodException {
    obj.getClass().getMethod(runMethod, new Class<?>[] {}).invoke(obj);
}
```

从代码片段上可以看出Azakan根据Java配置来决定调用JavaJobRunnerMain, 而JavaJobRunnerMain实例化了gobblin.azkaban.AzkabanJobLauncher 并调用其run 方法来launcher gobblin job, 使用canel方法来停止gobblin job。

AzkabanJobLauncher 的代码片段如下, 其中关于AzkabanJobLauncher的内容超出本文的界限, 在后续文章中详细介绍。

``` java
    @Override
    public void run()
        throws Exception {
      try {
        if (isCurrentTimeInRange()) {
          this.jobLauncher.launchJob(this.jobListener);
        }
      } finally {
        this.closer.close();
      }
    }

    @Override
    public void cancel()
        throws Exception {
      try {
        this.jobLauncher.cancelJob(this.jobListener);
      } finally {
        this.closer.close();
      }
    }
```

2.classpath配置项为需要调用的job.class的路径, 在本文也就是${GOBBLIN_HOME}/lib

3.ENV.JOB_PROP_FILE=hdfs_standalone_txt_test.pull

构造AzkabanJobLauncher实例的时候需要传入gobblin job的运行参数, 这就需要通过环境变量来传递配置文件路径。

下面依然是JavaJobRunnerMain是代码断

```java
public JavaJobRunnerMain() throws Exception {
    // 代码省略
    * * *

    _jobName = System.getenv(ProcessJob.JOB_NAME_ENV);
    //获取ENV.JOB_PROP_FILE的路径
    String propsFile = System.getenv(ProcessJob.JOB_PROP_ENV);

    // 加载配置文件内容
    Properties prop = new Properties();
    prop.load(new BufferedReader(new FileReader(propsFile)));

    // 通过反射机制实例化AzkabanJobLauncher
    _javaObject = getObject(_jobName, className, prop, _logger);

    // 代码省略
    * * *
}
```

至此已经完成了azkaban job的配置

#### 创建gobblin job

配置gobblin job跟standalone是相似的, 不同之处在于

1. 上文介绍的删除job.shedule
2. 增加launcher.type=LOCAL 和 job.class=gobblin.azkaban.AzkabanJobLauncher
3. 需要将gobblin-standalone.properties跟.pull配置里不同的配置项补充到.pull配置中

## 总结

本文介绍了gobblin如何结合azkaban配置工作流调度, 并简要介绍了调度源码。由于gobblin的资料实在太少, 所以往往只能查阅源码，不过这也是一件有趣的事情。

本文完
