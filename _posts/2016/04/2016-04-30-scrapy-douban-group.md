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

豆瓣小组的数据有啥用？爬取的初衷是为了获取各个小组的互相联系, 从而进行复杂网络分析。所以我主要爬取了豆瓣小组的基本属性以及相关的小组。接下来我们通过各个模块来分析如何爬取豆瓣小组的信息。

### 4.1 items

爬取的主要目标就是从非结构性的数据源提取结构性数据，例如网页。 Scrapy提供 Item 类来满足这样的需求。
Item 对象是种简单的容器，保存了爬取到得数据。 其提供了 类似于词典(dictionary-like) 的API以及用于声明可用字段的简单语法。

在item.py中定义了要抓取的数据结构. 在本例子中我们定义了一个DoubangroupItem对象, 属性包括groupName,groupURL,totalNumber, RelativeGroups。定义完DoubangroupItem后，就可以在spider的代码里返回DoubangroupItem的实例，Scrapy会自动序列化并导出到JSON/XML等。

```python
import scrapy
from scrapy import Field

class DoubangroupItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    groupName = Field()   //小组名字
    groupURL = Field()    //小组链接
    totalNumber = Field() //小组人数
    RelativeGroups = Field() // 小组的相关小组
```

### 4.2 spiders

Spider类定义了如何爬取某个(或某些)网站。包括了爬取的动作(例如:是否跟进链接)以及如何从网页的内容中提取结构化数据(爬取item)。 换句话说，Spider就是您定义爬取的动作及分析某个网页(或者是有些网页)的地方。所以Sipders是我们实现代码最多的地方。

Sipder的大量采用的回调的形式来处理逻辑:

* 以初始的URL初始化Request，并设置回调函数。 当该request下载完毕并返回时，将生成response，并作为参数传给该回调函数。
* spider中初始的request是通过调用 start_requests() 来获取的。 start_requests() 读取 start_urls 中的URL， 并以 parse 为回调函数生成 Request 。
* 在回调函数内分析返回的(网页)内容，返回 Item 对象或者 Request 或者一个包括二者的可迭代容器。 返回的Request对象之后会经过Scrapy处理，下载相应的内容，并调用设置的callback函数(函数可相同)。
* 在回调函数内，您可以使用 选择器(Selectors) (您也可以使用BeautifulSoup, lxml 或者您想用的任何解析器) 来分析网页内容，并根据分析的数据生成item。
* 最后，由spider返回的item将被存到数据库(由某些 Item Pipeline 处理)或使用 Feed exports 存入到文件中。

Scrapy提供了很多的Spider, 本文使用到的是CrawlSpider和Spider, 我们通过具体的例子来查看:

#### Spider

Spider是最简单的spider。每个其他的spider必须继承自该类(包括Scrapy自带的其他spider以及您自己编写的spider)。 Spider并没有提供什么特殊的功能。 其仅仅请求给定的 start_urls/start_requests ，并根据返回的结果(resulting responses)调用spider的 parse 方法。

```python
class DoubanGroupTestSpider(Spider):
    name = 'doubanGroupTest'
    allowed_domains = ["douban.com"]
    start_urls = ['https://www.douban.com/group/142121/']

    def __get_id_from_group_url(self, url):
        m =  re.search("^https://www.douban.com/group/([^/]+)/$", url)
        if(m):
            return m.group(1)
        else:
            return 0

    def parse(self, response):
        self.log("Fetch douban homepage page: %s" % response.url)
        sel = Selector(response)
        item = DoubangroupItem()
        item['groupName'] = sel.xpath('//h1/text()').re("^\s+(.*)\s+$")[0]
        #get group id
        item['groupURL'] = response.url
        groupid = self.__get_id_from_group_url(response.url)

        #get group members number
        members_url = "https://www.douban.com/group/%s/members" % groupid
        members_text = sel.xpath('//a[contains(@href, "%s")]/text()' % members_url).re("\((\d+)\)")
        item['totalNumber'] = members_text[0]

        #get relative groups
        item['RelativeGroups'] = []
        groups = sel.xpath('//div[contains(@class, "group-list-item")]')
        for group in groups:
            url = group.xpath('div[contains(@class, "title")]/a/@href').extract()[0]
            item['RelativeGroups'].append(url)

        return item
```

DoubanGroupTestSpider这个类很简单, 它只是对```https://www.douban.com/group/142121/```这个url进行抓取并调用parse方法进行解析并返回item。这也是Sipder能实现的功能。所以这里介绍其中的几个关键属性和方法。

name, 定义spider名字的字符串(string)。spider的名字定义了Scrapy如何定位(并初始化)spider，所以其必须是唯一的。 不过您可以生成多个相同的spider实例(instance)，这没有任何限制。 name是spider最重要的属性，而且是必须的。当在项目根目录使用```scrapy list```命令时, 返回的是就是这个name

allowed_domains, 可选。包含了spider允许爬取的域名(domain)列表(list)。 当 OffsiteMiddleware 启用时， 域名不在列表中的URL不会被跟进。

start_urls, URL列表。当没有制定特定的URL时，spider将从该列表中开始进行爬取。 因此，第一个被获取到的页面的URL将是该列表之一。 后续的URL将会从获取到的数据中提取。

parse, 当response没有指定回调函数时，该方法是Scrapy处理下载的response的默认方法。parse 负责处理response并返回处理的数据以及(/或)跟进的URL。

爬取的信息如下

```shell
(rcf-env)[bmw@data4 doubanGroup]$ scrapy crawl doubanGroupTest
2016-05-10 10:00:13 [scrapy] DEBUG: Scraped from <200 https://www.douban.com/group/142121/>
{'RelativeGroups': [u'https://www.douban.com/group/145305/',
                    u'https://www.douban.com/group/yueliang/',
                    u'https://www.douban.com/group/217247/',
                    u'https://www.douban.com/group/289901/',
                    u'https://www.douban.com/group/yueliang/',
                    u'https://www.douban.com/group/217247/',
                    u'https://www.douban.com/group/174832/',
                    u'https://www.douban.com/group/yueliangxiaozu/',
                    u'https://www.douban.com/group/410326/',
                    u'https://www.douban.com/group/291991/',
                    u'https://www.douban.com/group/midi/',
                    u'https://www.douban.com/group/145305/'],
 'groupName': u'\u6211\u4eec\u4ee3\u8868\u6708\u4eae\u6d88\u706d\u5c45\u5fc3\u4e0d\u826f\u7684\u4e50\u624b',
 'groupURL': 'https://www.douban.com/group/142121/',
 'totalNumber': u'15517'}
```

#### CrawlSpider

相比于Spider只能对单个网页进行爬取(不对网页内的link继续进行爬取), CrawlSpider支持从该网页为起点, 解析获取所需要的links, 并放入Scheduler中等待后续对links的爬取。因此使用CrawlSpider可以实现递归爬取。

```python
class DoubanGroupSpider(CrawlSpider):
    name = 'doubanGroup'
    allowed_domains = ["douban.com"]
    start_urls = [
        "http://www.douban.com/group/explore?tag=%E8%B4%AD%E7%89%A9",
        "http://www.douban.com/group/explore?tag=%E7%94%9F%E6%B4%BB",
        "http://www.douban.com/group/explore?tag=%E7%A4%BE%E4%BC%9A",
        "http://www.douban.com/group/explore?tag=%E8%89%BA%E6%9C%AF",
        "http://www.douban.com/group/explore?tag=%E5%AD%A6%E6%9C%AF",
        "http://www.douban.com/group/explore?tag=%E6%83%85%E6%84%9F",
        "http://www.douban.com/group/explore?tag=%E9%97%B2%E8%81%8A",
        "http://www.douban.com/group/explore?tag=%E5%85%B4%E8%B6%A3"

    ]

    rules = [
        ## 解析每一个group的主页, 爬取到这一层就不会再深入了
        Rule(LinkExtractor(allow=('/group/[^/]+/$', )), callback='parse_group_home_page'),

        ## 获取所有tag的路径, 并放入schedule, 后续要接着爬取各tag下的group url。
        Rule(LinkExtractor(allow=('/group/explore\?tag')), follow=True),

        ## 进入tag的页面后, 解析next, 获取下一页的group url
        Rule(LinkExtractor(allow=('/group/explore[?]start=.*?[&]tag=.*?$'),
                           restrict_xpaths=('//span[@class="next"]')),
                           callback='parse_next_page',
                           follow=True)
    ]

    def __get_id_from_group_url(self, url):
        m =  re.search("^https://www.douban.com/group/([^/]+)/$", url)
        if(m):
            return m.group(1)
        else:
            return 0

    def parse_next_page(self, response):
        self.log("Fetch next page: %s" % response.url)

    def parse_group_home_page(self, response):
        self.log("Fetch douban homepage page: %s" % response.url)
        sel = Selector(response)
        item = DoubangroupItem()
        item['groupName'] = sel.xpath('//h1/text()').re("^\s+(.*)\s+$")[0]
        #get group id
        item['groupURL'] = response.url
        groupid = self.__get_id_from_group_url(response.url)

        #get group members number
        members_url = "https://www.douban.com/group/%s/members" % groupid
        members_text = sel.xpath('//a[contains(@href, "%s")]/text()' % members_url).re("\((\d+)\)")
        item['totalNumber'] = members_text[0]

        #get relative groups
        item['RelativeGroups'] = []
        groups = sel.xpath('//div[contains(@class, "group-list-item")]')
        for group in groups:
            url = group.xpath('div[contains(@class, "title")]/a/@href').extract()[0]
            item['RelativeGroups'].append(url)

        return item
```

##### 在CrawlSpider中增加rules的属性, 它包含了多个爬取规则```class scrapy.contrib.spiders.Rule(link_extractor, callback=None, cb_kwargs=None, follow=None, process_links=None, process_request=None)```.

##### 其中link_extractor是一个 Link Extractor 对象。其定义了如何从爬取到的页面提取链接。

例如:

```LinkExtractor(allow=('/group/explore\?tag'))``` 只抓取复合```/group/explore\?tag```正则的link

```LinkExtractor(allow=('/group/explore[?]start=.*?[&]tag=.*?$'),restrict_xpaths=('//span[@class="next"]')))``` 只抓取符合```/group/explore[?]start=.*?[&]tag=.*?$```正则 且 约束在 ```//span[@class="next"]```该xpath路径下的连接。

##### follow 是一个布尔(boolean)值，指定了根据该规则从response提取的链接是否需要跟进(后续是否需要接着爬)。 如果 callback 为None， follow 默认设置为 True ，否则默认为 False 。

##### callback 是一个callable或string(该spider中同名的函数将会被调用)。 从link_extractor中每获取到链接时将会调用该函数。该回调函数接受一个response作为其第一个参数， 并返回一个包含 Item 以及(或) Request 对象(或者这两者的子类)的列表(list)。

例如：

```python
Rule(LinkExtractor(allow=('/group/explore[?]start=.*?[&]tag=.*?$'),
                   restrict_xpaths=('//span[@class="next"]')),
                   callback='parse_next_page',
                   follow=True)
```

当在```'//span[@class="next"]'```路径下找到符合```/group/explore[?]start=.*?[&]tag=.*?$```正则的连接时候, 就会回调函数

```python
def parse_next_page(self, response):
    self.log("Fetch next page: %s" % response.url)
```

对response进行处理。

还有很多Spider类型, 不过本文没用到, 所以就不展开了。 而如何解析豆瓣的html格式比较简单也不再深入展开。

### 4.3 pipelines

当Item在Spider中被收集之后，它将会被传递到Item Pipeline，一些组件会按照一定的顺序执行对Item的处理。

每个item pipeline组件(有时称之为“Item Pipeline”)是实现了简单方法的Python类。他们接收到Item并通过它执行一些行为，同时也决定此Item是否继续通过pipeline，或是被丢弃而不再进行处理。

以下是item pipeline的一些典型应用：

* 清理HTML数据
* 验证爬取的数据(检查item包含某些字段)
* 查重(并丢弃)
* 将爬取结果保存到数据库中

在这里我们使用pipelines来将爬取的item存储到mongo中。

```python
class MongoDBPipeline(object):

     def __init__( self ):
         host = settings[ 'MONGODB_SERVER' ]
         port = settings[ 'MONGODB_PORT' ]
         db = settings[ 'MONGODB_DB' ]
         username = settings[ 'MONGODB_USE' ]
         password = settings[ 'MONGODB_PASWD' ]
         table = settings[ 'MONGODB_COLLECTION' ]
         mongo_uri = 'mongodb://%s:%s@%s:%s/%s' % (username, password, host, port, db)
         conn = MongoClient(mongo_uri)
         self .collection = conn[db][table]

     def process_item( self , item, spider):
         valid = True
         for data in item:
           if not data:
             valid = False
             raise DropItem( "Missing {0}!" . format(data))
         if valid:
           self.collection.update({'groupURL': item['groupURL']}, dict(item), upsert=True)
           log.msg( "Question added to MongoDB database!" ,level = log.DEBUG, spider = spider)
         return item
```

### 4.4 settings

Settings包含了Scrapy的所有设置, 我们都可以通过该类来获取。设置可以在settings文件中进行。

```python
BOT_NAME = 'doubanGroup'

SPIDER_MODULES = ['doubanGroup.spiders']
NEWSPIDER_MODULE = 'doubanGroup.spiders'

## 两次爬取任务的时间间隔, 防止被反爬虫
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True

## 模拟浏览器请求, 防止被反爬虫
USER_AGENT = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_8_3) AppleWebKit/536.5 (KHTML, like Gecko) Chrome/19.0.1084.54 Safari/536.5'
COOKIES_ENABLED = True

ITEM_PIPELINES = ['doubanGroup.pipelines.MongoDBPipeline']
MONGODB_SERVER = 'lambo'
MONGODB_PORT = 27017
MONGODB_DB = 'Scrapy'
MONGODB_COLLECTION = 'DoubanGroup'
MONGODB_USE = 'scrapy'
MONGODB_PASWD = 'aaaaaa'
```

## 5. 总结

本文简单介绍了爬虫原理, 以及使用spider来爬取豆瓣小组。

```javascript
{
	"_id" : ObjectId("5721bae60af7dd1e5c14616d"),
	"groupName" : "MOON magazine",
	"groupURL" : "https://www.douban.com/group/MOONmag/",
	"totalNumber" : "2629",
	"RelativeGroups" : [
		"https://www.douban.com/group/24884/",
		"https://www.douban.com/group/jcremix/",
		"https://www.douban.com/group/sdzine/",
		"https://www.douban.com/group/24884/",
		"https://www.douban.com/group/Hi-low/",
		"https://www.douban.com/group/kinamagazine/",
		"https://www.douban.com/group/toomagazine/",
		"https://www.douban.com/group/soudian/",
		"https://www.douban.com/group/56145/",
		"https://www.douban.com/group/zhubian/",
		"https://www.douban.com/group/jcremix/"
	]
}
```

接下来我将对这些数据进行社交网络的分析, 这就是另外一篇文章的内容了。

## 参考文献

* [如何入门 Python 爬虫？](<https://www.zhihu.com/question/20899988>)
* [Scrapy文档](<http://scrapy-chs.readthedocs.io/zh_CN/0.24/topics/architecture.html>)
* [用Scrapy抓取豆瓣小组数据（一）](<http://my.oschina.net/chengye/blog/124157>)




本文完

* 原创文章，转载请注明： 转载自[Lamborryan](<http://www.lamborryan.com>)，作者：[Ruan Chengfeng](<http://www.lamborryan.com/about/>)
* 本文链接地址：[http://www.lamborryan.com/scrapy-douban-group](<http://www.lamborryan.com/scrapy-douban-group>)
* 本文基于[署名2.5中国大陆许可协议](<http://creativecommons.org/licenses/by/2.5/cn/>)发布，欢迎转载、演绎或用于商业目的，但是必须保留本文署名和文章链接。 如您有任何疑问或者授权方面的协商，请邮件联系我。
