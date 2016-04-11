---
layout: post
title: Hadoop系列(2)之HDFS通信框架
date: 2016-01-08 10:30:35
categories: 大数据
tags: Hadoop
---

## 1. 简介

前文[《HDFS框架》](<http://www.lamborryan.com/hadoop-hdfs/>)讲述HDFS如何实现文件系统的相关操作，以及分布式文件系统的构成NameNode，SecondNameNode，DataNode。上文提到，NameNode通过心跳信号来检查DataNode的运行状态，以及获取DataNode的Block列表，这些操作就是基于Hadoop通信框架Hadoop RPC，Hadoop RPC就是分布式系统的骨架，正式由于它的存在，分布式系统各组成部分才能协同工作。本文主要介绍Hadoop RPC的实现原理。

## 2. 什么是RPC

远程过程调用(Remote Procedure Call,RPC)是一种常用的分布式网络通信协议,它允许运行于一台计算机的程序调用另一台计算机的子程序,同时将网络的通信细节隐藏起来,使得用户无须额外地为这个交互作用编程。由于RPC大大简化了分布式程序开发,因此备受欢迎。作为一个分布式系统,Hadoop实现了自己的RPC通信协议,它是上层多个分布式子系统(如 MapReduce、YARN、HDFS 等)公用的网络通信模块。

RPC通常采用客户机/服务器模型，请求程序是一个客户机，而服务提供程序则是一个服务器，它的框架如下：

![img](../image/hadoop/hadoop-protocol-1.png)

* 通信模块：实现请求应答协议。主要分为同步方式和异步方式。
* Stub程序：客户端和服务器均包含Stub程序，可以看做代理程序，使得远程函数表现的跟本地调用一样，对用户程序完成透明。
* 调度程序：接受来自通信模块的请求消息，根据标识选择Stub程序处理。并发量大时一般采用线程池处理。
* 客户程序/服务过程：请求发出者和请求处理者。

一个RPC的请求过程：

1. 客户程序以本地方式调用系统产生的Stub程序；
2. 该Stub程序将函数调用信息按照网络模块的要求封装成信息包，并交给通信模块发送至远程服务器端；
3. 远程服务器端接收到此消息后，将此消息发送给相应的Stub程序；
4. Stub程序拆封消息，形成被调过程要求的形式，并调用对应的函数；
5. 被掉用函数按照所获参数执行，并将结果返回给Stub程序；
6. Stub程序将结果封装成消息，通过网络通信模块逐级地传送给客户程序。

## 3. HadoopRPC框架

#### Hadoop RPC主要分为：

* 序列化层：序列化主要作用是将结构化对象转为字节流以便于网络进行传输或写入持久存储。Hadoop RPC 支持三种序列化框架分别是Protocol Buffers,Apache Avro,和基于Writable接口实现的自身框架。
* 函数调用层：函数调用层主要功能是定位要调用的函数并执行该函数,Hadoop RPC采用了Java反射机制与动态代理实现了函数调用。
* 网络传输层:基于TCP/IP的Socket机制。
* 服务器端处理框架：而Hadoop RPC采用了基于Reactor设计模式的事件驱动I/O模型以提升服务端的并发处理能力。

#### Hadoop RPC使用举例：

Hadoop RPC 可分为以下4个步骤：

1.定义协议：RPC协议是客户端和服务器端之间的通信接口，定义了服务器端对外提供的服务接口，所有自定义的RPC接口都需要继承VersionedProtocol接口，它描述了协议的版本信息。

```java
interface ClientProtocol extends org.apache.hadoop.ipc.VersionedProtocol { //版本号,默认情况下,不同版本号的RPC Client和Server之间不能相互通信
        public static final long versionID = 1L;
        String echo(String value) throws IOException;
        int add(int v1, int v2) throws IOException;
}
```

2.实现协议：Hadoop RPC协议通常是一个JAVA接口，用户需要实现该接口。

```java
public static class ClientProtocolImpl implements ClientProtocol {
// 重载的方法,用于获取自定义的协议版本号,
public long getProtocolVersion(String protocol, long clientVersion) {
          return ClientProtocol.versionID;
         }
// 重载的方法,用于获取协议签名
public ProtocolSignature getProtocolSignature(String protocol, long clientVersion, inthashcode) {
          return new ProtocolSignature(ClientProtocol.versionID, null);
        }
        public String echo(String value) throws IOException {
          return value;
         }
        public int add(int v1, int v2) throws IOException {
          return v1 + v2;
         }
}
```

3.构建并启动RPC SERVER：直接使用静态类Builder构造一个RPC SERVER,并调用函数start()启动该Server：

```java
Server server = new RPC.Builder(conf).setProtocol(ClientProtocol.class)
       .setInstance(new ClientProtocolImpl()).setBindAddress(ADDRESS).setPort(0).setNumHandlers(5).build();
￼server.start();
```

4.构建RPC Client并发送RPC请求：

```java
proxy = (ClientProtocol)RPC.getProxy(
              ClientProtocol.class, ClientProtocol.versionID, addr, conf);
int result = proxy.add(5, 6);
String echoResult = proxy.echo("result");
```

## 4. Hadoop RPC内部实现

上文讲到Hadoop RPC使用4个步骤，那么本节主要将Hadoop RPC是怎么实现这4个步骤。Hadoop RPC由三个大类组成RPC,Client,Server分别对应接口，客户端，服务端。

#### 4.1 RPC类。

RPC类实际上是对底层客户机和服务器网络的封装。

* 它定义了一系列构建和销毁RPC客户端的方法，构建方法为getProxy，waitForProxy两大类，销毁只有stopProxy。
* 它定义了RPC服务器的静态构造方法RPC.build().
* 通过setXXX接口来设置RPC协议。
* 通过RPC.setProtocolEngine来修改设置序列化方式(Protocol Buffers，Apache Avro，Writable)。

Hadoop RPC服务器动态代理实现类图:
![img](../image/hadoop/hadoop-protocol-2.png)


#### 4.2 Client类。

Client主要完成的功能是发送远程过程调用并接受执行结果。它由两个子类组成：

* Call类，封装一个RPC请求，包含5个成员变量，分别是唯一标识 id、函数调用用信息 param、函数执行返回值 value、出错或者异常信息 error 和执行完成标识符done。当客户端向服务器端发送请求时,只需填充 id 和 param 两个变量,而剩下的3 个变量(value、error 和 done)则由服务器端根据函数执行情况填充。
* Connection类，Client与每个Server之间维护一个通信连接，连接的基本信息存于Connection类内。
* Client类调用Call函数实现远程方法的过程：
    * 创建一个 Connection 对象,并将远程方法调用信息封装成 Call 对象,放到 Connection对象中的哈希表中;
    * 调用 Connection 类中的 sendRpcRequest() 方法将当前 Call 对象发送给 Server 端 ;
    * Server 端处理完RPC请求后,将结果通过网络返回给Client 端,Client端通过receiveRpcResponse()函数获取结果;
    * Client 检查结果处理状态(成功还是失败),并将对应 Call 对象从哈希表中删除。

![img](../image/hadoop/hadoop-protocol-3.png)

#### 4.3 Server类

Hadoop采用的是Master/Slave框架，由于Master是单点的，所以很容易成为性能的瓶颈。Master通过Server接受并处理所有Server的请求，这就要求Master具有很高的并发处理能力。Hadoop RPC采用了包括线程池，事件驱动，和Reactor的设计模式等技术，本节主要将Hadoop RPC采用的Reacter设计模式。

1.Reactor设计模式：Reactor是并发编程中的一种基于事件驱动的设计模式,它具有以下两个特点:通过派发/分离I/O操作事件提高系统的并发性能 ;提供了粗粒度的并发控制,使用单线程实现, 避免了复杂的同步处理 。

![img](../image/hadoop/hadoop-protocol-4.png)

(1).Reactor:I/O 事件的派发者。

(2).Acceptor :接受来自 Client 的连接,建立与 Client 对应的 Handler,并向 Reactor 注册此 Handler。

(3).Handler :与一个 Client 通信的实体,并按一定的过程实现业务的处理。Handler 内部往往会有更进一步的层次划分,用来抽象诸如 read、decode、compute、encode 和send 等过程。

(4).Reader/Sender:Reactor模式一般分离 Handler 中的读和写两个过程,分别注册成单独的读事件和写事件, 并由对应的 Reader 和 Sender 线程处理。

2.Server处理过程分为：接受请求，处理请求，返回请求。

* 接受请求。该阶段主要任务就是接受来自各个客户端的RPC请求，并将他们封装成固定格式(Call类)放到callQueue共享队列中，以便后续处理。该阶段又分为建立连接和接受请求两个子阶段，分别由Listener和Reader两种线程完成。
    * 整个Server只会有一个Listerner，统一监听来自客户端的连接请求，一旦有新的请求到达，就会采用轮询的方式从线程池中选取一个Reader进行处理。
    * Reader 线程可同时存在多个,它们分别负责接收一部分客户端连接的RPC请求,至于每个Reader线程负责哪些客户端连接,完全由Listener决定,当前Listener只是采用了简单的轮询分配机制.
    * Listener和Reader线程内部各自包含一个Selector 对象,分别用于监听SelectionKey.OP_ACCEPT 和SelectionKey.OP_READ 事件。对于Listener线程,主循环的实现体是监听是否有新的连接请求到达,并采用轮询策略选择一个Reader线程处理新连接;对于Reader线程,主循环的实现体是监听(它负责的那部分)客户端连接中是否有新的RPC请求到达,并将新的RPC请求封装成Call对象,放到共享队列callQueue中.
* 处理请求。该阶段主要任务是从共享队列 callQueue 中获取 Call 对象,执行对应的函数调用,并将结果返回给客户端,这全部由Handler线程完成.Server端可同时存在多个Handler线程,它们并行从共享队列中读取Call对象,经执行对应的函数调用后,将尝试着直接将结果返回给对应的客户端。但考虑到某些函数调用返回结果很大或者网络速度过慢,可能难以将结果一次性发送到客户端,此时Handler将尝试着将后续发送任务交给Responder线程.
* 返回结果。每个 Handler 线程执行完函数调用后,会尝试着将执行结果返回给客户端对于特殊情况,比如函数调用返回结果过大或者网络异常情况(网速过慢),会将发送任 务交给 Responder 线程。Server 端仅存在一个 Responder 线程,它的内部包含一个 Selector 对象,用于监听SelectionKey.OP_WRITE 事件。当 Handler 没能将结果一次性发送到客户端时,会向该Selector 对象注册 SelectionKey.OP_WRITE 事件,进而由 Responder 线程采用异步方式继续 发送未发送完成的结果。
* 参数设置，上述Server的Reactor涉及到以下几个参数可以进行优化，Reader线程数目，每个Handle线程对应的最大Call数目，Handle线程数目。

![img](../image/hadoop/hadoop-protocol-5.png)

#### 4.4 Hadoop RPC怎么支持多个序列化框架。

前文提到Hadoop RPC支持Protocol Buffers，Apache Avro，以及自带的Writeable三个序列化框架，那么它是怎么实现的呢，先来看下类关系图。

![img](../image/hadoop/hadoop-protocol-6.png)

上图显示RPC通过RpcEngine来区分不同的序列化框架，每一个RpcEngine都实现了Client的构建，Server的获取，以及Invoke的实现。在创建Client和Server需要传入协议接口以及接口的实现。

```java
@Override
  public RPC.Server getServer(Class<?> protocol, Object protocolImpl,
      String bindAddress, int port, int numHandlers, int numReaders,
      int queueSizePerHandler, boolean verbose, Configuration conf,
      SecretManager<? extends TokenIdentifier> secretManager,
      String portRangeConfig)
      throws IOException {
    return new Server(protocol, protocolImpl, conf, bindAddress, port,
        numHandlers, numReaders, queueSizePerHandler, verbose, secretManager,
        portRangeConfig);
  }
  @Override
  @SuppressWarnings("unchecked")
  public <T> ProtocolProxy<T> getProxy(Class<T> protocol, long clientVersion,
      InetSocketAddress addr, UserGroupInformation ticket, Configuration conf,
      SocketFactory factory, int rpcTimeout, RetryPolicy connectionRetryPolicy,
      AtomicBoolean fallbackToSimpleAuth) throws IOException {
    final Invoker invoker = new Invoker(protocol, addr, ticket, conf, factory,
        rpcTimeout, connectionRetryPolicy, fallbackToSimpleAuth);
    return new ProtocolProxy<T>(protocol, (T) Proxy.newProxyInstance(
        protocol.getClassLoader(), new Class[]{protocol}, invoker), false);
  }
```

## 5. HDFS的通信协议

* DatanodeProtocol (DN && NN) 。
* InterDatanodeProtocol (DN && DN) .
* ClientDatanodeProtocol (Client && DN)
* ClientProtocol (Client && NN)
* NamenodeProtocol (SNN && NN)

![img](../image/hadoop/hadoop-protocol-7.png)

## 6. 总结

本文从整体上介绍了HDFS的通信框架, 希望通过本文的阅读能对HDFS的通信过程有个整体的认识.

本文完




* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/hadoop-hdfs-protocol
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
