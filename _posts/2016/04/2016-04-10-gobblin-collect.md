---
layout: post
title: Gobblin系列分析汇总
date: 2016-04-10 18:30:00
categories: 大数据
tags: Gobblin
---

## 简介

Gobblin是由linkin开源的Hadoop通用数据摄取框架，可以从各种数据源中提取，转换和加载海量数据。比如：数据库，rest APIs，kafka，等等。Gobblin 处理日常规划任务需要所有数据摄取ETLs，包括作业/任务规划，任务分配，错误处理，状态管理，数据质量检测，数据发布等等。

Gobblin 通过同样的执行框架从不同数据源摄取数据，在同一个地方管理所有不同数据源的元数据。同时结合了其他特性，比如自动伸缩，容错，数据质量保证，可扩展和处理数据模型改革等等。Gobblin 变得更容易使用，是个高效的数据摄取框架。

本文是Gobblin系列文章的目录索引。

## 目录

* [Gobblin系列(1)一之初探](<http://www.lamborryan.com/gobblin-first-exploration/>)
* [Gobblin系列(2)之History Store 和 Admin Server](<http://www.lamborryan.com/gooblin-history-store-admin-ui/>)
* [Gobblin系列(3)之Azkaban Schedule](<http://www.lamborryan.com/gobblin-azkaban/>)
* [Gobblin系列(4)之Runtime初探](<http://www.lamborryan.com/gobblin-runtime-view/>)
* [Gobblin系列(5)之Writer源码分析](<http://www.lamborryan.com/gobblin-partition-writer/>)
* [Gobblin系列(6)之State](<http://www.lamborryan.com/gobblin-state/>)
* [Gobblin系列(7)之Source源码分析](<http://www.lamborryan.com/gobblin-source/>)
* [Gobblin系列(8)之Extractor源码分析](<http://www.lamborryan.com/gobblin-extractor/>)


本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/gobblin-collect
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
