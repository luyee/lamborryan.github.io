---
layout: post
title: Hive源码分析系列(4)之Compile原理分析(2)-QB Tree
date: 2016-04-21 23:30:00
categories: 大数据
tags: Hive源码分析
---

## 1. 简介

前文[《Hive源码分析系列(3)之Compile原理分析(1)-抽象语法树AST》](<http://www.lamborryan.com/hive-src-compile/>)介绍Hive编译过程的第一步即利用Anstr将SQL转换成抽象语法树AST, 本文将接着讲Hive的编译过程。也就是以下几个步骤:

* Antlr定义SQL的语法规则，完成SQL词法，语法解析，将SQL转化为抽象语法树AST Tree
* 遍历AST Tree，抽象出查询的基本组成单元QueryBlock
* 遍历QueryBlock，翻译为执行操作树OperatorTree
* 逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量
* 遍历OperatorTree，翻译为MapReduce任务
* 物理层优化器进行MapReduce任务的变换，生成最终的执行计划

本文主要讲解编译过程的第二步, AST转换成QueryBlock. 经过AST转换后, AST Tree仍然非常复杂，不够结构化，不方便直接翻译为MapReduce程序，接下来将对SQL进一部抽象和结构化。本文主要的代码集中在SemanticAnalysis这个类中, 由于这个类有12000多行, 所以本文尽量少贴代码。

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

## 3. SQL基本组成单元QueryBlock

Hive的主要语义解析在SemanticAnalyzer中, 所以后续集中介绍SemanticAnalyzer。

本节主要内容就是Hiver遍历AST Tree,抽象出查询的基本组成单元QueryBlock, 代码主要在```genResolvedParseTree(ast, plannerCtx)```中。

### 3.1 处理位置别名

正常情况下TOK_SELECT,TOK_GROUPBY,TOK_TOK_ORDERBY是在同一层AST中的,```processPositionAlias```会遍历每一个节点的子树, 将```group by```或者```order by```后面的数字转换成```select```后面指出的列名。

例如```processPositionAlias```会将```SELECT id, count(id) FROM person GROUP BY 1``` 转换为```SELECT id, count(id) FROM person GROUP BY id```,
同理将```SELECT id, name FROM person ORDER BY 1 DESC```转换成```SELECT id, name FROM person ORDER BY id DESC```。

> 以前的hive在这个功能上有bug, 目前1.2.1可以正常运行。
> 运行该功能需要```set hive.groupby.orderby.position.alias=true;```

### 3.2 分析创建表命令

如果当前抽象语法树节点类型是HiveParser.TOK_CREATETABLE, 则需要对CREATE语句进行特殊处理。

有三种创建表的情况，

1)  正常的create-table语句。

例如 CREATE TABLE invites (foo INT, bar STRING) PARTITIONED BY (ds STRING);

2)  create-table-like 语句。

例如 CREATE TABLE x LIKE invites;

3)  create-table-as-select 语句。

例如 CREATE TABLE y AS SELECT * FROM invites;

这个阶段的语义分析主要是语义检查和语义处理。

语义检查:

1)  检查，create-table-like 与 create-table-as-select 不能共存，不能够 创建一个 table like xx 而又 as select ...;

2)  检查 create-table-like 与 create-table-as-select，确定它们不能有列名。换句话说，这两种情况创建的表的模式都是拷贝过来的;

3)  检查 create-table-as-select 现在不支持分区。

4)  检查其他的一些建表的逻辑。

语义处理:

1) 如果是create-table 和 create-table-like 这两种情况，它们只是影响了元数据，属于ddl的范围，直接交由 ddlwork去处理，返回null结束处理。

2) 如果是create-table-as-select则需要获取元信息并存入qb，返回select语句所在的树，表示后续还需要处理。填充默认的存储格式，对不同的文件格式 ，顺序文件 rcfile orc等之类的文件设置他们的输入格式类和输出格式类以及序列化反序列化类。创建表描述，存入qb中。

列出这三种情况的抽象语法树， 分析一下处理流程

CREATE TABLE invites (foo INT, bar STRING) PARTITIONED BY (ds STRING)

![img](../image/hive/hive-src-compile/create_table_1.jpg)

CREATE TABLE x LIKE invites

![img](../image/hive/hive-src-compile/create_table_2.jpg)

CREATE TABLE y AS SELECT * FROM invites

![img](../image/hive/hive-src-compile/create_table_3.jpg)

分析建表命令处理过程会逐一处理TOK_CREATETABLE下面的字节点，从第二个字节点开始处理。

如果是create-table  ，就把从TOK_TABCOLLIST取出列定义来创建来表描述。

如果是create-table-like，就取出like表名，用来创建表描述。

如果是create-table-as-select 也是从目标表明创建表描述，同时将TOK_QUERY 指向的节点返回。

### 3.3 分析创建视图命令

如果TOK_CREATEVIEW 和 TOK_ALTERVIEW(且TOK_QUERY, 也就是alter view as select) 两个节点，则进入分析创建视图命令过程。

相对于分析创建表命令而言，分析视图创建就很简单了，得到得到各列以及若干属性，用来创建视图描述，然后交由ddlwork处理，同时返回TOK_QUERY节点。

### 3.4 第一阶段处理(doPhase1)

在doPhase1之前先做两个准备工作, 1.初始化initPhase1Ctx,创建Phase1Ctx上下文, 2.遍历树, 查找insert into后的hdfs路径的文件是否加密, 如果加密则写入qb中。

#### 3.4.1 代码流程

现在开始进入doPhase1分析, doPhase1的过程比较复杂, 它通过递归的形式遍历AST将其转换成QB Tree.  由于doPhase1的代码较多, 而很大的部分也是重点在case语句对各种token的处理。

主要代码流程如下:

```java
public boolean doPhase1(ASTNode ast, QB qb, Phase1Ctx ctx_1, PlannerContext plannerCtx)
      throws SemanticException {
    boolean phase1Result = true;
    QBParseInfo qbp = qb.getParseInfo();
    boolean skipRecursion = false;

    if (ast.getToken() != null) {
        skipRecursion = true;
        switch (ast.getToken().getType()) {
            case XXX:
                todo(); // 对特定的token进行解析
                break;
            default:
                skipRecursion = false;
                break;
        }
    }

    if (!skipRecursion) {
        // Iterate over the rest of the children
        int child_count = ast.getChildCount();
        for (int child_pos = 0; child_pos < child_count && phase1Result; ++child_pos) {
            // Recurse
            phase1Result = phase1Result && doPhase1(
                (ASTNode)ast.getChild(child_pos), qb, ctx_1, plannerCtx);
        }
    }
    return phase1Result;
}
```

逻辑清晰地显示， dophase1 采用先根的方式递归遍历抽象语法树，从左右到右边递归处理子节点。

递归的出口是 在某些个节点(token的处理过程)上设置的skipRecursion，以及子节点处理的结果。

#### 3.4.2 QueryBlock

QueryBlock是一条SQL最基本的组成单元，包括三个部分：输入源，计算过程，输出。简单来讲一个QueryBlock就是一个子查询。

下图为Hive中QueryBlock相关对象的类图，解释图中几个重要的属性

* QB#aliasToSubq（表示QB类的aliasToSubq属性）保存子查询的QB对象，aliasToSubq key值是子查询的别名
* QB#qbp即QBParseInfo保存一个基本SQL单元中的各个操作部分的AST Tree结构，QBParseInfo#nameToDest这个HashMap保存查询单元的输出，key的形式是inclause-i（由于Hive支持Multi Insert语句，所以可能有多个输出），value是对应的ASTNode节点，即TOK_DESTINATION节点。类QBParseInfo其余HashMap属性分别保存输出和各个操作的ASTNode节点的对应关系。
* QBParseInfo#JoinExpr保存TOK_JOIN节点。QB#QBJoinTree是对Join语法树的结构化。
* QB#qbm保存每个输入表的元信息，比如表在HDFS上的路径，保存表数据的文件格式等。
* QBExpr这个对象是为了表示Union操作。

![img](../image/hive/hive-src-compile/qb_1.png)

#### 3.4.3 AST转QueryBlock

AST Tree生成QueryBlock的过程是一个递归的过程，先序遍历AST Tree，遇到不同的Token节点，保存到相应的属性中，

##### SELECT TOKENS

以TOK_SELECTDI和TOK_SELECT这两个token为例。

```java
switch (ast.getToken().getType()) {
    case HiveParser.TOK_SELECTDI:
        qb.countSelDi();
        // fall through
    case HiveParser.TOK_SELECT:
        qb.countSel();
        qbp.setSelExprForClause(ctx_1.dest, ast);

        int posn = 0;
        if (((ASTNode) ast.getChild(0)).getToken().getType() == HiveParser.TOK_HINTLIST) {
            qbp.setHints((ASTNode) ast.getChild(0));
            posn++;
        }

        if ((ast.getChild(posn).getChild(0).getType() == HiveParser.TOK_TRANSFORM))
        queryProperties.setUsesScript(true);

        LinkedHashMap<String, ASTNode> aggregations = doPhase1GetAggregationsFromSelect(ast,
          qb, ctx_1.dest);
        doPhase1GetColumnAliasesFromSelect(ast, qbp);
        qbp.setAggregationExprsForClause(ctx_1.dest, aggregations);
        qbp.setDistinctFuncExprsForClause(ctx_1.dest,
        doPhase1GetDistinctFuncExprs(aggregations));
        break;
}
```

主要进行了一下几步:

* 增加select/select distinct的计数器 ，若有hints，设置hints, 如使用transform 脚本则标记下。
* 将查询表达式的语法部分保存在destToSelExpr、exprToColumnAlias、destToAggregationExprs、destToDistinctFuncExprs四个属性中

可以下图来更进一步说明上下文:

![img](../image/hive/hive-src-compile/qb_2.png)

图可以放大查看。图中展示了qbp,qb,以及上下文的重要的几个属性。

> doPhase1GetAggregationsFromSelect 对select节点的孙子节点进行遍历，递归寻找聚集函数和窗口。并对自定义函数是否要求隐含排序进行语义检查。
> 这一步主要是对tok_select的孙节点层进行递归遍历，处理tok_function和 tok_functiondi 和tok_functionstar节点。如果该节点下面是 tok_windowspec则认为是窗口函数，填充到出口参数，返回。如果是identifier 则认为是自定义函数，hive中的自定义函数范围挺广的，包含常见的运算符等都是以自定义函数的方式实现的。从函数注册信息中去判断自定义函数是否需要隐含排序的条件，如果有，此时肯定缺少了over从句，因为over从句会被语法解析转换成tok_windowspec了.

##### OTHER TOKENS

其他的tokens处理如下图所示:

![img](../image/hive/hive-src-compile/qb_3.png)

#### 3.4.4 QBMetaData

根据QB生成QBMetaData.

至此, 已完成AST到QueryBlock Tree的转换过程。

## 4. 总结

本文主要介绍了AST转成QueryBlock Tree的过程， 转换的本质就是将AST转换成更符合HiveSQL的方式来组织。


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
