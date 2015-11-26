---
layout: post
title: Hive数据仓库之行分隔符line delimiter问题
date: 2015-11-26 12:30:00
categories: 大数据
tags: Hive Mongo-Hadoop
---
# Hive数据仓库之行分隔符line delimiter问题

## 场景

* 使用Mongo-Hadoop对Mongo的Product进行映射, 在进行查询的时候发现出现很多NULL, 查询productid发现出现字段的错位。
* 原来productname这个字段里面存在"\r\n", 而Hive默认的line delimiter 是"\n", 因此当读到productname时候就以为是多行而进行切分.

那么是否可以修改Hive line delimiter为其他的字符呢?

## 解决方案

Hive 的 Language Manual 说修改line delimiter是可以的,
{% highlight bash java %}
row_format
: DELIMITED [FIELDS TERMINATED BY char [ESCAPED BY char]] [COLLECTION ITEMS TERMINATED BY char]
[MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char]
[NULL DEFINED AS char] – (Note: Available in Hive 0.13 and later)
SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]
{% endhighlight java %}

但是 实际上FIELDS TERMINATED, COLLECTION ITEMS TERMINATED, MAP KEYS TERMINATED都可以进行修改, 唯独LINES TERMINATED 没法修改, 当试图修改LINES TERMINATED时候会报错
{% highlight bash java %}
Error: Error while compiling statement: FAILED: SemanticException 1:762 LINES TERMINATED BY only supports newline '\n' right now. Error encountered near token ''\u0007'' (state=42000,code=40000)
{% endhighlight java %}

由此可见Hive代码里面是禁止修改的。
可以查看Hive JIRA:
* https://issues.apache.org/jira/browse/HIVE-5999
* https://issues.apache.org/jira/browse/HIVE-11996

目前该问题提了好多年了, 但是Hive并没有有效的解决方案, 主要原因是:

* This is not fixable currently because the line terminator is determined by LineRecordReader.LineReader which is in the Hadoop land.

由此可见, 暂时是无法本质上解决该问题了, 虽然很多数据库的字段是支持"\r\n"的, 那么我们只能从Monogo-hadoop上解决了。
主要思路对STRING类型字段进行过滤'\r\n'
{% highlight bash java %}
private Object deserializePrimitive(final Object value, final PrimitiveTypeInfo valueTypeInfo) {
    switch (valueTypeInfo.getPrimitiveCategory()) {
        case BINARY:
            return value;
        case BOOLEAN:
            return value;
        case DOUBLE:
            return ((Number) value).doubleValue();
        case FLOAT:
            return ((Number) value).floatValue();
        case INT:
            return ((Number) value).intValue();
        case LONG:
            return ((Number) value).longValue();
        case SHORT:
            return ((Number) value).shortValue();
        case STRING:
            return value.toString().replaceAll("\r|\n","");  //过滤\r\n
        case TIMESTAMP:
            if (value instanceof Date) {
                return new Timestamp(((Date) value).getTime());
            } else if (value instanceof BSONTimestamp) {
                return new Timestamp(((BSONTimestamp) value).getTime() * 1000L);
            } else if (value instanceof String) {
                return Timestamp.valueOf((String) value);
            } else {
                return value;
            }
        default:
            return deserializeMongoType(value);
    }
}
{% endhighlight java %}

本文完
