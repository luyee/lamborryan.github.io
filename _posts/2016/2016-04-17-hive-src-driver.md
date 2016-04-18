---
layout: post
title: Hive源码分析系列(2)之Driver流程分析
date: 2016-04-17 23:30:00
categories: 大数据
tags: Hive源码分析
---

## 1. 简介

本文接着上文[《Hive源码分析系列(1)之框架与Client/Server入口》](<http://www.lamborryan.com/hive-src-entrance/>)讲, 上文讲到无论Hive是哪种方式运行(Cli, beeline, HiveServer2), 流程最后都汇总到Driver类. 它肩负着将用户的command进行编译，优化并生成MapReduce任务进行执行这一使命。所以Driver也是Hive的核心，主要的逻辑处理就在这里了, 相当于主线程。

本文主要来扒一扒Driver到底做了啥。

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

尽管如此, 但是仔细一看两个函数还是不同的, ```run(command, false)```和```run(null, true)```, 前者有查询命令commnad且设置alreadyCompiled为flase, 而后者查询command设为null且设置alreadyCompiled为true。一时好奇我跟踪了代码, 发现前者```run(command, false)```是由CliDriver和HiveCommandOperation调用的, 而后者```run(null, true)```是由SQLOperation调用的。关于这三个类的细节请查看[《Hive源码分析系列(1)之框架与Client/Server入口》](<http://www.lamborryan.com/hive-src-entrance/>)。

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

6.执行编译好后的查询```ret = execute();```,如果执行失败无论是否有锁都需要释放锁。

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

CliDriver由于alreadyCompiled为flase所以会调用compileInternal进行编译, 而SQLOperation则是由于alreadyCompiled为true, 则直接没有经过compileInternal而获取已有的QueryPlan, 那么这个QueryPlan是在哪里生成的？

SQLOperation的runInternal只有两步:

```java
prepare(opConfig);  // 本质调用driver.compileInternal()
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

由此可见SQLOperation将compile过程独立出来放在了prepare阶段(当然也加了另外的处理过程), execute过程放在了runQuery阶段.

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

首先, 说明下本节主要讲解compile的方法流程, 不会深入介绍Hive的编译原理, 这将会放在[《Hive源码分析系列(3)之Compile原理分析》](<http://www.lamborryan.com/hive-src-compile>)重点介绍。

1、保存当前查询状态, 将session的信息保存成QueryState.

```java
QueryState queryState = new QueryState();

if (plan != null) {
  close();
  plan = null;
}

if (resetTaskIds) {
  TaskFactory.resetId();
}
saveSession(queryState);
```

2.设置上下文变量

```java
command = new VariableSubstitution().substitute(conf,command);
ctx = new Context(conf);
ctx.setTryCount(getTryCount());
ctx.setCmd(command);
ctx.setHDFSCleanup(true);
```

3.创建ParseDriver对象，然后解析命令、生成AST树。语法和词法分析内容，不是本文重点故不做介绍。

```java
perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.PARSE);
ParseDriver pd = new ParseDriver();
ASTNode tree = pd.parse(command, ctx);
tree = ParseUtils.findRootNonNullToken(tree);
perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.PARSE);
```

简单归纳来说，解析程包括如下：

* 词法分析，生成AST树，ParseDriver完成。
* 分析AST树，AST拆分成查询子块，信息记录在QB，这个QB在下面几个阶段都需要用到,SemanticAnalyzer.doPhase1完成。
* 从metastore中获取表的信息，SemanticAnalyzer.getMetaData完成。
* 生成逻辑执行计划，SemanticAnalyzer.genPlan完成。
* 优化逻辑执行计划，Optimizer完成，ParseContext作为上下文信息进行传递。
* 生成物理执行计划，SemanticAnalyzer.genMapRedTasks完成。
* 物理计划优化，PhysicalOptimizer完成，PhysicalContext作为上下文信息进行传递。

![img](../image/hive/hive-src-driver/compile.png)

4.语法分析与hook.

通过```SemanticAnalyzerFactory.get(conf, tree)```生成分析器, 随后通过```sem.analyze(tree, ctx)```进行语法分析。如果还设置了```HiveSemanticAnalyzerHook```这个语法分析hook,就会分别调用```HiveSemanticAnalyzerHook.preAnalyze```和```HiveSemanticAnalyzerHook.postAnalyze```对语法分析前和语法分析后实现用户自定义操作。同样支持hook chain。

```java
perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.ANALYZE);
BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(conf, tree);
List<HiveSemanticAnalyzerHook> saHooks =
 getHooks(HiveConf.ConfVars.SEMANTIC_ANALYZER_HOOK,
     HiveSemanticAnalyzerHook.class);

// Do semantic analysis and plan generation
if (saHooks != null) {
    HiveSemanticAnalyzerHookContext hookCtx = new HiveSemanticAnalyzerHookContextImpl();
    hookCtx.setConf(conf);
    hookCtx.setUserName(userName);
    hookCtx.setIpAddress(SessionState.get().getUserIpAddress());
    hookCtx.setCommand(command);
    for (HiveSemanticAnalyzerHook hook : saHooks) {
        tree = hook.preAnalyze(hookCtx, tree);
    }
    sem.analyze(tree, ctx);
    hookCtx.update(sem);
    for (HiveSemanticAnalyzerHook hook : saHooks) {
        hook.postAnalyze(hookCtx, sem.getRootTasks());
    }
} else {
    sem.analyze(tree, ctx);
}
```

这里还做了```acidSinks = sem.getAcidFileSinks();```处理, 暂时不知道是做啥的。

5.检查执行计划：```sem.validate()```

6.创建查询计划QueryPlan。

```java
// Command should be redacted before passing it to the QueryPlan in order
// to avoid returning sensitive data
String queryStr = HookUtils.redactLogString(conf, command);

plan = new QueryPlan(queryStr, sem, perfLogger.getStartTime(PerfLogger.DRIVER_RUN), queryId,
    SessionState.get().getCommandType());

conf.setVar(HiveConf.ConfVars.HIVEQUERYSTRING, queryStr);

conf.set("mapreduce.workflow.id", "hive_" + queryId);
conf.set("mapreduce.workflow.name", queryStr);
```

7.初始化FetchTask。

```java
// initialize FetchTask right here
if (plan.getFetchTask() != null) {
    plan.getFetchTask().initialize(conf, plan, null);
}
```

8.得到schema ```schema = getSchema(sem, conf);```

9.如果设置```hive.security.authorization.enabled```设置为true, 则还需要进行权限验证。

```java
//do the authorization check
if (!sem.skipAuthorization() &&
    HiveConf.getBoolVar(conf, HiveConf.ConfVars.HIVE_AUTHORIZATION_ENABLED)) {

  try {
    perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.DO_AUTHORIZATION);
    doAuthorization(sem, command);
  } catch (AuthorizationException authExp) {
    console.printError("Authorization failed:" + authExp.getMessage()
        + ". Use SHOW GRANT to get more details.");
    errorMessage = authExp.getMessage();
    SQLState = "42000";
    return 403;
  } finally {
    perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.DO_AUTHORIZATION);
  }
}
```

本节只是简单介绍了Compile的流程, 关于更详细的分析将在下一篇文章中介绍。

### 2.5 executor

本节同样主要介绍executor的流程, 其中细节将在后续文章中展开。

1.抛开各种配置内容(比如query id, query str, hive.exec.parallel.thread.number等等)的获取与处理, 真正的执行流程是从```plan.setStarted();```开始的。

2.同样在执行前加入hive.exec.pre.hooks, 分为两种ExecuteWithHookContext和PreExecute。

3.获取job个数,job名字。

4.随着DriverContext执行上下文的初始化, 执行过程正式拉开帷幕。

它记录的信息主要包括可执行的任务队列(Queue runnable), 正在运行的任务队列(Queue running), 当前启动的任务数curJobNo, statsTasks（Map<String, StatsTask>）以及语义分析Semantic Analyzers依赖的Context对象等。

```java
// A runtime that launches runnable tasks as separate Threads through
// TaskRunners
// As soon as a task isRunnable, it is put in a queue
// At any time, at most maxthreads tasks can be running
// The main thread polls the TaskRunners to check if they have finished.

DriverContext driverCxt = new DriverContext(ctx);
driverCxt.prepare(plan);

public DriverContext(Context ctx) {
  this.runnable = new ConcurrentLinkedQueue<Task<? extends Serializable>>();
  this.running = new LinkedBlockingQueue<TaskRunner>();
  this.ctx = ctx;
}

public void prepare(QueryPlan plan) {
  // extract stats keys from StatsTask
  List<Task<?>> rootTasks = plan.getRootTasks();
  NodeUtils.iterateTask(rootTasks, StatsTask.class, new Function<StatsTask>() {
    public void apply(StatsTask statsTask) {
      statsTasks.put(statsTask.getWork().getAggKey(), statsTask);
    }
  });
}
```

5.DriverContext获取QueryPlan, 并将查询计划的root节点任务放入runable的队列, 等待任务的启动。

```java
// Add root Tasks to runnable
for (Task<? extends Serializable> tsk : plan.getRootTasks()) {
  // This should never happen, if it does, it's a bug with the potential to produce
  // incorrect results.
  assert tsk.getParentTasks() == null || tsk.getParentTasks().isEmpty();
  driverCxt.addToRunnable(tsk);
}
```

6.开始一个while循环, 通过```driverCxt.isRunning()```来判断driverCxt是否在处于运行状态。判断标准很简单, 只要driverCxt没有被关闭, 且```runnable```和```running```有任务存在即可。

```java
// Loop while you either have tasks running, or tasks queued up
while (!destroyed && driverCxt.isRunning()) {
    // 代码省略
}

public synchronized boolean isRunning() {
  return !shutdown && (!running.isEmpty() || !runnable.isEmpty());
}
```

6.1.从runable获取待执行任务,并执行任务。

```java
// Launch upto maxthreads tasks
Task<? extends Serializable> task;
while ((task = driverCxt.getRunnable(maxthreads)) != null) {
  perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.TASK + task.getName() + "." + task.getId());
  TaskRunner runner = launchTask(task, queryId, noName, jobname, jobs, driverCxt);
  if (!runner.isRunning()) {
    break;
  }
}
```

获取待执行的任务有两个前提, runable队列不为空,running队列的size小于maxthreads(hive的最大任务并行数)。

```java
public synchronized Task<? extends Serializable> getRunnable(int maxthreads) throws HiveException {
  checkShutdown();
  if (runnable.peek() != null && running.size() < maxthreads) {
    return runnable.remove();
  }
  return null;
}
```

获取到任务后通过launchTask进行提交, 实际上task是在TaskRunner内运行。

```java
private TaskRunner launchTask(Task<? extends Serializable> tsk, String queryId, boolean noName,
    String jobname, int jobs, DriverContext cxt) throws HiveException {
  if (SessionState.get() != null) {
    SessionState.get().getHiveHistory().startTask(queryId, tsk, tsk.getClass().getName());
  }
  // 判断任务类型MapReduceTask还是ConditionalTask
  if (tsk.isMapRedTask() && !(tsk instanceof ConditionalTask)) {
    if (noName) {
      conf.setVar(HiveConf.ConfVars.HADOOPJOBNAME, jobname + "(" + tsk.getId() + ")");
    }
    conf.set("mapreduce.workflow.node.name", tsk.getId());
    Utilities.setWorkflowAdjacencies(conf, plan);
    // 当前已启动任务数curJobNo加1
    cxt.incCurJobNo(1);
    console.printInfo("Launching Job " + cxt.getCurJobNo() + " out of " + jobs);
  }
  // 根据HiveConf,QueryPlan, DriverContext 初始化任务
  tsk.initialize(conf, plan, cxt);
  // 任务结果对象
  TaskResult tskRes = new TaskResult();
  // 任务执行对象
  TaskRunner tskRun = new TaskRunner(tsk, tskRes);
  // 将任务添加到running队列中, 表示该task正在运行
  cxt.launching(tskRun);
  // Launch Task
  if (HiveConf.getBoolVar(conf, HiveConf.ConfVars.EXECPARALLEL) && tsk.isMapRedTask()) {
    // Launch it in the parallel mode, as a separate thread only for MR tasks
    if (LOG.isInfoEnabled()){
      LOG.info("Starting task [" + tsk + "] in parallel");
    }
    tskRun.setOperationLog(OperationLog.getCurrentOperationLog());
    // 并行执行
    tskRun.start();
  } else {
    if (LOG.isInfoEnabled()){
      LOG.info("Starting task [" + tsk + "] in serial mode");
    }
    // 顺序执行
    tskRun.runSequential();
  }
  return tskRun;
}
```

6.2.每隔```SLEEP_TIME```时间(默认2秒), 检查running队列的任务是否已经完成任务, 如果完成就从running队列中删掉.并将完成的任务添加到钩子上下文HookContext的comlepltetTask数组中。

```java
// poll the Tasks to see which one completed
TaskRunner tskRun = driverCxt.pollFinished();
if (tskRun == null) {
  continue;
}
hookContext.addCompleteTask(tskRun);

/**
 * Polls running tasks to see if a task has ended.
 *
 * @return The result object for any completed/failed task
 */
public synchronized TaskRunner pollFinished() throws InterruptedException {
  while (!shutdown) {
    Iterator<TaskRunner> it = running.iterator();
    while (it.hasNext()) {
      TaskRunner runner = it.next();
      if (runner != null && !runner.isRunning()) {
        it.remove();
        return runner;
      }
    }
    wait(SLEEP_TIME);
  }
  return null;
}
```

6.3.检查任务的完成状态,并作相应处理

针对一个已完成的任务，首先获取任务的结果对象TaskResult和退出状态, 如果任务非正常退出，则第一步先判断任务是否支持Retry，如果支持，关闭当前DriverContext，设置jobTracker为初始状态，抛出CommandNeedRetry异常，这个异常会在CliDriver的processLocalCmd中捕获，然后尝试重新处理该命令(如果是通过远程调用,SQLOperations是不支持retry的)。如果任务不支持Retry，则启动备份任务backupTask，并添加到runnable队列，在下次循环过程中执行。如果没有backupTask，则查找用户配置“hive.exec.failure.hooks”,根据用户配置相应出错处理，并关闭DriverContext，返回退出码。

```java
Task<? extends Serializable> tsk = tskRun.getTask();
TaskResult result = tskRun.getTaskResult();

int exitVal = result.getExitVal();
if (exitVal != 0) {
  if (tsk.ifRetryCmdWhenFail()) {
    driverCxt.shutdown();
    // in case we decided to run everything in local mode, restore the
    // the jobtracker setting to its initial value
    ctx.restoreOriginalTracker();
    throw new CommandNeedRetryException();
  }
  Task<? extends Serializable> backupTask = tsk.getAndInitBackupTask();
  if (backupTask != null) {
    setErrorMsgAndDetail(exitVal, result.getTaskError(), tsk);
    console.printError(errorMessage);
    errorMessage = "ATTEMPT: Execute BackupTask: " + backupTask.getClass().getName();
    console.printError(errorMessage);

    // add backup task to runnable
    if (DriverContext.isLaunchable(backupTask)) {
      driverCxt.addToRunnable(backupTask);
    }
    continue;

  } else {
    hookContext.setHookType(HookContext.HookType.ON_FAILURE_HOOK);
    // Get all the failure execution hooks and execute them.
    for (Hook ofh : getHooks(HiveConf.ConfVars.ONFAILUREHOOKS)) {
      perfLogger.PerfLogBegin(CLASS_NAME, PerfLogger.FAILURE_HOOK + ofh.getClass().getName());

      ((ExecuteWithHookContext) ofh).run(hookContext);

      perfLogger.PerfLogEnd(CLASS_NAME, PerfLogger.FAILURE_HOOK + ofh.getClass().getName());
    }
    setErrorMsgAndDetail(exitVal, result.getTaskError(), tsk);
    SQLState = "08S01";
    console.printError(errorMessage);
    driverCxt.shutdown();
    // in case we decided to run everything in local mode, restore the
    // the jobtracker setting to its initial value
    ctx.restoreOriginalTracker();
    return exitVal;
  }
}
```

6.4.获取子任务

最后调用DriverContext的finished函数，对完成的任务进行处理（主要进行相应的统计处理）， 然后判断当前任务是否包含子任务，如果包含则依次将子任务添加到runnable队列，下次循环中被启动执行。

```java
driverCxt.finished(tskRun);

if (tsk.getChildTasks() != null) {
 for (Task<? extends Serializable> child : tsk.getChildTasks()) {
   if (DriverContext.isLaunchable(child)) {
     driverCxt.addToRunnable(child);
   }
 }
}
```

至此, 循环内的流程已经写完, 需要做下以下总结:

QueryPlan内的task是存在依赖关系的, 存在父任务和子任务. 所以在初始化的时候是先将root任务先放到runnable队列中, 大循环内每完成一个task, 都会将该task的子任务放到runnable队列中。

可以用以下流程图来展示下:

![img](../image/hive/hive-src-driver/excutor-1.png)

7.任务结束前的处理

当所有的任务都完成之后，如果发现DriverContext已经被关闭，表明任务取消，打印信息并返回对应的状态码。最后清除任务执行中不完整的输出，并加载执行用户指定的"hive.exec.post.hooks"，完成对应的钩子功能。对于执行过程中出现的异常，CommandNeedRetryException将会直接向上抛出，其他Exception，直接打印出错信息。无论是否发生异常，只要能够获取到任务执行过程中的MapReduce状态信息，都将在finally语句块中打印。（限于篇幅，此处只给出部分代码，钩子的处理方式前文已经给出不再详述，异常处理的部分，有兴趣的执行查看）

```java
//判断DriverContext是否被关闭
if (driverCxt.isShutdown()) {
        SQLState = "HY008";
        errorMessage = "FAILED: Operation cancelled";
        console.printError(errorMessage);
        return 1000;
}

//删除不完整的输出
HashSet<WriteEntity> remOutputs = new HashSet<WriteEntity>();
      for (WriteEntity output : plan.getOutputs()) {
        if (!output.isComplete()) {
          remOutputs.add(output);
        }
 }

 for (WriteEntity output : remOutputs) {
     plan.getOutputs().remove(output);
 }
```

最后通过输出```plan.setDone();```来结束executor过程。

## 3. 总结

本文主要介绍了Driver的核心流程, Compile和executor都是很大很复杂的模块, 所以将在后续文章中展开讲。

![img](../image/hive/hive-src-driver/driver-2.jpg)


## 参考文献

* [Hive Driver源码执行流程分析](<https://segmentfault.com/a/1190000002774731>)
* [Hive源码分析：Driver类运行过程](<http://blog.javachen.com/2013/08/22/hive-Driver.html>)


本文完

* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：[http://www.lamborryan.com/hive-src-driver](<http://www.lamborryan.com/hive-src-driver>)
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
