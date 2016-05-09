---
layout: post
title: Hadoop系列(4)之Yarn框架简介
date: 2016-01-25 10:30:35
categories: 大数据
tags: Hadoop
---

## 1. 简介

#### 什么是YARN

在Hadoop 1.0中，MapReduce由JobTracker和TaskTracker组成，JobTracker不但负责资源管理(由TaskScheduler模块实现)和作业控制(由JobTracker中多个模块共同实现)两部分组成，因此JobTracker赋予的功能过多而造成负载过重，从而在可扩展性，资源利用率和多框架支持等方面存在不足。

在第二代MapReduce框架中，将JobTracker的两个主要功能，即资源管理和作业控制(包括作业监控，容错等)分拆成两个独立进程。资源管理进程与具体应用程序无关，它负责整个集群的资源(内存、CPU、磁盘等)管理,而作业控制进程则是直接与应用程序相关的模块,且每个作业控制进程只负责管理一个作业。这样, 通过将原有JobTracker中与应用程序相关和无关的模块分开,不仅减轻了JobTracker负载,也使得Hadoop支持更多的计算框架。

因此在Hadoop2.0产生了YARN资源管理系统，主要由一个全局的资源管理器ResourceManager和每个应用程序特有的ApplicationMaster。其中ResourceManager负责整个系统的资源管理和分配，而ApplicationMaster负责单个应用程序的管理。其中不同的应用程序会产生不同的ApplicationMaster，比如MapReduce产生的是MRAppMaster。

#### 怎么理解YARN

![img](../image/hadoop/yarn-1.png)

在单机程序设计中,为了快速处理一个大的数据集,通常采用多线程并行编程,如上图所示,大体流程如下:先由操作系统启动一个主线程,由它负责数据切分、任务分配、子线程启动和销毁等工作,而各个子线程只负责计算自己的数据,当所有子线程处理完数据后,主线程再退出。类比理解,YARN上的应用程序运行过程与之非常相近,只不过它是集群上的分布式并行编程。可将YARN看做一个云操作系统,它负责为应用程序启动ApplicationMaster(相当于主线程),然后再由 ApplicationMaster负责数据切分、任务分配、 启动和监控等工作,而由 ApplicationMaster 启动的各个 Task(相当于子线程)仅负责自己的计 算任务。当所有任务计算完成后,ApplicationMaster 认为应用程序运行完成,然后退出

## 2. YARN的基本架构

YARN主要由ResourceManager，NodeManager，ApplicationMaster(给出了MapReduce和MPI两个计算框架的ApplicationMaster，分别为MR AppMstr和MPI AppMstr)和Container等几个组件构成。

![img](../image/hadoop/yarn-2.png)

* ResourceManager(RM):RM是一个全局的资源管理器，负责整个系统的资源管理和分配。主要包含两个组件：调度器(Scheduler)和应用程序管理器(Applications Manager，ASM)。
    * 调度器(Scheduler)。调度器根据容量、队列等限制条件。将系统中的资源分配给哥各个正在运行的应用程序。资源分配单位用Container表示，它是一个动态资源分配单位，将内存、CPU、磁盘、网络等资源封装在一起，从而限定每个任务使用的资源量。YARN的调度器是可插拔式的，目前支持以下几种：
        * FifoScheduler：最简单的调度器，按照先进先出的方式处理应用。只有一个队列可提交应用，所有用户提交到这个队列。可以针对这个队列设置ACL。没有应用优先级可以配置。
        * CapacityScheduler：可以看作是FifoScheduler的多队列版本。每个队列可以限制资源使用量。但是，队列间的资源分配以使用量作排列依据，使得容量小的队列有竞争优势。集群整体吞吐较大。延迟调度机制使得应用可以放弃，夸机器或者夸机架的调度机会，争取本地调度。
        * FairScheduler：多队列，多用户共享资源。特有的客户端创建队列的特性，使得权限控制不太完美。根据队列设定的最小共享量或者权重等参数，按比例共享资源。延迟调度机制跟CapacityScheduler的目的类似，但是实现方式稍有不同。资源抢占特性，是指调度器能够依据公平资源共享算法，计算每个队列应得的资源，将超额资源的队列的部分容器释放掉的特性。
    * 应用程序管理器(Applications Manager)。应用程序管理器负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协商资源以启动ApplicationMaster，监控ApplicationMaster运行状态并在失败时重新启动它等。换而言之，它就是管理ApplicationMaster的。
* ApplicationMaster(AM)。用户提交的每个应用程序均包含一个AM，主要功能如下。目前ApplicationMaster已经或者将要支持MapReduce，OpenMPI，Spark,Storm,HBase等
    * 申请资源。与RM调度器协商以获取资源(用Container表示)
    * 分配任务。将得到的任务进一步分配给内部的任务;(比如MapReduce的Split过程)
    * 操控任务。与NodeManager通信以启动/停止任务；
    * 监控任务。监控所有任务运行状态，并在任务运行失败时重新为任务申请资源以重启任务。
* NodeManager(NM)。NM是每个节点上的资源和任务管理器：
    * 定时地向RM汇报本节点上的资源使用情况和各个Container的运行状态；
    * 接受并处理来自AM的Container启动/停止等各个请求。
* Container。它是YARN中的资源抽象，封装了某个节点上的多维度资源，如内存，CPU，磁盘，网络等，目前仅支持CPU和内存。当AM向RM申请资源时候，RM为AM返回的资源便是用Container表示的。YARN会为每个任务分配一个Container，且该任务只能使用该Container种描述的资源。

## 3. YARN的通信协议

![img](../image/hadoop/yarn-3.png)

* JobClient与RM之间的协议——ApplicationClientProtocol: JobClient通过该RPC协议提交应用程序，查询应用程序状态。
* Admin与RM之间的通信协议--ResourceManagerAdministrationProtocol:Admin通过该RPC协议更新系统配置文件，比如节目黑白名单，用户队列权限等。
* AM与RM之间的协议--ApplicationMasterProtocol:AM通过该RPC协议向RM注册和撤销自己，并为各个任务申请资源
* AM与NM之间的协议--ContainerManagementProtocol：AM通过该RPC要求NM启动或者停止Container，获取各个Containner的使用状态等消息
* NM与RM之间的协议--ResourceTracker：NM通过该RPC协议向RM注册，并定时发送心跳信息汇报当前节点的资源使用情况和Container运行情况。

## 4. YARN的工作流程

![img](../image/hadoop/yarn-4.png)

* 步骤1:用户向YARN中提交应用程序，其中包括ApplicationMaster程序、启动ApplicationMaster的命令、用户程序等。
* 步骤2:ResourceManager为该应用程序分配第一个Container，并与对应的Node-Manager通信，要求它在这个Container中启动应用程序的ApplicationMaster。
* 步骤3:ApplicationMaster首先向ResourceManager注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将为各个任务申请资源，并监控它的运行状态，直到运行结束，即重复步骤4~7。
* 步骤4:ApplicationMaster采用轮询的方式通过RPC协议向ResourceManager申请和领取资源。
* 步骤5:一旦ApplicationMaster申请到资源后，便与对应的NodeManager通信，要求它启动任务。
* 步骤6:NodeManager为任务设置好运行环境（包括环境变量、JAR包、二进制程序等）后，将任务启动命令写到一个脚本中，并通过运行该脚本启动任务。
* 步骤7:各个任务通过某个RPC协议向ApplicationMaster汇报自己的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务。在应用程序运行过程中，用户可随时通过RPC向ApplicationMaster查询应用程序的当前运行状态。
* 步骤8:应用程序运行完成后，ApplicationMaster向ResourceManager注销并关闭自己。

## 5. YARN的通信框架

#### 5.1 Protocol Buffers

相比于Hadoop RPC默认以Writeable接口实现序列化，YARN RPC则是以Protocol Buffers为序列化基础。那么ProtocolBuffer做序列化的好处在哪里。

YARN之所以在引入Protocal buffer，最直接的原因是提高Hadoop的向后兼容性，即不同版本的Client、ResourceManager、NodeManager和ApplicationMaster之间通信。在Hadoop 1.0中，如果新版本的通信接口参数定义被修改了，则无法与旧版本进行通信。下面举例说明，在YARN中，Client与ResourceManager之间的通信协议是ClientRMProtocol，该协议中有一个RPC函数为：

```java
public SubmitApplicationResponse submitApplication(SubmitApplicationRequest request)
```

一看名字大家就能猜到，该函数用于Client向RM提交一个应用程序，其中参数SubmitApplicationRequest中的ApplicationSubmissionContext字段描述了应用程序的一些属性，主要包括以下几个方面：

```shell
（1）       application_id  //application ID
（2）       application_name //application
（3）       user //application owner
（4）       queue//queue that application submits to
（5）       priority  //application priority
（6）       am_container_spec //AM所需资源描述
（7）       cancel_tokens_when_complete
（8）       unmanaged_am
```

如果采用Java，ApplicationSubmissionContext定义可能如下：

```java
public class SubmitApplicationRequest implements Writable {
    private ApplicationId application_id;
    private String application_name;
    private String user;
    private String queue;
    private Priority priority;
    private ContainerLaunchContext am_container_spec;
    private bool cancel_tokens_when_complete;
    private bool unmanaged_am;
    @Override
    public void readFields(DataInput in) throws IOException {}
    @Override
    public void writeFields(DataInput in) throws IOException {}
}
```

如果在一个新的YARN版本中，需要在ApplicationSubmissionContext中添加一个新的属性，比如deadline（期望应用程序在deadline时间内运行完成），则所有旧的Client将无法与升级后的ResourceManager通信，原因是接口不兼容，即客户端发送的ApplicationSubmissionContext请求包中多出了deadline字段导致ResourceManager无法对其进行反序列化。这意味着所有客户端必须升级，不然无法使用。这是客户端与ResourceManager之间的一个例子，同样，对于NodeManager与ResourceManager，ApplicationMaster与ResourceManager，ApplicationMaster与NodeManager之间的通信也是类似的，一旦一端修改了通信协议内容（RPC函数名不能改），则另外一端必须跟着改，不然对方与之通信（反序列化失败），这可能导致a.b.0版本的NodeManager，无法与a.b.a版本的ResourceManager通信。

为了解决该问题，可使用Protocal Buffer，在PB中，可以采用如下的规范定义ApplicationSubmissionContext：

```java
message ApplicationSubmissionContextProto {
optional ApplicationIdProto application_id = 1;
optional string application_name = 2 [default = "N/A"];
optional string user = 3;
optional string queue = 4 [default = "default"];
optional PriorityProto priority = 5;
optional ContainerLaunchContextProto am_container_spec = 6;
optional bool cancel_tokens_when_complete = 7 [default = true];
optional bool unmanaged_am = 8 [default = false];
}
```

当需要增加一个新的deadline字段时，可直接在最后面添加一个optional字段即可，即：

```java
message ApplicationSubmissionContextProto {
……
optional bool cancel_tokens_when_complete = 7 [default = true];
optional bool unmanaged_am = 8 [default = false];
optional int deadline=9[default=-1];
}
```

由于旧的客户端请求中没有deadline这一字段，ResourceManager端进行反序列化时会跳过该字段，直接赋予该值为默认值-1.

#### 5.2 YARN RPC

上文讲到了为什么YARN RPC会以ProtocolBuffers为默认的序列化框架，本节将要将YARN RPC是怎么实现的。
YARN RPC只是对HADOOP RPC进行了封装。

![img](../image/hadoop/yarn-5.png)

YarnRPC是抽象类，实际的实现由参数yarn.ipc.rpc.class指定，默认情况下就是HadoopYarnProtoRPC类。

HadoopYarnProtoRPC 通过 RPC 工厂生成器(工厂 设计模式)RpcFactoryProvider 生成客户端工厂(由参数 yarn.ipc.client.factory.class 指定, 默认值是 org.apache.hadoop.yarn.factories.impl.pb.RpcClientFactoryPBImpl)和服务器工厂(由参数 yarn.ipc.server.factory.class 指定,默认值是 org.apache.hadoop.yarn.factories.impl. pb.RpcServerFactoryPBImpl),以根据通信协议的 Protocol Buffers 定义生成客户端对象和服务器对象.

RpcServerFactoryPBImpl其实就是Server的工厂方式的实现，内部调用的HadoopRPC来构建Server。

```java
private Server createServer(Class<?> pbProtocol, InetSocketAddress addr, Configuration conf,
      SecretManager<? extends TokenIdentifier> secretManager, int numHandlers,
      BlockingService blockingService, String portRangeConfig) throws IOException {
    RPC.setProtocolEngine(conf, pbProtocol, ProtobufRpcEngine.class);
    RPC.Server server = new RPC.Builder(conf).setProtocol(pbProtocol)
        .setInstance(blockingService).setBindAddress(addr.getHostName())
        .setPort(addr.getPort()).setNumHandlers(numHandlers).setVerbose(false)
        .setSecretManager(secretManager).setPortRangeConfig(portRangeConfig)
        .build();
    LOG.info("Adding protocol "+pbProtocol.getCanonicalName()+" to the server");
    server.addProtocol(RPC.RpcKind.RPC_PROTOCOL_BUFFER, pbProtocol, blockingService);
    return server;
  }
```

同理RpcClientFactoryPBImpl也就是Client的工厂方式的实现，同样调用HADOOP RPC实现。

```java
@Override
  public Object getProxy(Class protocol, InetSocketAddress addr,
      Configuration conf) {
    LOG.debug("Creating a HadoopYarnProtoRpc proxy for protocol " + protocol);
    return RpcFactoryProvider.getClientFactory(conf).getClient(protocol, 1,
        addr, conf);
  }
```

最后以ResourceTracker协议即NodeManage同ResourceManage通信为例，查看Hadoop RPC怎么使用ProtocolBuffers

![img](../image/hadoop/yarn-6.png)

## 6. 总结

本文从整体上介绍了yarn的框架和基本原理, 希望通过本文能让你对yarn有初步的理解。


本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/hadoop-yarn
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
