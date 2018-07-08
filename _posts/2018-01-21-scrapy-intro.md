---
layout: post
title: Scrapy框架介绍
date: 2018-01-21 12:21:56
tags: [Scrapy]
categories: 爬虫
mathjax: true
---

* content
{:toc}

本文对Scrapy的架构进行简要介绍并实现了一个最简单的爬虫。




Scrapy是一个为了爬取网站数据, 提取结构性数据而编写的应用框架。 可以应用在包括数据挖掘，信息处理或存储历史数据等一系列的程序中。

其最初是为了 页面抓取 ( 更确切来说, 网络抓取 ) 所设计的，也可以应用在获取API所返回的数据或者通用的网络爬虫。你可以在[这里](http://scrapy.readthedocs.io/en/latest/)看到Scrapy的更多介绍。

## Scrapy流程图

![Scrapy执行流程](/img/scrapy_2.png)

上图包括了Scrapy的主要组件及系统的数据处理流程（绿色箭头所示）。
Scrapy数据流是由执行的核心引擎(Engine)控制，流程是这样的：

> 1、爬虫引擎获得初始请求开始抓取。
> 2、爬虫引擎开始请求调度程序，并准备对下一次的请求进行抓取。
> 3、爬虫调度器返回下一个请求给爬虫引擎。
> 4、引擎将请求发送到下载器，通过下载中间件下载网络数据。
> 5、一旦下载器完成页面下载，将下载结果返回给爬虫引擎。
> 6、引擎将下载器的响应通过中间件返回给爬虫进行处理。
> 7、爬虫处理响应，并通过中间件返回处理后的items，以及新的请求给引擎。
> 8、引擎发送处理后的items到项目管道，然后把处理结果返回给调度器，调度器计划处理下一个请求抓取。
> 9、重复该过程（继续步骤1），直到爬取完所有的url请求。

现在我们安装好了scrapy，如有疑问可参考[安装指南](http://scrapy.readthedocs.io/en/latest/intro/install.html)。

### 接下来我们要完成以下任务：

> * 创建一个Scrapy项目
> * 定义提取的Item
> * 编写爬取网站的 spider 并提取 Item

## 创建项目
在开始爬取之前，必须创建一个新的Scrapy项目。进入打算存储代码的目录中，运行下列命令:
>scrapy startproject tutorial

该命令将会创建包含下列内容的 `tutorial` 目录:
![tutorial目录](/img/scrapy_1.png)

### 这些文件分别是:
* `scrapy.cfg` : 项目的配置文件。
* `tutorial/` : 该项目的python模块。之后将在此加入代码。
* `tutorial/items.py` : 项目中的item文件。
* `tutorial/pipelines.py` : 项目中的pipelines文件。
* `tutorial/settings.py` : 项目的设置文件。
* `tutorial/spiders/` : 放置spider代码的目录。

## 定义Item
Item 是保存爬取到的数据的容器；其使用方法和python字典类似， 并且提供了额外保护机制来避免拼写错误导致的未定义字段错误。
类似在ORM中做的一样，我们可以通过创建一个 scrapy.Item 类， 并且定义类型为 scrapy.Field 的类属性来定义一个Item。
首先根据需要从dmoz.org获取到的数据对item进行建模。 我们需要从dmoz中获取名字，url，以及网站的描述。 对此，在item中定义相应的字段。编辑 tutorial 目录中的 items.py 文件:

```python
import scrapy
class DmozItem(scrapy.Item):
    title = scrapy.Field()
    link  = scrapy.Field()
    desc  = scrapy.Field()
```

## 编写第一个爬虫(Spider)
Spider是用户编写用于从单个网站(或者一些网站)爬取数据的类。
其包含了一个用于下载的初始URL，如何跟进网页中的链接以及如何分析页面中的内容， 提取生成 item 的方法。
为了创建一个Spider，必须继承 scrapy.Spider 类， 且定义以下三个属性:

> `name` : 用于区别Spider。 该名字必须是唯一的，您不可以为不同的Spider设定相同的名字。
> `start_urls` : 包含了Spider在启动时进行爬取的url列表。 因此，第一个被获取到的页面将是其中之一。 后续的URL则从初始的URL获取到的数据中提取。
> `parse()` : spider的一个方法。被调用时，每个初始URL完成下载后生成的Response 对象将会作为唯一的参数传递给该函数。该方法负责 `解析` 返回的数据(response data)，`提取` 数据(生成item)以及 `生成` 需要进一步处理的URL的 Request 对象。

以下为我们的第一个Spider代码，保存在 tutorial/spiders 目录下的 dmoz_spider.py 文件中:
```python
import scrapy
class DmozSpider(scrapy.Spider):
    name = "dmoz"
    allowed_domains = ["dmoz.org"]
    start_urls = [
        "http://www.dmoz.org/Computers/Programming/Languages/Python/Books/",
        "http://www.dmoz.org/Computers/Programming/Languages/Python/Resources/"
    ]
    def parse(self, response):
        filename = response.url.split("/")[-2]
        with open(filename, 'wb') as f:
            f.write(response.body)
```

### 爬取
进入项目的根目录，执行下列命令启动spider:
> scrapy crawl dmoz

`crawl dmoz` 启动用于爬取 `dmoz.org` 的spider，您将得到类似的输出:

![运行结果](/img/scrapy_3.png)

查看包含 `[dmoz]` 的输出，可以看到输出的log中包含定义在 `start_urls` 的初始URL，并且与spider中是一一对应的。

## 提取Item
### Selectors选择器简介
从网页中提取数据有很多方法。Scrapy使用了一种基于 `XPath` 和 `CSS` 表达式机制: `Scrapy Selectors` 。 关于selector和其他提取机制的信息请参考 [Selector文档](http://scrapy.readthedocs.io/en/latest/topics/selectors.html) 。

下面在我们的spider中加入下面代码：
```python
import scrapy
class DmozSpider(scrapy.Spider):
    name = "dmoz"
    allowed_domains = ["dmoz.org"]
    start_urls = [
        "http://www.dmoz.org/Computers/Programming/Languages/Python/Books/",
        "http://www.dmoz.org/Computers/Programming/Languages/Python/Resources/"
    ]
    def parse(self, response):
        for sel in response.xpath('//ul/li'):
            title = sel.xpath('a/text()').extract()
            link = sel.xpath('a/@href').extract()
            desc = sel.xpath('text()').extract()
            print title, link, desc
```
再次运行爬虫：

> scrapy crawl dmoz

此时我们可以看到网站信息被成功输出。

### 使用item
`Item` 对象是自定义的python字典。我们可以使用标准的字典语法来获取到其每个字段的值(字段即是我们之前用Field赋值的属性)。一般来说，Spider将会将爬取到的数据以 Item对象返回。所以为了将爬取的数据返回，我们最终的代码将是:
```python
import scrapy
from tutorial.items import DmozItem
class DmozSpider(scrapy.Spider):
    name = "dmoz"
    allowed_domains = ["dmoz.org"]
    start_urls = [
        "http://www.dmoz.org/Computers/Programming/Languages/Python/Books/",
        "http://www.dmoz.org/Computers/Programming/Languages/Python/Resources/"
    ]
    def parse(self, response):
        for sel in response.xpath('//ul/li'):
            item = DmozItem()
            item['title'] = sel.xpath('a/text()').extract()
            item['link'] = sel.xpath('a/@href').extract()
            item['desc'] = sel.xpath('text()').extract()
            yield item
```
现在对 dmoz.org 进行爬取将会产生 `DmozItem` 对象:
![DmozItem](/img/scrapy_4.png)

### 保存爬取到的数据

最简单存储爬取的数据的方式是使用 [Feed exports](http://scrapy.readthedocs.io/en/latest/topics/feed-exports.html):
> scrapy crawl dmoz -o items.json

该命令将采用 `JSON` 格式对爬取的数据进行序列化，生成 `items.json` 文件。(在生成json文件后,这里推荐将文件中的内容复制，使用Chrome中的 `Json Handle` 进行查看)

在类似本篇教程里这样小规模的项目中，这种存储方式已经足够。

如果需要对爬取到的item做更多更为复杂的操作，您可以编写 `Item Pipeline` 。 类似于我们在创建项目时对Item做的，用于您编写自己的 `tutorial/pipelines.py` 也被创建。不过如果仅仅想要保存item，那么不需要实现任何的pipeline。
