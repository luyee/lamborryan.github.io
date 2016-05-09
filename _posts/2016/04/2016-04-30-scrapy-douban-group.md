---
layout: post
title: 利用Scrapy爬取豆瓣小组
date: 2016-04-30 18:30:00
categories: 网络爬虫
tags: Scrapy
---

## 1. 简介

当初刚做码农时候, 是从做搜索引擎开发开始的, 那个时候只是简单了解了Nutch, 却是意犹未尽。 所以一直想玩玩爬虫, 直到现在想学下数据挖掘时候需要利用爬虫爬取需要的数据时, 才好好体验了一把爬虫。果然很有趣。

本系列文章主要是记录我爬取各数据的笔记, 以及研究Scrapy过程中遇到的问题。 本文是我使用Scrapy的第一篇, 所以将简单介绍爬虫原理, Scrapy的框架以及使用, 和如何使用Scrapy爬取豆瓣小组。

豆瓣是个有趣的网站, 因为有趣它也成为新手爬虫起航的地方, 那我就用它开始爬虫的征途。

## 2. 爬虫简介

什么是爬虫, 在知乎上[如何入门 Python 爬虫？](<https://www.zhihu.com/question/20899988>)有人已经给出了通熟易懂的解答。我们这里主要使用它的答案来介绍下爬虫的基本原理。

> 想象你是一只蜘蛛，现在你被放到了互联“网”上。那么，你需要把所有的网页都看一遍。怎么办呢？没问题呀，你就随便从某个地方开始，比如说人民日报的首页，这个叫initial pages，用$表示吧。

> 在人民日报的首页，你看到那个页面引向的各种链接。于是你很开心地从爬到了“国内新闻”那个页面。太好了，这样你就已经爬完了俩页面（首页和国内新闻）！暂且不用管爬下来的页面怎么处理的，你就想象你把这个页面完完整整抄成了个html放到了你身上。

> 突然你发现， 在国内新闻这个页面上，有一个链接链回“首页”。作为一只聪明的蜘蛛，你肯定知道你不用爬回去的吧，因为你已经看过了啊。所以，你需要用你的脑子，存下你已经看过的页面地址。这样，每次看到一个可能需要爬的新链接，你就先查查你脑子里是不是已经去过这个页面地址。如果去过，那就别去了。

如果用伪代码来实现这个过程， 就是一下。

```python
import Queue

initial_page = "http://www.renminribao.com"

url_queue = Queue.Queue()
seen = set()

seen.insert(initial_page)
url_queue.put(initial_page)

while(True): #一直进行直到海枯石烂
    if url_queue.size()>0:
        current_url = url_queue.get()    #拿出队例中第一个的url
        store(current_url)               #把这个url代表的网页存储好
        for next_url in extract_urls(current_url): #提取把这个url里链向的url
            if next_url not in seen:     #如果没有爬过就接着爬
                seen.put(next_url)
                url_queue.put(next_url)
    else:
        break
```

当然实际应用的爬虫不可能这么简单, 往往涉及到网页分析,网页去重,存储,爬虫效率(分布式),增量爬取(更新问题),与网站的斗智斗勇各种反爬虫技术等等问题, 每一块技术都可以深入讲一大堆。而到现在为止, 我也并了解不多, 这些都是我学习这个爬虫需要取克服和翻越的大山。

## 3. Scrapy 简介

开始爬虫的时候, 一直在造轮子(自己写爬虫)还是使用轮子上(使用开源框架)犹豫着。奈何目前的重点是学习数据挖掘, 所以最后还是决定使用别人造的轮子。

Scrapy是Python的一个鼎鼎大名的开源项目, 由于之前一直使用的是python且数据挖掘算法也将用python写, 所以选定了scrapy这个爬虫框架, 体验的感觉还不错。

### 3.1 框架介绍

Scrapy有[中文文档](<http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/architecture.html>), 但是不是最新的。下图是scrapy的框架介绍。

![img](../image/scrapy/scrapy-douban-group/scrapy_architecture.png)

Scrapy主要由以下几个模块组成:

* Scrapy Engine。引擎负责控制数据流在系统中所有组件中流动，并在相应动作发生时触发事件。 详细内容查看下面的数据流(Data Flow)部分。
* 调度器(Scheduler)。调度器从引擎接受request并将他们入队，以便之后引擎请求他们时提供给引擎。
* 下载器(Downloader)。 下载器负责获取页面数据并提供给引擎，而后提供给spider。
* Spiders。 Spider是Scrapy用户编写用于分析response并提取item(即获取到的item)或额外跟进的URL的类。 每个spider负责处理一个特定(或一些)网站。
* Item Pipeline。 Item Pipeline负责处理被spider提取出来的item。典型的处理有清理、 验证及持久化(例如存取到数据库中)。
* 下载器中间件(Downloader middlewares)。 下载器中间件是在引擎及下载器之间的特定钩子(specific hook)，处理Downloader传递给引擎的response。 其提供了一个简便的机制，通过插入自定义代码来扩展Scrapy功能。
* Spider中间件(Spider middlewares)。 Spider中间件是在引擎及Spider之间的特定钩子(specific hook)，处理spider的输入(response)和输出(items及requests)。 其提供了一个简便的机制，通过插入自定义代码来扩展Scrapy功能。

Scrapy的数据流程:

* 引擎打开一个网站(open a domain)，找到处理该网站的Spider并向该spider请求第一个要爬取的URL(s)。
* 引擎从Spider中获取到第一个要爬取的URL并在调度器(Scheduler)以Request调度。
* 引擎向调度器请求下一个要爬取的URL。
* 调度器返回下一个要爬取的URL给引擎，引擎将URL通过下载中间件(请求(request)方向)转发给下载器(Downloader)。
* 一旦页面下载完毕，下载器生成一个该页面的Response，并将其通过下载中间件(返回(response)方向)发送给引擎。
* 引擎从下载器中接收到Response并通过Spider中间件(输入方向)发送给Spider处理。
* Spider处理Response并返回爬取到的Item及(跟进的)新的Request给引擎。
* 引擎Scrapy Engine将(Spider返回的)爬取到的Item给Item Pipeline，将(Spider返回的)Request给调度器。
* (从第二步)重复直到调度器中没有更多地request，引擎关闭该网站。

Scrapy基于事件驱动网络框架 Twisted 编写。因此，Scrapy基于并发性考虑由非阻塞(即异步)的实现。

### 3.2 安装

Scrapy的安装看上去很简单, 只要```pip install scrapy```就行, 但是实际安装过程中, 3台服务器我只顺利安装成功了一台。 其他或多或少出现了问题, 主要就是缺少本地依赖库或者版本匹配问题。这部分遇到的问题通过google基本上都能解决, 所以就不详叙了。

### 3.3 使用

一般情况下我们创建一个Scrapy项目要分为以下几步:

* 创建一个Scrapy项目
* 定义提取的Item
* 编写爬取网站的 spider 并提取 Item
* 编写 Item Pipeline 来存储提取到的Item(即数据)

#### 创建项目

进入你需要创建的项目根目录```$WORKSPACE```, 输入以下命令:

```shell
scrapy startproject doubanGroup
```

这样就会在当前目录下创建doubanGroup的文件夹:

``` shell
(master) % tree doubanGroup                             ~/Workspace/git/Creeper
doubanGroup
├── doubanGroup
│   ├── __init__.py
│   ├── items.py
│   ├── pipelines.py
│   ├── settings.py
│   └── spiders
│       ├── DoubanDroupTestSpider.py
│       ├── DoubanGroupSpider.py
│       ├── __init__.py
└── scrapy.cfg

2 directories, 15 files
```

其中:

* scrapy.cfg: 项目的配置文件
* doubanGroup/: 该项目的python模块。之后您将在此加入代码。
* doubanGroup/items.py: 项目中的item文件.
* doubanGroup/pipelines.py: 项目中的pipelines文件, 主要是对经过spider后的item进行存储的等处理.
* doubanGroup/settings.py: 项目的设置文件.
* doubanGroup/spiders/: 放置spider代码的目录.

后续将结合爬取豆瓣小组的例子来介绍各个文件。

#### 操作

Scrapy的很多操作都是基于```scrapy```触发的, 比如:

* ```scrapy list```查看当前项目下有哪些spider(该命令必须在项目目录下。)
* ```scrapy crawl doubanGroup```来运行某项目中某个spider。
* ```scrapy shell https://www.douban.com/group/320130/```进入命令行模式对spider进行调试。

等等, 其他的命令可以通过```scrapy -help```进行查看。

这里要重点提一下调试神器```scrapy shell```, 通过```scrapy shell url``` scrapy会爬取url这个网址, 并储存在response对象里。```scrapy shell```本质是加载了scrapy的python shell。我们可以通过scrapy shell来对reponse的解析进行调试。

如果您安装了 IPython ，Scrapy终端将使用 IPython (替代标准Python终端)。 IPython 终端与其他相比更为强大，提供智能的自动补全，高亮输出，及其他特性。

```python
[s] Available Scrapy objects:
[s]   crawler    <scrapy.crawler.Crawler object at 0x1e16b50>
[s]   item       {}
[s]   request    <GET http://scrapy.org>
[s]   response   <200 http://scrapy.org>
[s]   sel        <Selector xpath=None data=u'<html>\n  <head>\n    <meta charset="utf-8'>
[s]   settings   <scrapy.settings.Settings object at 0x2bfd650>
[s]   spider     <Spider 'default' at 0x20c6f50>
[s] Useful shortcuts:
[s]   shelp()           Shell help (print this help)
[s]   fetch(req_or_url) Fetch request (or URL) and update local objects
[s]   view(response)    View response in a browser

In [7]: print(response.xpath('//h1/text()').extract()[0])

        每天一首古典诗词

In [8]: view(response)

......
```

我们完全可以把spider里面的代码拿到这里来进行调试。

## 4. 如何爬取豆瓣小组

## 5. 总结


## 参考文献

* [如何入门 Python 爬虫？](<https://www.zhihu.com/question/20899988>)
* [Scrapy文档](<http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/architecture.html>)
* [用Scrapy抓取豆瓣小组数据（一）](<http://my.oschina.net/chengye/blog/124157>)
