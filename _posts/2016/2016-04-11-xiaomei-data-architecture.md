---
layout: post
title: 小美数据平台总结
date: 2016-04-11 10:30:00
categories: 在学习架构的路上
tags: 数据架构
---

## 1. 前言

小美项目即将结束, 所以有必要抽时间来总结下在小美项目上的得失。

很庆幸在小美项目的7-8个月里, 全程参与了小美大数据平台从无到有的过程, 更加难得的是作为主要的后端开发人员参与了数据平台的架构变迁。中间踩坑的经验是很宝贵的, 所以就有了本文了。

虽然我们团队一直没超过4个人, 但是我们是个很好的团队, 很幸运能遇到他们。

虽然最后数据平台还很不完善, 但是它也经历了3个阶段:

* 第1阶段, 单机的数据平台
* 第2阶段, 混合的数据平台
* 第3阶段, 较完全的大数据平台

## 2. 第1阶段

在第一阶段, 数据的需求突然大量增加, 各部门需要大量的数据报表。 需求的增加与项目初期开发人员短缺, 技术积累不够形成了较大矛盾。

考虑到需要快速上手以及快速满足需求, 我们选了Pandas框架来进行数据仓库建模和数据分析, 使用Flask来实现后台数据交互. 同时由于对业务的理解不够深入, 以及产品设计上的不完善, 使得数据仓库的维度模型比较混乱, 维度表与事实表比较模糊。这样造成的结果就是每一个需求都是定制的, 每个开发人员疲于奔命, 数据分析变成了存体力活, 效率底下。

在第1阶段中期, 我们开始慢慢开发我们自己的基于Pandas的XiaoMei ETL-Engineer和XiaoMei Query Engineer来抽象并快速提升开发效率。它们基于jinjia2模板引擎可通过配置文件快速部署开发。在此阶段的任务调度完全就是基于Crontab的顺序调度.

由于此时的业务系统数据库是在Mysql和Mongo上, 在ETL过程中要同时摄取上述两个源. 而Mongo的Nosql特性使得业务系统的同事经常变动里面的字段, 给我们的数据统计造成较大的麻烦。

在第1阶段后期, 通过Pandas的ETL Engineer和Query Engineer, 我们基本上已经能满足日常的数据需求和报表需求。但是我们组在报表等临时需求上投入了过多精力, 大大降低平台搭建速度. 此时的建完模型的数据是存在MySql上的, 但是公司内部会SQL的运营基本没有, 因此我们调研了tableau, 并给需求方进行培训, 让他们通过tableau来满足他们自己的需求, 而我们只需要维护好数据仓库和数据平台即可。

此时的基本框架如下:

![img](../image/architecture/xiaomei-1.png)

该框架已经能基本满足初创团队对于数据统计分析的需求, 它维护成本低,只需部署在单台节点上,操作简单。

## 3. 第2阶段

随着业务增加, 更多的数据接入, 数据量得到了较大的增加. 实时统计的数据需求也开始产生。这时第一阶段的数据框架已经不满足现在的业务需求了。为了之后更好的扩展性, 本阶段将开始上线大数据平台的部分框架来代替ETL功能, 同时优化数据仓库的维度模型。

第2阶段的数据平台框架图:
![img](../image/architecture/xiaomei-2.png)

在这一阶段中我们有两条主线, 实时计算和离线计算:

#### 3.1 离线计算

在离线计算中我们使用Hadoop平台来实现数据的存储以及通过Hive来实现ETL和维度建模.

* Mongo到Hadoop的数据摄入我们采用Mongo-Hadoop, 请看我的博客
    * [《Hive数据仓库(4)之Mongo-Hadoop的使用》](<http://www.lamborryan.com/mongo-hadoop-hive/>),
    * [《Hive数据仓库(5)之Mongo-Hadoop的split》](<http://www.lamborryan.com/mongo-hadoop-hive-split/>),
    * [《Hive数据仓库(7)之Mongo-Hadoop的行分隔符line delimiter问题》](<http://www.lamborryan.com/end-of-record-delimiter/>);
    通过Mongo-Hive的映射, 我们在底层过滤了Mongo bson这种自由的文档类型格式对维度模型的影响。
* MySql数据摄入到Hadoop以及最后的维度数据导出到MySql, 都是通过Sqoop全量实现的。

由于业务系统变更较快，因此业务数据库的数据模型也经常变动, 常常上一个版本有这个字段， 这个版本该字段需要从别的地方获取了。 如果是第1阶段的那个模式, 就会使的我们越弄越混乱。在这阶段我通过Hive来屏蔽这种业务变更对上乘应用分析的影响。

```sql
CREATE TABLE deal_current
AS SELECT x
FROM deal_v_1.0
UNION
SELECT O AS X
FROM deal_v_1.1
```

每次业务版本变更, 我只需将deal_current转换成相应的版本就行。对上乘的维度模型影响较小。

#### 3.2 实时计算

在实时计算中我们采用Kafka, Spark Streaming 和 hbase的经典框架来实现。目前阶段的实时业务较简单, 无非就是pv, uv的计算以及一些页面留存等需求。Spark的入门还是有点难度的。

hbase的设计我们也下了不少的心思, 以统计某页面的pv,uv为例,

> UV/PV of events => count/sum(pv) of hbase rows prefix by {action}@U

> UV/PV of events on specified date => count/sum(pv) of hbase rows prefix by {action}@D{date}@U

> UV/PV of events with specified param => count/sum(pv) of hbase rows prefix by {action}@P{param}@U

> UV/PV of events with specified param on specified date => count/sum(pv) of hbase rows prefix by {action}@P{param}@D{date}@U

在Query Engineer我们采用python的happy-hbase对数据进行进一步的处理。

#### 3.3 工作流调度

业务量和需求的增长带来的另一个问题就是job数量增加, job之间的依赖并不能再简单的使用crontab了。这个时候我们调研了Oozie和Azkaban, 相比于Oozie的笨重, 我们最后选择了Azkaban来做为我们的工作流调度引擎。

关于Oozie的调研情况, 请查看我的博客:

* [《Oozie系列(1)之Oozie的安装》](<http://www.lamborryan.com/oozie-install/>)
* [《Oozie系列(2)之OOZIE运行Hive、Sqoop Action死锁问题分析》](<http://www.lamborryan.com/oozie-action-dead-lock/>)
* [《Oozie系列(3)之解决Sqoop Job无法运行的问题》](<http://www.lamborryan.com/oozie-sqoop-fail/>)

#### 3.4 维度模型

在该阶段我们已经建立初步的维度模型, 以商品订单为例:

![img](../image/architecture/xiaomei-3.png)

该阶段是单机向大数据平台的过渡模型, 我们在这个阶段持续了不少时间, 因为需要调研的东西实在不少。

## 4. 第3阶段

第3阶段是完整的大数据平台了, 我们需要在第2阶段的基础上要进行以下的优化:

* 用大数据的OLAP方案来代替Query Engineer, 因此我们引入了ElasticSearch, 通过它丰富的DSL语法来进行近实时的数据查询;
* 完善实时计算流, 因此我们开发了基于Spark-Streaming的实时流处理框架Plumber；
* 由于之前的数据后台只是简单的报表形式, 我们更丰富的展现方式和元素, 因此我们开发了基于AdminLTE,Vue.js和Meteor的数据可视化框架ArgonathUI;
* 优化数据源的摄入方式, 增加更多数据的落地, 因此我们引进了Gobblin框架来实现kafka, MySql数据的摄入以及初步的ETL过程;
* 增加更细颗粒度的数据管理, 因此我们采用Canal来订阅MySql的binlog, 为后续的监控做准备。

框架示意图如下:

![img](../image/architecture/xiaomei-4.png)

#### 4.1 ElasticSearch

最后选取的OLAP方案是通过ElasticSearch自带的丰富的DSL来实现, 因此在DSL的基础上我们开发了ElasticSearch的中间件来负责与ArgonathUI通信。我们可以通过Spark-Streaming实时为ElasticSearch建立索引, 也可以通过ElasticSearch－Hadoop快速的进行索引重建. 关于Hive如何实现ElasticSearch的索引重建可以查看[《Hive数据仓库(10)之Hive与Elasticsearch的结合》](<http://www.lamborryan.com/hive-elasticsearch/>).

#### 4.2 实时流处理框架Plumber

Plumber的原理示意图如下所示:
![img](../image/architecture/xiaomei-5.png)

* Plumber选取kafka来作为管道称之为pip, 作为数据的临时存放处。
* 当数据源Source接入到kafka后, 数据变为初始版本version 0, 经过各个Plumber Convert处理后, kafka内储存了不同version的数据。我们可以订阅这些不同version的数据。
* 从version 0 经过 Plumber Convert 变为 version1 这个过程我们称之为一次Plumber, 如图右边所示.
* Plumber分为inlet, Convert，outLet。
    * inlet即输入数据源, 即可以是kafka源, 也可以是其他数据源. 如果是其他的数据源, 那么它在pip里就是version 0。
    * Convert的过程由用户自己定义。
    * OutLet即输出, 即可以输出到pip中,则version＋1, 也可以sink到Elasticsearch或者hbase中.
* 我们可以通过配置来实现Plumber, 这个过程称为Bind.

Plumber即将开源, 敬请期待。

#### 4.3 ArgonathUI

关于ArgonathUI请看github[《ArgonathUI》](<https://github.com/FollowDataKing/ArgonathUI/blob/master/README.md>)

#### 4.4 Gobblin和Binlog的引入

Gobblin和Binlog的引入主要是为了完善对各种数据源数据的摄入以及优化ETL过程.

关于Gobblin的相关请看我的博客[《Gobblin系列分析汇总》](<http://www.lamborryan.com/gobblin-collect/>)

## 5. 总结与展望

第3阶段的大数据架构在ETL和数据统计分析上已经开始完善, 同时加入了数据可视化使得用户体验更好。

但是, 第3阶段的数据平台依然是很初级的阶段, 依然处在数据积累阶段上即存储层。 还没达到数据使用以及数据挖掘阶段(业务层和应用层)，项目就关闭了。

因此关于后续可以继续完善的地方, 我觉得还有以下方向:

* 加入权限管理模块, 不但对hadoop集群增加权限, 更需要对数据进行权限管理, 因此需要开发中间件。
* 加入监控模块, 用好binlog和log日志, 通过它们来实现实时监控。
* 在现有的数据仓库上, 加入特征,模型, 推荐, 搜索等业务应用层的架构。


数据平台建设的道路, 任重而道远。

本文完




* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：[http://www.lamborryan.com/xiaomei-data-architecture](<http://www.lamborryan.com/xiaomei-data-architecture>)
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
