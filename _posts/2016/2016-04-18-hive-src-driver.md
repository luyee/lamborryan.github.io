---
layout: post
title: Hive源码分析系列(2)之Driver流程分析
date: 2016-04-18 12:30:00
categories: 大数据
tags: Hive源码分析
---

## 1. 简介

本文接着上文[《Hive源码分析系列(1)之框架与Client/Server入口》](<http://www.lamborryan.com/hive-src-entrance/>)讲, 上文讲到无论Hive是哪种方式运行(Cli,beeline,HiveServer2), 流程最后都汇总到Driver类. 它肩负着将用户的command进行编译，优化并生成MapReduce任务进行执行这一使命。所以Driver也是Hive的核心，主要的逻辑处理就在这里了, 相当于主线程。 本文主要来扒一扒Driver到底做了啥。

## 2. 源码分析

从上文的逻辑中看出Driver的入口函数是run, 那么我们流程就以run函数开始讲。

在开始讲run函数之前, 先来看下Driver的构造函数以及初始化函数. Driver有三个构造函数，主要功能也就是设置类的实例变量HiveConf。 代码中的构造函数主要是从SessionState中获取HiveConf。而init函数只是重置了Operator的id。这里的Operator对应的是sql解析好后需要执行的Operator, 比如SelectOperator, 这不在本文介绍范围内。

```java
public Driver() {
  if (SessionState.get() != null) {
    conf = SessionState.get().getConf();
  }
}

@Override
public void init() {
  Operator.resetId();
}

```

### 2.1 run

Driver 有两个run入口函数, 它们最后都调用了runInternal, 所以主要逻辑在runInternal内.

```java
@Override
public CommandProcessorResponse run(String command)
    throws CommandNeedRetryException {
  return run(command, false);
}

public CommandProcessorResponse run()
    throws CommandNeedRetryException {
  return run(null, true);
}

public CommandProcessorResponse run(String command, boolean alreadyCompiled)
        throws CommandNeedRetryException {
    CommandProcessorResponse cpr = runInternal(command, alreadyCompiled);
    // 对cpr response进行处理, 如果执行过程中出现出错，还附加了对错误码和错误信息的处理. 代码省略
}
```

尽管如此, 但是仔细一看两个函数还是不同的, ```run(command, false)```和```run(null, true)```, 前者有查询命令commnad且设置alreadyCompiled为flase, 而后者查询command设为null且设置alreadyCompiled为false。一时好奇我跟踪了代码, 发现前者```run(command, false)```是由CliDriver和HiveCommandOperation调用的, 而后者```run(null, true)```是由SQLOperation调用的。关于这三个类的细节请查看[《Hive源码分析系列(1)之框架与Client/Server入口》](<http://www.lamborryan.com/hive-src-entrance/>)。

而```alreadyCompiled```的值不同将影响runInternal的流程，我们将在runInternal小节展开讲。

### 2.2 runInternal

#### 2.2.1 流程分析

runInternal 的逻辑还是比较清晰的:

1.进行配置参数检查

```java
if (!validateConfVariables()) {
  return createProcessorResponse(12);
}
```

2.设置HiveDriverRunHook, 运行前实现钩子

```java
HiveDriverRunHookContext hookContext = new HiveDriverRunHookContextImpl(conf, command);
// Get all the driver run hooks and pre-execute them.
List<HiveDriverRunHook> driverRunHooks;
driverRunHooks = getHooks(HiveConf.ConfVars.HIVE_DRIVER_RUN_HOOKS,
  HiveDriverRunHook.class);
for (HiveDriverRunHook driverRunHook : driverRunHooks) {
  driverRunHook.preDriverRun(hookContext);
}
```

3.添加运行起始时间

```java
// Reset the perf logger
PerfLogger perfLogger = PerfLogger.getPerfLogger(true);
perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.DRIVER_RUN);
perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.TIME_TO_SUBMIT);
```

4.如果还未对command进行编译即```alreadyCompiled＝false```, 则开始调用```compileInternal```编译command, 否则获取已有的QueryPlan并重设运行起始时间。

```java
int ret;
if (!alreadyCompiled) {
  ret = compileInternal(command);
  if (ret != 0) {
    return createProcessorResponse(ret);
  }
} else {
  // Since we're reusing the compiled plan, we need to update its start time for current run
  plan.setQueryStartTime(perfLogger.getStartTime(PerfLogger.DRIVER_RUN));
}
```

5.如果要求对任务进行读写加锁, 则从SessionState中获取HiveTxnManager并通过HiveTxnManager加锁。如果加锁失败则调用```releaseLocksAndCommitOrRollback```释放锁。hive中可以配置hive.support.concurrency值为true并设置zookeeper的服务器地址和端口，基于zookeeper实现分布式锁以支持hive的多并发访问。关于Hive的锁机制不在本文范围内,不过后续我单独介绍。

```java
// 通过ctx.setHiveTxnManager共享多个query之间的HiveTxnManager从而实现锁功能。
ctx.setHiveTxnManager(SessionState.get().getTxnMgr());

if (requiresLock()) {
  ret = acquireLocksAndOpenTxn();
  if (ret != 0) {
    try {
      releaseLocksAndCommitOrRollback(ctx.getHiveLocks(), false);
    } catch (LockException e) {
      // Not much to do here
    }
    return createProcessorResponse(ret);
  }
}
```

6.执行编译好后的query ```ret = execute();```,如果执行失败无论是否有锁都需要释放锁。

7.添加运行结束时间

```java
perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.DRIVER_RUN);
perfLogger.close(LOG, plan);
```

8.执行HiveDriverRunHook的运行后钩子

```java
    for (HiveDriverRunHook driverRunHook : driverRunHooks) {
        driverRunHook.postDriverRun(hookContext);
    }
```

从上述过程中不难看出, runInternal的核心流程就是两个compileInternal和execute.

#### 2.2.2 alreadyCompiled

在2.1节中提到两个run入口函数不同, 因为调用它们的类也是不同的, 分别是CliDriver和SQLOperation。

CliDriver由于alreadyCompiled为true所以会调用compileInternal进行编译, 而SQLOperation则是由于alreadyCompiled为flase, 则直接没有经过compileInternal而是获取已有的QueryPlan, 那么这个QueryPlan是在哪里生成的？

SQLOperation的runInternal只有两步:

```java
prepare(opConfig);
runQuery(opConfig); // 本质调用driver.run()
```

那么compile过程只有可能在```prepare```中:

```java
public void prepare(HiveConf sqlOperationConf) throws HiveSQLException {
    // 省略
    driver = new Driver(sqlOperationConf, getParentSession().getUserName());
    // 省略
    String subStatement = new VariableSubstitution().substitute(sqlOperationConf, statement);
    response = driver.compileAndRespond(subStatement);
    // 省略
}
public CommandProcessorResponse compileAndRespond(String command) {
  return createProcessorResponse(compileInternal(command));
}      
```

由此可见SQLOperation将compile过程独立出来放在了prepare阶段, execute过程放在了runQuery阶段.

只是我不知道为啥要这么设计？

### 2.3 HiveDriverRunHook

HiveDriverRunHook会在任务运行前和运行结束后分别调用prerDriverRun和postDriverRun来对任务实现钩子处理, 我们只需继承HiveDriverRunHook接口, 且设置配置hive.exec.driver.run.hooks(如果有多个钩子, 则按分号作为分隔符)即可。

```java
/**
 * HiveDriverRunHook allows Hive to be extended with custom logic for processing commands.
 * Note that the lifetime of an instantiated hook object is scoped to the analysis of a single
 * statement; hook instances are never reused.
 */
public interface HiveDriverRunHook extends Hook {
  /**
   * Invoked before Hive begins any processing of a command in the Driver,
   * notably before compilation and any customizable performance logging.
   */
  public void preDriverRun(
    HiveDriverRunHookContext hookContext) throws Exception;

  /**
   * Invoked after Hive performs any processing of a command, just before a
   * response is returned to the entity calling the Driver.
   */
  public void postDriverRun(
    HiveDriverRunHookContext hookContext) throws Exception;
}
```

### 2.4 compile

首先, 进行下说明本节主要讲解compile的方法流程, 不会深入介绍Hive的编译原理, 这将会放在[《Hive源码分析系列(3)之Compile原理分析》](<http://www.lamborryan.com/hive-src-compile>)重点介绍。



### 2.5 executor

## 3. Hook

## 4. 总结


## 参考文献

* [Hive Driver源码执行流程分析](<https://segmentfault.com/a/1190000002774731>)
* [Hive源码分析：Driver类运行过程](<http://blog.javachen.com/2013/08/22/hive-Driver.html>)


本文完

* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：[http://www.lamborryan.com/hive-src-driver](<http://www.lamborryan.com/hive-src-driver>)
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
