---
layout: post
title: Gobblin系列(7)之Source源码分析
date: 2016-03-28 13:30:00
categories: 大数据
tags: Gobblin
---

## 一.简介

Source在整个Gobblin任务流中负责以下三点:

1. 对数据源进行预切分获取```WorkUnits```。所谓预切分即是在不知道数据源是啥样的前提下, 划分好数据。
2. 为每一个```task```生成一个```extractor```, 一般情况下一个```task```对应一个```WorkUnit```(当然也存在多对多的情况), 从而实现对数据的摄取。
3. 提供```shutdown```处理机制, 即在job完成时候gobblin会回调```shutdown```函数, 我们可以在这里进行相应的逻辑处理。

这些功能都是通过Source接口实现的。

``` java
/*
 * An interface for classes that the end users implement to work with a data source from which
 * schema and data records can be extracted.
 * An implementation of this interface should contain all the logic required to work with a specific data source. This usually includes work determination and partitioning, and details of the connection protocol to work with the data source.
 */
public interface Source<S, D> {

  /**
   * Get a list of WorkUnits, each of which is for extracting a portion of the data.

   * Each WorkUnit will be used instantiate a WorkUnitState that gets passed to the
   * getExtractor(WorkUnitState) method to get an Extractor for extracting schema and data records
   * from the source. The WorkUnit instance should have all the properties needed for the Extractor
   * to work.

   * Typically the list of WorkUnits for the current run is determined by taking into account the
   * list of WorkUnits from the previous run so data gets extracted incrementally. The method  
   * SourceState.getPreviousWorkUnitStates can be used to get the list of WorkUnits from the
   * previous run.
   */
  public abstract List<WorkUnit> getWorkunits(SourceState state);

  /**
   * Get an Extractor based on a given WorkUnitState.

   * The Extractor returned can use WorkUnitState to store arbitrary key-value pairs that will be
   * persisted to the state store and loaded in the next scheduled job run.
   */
  public abstract Extractor<S, D> getExtractor(WorkUnitState state)
      throws IOException;

  /**
   * This method is called once when the job completes. Properties (key-value pairs) added to the
   * input SourceState instance will be persisted and available to the next scheduled job run
   * through the method getWorkunits(SourceState). If there is no cleanup or reporting required for
   * a particular implementation of this interface, then it is acceptable to have a default
   * implementation of this method.
   */
  public abstract void shutdown(SourceState state);
}
```

所有的数据源都是基于Source接口实现的, 通过继承并实现这三个方法我们就可以实现自己的Source了。

下图是 gobblin-0.6.2自带的Source的子类。

![img](../image/gobblin-source-1.png)

其中

1. ```SimpleJsonSource```和```WikipediaSource```都是自带的Source example
2. ```SourceDecorator```是Source的适配器, Gobblin-Runtime对source的调用都是通过```SourceDecorator```实现的。
3. 复杂点的Source都是从```AbstractSource```, ```AbstractSource```有啥作用, 它加入了对上一个job的```WorkUnits```处理, 如果上一个job周期内有```WorkUnits```处理失败了，这些```WorkUnits```就会加入到本次```WorkUnits```中，具体过程待我下节慢慢讲来。

目前使用较多的数据源主要是```MysqlSource```和```KafkaSimpleSource```。

本文主要以```AbstractSource```->```QueryBasedSource```－>```MysqlSource```这一继承关系为例来介绍Gobblin是如何实现Source的。由于关于```Extractor```将会在下篇文章中单独介绍, 因此本文主要介绍Source的```getWorkunits()```。

## 三.AbstractSource

Gobblin在获取WorkUnits时候不但会根据water marks来切分当前job的WorkUnits, 而且也会根据SourceState中获取上一个job的运行状态并根据策略配置选择是否要把上一个job的WorkUnits也加进去。(关于SourceState的介绍请看[《Gobblin系列六之State》](http://lamborryan.github.io/gobblin-state/))

因此AbstractSource就是在Source的基础上增加了获取上次job中需要在本次job重新运行的WorkUnits的方法。

> We use two keys for configuring work unit retries. The first one specifies whether work unit retries are enabled or not. This is for individual jobs or a group of jobs that following the same rule for work unit retries. The second one that is more advanced is for specifying a retry policy. This one is particularly useful for being a global policy for a group of jobs that have different job commit policies and want work unit retries only for a specific job commit policy. The first one probably is sufficient for most jobs that only need a way to enable/disable work unit retries. The second one gives users more flexibilities.

关于WorkUnitRetryPolicy策略有以下几种:

* ```WorkUnitRetryPolicy.ALWAYS```: 总会对失败或者异常的work units进行retry, 不管采用了何种提交策略。
* ```WorkUnitRetryPolicy.ONPARTIAL```: 只有当提交策略是COMMIT_ON_PARTIAL_SUCCESS时候才会失败或者异常的work units进行retry.
 该选项往往用在对具有不同提交策略的job group上,做为全局的Retry策略。
* ```WorkUnitRetryPolicy.ONFULL```: 只有当提交策略是COMMIT_ON_FULL_SUCCESS时候才会失败或者异常的work units进行retry.
  该选项往往用在对具有不同提交策略的job group上,做为全局的Retry策略。
* ```WorkUnitRetryPolicy.NEVER```: 从不对失败或者异常的work units进行retry.

```java
protected List<WorkUnitState> getPreviousWorkUnitStatesForRetry(SourceState state) {
  * * *
  // 获取retry策略
  WorkUnitRetryPolicy workUnitRetryPolicy;
  if (state.contains(ConfigurationKeys.WORK_UNIT_RETRY_POLICY_KEY)) {
    // Use the given work unit retry policy if specified
    workUnitRetryPolicy = WorkUnitRetryPolicy.forName(state.getProp(ConfigurationKeys.WORK_UNIT_RETRY_POLICY_KEY));
  } else {
    // 根据WORK_UNIT_RETRY_ENABLED_KEY这个配置来决定是否打开WorkUnitRetryPolicy策略
    boolean retryFailedWorkUnits = state.getPropAsBoolean(ConfigurationKeys.WORK_UNIT_RETRY_ENABLED_KEY, true);
    workUnitRetryPolicy = retryFailedWorkUnits ? WorkUnitRetryPolicy.ALWAYS : WorkUnitRetryPolicy.NEVER;
  }

  // 如果是never策略则返回空的workunit
  if (workUnitRetryPolicy == WorkUnitRetryPolicy.NEVER) {
    return ImmutableList.of();
  }

  List<WorkUnitState> previousWorkUnitStates = Lists.newArrayList();
  // 获取上一个job的没有成功的workunit。
  for (WorkUnitState workUnitState : state.getPreviousWorkUnitStates()) {
    if (workUnitState.getWorkingState() != WorkUnitState.WorkingState.COMMITTED) {
      if (state.getPropAsBoolean(ConfigurationKeys.OVERWRITE_CONFIGS_IN_STATESTORE,
          ConfigurationKeys.DEFAULT_OVERWRITE_CONFIGS_IN_STATESTORE)) {
        // We need to make a copy here since getPreviousWorkUnitStates returns ImmutableWorkUnitStates
        // for which addAll is not supported
        WorkUnitState workUnitStateCopy = new WorkUnitState(workUnitState.getWorkunit());
        workUnitStateCopy.addAll(workUnitState);
        workUnitStateCopy.overrideWith(state);
        previousWorkUnitStates.add(workUnitStateCopy);
      } else {
        previousWorkUnitStates.add(workUnitState);
      }
    }
  }

  // 如果是always策略, 则直接返回上一个job所有失败的workunits
  if (workUnitRetryPolicy == WorkUnitRetryPolicy.ALWAYS) {
    return previousWorkUnitStates;
  }

  // 获取提交策略，默认是全部提交。
  JobCommitPolicy jobCommitPolicy = JobCommitPolicy
      .forName(state.getProp(ConfigurationKeys.JOB_COMMIT_POLICY_KEY, ConfigurationKeys.DEFAULT_JOB_COMMIT_POLICY));

  // 根据提交策略和retry策略来决定是否需要retry上一个job失败的workunits
  if ((workUnitRetryPolicy == WorkUnitRetryPolicy.ON_COMMIT_ON_PARTIAL_SUCCESS
      && jobCommitPolicy == JobCommitPolicy.COMMIT_ON_PARTIAL_SUCCESS)
      || (workUnitRetryPolicy == WorkUnitRetryPolicy.ON_COMMIT_ON_FULL_SUCCESS
          && jobCommitPolicy == JobCommitPolicy.COMMIT_ON_FULL_SUCCESS)) {
    return previousWorkUnitStates;
  } else {
    // Return an empty list if job commit policy and work unit retry policy do not match
    return ImmutableList.of();
  }
}
```

由代码可以看出, 对于上一个失败的workunits, 要么全部retry, 要么都不retry, 在颗粒度比较粗。

同时AbstractSource还会生成Extract State. 相同的namespace和table具有相同的Extract。比如kafka sourc, 虽然只是一次job, 但是有可能因为存在多个topic从而产生了不同的Extract, 即每一个top的Extract id不同。而Extract不同的一个好处是可以使用不同的发布策略。

``` java
public Extract createExtract(TableType type, String namespace, String table) {
    return this.extractFactory.getUniqueExtract(type, namespace, table);
}
```

## 四.QueryBasedSource

QueryBasedSource在AbstractSource基础上又实现了query－based的getWorkunits， 已经具有很鲜明的sql特点了。
本小节主要介绍QueryBasedSource如何通过getWorkunits来获取WorkUnits.

QueryBasedSource的getWorkunits需要解决以下几个问题:

* 怎么获取上一个job的Latest Water mark, 从而可以做为本次job的low Water mark
* 怎么进行partition

因此分为两小节来分别介绍。

### 4.1. getLatestWatermarkFromMetadata

* 如果commit policy是full且有task失败了, 则Latest Water mark是所有WorkUnits中最小的low water mark。
* 如果commit policy是full且所有task成功, 则Latest Water mark是所有WorkUnits中最大的high water mark。
* 如果commit policy不是full且有task成功, 则Latest Water mark是所有成功的WorkUnits中最大的high water mark。失败的task由retry policy控制。
* 如果commit policy不是full且所有task都失败, 则Latest Water mark是所有WorkUnits中最小的low water mark。

> 这里我有疑惑: 如果设置了retry policy = JobCommitPolicy.COMMIT_ON_FULL_SUCCESS, retry policy会把上一个job的failed workunit加入到新job的workunits。 而getLatestWatermarkFromMetadata又会计算上一个job的最小的watermark，从而再次计算这些workunit。使得上一job的workunits重新跑两次。

``` java
private long getLatestWatermarkFromMetadata(SourceState state) {
   ...
   boolean hasFailedRun = false;
   boolean isCommitOnFullSuccess = false;
   boolean isDataProcessedInPreviousRun = false;

   JobCommitPolicy commitPolicy = JobCommitPolicy
       .forName(state.getProp(ConfigurationKeys.JOB_COMMIT_POLICY_KEY, ConfigurationKeys.DEFAULT_JOB_COMMIT_POLICY));
   if (commitPolicy == JobCommitPolicy.COMMIT_ON_FULL_SUCCESS) {
     isCommitOnFullSuccess = true;
   }

   for (WorkUnitState workUnitState : previousWorkUnitStates) {
     long processedRecordCount = 0;
     if (workUnitState.getWorkingState() == WorkingState.FAILED
         || workUnitState.getWorkingState() == WorkingState.CANCELLED
         || workUnitState.getWorkingState() == WorkingState.RUNNING
         || workUnitState.getWorkingState() == WorkingState.PENDING) {
       hasFailedRun = true;
     } else {
       processedRecordCount = workUnitState.getPropAsLong(ConfigurationKeys.EXTRACTOR_ROWS_EXPECTED);
       if (processedRecordCount != 0) {
         isDataProcessedInPreviousRun = true;
       }
     }

     // Consider high water mark of the previous work unit, if it is
     // extracted any data
     if (processedRecordCount != 0) {
       previousWorkUnitStateHighWatermarks.add(workUnitState.getHighWaterMark());
     }

     previousWorkUnitLowWatermarks.add(this.getLowWatermarkFromWorkUnit(workUnitState));
   }

   // If commit policy is full and it has failed run, get latest water mark
   // as
   // minimum of low water marks from previous states.
   if (isCommitOnFullSuccess && hasFailedRun) {
     long previousLowWatermark = Collections.min(previousWorkUnitLowWatermarks);

     WorkUnitState previousState = previousWorkUnitStates.get(0);
     ExtractType extractType =
         ExtractType.valueOf(previousState.getProp(ConfigurationKeys.SOURCE_QUERYBASED_EXTRACT_TYPE).toUpperCase());

     // add backup seconds only for snapshot extracts but not for appends
     if (extractType == ExtractType.SNAPSHOT) {
       int backupSecs = previousState.getPropAsInt(ConfigurationKeys.SOURCE_QUERYBASED_LOW_WATERMARK_BACKUP_SECS, 0);
       String watermarkType = previousState.getProp(ConfigurationKeys.SOURCE_QUERYBASED_WATERMARK_TYPE);
       latestWaterMark = this.addBackedUpSeconds(previousLowWatermark, backupSecs, watermarkType);
     } else {
       latestWaterMark = previousLowWatermark;
     }
   }

   // If commit policy is full and there are no failed tasks or commit
   // policy is partial,
   // get latest water mark as maximum of high water marks from previous
   // tasks.
   else {
     if (isDataProcessedInPreviousRun) {
       latestWaterMark = Collections.max(previousWorkUnitStateHighWatermarks);
     } else {
       latestWaterMark = Collections.min(previousWorkUnitLowWatermarks);
     }
   }

   return latestWaterMark;
 }
```

### 4.2. Partitioner

Partitioner 的作用就是根据latestWaterMark来对数据进行partition, 并得到WorkUnits. Partition过程涉及到以下几个变量:

* interval, partition时候的最小单位, 由source.querybased.partition.interval获取。
* maxPartitions, partition的最大个数, 由配置source.max.number.of.partitions获取。
* lowWatermark, 根据previousWatermark(latestWaterMark)获取partition的左边界。
* highWatermark, 计算partition的右边界。
* source.querybased.watermark.type决定了是以哪种类型进行partition。

如果lowWatermark或者highWatermark等于DEFAULT_WATERMARK_VALUE, 则只会形成一个partition。

``` java
public HashMap<Long, Long> getPartitions(long previousWatermark) {
   HashMap<Long, Long> defaultPartition = new HashMap<Long, Long>();
   ...
   // extract type 比如snapshot等
   ExtractType extractType =
       ExtractType.valueOf(this.state.getProp(ConfigurationKeys.SOURCE_QUERYBASED_EXTRACT_TYPE).toUpperCase());
    // watermarkType 类型 比如 timestamp date hour simple
   WatermarkType watermarkType =
       WatermarkType.valueOf(this.state.getProp(ConfigurationKeys.SOURCE_QUERYBASED_WATERMARK_TYPE,
           ConfigurationKeys.DEFAULT_WATERMARK_TYPE).toUpperCase());
   // 分区步长,最小单位
   int interval =
       this.getUpdatedInterval(this.state.getPropAsInt(ConfigurationKeys.SOURCE_QUERYBASED_PARTITION_INTERVAL, 0),
           extractType, watermarkType);
   int sourceMaxAllowedPartitions = this.state.getPropAsInt(ConfigurationKeys.SOURCE_MAX_NUMBER_OF_PARTITIONS, 0);
   // 最大可以分区的个数
   int maxPartitions =
       (sourceMaxAllowedPartitions != 0 ? sourceMaxAllowedPartitions
           : ConfigurationKeys.DEFAULT_MAX_NUMBER_OF_PARTITIONS);

   WatermarkPredicate watermark = new WatermarkPredicate(null, watermarkType);
   int deltaForNextWatermark = watermark.getDeltaNumForNextWatermark();

   // 可以分区的最小watermark
   long lowWatermark = this.getLowWatermark(extractType, watermarkType, previousWatermark, deltaForNextWatermark);
   // 可以分区的最大watermark
   long highWatermark = this.getHighWatermark(extractType, watermarkType);
   // 如果最小watermark或者最大watermark为－1, 则只有一个分区
   if (lowWatermark == ConfigurationKeys.DEFAULT_WATERMARK_VALUE
       || highWatermark == ConfigurationKeys.DEFAULT_WATERMARK_VALUE) {
     defaultPartition.put(lowWatermark, highWatermark);
     return defaultPartition;
   }
   // 根据相应的watertype进行分区, 实际上调用的是watermark的getIntervals
   return watermark.getPartitions(lowWatermark, highWatermark, interval, maxPartitions);
 }

```
这里要介绍下source.querybased.watermark.type这个配置, 它决定了是以哪种类型来进行partition。默认支持TIMESTAMP, DATE, HOUR, SIMPLE. 所谓的SIMPLE, 其实就是整数, Gobblin会根据这个type来决定哪种watermark。

```java
public WatermarkPredicate(String watermarkColumn, WatermarkType watermarkType) {
    super();
    this.watermarkColumn = watermarkColumn;
    this.watermarkType = watermarkType;

    switch (watermarkType) {
    case TIMESTAMP:
      this.watermark = new TimestampWatermark(watermarkColumn, DEFAULT_WATERMARK_VALUE_FORMAT);
      break;
    case DATE:
      this.watermark = new DateWatermark(watermarkColumn, DEFAULT_WATERMARK_VALUE_FORMAT);
      break;
    case HOUR:
      this.watermark = new HourWatermark(watermarkColumn, DEFAULT_WATERMARK_VALUE_FORMAT);
      break;
    case SIMPLE:
      this.watermark = new SimpleWatermark(watermarkColumn, DEFAULT_WATERMARK_VALUE_FORMAT);
      break;
    default:
      this.watermark = new SimpleWatermark(watermarkColumn, DEFAULT_WATERMARK_VALUE_FORMAT);
      break;
    }
}
```

假设我们选择了TIMESTAMP, 则gobblin会实例化TimestampWatermark来进行partition。

```java
synchronized public HashMap<Long, Long> getIntervals(long lowWatermarkValue, long highWatermarkValue, long partitionInterval, int maxIntervals) {
    ...
    HashMap<Long, Long> intervalMap = new HashMap<Long, Long>();
    Date startTime = new Date(lowWatermark);
    Date endTime = new Date(highWatermark);
    LOG.debug("Sart time:" + startTime + "; End time:" + endTime);
    long lwm;
    long hwm;
    while (startTime.getTime() <= endTime.getTime()) {
      lwm = Long.parseLong(INPUTFORMATPARSER.format(startTime));
      calendar.setTime(startTime);
      calendar.add(Calendar.HOUR, (int) interval);
      nextTime = calendar.getTime();
      hwm = Long.parseLong(INPUTFORMATPARSER.format(nextTime.getTime() <= endTime.getTime() ? nextTime : endTime));
      intervalMap.put(lwm, hwm);
      LOG.debug("Partition - low:" + lwm + "; high:" + hwm);
      calendar.add(Calendar.SECOND, deltaForNextWatermark);
      startTime = calendar.getTime();
    }

    return intervalMap;
}

```
需要说明一点就是, 如果watertype是timestamp,date,hour的highwatermark是当前时间, 如果是simple且snapshot的话, highwatermark是－1, 也就是说如果是simple的话只会有一个partition。

### 4.3. 总结

最后回顾下getWorkunits的过程

``` java
public List<WorkUnit> getWorkunits(SourceState state) {
    ...
    // 获取上一个job的latest watermark
    long previousWatermark = this.getLatestWatermarkFromMetadata(state);

    // 根据latest watermark进行partition,并作排序
    Map<Long, Long> sortedPartitions = Maps.newTreeMap();
    sortedPartitions.putAll(new Partitioner(state).getPartitions(previousWatermark));

    // Use extract table name to create extract
    SourceState partitionState = new SourceState();
    partitionState.addAll(state);
    Extract extract = partitionState.createExtract(tableType, nameSpaceName, extractTableName);

    // Setting current time for the full extract
    if (Boolean.valueOf(state.getProp(ConfigurationKeys.EXTRACT_IS_FULL_KEY))) {
      extract.setFullTrue(System.currentTimeMillis());
    }

    // 将各paritions的watermark写入到workunits
    for (Entry<Long, Long> entry : sortedPartitions.entrySet()) {
      partitionState.setProp(ConfigurationKeys.WORK_UNIT_LOW_WATER_MARK_KEY, entry.getKey());
      partitionState.setProp(ConfigurationKeys.WORK_UNIT_HIGH_WATER_MARK_KEY, entry.getValue());
      workUnits.add(partitionState.createWorkUnit(extract));
    }

    // 将上一个job失败需要retry的workunit添加到本job的workunit中
    List<WorkUnit> previousWorkUnits = this.getPreviousWorkUnitsForRetry(state);
    LOG.info("Total number of incomplete tasks from the previous run: " + previousWorkUnits.size());
    workUnits.addAll(previousWorkUnits);

    return workUnits;
  }
```

## 五.MysqlSource

最后MysqlSource只实现了getExtractor接口，即返回一个getExtractor对象.

```java
public class MysqlSource extends QueryBasedSource<JsonArray, JsonElement> {
    private static final Logger LOG = LoggerFactory.getLogger(MysqlSource.class);

    public Extractor<JsonArray, JsonElement> getExtractor(WorkUnitState state)
      throws IOException {
        Extractor<JsonArray, JsonElement> extractor = null;
        try {
          extractor = new MysqlExtractor(state).build();
        } catch (ExtractPrepareException e) {
          LOG.error("Failed to prepare extractor: error - " + e.getMessage());
          throw new IOException(e);
        }
        return extractor;
    }
}
```

## 总结

本文主要介绍了Source的几个重要功能, 以MysqlSource为例介绍了Source是如何getWorkunits的, 在这过程中又结合watermark简单描述了gobblin的retry策略。

本文完


* 原创文章，转载请注明： 转载自[Lamborryan](<http://lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/gobblin-source
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
