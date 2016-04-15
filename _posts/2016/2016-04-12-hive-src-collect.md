---
layout: post
title: Hive源码分析系列汇总
date: 2016-04-12 18:30:00
categories: 大数据
tags: Hive源码分析
---

## 简介

Hive是基于Hadoop的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的sql查询功能，可以将sql语句转换为MapReduce任务进行运行。我们常常使用Hive来进行数据维度建模以及ETL。

Hive源码分析系列着眼于通过源码来更深入的了解Hive, 希望了解什么是Hive以及如何使用Hive的同学可以看我的另外一个专栏[《Hive数据仓库系列分析》](<http://www.lamborryan.com/hive-warehouse/>)以及《Hive权威指南》。

## 目录

* [Hive源码分析系列(1)之框架与Client/Server入口](<http://www.lamborryan.com/hive-src-entrance>)
* [Hive源码分析系列(2)之Driver流程分析](<http://www.lamborryan.com/hive-src-driver>)




本文完



* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：http://www.lamborryan.com/hive-src-collect
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
