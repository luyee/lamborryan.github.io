---
layout: post
title: Hive数据仓库之数据去重
date: 2015-11-24 13:30:00
categories: 大数据
tags: Hive
---
# Hive数据仓库之数据去重

## 使用场景

* 对于Hive来说没有primary key这样的概念, 所以没法根据primary_key进行去重.
* Hive没有update操作, 所以对于那种经常会改变状态的数据来说, Hive没法直接实现更新. 当有新的数据生产时候, 我们只能追加到Hive里面, 这时候会有id一样但是状态不一样的数据, 这时候我们就可以通过数据去重实现update。

当碰到以上两种情况时候, 我们只能通过去重来实现数据更新, 还好Hive内置的row_number()函数很好的帮助我们实现去重。

## 使用例子

### 新建raw_number文件存放以下数据:
{% highlight bash sql %}
1,hangzhou,28
1,shanghai,30
1,shenzhen,2
2,beijing,10
3,shangyu,5
3,shaoxing,10
{% endhighlight sql %}

### 新建表并加载数据:
{% highlight bash sql %}
CREATE TABLE test_row_number(id int,city STRING,num int)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;
LOAD DATA LOCAL INPATH '/home/bmw/raw_number' OVERWRITE INTO TABLE test_row_number;
{% endhighlight sql %}

### 去重

{% highlight bash sql %}
select t.id, t.city, t.num
from(
    select *, ROW_NUMBER() over(distribute by id sort by num desc) rownumber
    from test_row_number
) t
where t.rownumber = 1

+-----+-----------+------+--+
| id  |   city    | num  |
+-----+-----------+------+--+
| 1   | shanghai  | 30   |
| 2   | beijing   | 10   |
| 3   | shaoxing  | 10   |
+-----+-----------+------+--+
3 rows selected (0.478 seconds)
{% endhighlight sql %}

### 说明
* 其中distribute by id 表示以id进行分区, sort by num desc 以num进行倒序(当然可以改成asc来获取最小的一个), t.rownumber = 1表示只取最大的一个。
* distribute by + sort 实现的是分区排序, 它只保证该分区的有序; order by 是全局排序, 它在单个reduce里面进行排序, 因此性能可能会很差。 这里使用分区排序刚好。详见 http://www.crazyant.net/1456.html

本文完


* 原创文章，转载请注明： 转载自[Lamborryan](<lamborryan.github.io>)，作者：[Ruan Chengfeng](<http://lamborryan.github.io/about/>)
* 本文链接地址：http://lamborryan.github.io/hive-row-number-duplicate
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
