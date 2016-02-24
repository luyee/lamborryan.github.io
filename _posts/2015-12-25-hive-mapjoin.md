---
layout: post
title: Hive优化之MapJoin
date: 2015-12-25 13:30:00
categories: 大数据
tags: Hive
---

# Hive优化之MapJoin

## 前言

在使用hive过程, 经常会碰到以下错误日志:
``` java
return code 3 from org.apache.hadoop.hive.ql.exec.mr.MapredLocalTask
```
当出现这个错误的时候,多半跟OOM有关了,而为啥出现OOM了多半跟join有关。我们以以下这个SQL来说明Hive是怎么进行优化的
{% highlight java linenos %}
SELECT
   dl.deal_id,
   dl.ship_price,
   dc.canceltype,
   ad.assignstatus,
   CASE WHEN ss.settlementStarNum is NULL THEN ss.starNum ELSE ss.settlementStarNum END as appraiseResult,
   SQRT(POW(ds.shop_area_map[0] - ap.maploc[0],2.0) + POW(ds.shop_area_map[1] - ap.maploc[1],2.0)) AS distance,
   CASE WHEN ds.initshippingfee is NULL THEN sc.initshippingfee ELSE  ds.initshippingfee END - dl.ship_price as ship_fee
FROM fact_deal dl
LEFT OUTER JOIN dealcancelreasondb dc
ON dl.deal_id = dc.id
LEFT OUTER JOIN assigndeal ad
ON dl.deal_id = ad.id
LEFT OUTER JOIN supershopperrating ss
ON dl.deal_id = ss.dealid
LEFT OUTER JOIN address ap
ON dl.ship_address_id = ap.addess_id.oid
Left OUTER JOIN dim_shop ds
ON dl.shop_id = ds.shop_id
LEFT OUTER JOIN supportedcity sc
ON ds.city_code == sc.city_code;
{% endhighlight %}
