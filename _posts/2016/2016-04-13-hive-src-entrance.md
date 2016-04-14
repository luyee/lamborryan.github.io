---
layout: post
title: Hive源码分析系列(1)之框架与Client/Server入口
date: 2016-04-13 23:30:00
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

* Hive有3种Service，分别是Hiveserver2, Hwi 以及 Metastore Server。
    * HiveServer2通过thrift对外提供服务，默认端口10000，启动方式为$HIVE_HOME/bin/hive--service hiveserver。像Beeline, Jdbc, Odbc 等客户端皆都是链接HiveServer2来实现的, 这将在下文重点介绍。为啥叫HiveServer2, 是因为当初设计的HiveServer1存在缺陷, 尤其是权限问题, 所以HiveServer已经废弃
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

### 3.1 bin/hive

首先, 我们来搞懂下hive这个客户端脚本. 在$HIVE_HOME/bin目录下存在ext目录, 他里面存放了各种启动脚本比如cli.sh, beeline.sh。 而这些脚本都是在bin/hive中被调用的。

在bin/hive脚本中存在以下几个service, 当然它只是参数名叫service, 并不是真正的service:

```shell
while [ $# -gt 0 ]; do
  case "$1" in
    --version)
      shift
      SERVICE=version
      ;;
    --service)
      shift
      SERVICE=$1
      shift
      ;;
    --rcfilecat)
      SERVICE=rcfilecat
      shift
      ;;
    --orcfiledump)
      SERVICE=orcfiledump
      shift
      ;;
    --help)
      HELP=_help
      shift
      ;;
    --debug*)
      DEBUG=$1
      shift
      ;;
    *)
      break
      ;;
  esac
done

if [ "$SERVICE" = "" ] ; then
  if [ "$HELP" = "_help" ] ; then
    SERVICE="help"
  else
    SERVICE="cli"
  fi
fi
```

这些SERVICE都分别对应 bin/ext/XX.sh。bin/hive 会把所有bin/ext和bin/ext/util目录下的sh脚本通过```. XX.sh```的方式加载到环境变量中, 然后只需要运行XX命令就行了。

```shell
SERVICE_LIST=""

for i in "$bin"/ext/*.sh ; do
  . $i
done

for i in "$bin"/ext/util/*.sh ; do
  . $i
done

TORUN=""
for j in $SERVICE_LIST ; do
  if [ "$j" = "$SERVICE" ] ; then
    TORUN=${j}$HELP
  fi
done

 $TORUN "$@"
```

比如当我运行```hive -e 'show tables';```, 因为没指定service, 所以bin/hive默认service为cli. 当运行以上代码时候, bin/hive会把bin/ext/cli.sh加载到环境变量中, 后续通过```cli -e 'show tables;'```就直接运行了bin/ext/cli.sh这个脚本。同理beeline。

### 3.2 启动脚本

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

### 3.3 入口类

java中的类如果有main方法就能运行，故直接查找org.apache.hadoop.hive.cli.CliDriver中的main方法即可。

```java
public static void main(String[] args) throws Exception {
  int ret = new CliDriver().run(args);
  System.exit(ret);
}
```

那么我们来查看下```Run```到底干了啥。

* 读取main方法的参数.
* 重置默认的log4j配置并为hive重新初始化log4j，注意，在这里是读取hive-log4j.properties来初始化log4j。
* 创建CliSessionState，并初始化in、out、info、error等stream流。CliSessionState是一次命令行操作的session会话，其继承了SessionState。
* 从命令行参数中读取参数并设置到CliSessionState中。
* 启动SessionState。

> 根据OptionsProcessor类的源码来看, 虽然CliDriver里面出现了CliSessionState, 但是它并不提供远程调用。也就是说CliDriver其实只支持本地调用。OptionsProcessor主要将命令行参数转换为CliSessionState. 而在Run的流程中SessionState只有当HIVE_EXECUTION_ENGINE为tez才会去连接server。一般情况下这个CliSessionState是本地的,只是定义了这个类且作为存储数据用而已。不要被```CliDriver.run()```里面的```SessionState.start(ss)```迷惑了。

```java
if (HiveConf.getVar(startSs.getConf(), HiveConf.ConfVars.HIVE_EXECUTION_ENGINE)
        .equals("tez") && (startSs.isHiveServerQuery == false)) {
      try {
        if (startSs.tezSessionState == null) {
          startSs.tezSessionState = new TezSessionState(startSs.getSessionId());
        }
        if (!startSs.tezSessionState.isOpen()) {
          startSs.tezSessionState.open(startSs.conf); // should use conf on session start-up
        }
      } catch (Exception e) {
        throw new RuntimeException(e);
      }
    }
```

* 创建一个CliDriver对象，并设置当前选择的数据库。可以在命令行参数添加-database database来选择连接那个数据库，默认为default数据库。
* 加载初始化文件.hiverc，该文件位于当前用户主目录下，读取该文件内容后，然后调用processFile方法处理文件内容。
* 如果命令行中有-e参数，则运行指定的sql语句, ```CliDriver.processLine```；如果有-f参数，则读取该文件内容并运行,```CliDriver.processFile```。注意：不能同时指定这两个参数。 processFile实质上也是用一个buffer来存储文件的内容, 内部调用processLine。 processLine支持以分号作为分隔符的多个语句的处理;
* 如果即没有-e, 也没有-f, 则说明是交互式的方式启动的。那么就会调用```setupConsoleReader();
```来启动Console交互。在while循环里不断读取控制台的输入内容，每次读取一行，如果行末有分号，则调用CliDriver的processLine方法运行读取到的内容。
* 每次调用processLine方法时，都会创建SignalHandler用于捕捉用户的输入，当用户输入Ctrl+C时，会kill当前正在运行的任务以及kill掉当前进程。kill当前正在运行的job的代码如下. ```HadoopJobExecHelper.killRunningJobs();
```
* 无论最后是-e, -f还是都没有, 最后都是调用```processCmd```来处理处理hive命令。

### 3.4 处理hive命令

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

5.其他则调用processLocalCmd方法运行本地命令。

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

Beeline的步骤大致与Cli类似, 这里就简单介绍下:

* 由```Beeline.main```启动。
* 首先```getOpts().load();```加载beeline配置文件、历史记录文件
* 接着```BeeLine.initArgs```解析并检查命令行参数, 并存储到BeeLineOpts中. 如果是```!connect```, 经过Beeline的处理后，最终会调用到Commands对象的connect方法.
* 无论是使用-e执行单条语句、-f执行单个SQL文件，或者是不指定参数进入交互模式，beeline最终都将各种输入源转换成逐行命令。这个过程中会引用第三方组件jline。注意Hive的jline和hadoop的jline版本要一致, 否则
* 如果是```!```开头的命令表示本地的shell命令, 则直接运行本地的shell。否则进入```Command.sql```来运行sql。
* ```Command.sql```会对命令按分号切成多个查询, 并调用Hive Jdbc来进行查询.

```java
stmnt = beeLine.createStatement();
    if (beeLine.getOpts().isSilent()) {
    hasResults = stmnt.execute(sql);
} else {
    logThread = new Thread(createLogRunnable(stmnt));
    logThread.setDaemon(true);
    logThread.start();
    hasResults = stmnt.execute(sql);
    logThread.interrupt();
    logThread.join(DEFAULT_QUERY_PROGRESS_THREAD_TIMEOUT);
}
Statement createStatement() throws SQLException {
  Statement stmnt = getDatabaseConnection().getConnection().createStatement();
  if (getOpts().timeout > -1) {
    stmnt.setQueryTimeout(getOpts().timeout);
  }
  if (signalHandler != null) {
    signalHandler.setStatement(stmnt);
  }
  return stmnt;
}
```

这里要重点说下Command类, 它每一个方法对应Beeline一次请求的需求. 方法实在太多, 下面主要说一下metadata，connect，sql方法。

* metadata：beeline可以直接使用tables,columns等命令获取信息，这些命令都不会经过HiveServer，而是直接调用本方法，通过JDBC驱动获取元数据信息。因此，在版本升级、版本迁移等场景中，元数据中记录的信息可能和show tables等DDL查询的内容不一致，可以通过这些接口来查询确认。
* connect: 该方法将传入的jdbc字符串解析，查找对应的JDBC驱动，并建立与服务端的连接。Beeline支持管理多个连接，可以使用!go在多个连接中进行切换，具体管理连接的类有DatabaseConnections和DatabaseConnection，都在rg.apache.hive.beeline包中。
* sql: sql方法将传入的SQL语句，通过JDBC接口，向服务端发起请求，并向客户端返回结果。


## 5. HiveServer2

HiveServer2负责所有通过thrift连接的请求, 以及相应的权限印证. 关于Hive的权限管理请查看我的博客:

* [《Hive数据仓库(1)之如何使用Custom方式进行认证 ](<http://www.lamborryan.com/hive-custom-authentication/>)
* [《Hive数据仓库(2)之超级管理员实现》](<http://www.lamborryan.com/hive-admin-setting/>)
* [《Hive数据仓库(3)之权限操作》](<http://www.lamborryan.com/hive-authority/>)

首先查看跟HiveServer2有关的几个类。HiveServer2继承CompositeService类，CompositeService类内部维护一个serviceList，能够加入、删除、启动、停止不同的服务。

```java
AbstractService (org.apache.hive.service)
    CompositeService (org.apache.hive.service)
        CLIService (org.apache.hive.service.cli)
        HiveServer2 (org.apache.hive.service.server)
        SessionManager (org.apache.hive.service.cli.session)
    OperationManager (org.apache.hive.service.cli.operation)
    ThriftCLIService (org.apache.hive.service.cli.thrift)
        ThriftBinaryCLIService (org.apache.hive.service.cli.thrift)
        ThriftHttpCLIService (org.apache.hive.service.cli.thrift)
```

HiveServer2的过程依然还得从main函数将起, 它会根据启动参数来决定初始化哪个```ServerOptionsProcessorResponse```, 不同的```ServerOptionsProcessorResponse```影响hiveserver不同的启动方式.

(Process --help) ```HelpOptionExecutor```,
(Process --deregister) ```DeregisterOptionExecutor```,
(default)```StartOptionExecutor``` 具体作用看名字就可秒懂.
我们只关注默认的```StartOptionExecutor```

```java
server = new HiveServer2();
server.init(hiveConf);
server.start();
```

### 5.1 ThriftCLIService

ThriftCLIService是HiveServer2的thrift服务端, JDBC客户端这边通过thrift发过来的请求将全部落入这个对象进行处理，从他的类定义可以看到标准的thrift用法。从他的类定义可以看到标准的thrift用法。

```java
public abstract class ThriftCLIService extends AbstractService implements TCLIService.Iface, Runnable{

}
```
HiveServer2在init(hiveConf)的时候，会加入CLIService和ThriftCLIService两个Service。根据传输模式，如果是http或https的话，就使用ThriftHttpCLIService，否则使用ThriftBinaryCLIService。无论是哪个ThriftCLIService，都传入了CLIService的引用，thrift只是一个封装。也就说ThriftCLIService负责处理通信, CLIService 负责业务处理。

```java
@Override
public synchronized void init(HiveConf hiveConf) {
  cliService = new CLIService(this);
  addService(cliService);
  if (isHTTPTransportMode(hiveConf)) {
    thriftCLIService = new ThriftHttpCLIService(cliService);
  } else {
    thriftCLIService = new ThriftBinaryCLIService(cliService);
  }
  addService(thriftCLIService);
  super.init(hiveConf);

  // Add a shutdown hook for catching SIGTERM & SIGINT
  final HiveServer2 hiveServer2 = this;
  Runtime.getRuntime().addShutdownHook(new Thread() {
    @Override
    public void run() {
      hiveServer2.stop();
    }
  });
}
```

### 5.2 CLIService

CLIService也继承自CompositeService，CLIService 在init的时候会加入SessionManager服务，并且根据hiveConf，从 hadoop shims里得到UGI里的serverUsername。

SessionManager管理hive连接的开启、关闭等管理功能，已有的连接会维护在一个HashMap里，value为HiveSession类，里面大致是用户名、密码、hive配置等info。

所以CLIService里几乎所有的事情都是委托给SessionManager做的。

比如执行命令就是依靠SessionManager完成的


### 5.3 SessionManager

SessionManager内主要是OperationManager这个服务，是最重要的和执行逻辑有关的类

上小节讲到CLIService里几乎所有的事情都是委托给SessionManager做的。 那么是如何实现的呢？

#### 5.3.1 创建Session

在SessionManager建立好连接后，会将其管理在一个SessionManager对象的一个Map中handleToSession，以SessionHandle为key，并将key返回给客户端，后续客户端在这个会话中的请求，都会携带这个sessionHandle，作为寻找Session的唯一ID。

```handleToSession```的数据结构如下

``` java
private final Map<SessionHandle, HiveSession> handleToSession =
      new ConcurrentHashMap<SessionHandle, HiveSession>();
```

HiveSession有两个子类HiveSessionImpl和HiveSessionImplwithUGI, 它们包含了后续对session处理的各种方法。HiveSessionImpl 和 HiveSessionImplwithUGI的差别就在于前者运行hive命令是以连接登入的用户名去执行,而后者则是以启动hive server的用户名去执行, 详细可以查看```doAs```这个配置。

> If doAs is set to true for HiveServer2, we will create a proxy object for the session impl.Within the proxy object, we wrap the method call in a UserGroupInformation#doAs

以下是创建一个新的session的代码:

```java
@Override
public SessionHandle openSession(String username, String password, Map<String, String> configuration)
    throws HiveSQLException {
  SessionHandle sessionHandle = sessionManager.openSession(SERVER_VERSION, username, password, null, configuration, false, null);
  LOG.debug(sessionHandle + ": openSession()");
  return sessionHandle;
}
```

#### 5.3.2 获取Session

当需要执行hive命令时, client会根据sessionHandle向handleToSession获取其对应的HiveSession, 然后调用HiveSession的executeStatement等方法进行相应处理。

以下是CliServer的executeStatement的方法。

```java
@Override
public OperationHandle executeStatement(SessionHandle sessionHandle, String statement,
    Map<String, String> confOverlay)
        throws HiveSQLException {
  OperationHandle opHandle = sessionManager.getSession(sessionHandle)
      .executeStatement(statement, confOverlay);
  LOG.debug(sessionHandle + ": executeStatement()");
  return opHandle;
}
```

#### 5.3.3 总结

写到这里需要再理一下CLIService与SessionManager的关系, 我用下图进行表示吧:

![img](../image/hive/hive-src-extrance/hive-5.png)

执行hive查询时候, executeStatement先调用了SessionManager的getSession来获取hiveSession, 随后对返回的hiveSession调用了它的executeStatement, 实际上调用了executeStatementInternal.

```java
private OperationHandle executeStatementInternal(String statement, Map<String, String> confOverlay,
      boolean runAsync)
          throws HiveSQLException {
    acquire(true);

    OperationManager operationManager = getOperationManager();
    ExecuteStatementOperation operation = operationManager
        .newExecuteStatementOperation(getSession(), statement, confOverlay, runAsync);
    OperationHandle opHandle = operation.getHandle();
    try {
      operation.run();
      opHandleSet.add(opHandle);
      return opHandle;
    } catch (HiveSQLException e) {
      // Refering to SQLOperation.java,there is no chance that a HiveSQLException throws and the asyn
      // background operation submits to thread pool successfully at the same time. So, Cleanup
      // opHandle directly when got HiveSQLException
      operationManager.closeOperation(opHandle);
      throw e;
    } finally {
      release(true);
    }
  }
```

由此可见最后通过OperationManager的newExecuteStatementOperation获取Operation, 然后通过```operation.run();```来执行hive命令。

### 5.4 OperationManager

顾名思义, OperationManager就是跟操作有关的管理类. 我们来看看newExecuteStatementOperation做了啥:

```java
public static ExecuteStatementOperation newExecuteStatementOperation(
      HiveSession parentSession, String statement, Map<String, String> confOverlay, boolean runAsync)
          throws HiveSQLException {
    String[] tokens = statement.trim().split("\\s+");
    CommandProcessor processor = null;
    try {
      processor = CommandProcessorFactory.getForHiveCommand(tokens, parentSession.getHiveConf());
    } catch (SQLException e) {
      throw new HiveSQLException(e.getMessage(), e.getSQLState(), e);
    }
    if (processor == null) {
      return new SQLOperation(parentSession, statement, confOverlay, runAsync);
    }
    return new HiveCommandOperation(parentSession, statement, processor, confOverlay);
  }
```

CommandProcessorFactory 这个类是不是很熟悉?没错就是<3.4节>已经介绍过了, 它的作用根据输入的命令选择对应的操作CommandProcessor(比如set/dfs/add/delete/reset),

它将命令分为两种HiveCommandOperation，包含set/dfs/add/delete/reset，以及SQLOperation(真正的sql命令)。 它们两都继承自ExecuteStatementOperation。 当然接下来我们只关注SQLOperation了。

### 5.5 SQLOperation

SQLOperation 的核心方法就是runQuery

```java
private void runQuery(HiveConf sqlOperationConf) throws HiveSQLException {
    try {
      // In Hive server mode, we are not able to retry in the FetchTask
      // case, when calling fetch queries since execute() has returned.
      // For now, we disable the test attempts.
      driver.setTryCount(Integer.MAX_VALUE);
      response = driver.run();
      if (0 != response.getResponseCode()) {
        throw toSQLException("Error while processing statement", response);
      }
    } catch (HiveSQLException e) {
      // If the operation was cancelled by another thread,
      // Driver#run will return a non-zero response code.
      // We will simply return if the operation state is CANCELED,
      // otherwise throw an exception
      if (getStatus().getState() == OperationState.CANCELED) {
        return;
      }
      else {
        setState(OperationState.ERROR);
        throw e;
      }
    } catch (Exception e) {
      setState(OperationState.ERROR);
      throw new HiveSQLException("Error running query: " + e.toString(), e);
    }
    setState(OperationState.FINISHED);
  }
```

由此可见, 绕了一大圈, 最后还是回到Driver类. 也就是说一切的sql查询都是从Driver发起的。

## 6. 总结

本文首先介绍了Hive的框架, 工作流程.  

最后详细介绍了Cli, Beeline, hiveServer2执行hive命令的流程，它们最后都汇总到了Driver类。

最后用下图来表示下类调用关系。

![img](../image/hive/hive-src-extrance/hive-6.png)

希望本文对你有帮助

本文完

## 参考文献

[Hive源码分析：CLI入口类](<http://blog.javachen.com/2013/08/21/hive-CliDriver.html>)

[Hive学习笔记 之 Hive运行流程简析1](<http://www.cnblogs.com/Rudd/p/5140936.html>)



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：[http://www.lamborryan.com/hive-src-entrance](<http://www.lamborryan.com/hive-src-entrance>)
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
