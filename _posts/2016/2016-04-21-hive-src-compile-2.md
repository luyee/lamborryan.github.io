---
layout: post
title: Hive源码分析系列(4)之Compile原理分析(2)-语义分析
date: 2016-04-21 23:30:00
categories: 大数据
tags: Hive源码分析
---

## 1. 简介

前文[《Hive源码分析系列(3)之Compile原理分析(1)-抽象语法树AST》](<http://www.lamborryan.com/hive-src-compile/>)介绍Hive编译过程的第一步即利用Anstr将SQL转换成抽象语法树AST, 本文将接着讲Hive的编译过程。也就是以下几个步骤:

* 遍历AST Tree，抽象出查询的基本组成单元QueryBlock
* 遍历QueryBlock，翻译为执行操作树OperatorTree
* 逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量
* 遍历OperatorTree，翻译为MapReduce任务
* 物理层优化器进行MapReduce任务的变换，生成最终的执行计划

经过AST转换后, AST Tree仍然非常复杂，不够结构化，不方便直接翻译为MapReduce程序，接下来将对SQL进一部抽象和结构化。本文主要的代码集中在SemanticAnalysis这个类中, 由于这个类有12000多行, 所以本文尽量少贴代码。

## 2. 代码结构

在```Driver.compile```中讲到, 生成语法树之后进一步解析其实只有两步:

```java
public int compile(String command, boolean resetTaskIds) {
    // ...
    BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(conf, tree);
    // ...
    sem.analyze(tree, ctx);
    // ...
}
```

所以后续的语义分析都从SemanticAnalyzerFactory和BaseSemanticAnalyzer开始展开。

### 2.1 SemanticAnalyzerFactory

Hive采用了SemanticAnalyzerFactory这个工厂来实现语义模块之间的关系, 它会根据相应的token类型来决定生成哪个BaseSemanticAnalyzer的子类。

![img](../image/hive/hive-src-compile/compile2.png)

各个具体实现类的具体意义如下所示:

* ExplainSemanticAnalyzer,只分析执行计划
* LoadSemanticAnalyzer,装载语句的语义解析
* ExportSemanticAnalyzer,导出语句的语义解析
* ImportSemanticAnalyzer,导入语句的语义解析
* DDLSemanticAnalyzer,数据定义语言的语义解析
* FunctionSemanticAnalyzer,函数的语义解析
* ColumnStatsSemanticAnalyzer,列统计语义分析
* MacroSemanticAnalyzer,宏语义分析
* SemanticAnalyzer,常用的语义分析，主要是查询

每一个实现类都对应不同的token类型。

![img](../image/hive/hive-src-compile/compile3.png)

上面的图都比较小, 可以放大查看。

### 2.2 BaseSemanticAnalyzer

查看BaseSemanticAnalyzer的analysis方法:

```java
public void analyze(ASTNode ast, Context ctx) throws SemanticException {
  // 初始化
  initCtx(ctx);
  init(true);
  // 语义解析
  analyzeInternal(ast);
}

public abstract void analyzeInternal(ASTNode ast) throws SemanticException;
```

由此可见各种BaseSemanticAnalyzer的实现类都只要实现analyzeInternal这个方法就行了。

### 2.3 ExplainSemanticAnalyzer

BaseSemanticAnalyzer的实现类实在太多了, 我就不一一展开了, 由于除了SemanticAnalyzer之外其他的都是特殊的语义分析器, 所以以ExplainSemanticAnalyzer来看下analyzeInternal怎么实现的。

```java
@Override
public void analyzeInternal(ASTNode ast) throws SemanticException {
    // ...
  for (int i = 1; i < ast.getChildCount(); i++) {
  int explainOptions = ast.getChild(i).getType();
  if (explainOptions == HiveParser.KW_FORMATTED) {
        formatted = true;
      } else if (explainOptions == HiveParser.KW_EXTENDED) {
        extended = true;
      } else if (explainOptions == HiveParser.KW_DEPENDENCY) {
        dependency = true;
      } else if (explainOptions == HiveParser.KW_LOGICAL) {
        logical = true;
      } else if (explainOptions == HiveParser.KW_AUTHORIZATION) {
        authorize = true;
      }
    }

    // ...

    // Create a semantic analyzer for the query
    ASTNode input = (ASTNode) ast.getChild(0);
    BaseSemanticAnalyzer sem = SemanticAnalyzerFactory.get(conf, input);
    sem.analyze(input, ctx);
    sem.validate();

    // ...
}
```

从上面代码看:

1.不同的语义分析器BaseSemanticAnalyzer都是按照token对应rule进行解析的。

2.BaseSemanticAnalyzer解析AST Tree是以递归的形式的。

比如```EXPLAIN SELECT * FROM A```, 它的AST树中, ```HiveParser.TOK_EXPLAIN```是```HiveParser.TOK_SELECT```, 所以在进行解析的时候第一层调用ExplainSemanticAnalyzer,第二层调用SemanticAnalyzer进行解析。

## 3. SemanticAnalyzer

Hive的主要语义解析在SemanticAnalyzer中, 所以后续集中介绍SemanticAnalyzer。

### 3.1 SQL基本组成单元QueryBlock

本小节主要内容就是Hiver遍历AST Tree,抽象出查询的基本组成单元QueryBlock, 它分为以下几步:

#### 3.1.1 处理位置别名

正常情况下TOK_SELECT,TOK_GROUPBY,TOK_TOK_ORDERBY是在同一层AST中的,```processPositionAlias```会遍历每一个节点的子树, 将```group by```或者```order by```后面的数字转换成```select```后面指出的列名。

例如```processPositionAlias```会将```SELECT id, count(id) FROM person GROUP BY 1``` 转换为```SELECT id, count(id) FROM person GROUP BY id```,
同理将```SELECT id, name FROM person ORDER BY 1 DESC```转换成```SELECT id, name FROM person ORDER BY id DESC```。

> 以前的hive在这个功能上有bug, 目前1.2.1可以正常运行。
> 运行该功能需要```set hive.groupby.orderby.position.alias=true;```

#### 3.1.2 分析创建表命令

如果当前抽象语法树节点类型是HiveParser.TOK_CREATETABLE, 则需要对CREATE语句进行特殊处理。处理方案如下:



### 3.2 Gen OP Tree from resolved Parse Tree

### 3.3 Deduce Resultset Schema

### 3.4 Generate Parse Context for Optimizer & Physical compiler

### 3.5 Take care of view creation

### 3.6 Generate table access stats if required

### 3.7 Perform Logical optimization

### 3.8 Generate column access stats if required - wait until column pruning takes place during optimization

### 3.9 Optimize Physical op tree & Translate to target execution engine (MR, TEZ..)

### 3.10 put accessed columns to readEntity

### 3.11 if desired check we're not going over partition scan limits

## 4. 总结


## 参考文献

* [Hive SQL的编译过程](<http://tech.meituan.com/hive-sql-to-mapreduce.html>)
* [技术乱弹之hive源码之词法语法解析](<https://github.com/alan2lin/hive_ql_parser#_Toc368923114>)
* [Compiler - lexical analysis](<http://www.cnblogs.com/RicCC/archive/2008/10/04/compiler-lexical-analysis.html>)
* [使用 Antlr 开发领域语言](<http://www.ibm.com/developerworks/cn/java/j-lo-antlr/>)
* [技术乱弹之hand in hand with antlr](<https://github.com/alan2lin/hand_in_hand_with_antlr>)


本文完

* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：[http://www.lamborryan.com/hive-src-compile－2](<http://www.lamborryan.com/hive-src-compile-2>)
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
