---
layout: post
title: Hadoop系列(3)之MapReduce原理及应用举例
date: 2016-01-18 10:30:35
categories: 大数据
tags: Hadoop
---

## 1. 简介

什么是MapReduce

MapReduce是一个针对大规模群组中的海量数据处理的分布式编程模型。

* 我们要数图书馆中的所有书。你数1号书架，我数2号书架。这就是“Map”。我们人越多，数书就更快。
* 现在我们到一起，把所有人的统计数加起来。这就是“Reduce”

## 2. 第一代MapReduce框架

第一代MapReduce采用Master/Slave架构，主要由以下组件组成:Client,JobTracker,TaskTracker,Task.

![img](../image/hadoop/mapreduce-1.png)

* Client。用户编写MapReduce程序通过Client提交到JobTracker端，同时用户通过Client提供一些接口查看作业运行状态。Client启动作业后，向JobTracker请求Job ID,将运行作业所需要的资源文件复制到HDFS上，包括MapReduce程序打包的JAR文件、配置文件和客户端计算所得的输入划分信息。这些文件都存放在JobTracker专门为该作业创建的文件夹中。同时检查作业的输出说明，如果没有指定输出目录或者已经存在，作业就会提交失败。计算作业的输入划分，如果划分无法计算，比如输入路径不存在，作业就会提交失败。
* JobTracker。主要负责资源监控和作业调度，JobTracker监控所有TaskTracker与作业健康状态，一旦发现失败情况会将相应任务转移到其他节点；同时，JobTracker会跟踪任务的执行进度，资源使用量等信息，并将这些信息告诉任务调度器，调度器会在资源出现空闲时，选择合适的任务使用这些资源。
* TaskTracker。TaskTracker会周期性地通过HeartBeat将本节点上资源的使用情况和任务的运行进度汇报给JobTracker，同时接受JobTracker发送过来的命令并执行相应的操作(如启动新任务，杀死任务等)
* Task。Task分为Map Task 和 Reduce Task两种，均由TaskTracker启动，对于MapReduce，一个Split就是一个MapTask。
    * Mask Task流程：MapTask先将对于的Split迭代解析城一个个Key/Value，依次调用用户自定义的Map()函数进行处理，最近将临时结果存放在本地磁盘上。临时文件分成多个Partition，每个Partition对应一个ReduceTask处理
    * Reduce Task流程：
        * 从远处节点上读取MapTask中间结果 称为Shuffle阶段
        * 按照Key对Key/Value进行排序，称为Sort阶段
        * 依次读取<Key，Value List>，调用用户自定义的Reduce()函数处理，并将结果存在HDFS上，称为Reduce阶段。

Mask Task流程图:
![img](../image/hadoop/mapreduce-2.png)
Reduce Task流程图:
![img](../image/hadoop/mapreduce-3.png)

## 3. MapReduce流程

![img](../image/hadoop/mapreduce-4.png)

MapReduce编程框架

![img](../image/hadoop/mapreduce-5.png)

### 3.1 InputFormt

InputFormat主要负责两个功能：

1.数据切分，按照某个策略将输入数据切分成若干个Split，以便确定MapTask个数以及对应的Split。数据切分工作在Job初始化的时候完成。InputFormt有两个接口：getSplits负责切分数据并选出对应的Host节点，createRecordReader读取Split的接口，这个接口是解析Key和Value。

```java
public abstract class InputFormat<K, V> {
  /**
   * Logically split the set of input files for the job.  
   *
   * <p>Each {@link InputSplit} is then assigned to an individual {@link Mapper}
   * for processing.</p>
   *
   * <p><i>Note</i>: The split is a <i>logical</i> split of the inputs and the
   * input files are not physically split into chunks. For e.g. a split could
   * be <i><input-file-path, start, offset></i> tuple. The InputFormat
   * also creates the {@link RecordReader} to read the {@link InputSplit}.
   *
   * @param context job configuration.
   * @return an array of {@link InputSplit}s for the job.
   */
  public abstract
    List<InputSplit> getSplits(JobContext context
                               ) throws IOException, InterruptedException;

  /**
   * Create a record reader for a given split. The framework will call
   * {@link RecordReader#initialize(InputSplit, TaskAttemptContext)} before
   * the split is used.
   * @param split the split to be read
   * @param context the information about the task
   * @return a new record reader
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract
    RecordReader<K,V> createRecordReader(InputSplit split,
                                         TaskAttemptContext context
                                        ) throws IOException,
                                                 InterruptedException;
}
```

* 文件切分算法：文件切分算法主要用于确定 InputSplit 的个数以及每个 InputSplit 对应的数据段。FileInputFormat 以文件为单位切分生成 InputSplit 。对于每个文件,由以下三个属性值确定 其对应的 InputSplit 的个数.
    * goalSize :它是根据用户期望的 InputSplit 数目计算出来的,即 totalSize/numSplits。其中,totalSize 为文件总大小;numSplits 为用户设定的 Map Task 个数,默认情况下 是 1。
    * minSize:InputSplit 的最小值,由配置参数 mapred.min.split.size 确定,默认是 1.
    * blockSize:文件在 HDFS 中存储的 block 大小,不同文件可能不同,默认是 64 MB。
    * splitSize = max{minSize, min{goalSize, blockSize}}
* host算法
    * Split大于Block时候，一个Split可能有多个Block，而这杯Block不可能同时在本地的一个节点上，所以要完全实现数据本地化是不可能的。
    * Hadoop 将数据本地性按照代价划分成三个等级 :node locality、rack locality 和 data center locality(Hadoop 还未实现该 locality 级别)。在进行任务调度时,会依次考虑这 3 个 节点的 locality,即优先让空闲资源处理本节点上的数据。
    * 首先按照 rack 包含的数据量对 rack 进行排序,然后在 rack 内部按照每个 node 包含的数据 量对 node 排序,最后取前 N 个 node 的 host 作为 InputSplit 的 host 列表,这里的 N 为 block副本数。这样,当任务调度器调度 Task 时,只要将 Task 调度给位于 host 列表的节点,就 认为该 Task 满足本地性。
    * 当使用基于 FileInputFormat 实现 InputFormat 时,为了提高 Map Task 的数据本地 性,应尽量使 InputSplit 大小与 block 大小相同。
    * 举列：某个 Hadoop 集群的网络拓扑结构如图 3-10 所示,HDFS 中 block 副本数为3,某个 InputSplit 包含 3 个 block,大小依次是 100、150 和 75,很容易计算,4 个 rack 包 含的(该 InputSplit 的)数据量分别是 175、250、150 和 75。rack2 中的 node3 和 node4,rack1 中的 node1 将被添加到该 InputSplit 的 host 列表中.

![img](../image/hadoop/mapreduce-6.png)

2.为Mapper提供输入数据，给定某个Split，能将其解析成一个个Key/Value对。

目前Mapreduce默认支持以下几种格式

![img](../image/hadoop/mapreduce-7.png)

### 3.2 OutputFormt

OutputFormat跟InputFormat对应。主要有两个接口：

1.实现checkOutputSpecs接口：该接口在作业运行之前被调用，默认功能是检查用户配置的输出目录是否存在，如果存在则跑出异常，以防止之前的数据被覆盖。

2.处理Side-Effect File：何为Side-Effect File，这就得讲到MapReduce的推测式任务。在 Hadoop 中,因为硬件老化、网络故障等原因,同 一个作业的某些任务执行速度可能明显慢于其他任务,这种任务会拖慢整个作业的执行速 度。为了对这种“慢任务”进行优化,Hadoop 会为之在另外一个节点上启动一个相同的 任务,该任务便被称为推测式任务,最先完成任务的计算结果便是这块数据对应的处理结 果。为防止这两个任务同时往一个输出文件中写入数据时发生写冲突,FileOutputFormat会为每个 Task 的数据创建一个 side-effect file,并将产生的数据临时写入该文件,待 Task完成后,再移动到最终输出目录中。

```java
public abstract RecordWriter<K, V>
    getRecordWriter(TaskAttemptContext context
                    ) throws IOException, InterruptedException;
  /**
   * Check for validity of the output-specification for the job.
   *  
   * <p>This is to validate the output specification for the job when it is
   * a job is submitted.  Typically checks that it does not already exist,
   * throwing an exception when it already exists, so that output is not
   * overwritten.</p>
   *
   * @param context information about the job
   * @throws IOException when output should not be attempted
   */
  public abstract void checkOutputSpecs(JobContext context
                                        ) throws IOException,
                                                 InterruptedException;
  /**
   * Get the output committer for this output format. This is responsible
   * for ensuring the output is committed correctly.
   * @param context the task context
   * @return an output committer
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract
  OutputCommitter getOutputCommitter(TaskAttemptContext context
                                     ) throws IOException, InterruptedException;
}
```

### 3.3 Partitioner

Partitioner是将对Mapper产生的中间结果进行分片，以便将统一分组的数据交给同一个Reducer处理，所以它直接影响Reducer阶段的负载均衡。MapReduce提供两个Partitioner实现：HashPartitioner和TotalOrderPartitioner，前者是基域哈希值的分片方法：

```java
public int getPartition(K2 key, V2 value,
                              int numReduceTasks) {
        return (key.hashCode() & Integer.MAX_VALUE) % numReduceTasks;
      }
```

后者是基于区间的分片方法，通常在数据全排序中。它不同于归并排序，归并排序,即在 Map 阶段,每个 Map Task进行局部排序 ;在 Reduce 阶段,启动一个 Reduce Task 进行全局排序。由于作业只能有一 个 Reduce Task,因而 Reduce 阶段会成为作业的瓶颈，它能够按照大小将数据分成若干个区间(分片),并保证后一个区间的所有数据均大于前一个区间数据 。

### 3.4 Mapper

MapReduce框架会通过InputFormt中的RecordReader从InputSolit获取一个个Key/value对，并交给下面的Map()函数处理。

```java
protected void map(KEYIN key, VALUEIN value,
                     Context context) throws IOException, InterruptedException {
    context.write((KEYOUT) key, (VALUEOUT) value);
  }
```

* Map函数开始产生输出结果时，并不是简单地将它写到磁盘，这个过程更复杂，它利用缓冲的方式写到内存，并处理效率的原因预先进行排序。每个Map任务都有一个环形内存缓冲区，任务会把输出写到此。默认情况下，缓冲区的大小为100MB，此值可以通过io.sort.mb属性来修改。当缓冲内容达到指定大小时(io.sort.spill.percent，默认为0.8)，一个后台线程便开始把内容溢写spill到磁盘中。在线程工作的同时，map输出继续被写到缓冲区，但如果在此期间缓冲区被填满，map会阻塞直到溢写过程结束。
* 溢写将按轮询的方式写到mapred.local.dir属性指定的目录，在一个作业相关子目录中。在写磁盘前，线程根据数据最终被传送的reducer，将数据划分成相应的Partitioner，在每个Partitioner中，后台线程按键进行内排序。此时如果有一个Combiner，它将基于排序后输出运行。
* 一旦内存缓冲区达到溢写阀值，就会新建一个溢写文件，因此在Map任务写入其最后一个输出记录后，就会有若干个溢写文件。在任务完成之前，溢写文件被合并成一个已分区且已排序的输出文件。
* 如果已经指定combine，且溢写次数至少为3(min.num.spills.for.combine)时，combine就在输出文件被写之前执行。

### 3.5 Reducer

Reduce同Mapper相似,只不过输入的是Key，List<Value>

```java
/**
   * This method is called once for each key. Most applications will define
   * their reduce class by overriding this method. The default implementation
   * is an identity function.
   */
  @SuppressWarnings("unchecked")
  protected void reduce(KEYIN key, Iterable<VALUEIN> values, Context context
                        ) throws IOException, InterruptedException {
    for(VALUEIN value: values) {
      context.write((KEYOUT) key, (VALUEOUT) value);
    }
  }
```

* Map任务可以在不同时间完成，因此只要有一个任务结束，reduce任务就开始复制其输出。Reduce任务有少量复制线程，因此能够并行的取得map输出。这是因为Map任务完成后，它们会通知其父TaskTracker状态已更新，然后TaskTracker进而通知JobTracker，这些通知在心跳机制内传输。由于JobTracker知道Map输出与TaskTracker之间的映射关系，所以Reducer的一个线程会定期向JobTracker获取Map输出位置，直到得到所有输出位置，
* TaskTracker并没有在Reducer检索之后立即删除Map输出，因为Reducer可能失败，它们会等到JobTrasker告知可以删除才进行删除。
* 如果Map输出相当小，就会复制到Reduce TaskTracker的内存(缓冲区大小mapred.job.shuffle.inpit.buffer.percent)中，否则被复制到磁盘。内存缓冲区达到阀值大小(mapred.job.shuffle.merge.percent)或者Map输出阀值(mapred.inmen.merge.thresold)，才会被合并进而被溢写到磁盘中。
* 后台线程将磁盘上积累的副本合并成一个更大的，排序好的文件，为后续合并节省时间。
* 有几个Reducer会产生几个part-XXX文件。

### 3.6 Combiner

如果在Client设置过Combiner，那么Mapper在进行Partitioner之前还会进行Combiner,该操作就是将相同的Key的Key/Value中的Value加起来，减少写到磁盘的数据量。Combiner的输出是Reducer的输入，Combiner绝不能改变最终的计算结果。所以从我的想法来看，Combiner只应该用于那种Reduce的输入key/value与输出key/value类型完全一致，且不影响最终结果的场景。比如累加，最大值等。Combiner的使用一定得慎重，如果用好，它对job执行效率有帮助，反之会影响reduce的最终结果。

### 3.7 作业的提交

在 MapReduce 中,每个作业由两部分组成 :应用程序和作业配置。其中,作业配置内 容包括环境配置和用户自定义配置两部分。环境配置由 Hadoop 自动添加,主要由 mapred- default.xml 和 mapred-site.xml 两个文件中的配置选项组合而成 ;用户自定义配置则由用 户自己根据作业特点个性化定制而成,比如用户可设置作业名称,以及 Mapper/Reducer、Reduce Task 个数等。

```shell
Configuration conf = new Configuration();
Job job = new Job(conf, "myjob ");
job.setJarByClass(MyJob.class);
job.setMapperClass(MyJob.MyMapper.class);
job.setReducerClass(MyJob.MyReducer.class);
System.exit(job.waitForCompletion(true) ? 0 : 1)
```

## 4. MapReduce工作流

上文讲到的MapReduce只支持单个Map-Reduce运算，很多时候需要多个Map之间的依赖等复杂操作。这时候可以用工作流的方式来实现。

### 4.1 JobControl

以下例子是通过JobControl的addDepending()函数来添加作业的依赖关系，而JobControl就会按照依赖的关系调度各个作业。

```java
Configuration extractJobConf = new Configuration();
Configuration classPriorJobConf = new Configuration();
Configuration conditionalProbilityJobConf = new Configuration(); Configuration predictJobConf = new Configuration();
...
// 设置各个 Configuration
// 创建 Job 对象。注意,JobControl 要求作业必须封装成 Job 对象
Job extractJob = new Job(extractJobConf);
Job classPriorJob = new Job(classPriorJobConf);
Job conditionalProbilityJob = new Job(conditionalProbilityJobConf);
 Job predictJob = new Job(predictJobConf);
// 设置依赖关系,构造一个 DAG 作业
classPriorJob.addDepending(extractJob);
conditionalProbilityJob.addDepending(extractJob);
predictJob.addDepending(classPriorJob);
predictJob.addDepending(conditionalProbilityJob);
// 创建 JobControl 对象,由它对作业进行监控和调度
JobControl JC = new JobControl("Native Bayes");
JC.addJob(extractJob);// 把 4 个作业加入 JobControl 中 JC.addJob(classPriorJob)；
```

#### 4.1.1 Job

JobControl 由两个类组成 :Job 和 JobControl。其中,Job 类封装了一个 MapReduce 作 业及其对应的依赖关系,主要负责监控各个依赖作业的运行状态,以此更新自己的状态。

![img](../image/hadoop/mapreduce-8.png)

作业刚开始处于 WAITING 状态。如果没有依赖作业或者所 有依赖作业均已运行完成,则进入 READY 状态。一旦进入 READY 状态,则作业可被提 交到 Hadoop 集群上运行,并进入 RUNNING 状态。在 RUNNING 状态下,根据作业运行 情况,可能进入 SUCCESS 或者 FAILED 状态。需要注意的是,如果一个作业的依赖作业 失败,则该作业也会失败,于是形成“多米诺骨牌效应”,后续所有作业均会失败。

#### 4.1.2 JobControl

JobControl包含一个线程用于周期性地监控和更新各个作业的运行状态,调度依赖作业运行完成的作业,提交处于 READY 状态的作业等。同时,它还提供了一些API 用于挂起、恢复和暂停该线程 。

### 4.2 线性链式Mapper实现与原理(ChainMapper和ChainReducer)

线性链式Mapper实现的场景：在Map 阶段,数据依次经过 Mapper1 和 Mapper2 处理 ;在 Reduce 阶段,数据经过 shuffle 和sort 后;交由对应的 Reducer 处理,但 Reducer 处理之后并没有直接写到 HDFS 上,而是交 给另外一个 Mapper 处理,它产生的结果写到最终的 HDFS 输出目录中。需要注意的是无论有多少个Mapper，只能有一个Reducer.

![img](../image/hadoop/mapreduce-9.png)

```java
conf.setJobName("chain");
conf.setInputFormat(TextInputFormat.class);
conf.setOutputFormat(TextOutputFormat.class);
JobConf mapper1Conf = new JobConf(false);
JobConf mapper2Conf = new JobConf(false);
JobConf reduce1Conf = new JobConf(false);
JobConf mapper3Conf = new JobConf(false); ...
ChainMapper.addMapper(conf, Mapper1.class, LongWritable.class, Text.class,Text. class, Text.class, true, mapper1Conf);
ChainMapper.addMapper(conf, Mapper2.class, Text.class, Text.class,LongWritable.class, Text.class, false, mapper2Conf);
ChainReducer.setReducer(conf, Reducer.class, LongWritable.class, Text.class,Text. class, Text.class, true, reduce1Conf);
ChainReducer.addMapper(conf, Mapper3.class, Text.class, Text.class,LongWritable.class, Text.class, false, null);
JobClient.runJob(conf);
```

ChainMapper/ChainReducer 实 现 的 关 键 技 术 点 是 修 改 Mapper 和 Reducer 的 输 出 流,将本来要写入文件的输出结果重定向到另外一个 Mapper 中。

第一个调用Mapper是ChainMapper的导火线，它将Mapper的输出重定向到chain的下一个Mapper里了。当链式作业开始执行的时候,首先将各个 Mapper 的 JobConf 对象反序列化,并构造对 应的 Mapper 和 Reducer 对象,添加到数据结构 mappers(List<Mapper> 类型)和 reducer(Reducer 类型)中。Chain内存放了Mapper的List数组，mapperIndex是Mapper的索引。Reduce过程跟Mapper类似。

```java
public void map(Object key, Object value, OutputCollector output,
                  Reporter reporter) throws IOException {
    Mapper mapper = chain.getFirstMap();
    if (mapper != null) {
      mapper.map(key, value, chain.getMapperCollector(0, output, reporter),
                 reporter);
    }
  }

  /**
    * Returns the OutputCollector to be used by a Mapper instance in the chain.
    *
    * @param mapperIndex index of the Mapper instance to get the OutputCollector.
    * @param output      the original OutputCollector of the task.
    * @param reporter    the reporter of the task.
    * @return the OutputCollector to be used in the chain.
    */
   @SuppressWarnings({"unchecked"})
   public OutputCollector getMapperCollector(int mapperIndex,
                                             OutputCollector output,
                                             Reporter reporter) {
     Serialization keySerialization = mappersKeySerialization.get(mapperIndex);
     Serialization valueSerialization =
       mappersValueSerialization.get(mapperIndex);
     return new ChainOutputCollector(mapperIndex, keySerialization,
                                     valueSerialization, output, reporter);
   }
```

在ChainOutputCollector实现了collect接口：

```java
public void collect(K key, V value) throws IOException {
      if (nextMapperIndex < mappers.size()) {
        // there is a next mapper in chain
        // only need to ser/deser if there is next mapper in the chain
        if (keySerialization != null) {
          key = makeCopyForPassByValue(keySerialization, key);
          value = makeCopyForPassByValue(valueSerialization, value);
        }
        // gets ser/deser and mapper of next in chain
        Serialization nextKeySerialization =
          mappersKeySerialization.get(nextMapperIndex);
        Serialization nextValueSerialization =
          mappersValueSerialization.get(nextMapperIndex);
        Mapper nextMapper = mappers.get(nextMapperIndex);
        // invokes next mapper in chain
        nextMapper.map(key, value,
                       new ChainOutputCollector(nextMapperIndex,
                                                nextKeySerialization,
                                                nextValueSerialization,
                                                output, reporter),
                       reporter);
      } else {
        // end of chain, user real output collector
        output.collect(key, value);
      }
    }
```

## 5. Hadoop Streaming

虽然MapReduce使用JAVA编写的，但是我们可以通过Hadoop Streaming工具用其他语言来编写MapReduce，只需要支持标准输入输出的就行。Hadoop Streaming 使用了 JDK 中的 java.lang. ProcessBuilder 类。该类提供了一整套管理操作系统进程的方法,包括创建、启动和停止进 程(也就是应用程序)等.

![img](../image/hadoop/mapreduce-10.png)

Hadoop Streaming的原理如下:

1.在CLient构建Job时候指定Mapper类为PipeMapper.class,PipeReducer.class,并将该Job提交给JobTracker。

2.在初始化Mapper的时候，JobTracker会初始化PipeMapper和PipeReducer，并利用JAVA.LANG.ProcessBuilder来启动PipeMapper和PipeReducer中用户上传的Mapper的Reducer程序;

```java
ProcessBuilder builder = new ProcessBuilder(argvSplit);
      builder.environment().putAll(childEnv.toMap());
      sim = builder.start(); //运行Mapper程序
      clientOut_ = new DataOutputStream(new BufferedOutputStream(
                                              sim.getOutputStream(),
                                              BUFFER_SIZE));
      clientIn_ = new DataInputStream(new BufferedInputStream(
                                              sim.getInputStream(),
                                              BUFFER_SIZE));
      clientErr_ = new DataInputStream(new BufferedInputStream(sim.getErrorStream()));
```

3.通过标准输出输入流，InputSplit输出的Value作为用户Mapper程序的输入，用户Mapper程序的输出作为PipeMapper的输出。

4.Python的Hadoop Streaming最基本的用例：

Mapper:

```python
#!/usr/bin/env python
import sys
# input comes from STDIN (standard input)
for line in sys.stdin:
    # remove leading and trailing whitespace
    line = line.strip()
    # split the line into words
    words = line.split()
    # increase counters
    for word in words:
        # write the results to STDOUT (standard output);
        # what we output here will be the input for the
        # Reduce step, i.e. the input for reducer.py
        #
        # tab-delimited; the trivial word count is 1
        print '%s\t%s' % (word, 1)
```

Reducer:

```python
#!/usr/bin/env python
from operator import itemgetter
import sys
current_word = None
current_count = 0
word = None
# input comes from STDIN
for line in sys.stdin:
    # remove leading and trailing whitespace
    line = line.strip()
    # parse the input we got from mapper.py
    word, count = line.split('\t', 1)
    # convert count (currently a string) to int
    try:
        count = int(count)
    except ValueError:
        # count was not a number, so silently
        # ignore/discard this line
        continue
    # this IF-switch only works because Hadoop sorts map output
    # by key (here: word) before it is passed to the reducer
    if current_word == word:
        current_count += count
    else:
        if current_word:
            # write result to STDOUT
            print '%s\t%s' % (current_word, current_count)
        current_count = count
        current_word = word
# do not forget to output the last word if needed!
if current_word == word:
    print '%s\t%s' % (current_word, current_count)
```

任务运行:

```shell
hduser@ubuntu:/usr/local/hadoop$ bin/hadoop jar contrib/streaming/hadoop-*streaming*.jar \
-file /home/hduser/mapper.py    -mapper /home/hduser/mapper.py \
-file /home/hduser/reducer.py   -reducer /home/hduser/reducer.py \
-input /user/hduser/gutenberg/* -output /user/hduser/gutenberg-output
```

## 6. MapReduce实例

以WordCount为例：

![img](../image/hadoop/mapreduce-11.png)

```java
package com.lamborryan;
import java.io.IOException;
import java.util.Iterator;
import java.util.StringTokenizer;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapred.FileInputFormat;
import org.apache.hadoop.mapred.FileOutputFormat;
import org.apache.hadoop.mapred.JobClient;
import org.apache.hadoop.mapred.JobConf;
import org.apache.hadoop.mapred.MapReduceBase;
import org.apache.hadoop.mapred.Mapper;
import org.apache.hadoop.mapred.OutputCollector;
import org.apache.hadoop.mapred.Reducer;
import org.apache.hadoop.mapred.Reporter;
import org.apache.hadoop.mapred.TextInputFormat;
import org.apache.hadoop.mapred.TextOutputFormat;
/**
 *
 * 描述：WordCount explains by Felix
 * @author Hadoop Dev Group
 */
public class WordCount
{
    /**
     * MapReduceBase类:实现了Mapper和Reducer接口的基类（其中的方法只是实现接口，而未作任何事情）
     * Mapper接口：
     * WritableComparable接口：实现WritableComparable的类可以相互比较。所有被用作key的类应该实现此接口。
     * Reporter 则可用于报告整个应用的运行进度，本例中未使用。
     *
     */
    public static class Map extends MapReduceBase implements
            Mapper<LongWritable, Text, Text, IntWritable>
    {
        /**
         * LongWritable, IntWritable, Text 均是 Hadoop 中实现的用于封装 Java 数据类型的类，这些类实现了WritableComparable接口，
         * 都能够被串行化从而便于在分布式环境中进行数据交换，你可以将它们分别视为long,int,String 的替代品。
         */
        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();

        /**
         * Mapper接口中的map方法：
         * void map(K1 key, V1 value, OutputCollector<K2,V2> output, Reporter reporter)
         * 映射一个单个的输入k/v对到一个中间的k/v对
         * 输出对不需要和输入对是相同的类型，输入对可以映射到0个或多个输出对。
         * OutputCollector接口：收集Mapper和Reducer输出的<k,v>对。
         * OutputCollector接口的collect(k, v)方法:增加一个(k,v)对到output
         */
        public void map(LongWritable key, Text value,
                OutputCollector<Text, IntWritable> output, Reporter reporter)
                throws IOException
        {
            String line = value.toString();
            StringTokenizer tokenizer = new StringTokenizer(line);
            while (tokenizer.hasMoreTokens())
            {
                word.set(tokenizer.nextToken());
                output.collect(word, one);
            }
        }
    }
    public static class Reduce extends MapReduceBase implements
            Reducer<Text, IntWritable, Text, IntWritable>
    {
        public void reduce(Text key, Iterator<IntWritable> values,
                OutputCollector<Text, IntWritable> output, Reporter reporter)
                throws IOException
        {
            int sum = 0;
            while (values.hasNext())
            {
                sum += values.next().get();
            }
            output.collect(key, new IntWritable(sum));
        }
    }
    public static void main(String[] args) throws Exception
    {
        /**
         * JobConf：map/reduce的job配置类，向hadoop框架描述map-reduce执行的工作
         * 构造方法：JobConf()、JobConf(Class exampleClass)、JobConf(Configuration conf)等
         */
        JobConf conf = new JobConf(WordCount.class);
        conf.setJobName("wordcount");           //设置一个用户定义的job名称
        conf.setOutputKeyClass(Text.class);    //为job的输出数据设置Key类
        conf.setOutputValueClass(IntWritable.class);   //为job输出设置value类
        conf.setMapperClass(Map.class);         //为job设置Mapper类
        conf.setCombinerClass(Reduce.class);      //为job设置Combiner类
        conf.setReducerClass(Reduce.class);        //为job设置Reduce类
        conf.setInputFormat(TextInputFormat.class);    //为map-reduce任务设置InputFormat实现类
        conf.setOutputFormat(TextOutputFormat.class);  //为map-reduce任务设置OutputFormat实现类
        /**
         * InputFormat描述map-reduce中对job的输入定义
         * setInputPaths():为map-reduce job设置路径数组作为输入列表
         * setInputPath()：为map-reduce job设置路径数组作为输出列表
         */
        FileInputFormat.setInputPaths(conf, new Path(args[0]));
        FileOutputFormat.setOutputPath(conf, new Path(args[1]));
        JobClient.runJob(conf);         //运行一个job
    }
}
```

## 7. 总结

本文从整体的角度介绍了MapReduce的原理和框架, 并给出了相应的实例. 希望通过本文让你对MapReduce有初步的了解。


本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/hadoop-mapreduce
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
