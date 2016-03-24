---
layout: post
title: Gobblin系列五之Source和Extractor源码分析
date: 2016-03-21 13:30:00
categories: 大数据
tags: Gobblin
---
# Gobblin系列五之Source和Extractor源码分析

{% highlight java %}
private void buildWriterIfNotPresent() throws IOException {
  if (!this.writer.isPresent()) {
    try {
      this.writer = Optional.of(this.closer.register(buildWriter()));
    } catch (SchemaConversionException sce) {
      throw new IOException("Failed to build writer for fork " + this.index, sce);
    }
  }
}
{% endhighlight %}
