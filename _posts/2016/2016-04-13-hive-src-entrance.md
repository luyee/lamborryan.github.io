---
layout: post
title: Hive源码分析系列(1)之框架与Client/Server入口
date: 2016-04-12 10:30:00
categories: 大数据
tags: Hive源码分析
---

## 1. 简介

Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的sql查询功能，可以将sql语句转换为MapReduce任务进行运行。我们常常使用Hive来进行数据维度建模以及ETL。

Hive源码分析系列着眼于通过源码来更深入的了解Hive, 希望了解什么是Hive以及如何使用Hive的同学可以看我的另外一个专栏[《Hive数据仓库系列分析》](<http://www.lamborryan.com/hive-warehouse/>)以及《Hive权威指南》。

本系列的源码分析都是基于Apache Hive 1.2.1 的, 后续不再说明。

作为本系列的第一文章, 我将简单介绍了Hive的框架以及工作流程, 随后重点描述Hive的Client和Server的入口。

## 2. 框架与流程

### 2.1 框架

Hive的框架如下图所示:

![img](../image/hive/hive-src-extrance/hive-1.png)

* Hive有4种Service，分别是Cli，Hiveserver2, Hwi 以及 Metastore Server。
    * 当单单运行```hive```命令时, 其实它就是在本地运行了一个CliServer。
    * HiveServer2通过thrift对外提供服务，默认端口10000，启动方式为$HIVE_HOME/bin/hive--service hiveserver。像Beeline, Jdbc, Odbc 等客户端皆都是链接HiveServer2来实现的, 这将在下节重点介绍。为啥叫HiveServer2, 是因为当初设计的HiveServer1存在缺陷, 尤其是权限问题, 所以HiveServer1已经废弃
    * Hwi为web接口，可以通过浏览器访问hive，默认端口9999，启动方式为$HIVE_HOME/bin/hive--service hwi。具体细节请看我的博客[《Hive数据仓库(9)之Hive Web Interface》](<http://www.lamborryan.com/hive-hwi/>)
    * Metastore Server是负责管理Hive的元数据, 比如表结构,分区, Serde等等重要信息.
    * 每个服务间互相独立，有各自的配置文件(配置metasotre/namenode/jobtracker等)，如果metasotre的配置一样则物理上对应同一hive库。
* Hive同样存在Client, 即Hive 和 Beeline Client, 前者只能在本地调用, 后者可通过Hive Jdbc远程调用。
Hive还支持Jdbc和Odbc调用。JDBC Hive已经自带, HiveServer2的odbc hive并不支持, 需要去第三方寻找比如cdh.
* Driver用于解释、编译、优化、执行HQL，每个service的Driver相互独立。

通过上图可以看出Hive的核心内容在Driver中, 因此Driver将是后续系列文章的重点, 本文主要介绍Cli, HiveServer2是如何调用到Driver, 以及在这个过程中发生了什么。

### 2.2 流程

上一节介绍了Hive的框架, 本文从数据流的角度看看Hive到底干了啥。

![img](../image/hive/hive-src-extrance/hive-2.png)

## 3. Cli入口

### 3.1 启动脚本

从shell脚本/usr/lib/hive/bin/ext/cli.sh可以看到hive cli的入口类为org.apache.hadoop.hive.cli.CliDriver

```shell
cli () {
  CLASS=org.apache.hadoop.hive.cli.CliDriver
  execHiveCmd $CLASS "$@"
}

cli_help () {
  CLASS=org.apache.hadoop.hive.cli.CliDriver
  execHiveCmd $CLASS "--help"
}
```

### 3.2 入口类

java中的类如果有main方法就能运行，故直接查找org.apache.hadoop.hive.cli.CliDriver中的main方法即可。

```java
public static void main(String[] args) throws Exception {
  int ret = new CliDriver().run(args);
  System.exit(ret);
}
```

那么我们来查看下```Run```到底干了啥。

* 读取main方法的参数.(```OptionsProcessor.process_stage1```)
* 重置默认的log4j配置并为hive重新初始化log4j，注意，在这里是读取hive-log4j.properties来初始化log4j。(```LogUtils.initHiveLog4j()```)
* 创建CliSessionState，并初始化in、out、info、error等stream流。CliSessionState是一次命令行操作的session会话，其继承了SessionState。(``` CliSessionState ss = new CliSessionState(new HiveConf(SessionState.class));```)
* 重命令行参数中读取参数并设置到CliSessionState中。((```OptionsProcessor.process_stage2```))
* 启动SessionState并连接到hive server。

```java
session = TezClient.create("HIVE-" + sessionId, tezConfig, true,
    commonLocalResources, null);
TezJobMonitor.initShutdownHook();
session.start();
```

* 创建一个CliDriver对象，并设置当前选择的数据库。可以在命令行参数添加-database database来选择连接那个数据库，默认为default数据库。(``cli.processSelectDatabase(ss);```)
* 加载初始化文件.hiverc，该文件位于当前用户主目录下，读取该文件内容后，然后调用processFile方法处理文件内容。(```CliDriver.processInitFiles```)
* 如果命令行中有-e参数，则运行指定的sql语句；如果有-f参数，则读取该文件内容并运行。注意：不能同时指定这两个参数。```hive -e 'show tables'; hive -f /root/hive.sql```。
* 如果没有指定上面两个参数，则从当前用户主目录读取.hivehistory文件，如果不存在则创建。该文件保存了当前用户所有运行的hive命令。
* 在while循环里不断读取控制台的输入内容，每次读取一行，如果行末有分号，则调用CliDriver的processLine方法运行读取到的内容。
* 每次调用processLine方法时，都会创建SignalHandler用于捕捉用户的输入，当用户输入Ctrl+C时，会kill当前正在运行的任务以及kill掉当前进程。kill当前正在运行的job的代码如下. ```HadoopJobExecHelper.killRunningJobs();
```
* 处理hive命令。

### 3.3 处理hive命令

Hive Shell 不但支持sql语句, 还支持shell, hadoop shell等命令, 甚至add, set环境变量等名。 那么我们来看Hive是怎么处理这里命令的。 这部分的内容尽都在```CliDriver.processCmd```方法内。

1.如果输入的是quit或者exit,则程序退出。

2.如果命令开头是source，则会读取source 后面文件内容，然后执行该文件内容。通过这种方式，你可以在hive命令行模式运行一个文件中的hive命令。

3.如果命令开头是感叹号，执行操作系统命令（如!ls，列出当前目录的文件信息）。通过以下代码来运行：

```java
// shell_cmd = "/bin/bash -c \'" + shell_cmd + "\'";
try {
  ShellCmdExecutor executor = new ShellCmdExecutor(shell_cmd, ss.out, ss.err);
  ret = executor.execute();
  if (ret != 0) {
    console.printError("Command failed with exit code = " + ret);
  }
} catch (Exception e) {
  console.printError("Exception raised from Shell command " + e.getLocalizedMessage(),
      stringifyException(e));
  ret = 1;
}
```

其本质是调用了 ```Process executor = Runtime.getRuntime().exec(shell_cmd);```

4.如果命令开头是list，列出jar/file/archive

5.如果是远程模式运行命令行，则通过HiveClient来运行命令；否则，调用processLocalCmd方法运行本地命令。

以本地模式运行时，会通过CommandProcessorFactory工厂解析输入的语句来获得一个CommandProcessor，CommandProcessor接口的实现类见下图：

![img](../image/hive/hive-src-extrance/hive-3.jpg)

从上图可以看到指定的命令(set/dfs/add/delete/reset)交给指定的CommandProcessor处理, 它们都继承了run方法，其余的(指hql语句)交给Driver类来处理。

```java
public static CommandProcessor getForHiveCommandInternal(String[] cmd, HiveConf conf,
                                                         boolean testOnly){
    ***
    switch (hiveCommand) {
      case SET:
        return new SetProcessor();
      case RESET:
        return new ResetProcessor();
      case DFS:
        SessionState ss = SessionState.get();
        return new DfsProcessor(ss.getConf());
      case ADD:
        return new AddResourceProcessor();
      case LIST:
        return new ListResourceProcessor();
      case DELETE:
        return new DeleteResourceProcessor();
      case COMPILE:
        return new CompileProcessor();
      case RELOAD:
        return new ReloadProcessor();
      case CRYPTO:
        try {
          return new CryptoProcessor(SessionState.get().getHdfsEncryptionShim(), conf);
        } catch (HiveException e) {
          throw new SQLException("Fail to start the command processor due to the exception: ", e);
        }
      default:
        throw new AssertionError("Unknown HiveCommand " + hiveCommand);
    }
}
```

故，org.apache.hadoop.hive.ql.Driver类是hql查询的起点，而run()方法会先后调用compile()和execute()两个函数来完成查询，所以一个command的查询分为compile和execute两个阶段。 本文的分析也到Driver结束了。

CliDriver类的运行逻辑的思维导图如下所示:

![img](../image/hive/hive-src-extrance/hive-4.jpg)

## 4. Beeline入口

## 5. HiveServer2

## 6. 总结


## 参考文献

* [Hive源码分析：CLI入口类](<http://blog.javachen.com/2013/08/21/hive-CliDriver.html>)
