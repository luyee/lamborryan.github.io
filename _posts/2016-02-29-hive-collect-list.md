---
layout: post
title: Hive数据仓库之collect_list浅谈
date: 2016-02-29 13:00:01
categories: 大数据
tags: Hive
---

## 前言
Hive的collect_list函数是个聚合函数, 它可以将Group By后的列聚合为一个list(不去重)。类似的有collect_set(去重)。

函数        |作用     |返回类型
-------    |------   |------
collect_set(col) |Returns a set of objects with duplicate elements eliminated |array
collect_list(col)|Returns a list of objects with duplicates |array

现在有这样结构的数据需要聚合得到每个用户的skuId, skuId按count,buyTime倒序, 生成的数据保存到mongo上去

``` Java
class UserCollectDB {
    private ObjectId id;
	private long userId;// 用户ID
	private List<UserCollectItem> items;
}
class UserCollectItem {
    private String templateId;
    private String skuId;
    private int count;
    private long buyTime;
}
```

userId        |skuId     |templateId | count |buyTime
-------       |------    |------     |------ |------
a             | A01      |dsadasfd   | 2     | 2015-10-01 00:00:01
a             | A02      |efeddsad   | 3     | 2015-10-02 00:00:01
b             | A02      |efeddsad   | 5     | 2015-10-03 10:00:02

这时需要用到collect_list进行聚合。

## 实现

### 生成struct

Constructor Function | Operands | Description
---------------------|----------|------------
map|(key1, value1, key2, value2, ...)|Creates a map with the given key/value pairs.
struct| (val1, val2, val3, ...)|Creates a struct with the given field values. Struct field names will be col1, col2, ....
named_struct|(name1, val1, name2, val2, ...)|Creates a struct with the given field names and values. (As of Hive 0.8.0.)
array|(val1, val2, ...)|Creates an array with the given elements.
create_union|(tag, val1, val2, ...)|Creates a union type with the value that is being pointed to by the tag parameter.

所以可以用named_struct来组合多个filed为struct.

``` sql
NAMED_STRUCT (
    "templateId", templateId,
    "skuId", skuId,
    "count", count,
    "buyTime", buyTime
)
```

* 注意: 使用struct(val1, val2, val3, ...) 方式, struct内部的field name自动为col1, col2, col3
* 使用named_struct 可以重命名field name

### 使用collect_list

使用以下collect_list进行聚合会出现错误, 由此可见本人的hive(1.2.1)不支持collect_list 对struct和map进行聚合。

* UDFArgumentTypeException Only primitive type arguments are accepted.

``` sql
SELECT
userId,
collect_list(
    NAMED_STRUCT (
        "templateId", templateId,
        "skuId", skuId,
        "count", count,
        "buyTime", buyTime
    )
) AS items
FROM st
GROUP BY userId;
```

查看hive-jira 发现了这个jira [Hive-10427](https://issues.apache.org/jira/browse/HIVE-10427), 由此可见可以通过打补丁来修复这个功能, 而不需要自定义UDAF

``` Java
@Override
public GenericUDAFEvaluator getEvaluator(TypeInfo[] parameters)
    throws SemanticException {
  if (parameters.length != 1) {
    throw new UDFArgumentTypeException(parameters.length - 1,
        "Exactly one argument is expected.");
  }

  switch (parameters[0].getCategory()) {
    case PRIMITIVE:
    case STRUCT:
    case MAP:
      break;
    default:
      throw new UDFArgumentTypeException(0,
          "Only primitive, struct or map type arguments are accepted but "
              + parameters[0].getTypeName() + " was passed as parameter 1.");
  }
  return new GenericUDAFMkCollectionEvaluator(BufferType.LIST);
}
```

### 打patch

* 下载jira内的4个patch, 并copy到hive的源码根目录下新建的patch目录内
* 使用patch -p0 < ./patch/HIVE-10427.<i>.patch 依次打补丁, i替换为1-4
* 重新编译HIVE mvn clean package -Phadoop-2
* 将ql/target/hive-exec-1.2.1.jar替换线上的hive-exec-1.2.1.jar, 重启hive.
* 本次补丁主要涉及GenericUDAFCollectList.java, GenericUDAFCollectSet.java, GenericUDAFMkCollectionEvaluator.java, GenericUDFSortArray.java

关于如何实现udaf, 将在下文单独介绍.

本文完


* 原创文章，转载请注明： 转载自[Lamborryan](<lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/hive-collect-list
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
