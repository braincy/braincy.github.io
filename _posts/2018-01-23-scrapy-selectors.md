---
layout: post
title: Scrapy提取网页数据
date: 2018-01-23 15:10:47
tags: [Scrapy]
categories: 爬虫
mathjax: true
---

* content
{:toc}

本文对Scrapy的选择器 *Selector* 进行详细介绍。




## 选择器Selectors介绍

在抓取网页时，需要执行的最常见的任务是从HTML源中提取数据。有几个库可以实现这一点：
> * [BeautifulSoup](https://www.crummy.com/software/BeautifulSoup/)是一个很方便的网页抓取库，它基于HTML代码的结构构建了一个Python对象。但有一个缺点：速度慢。
> * [lxml](http://lxml.de/)是一个基于[ElementTree](https://docs.python.org/2/library/xml.etree.elementtree.html)的pythonic API的XML／HTML解析库。*lxml不是Python标准库*

Scrapy有自身提取数据的机制，被称为选择器--Selectors。它可以通过XPath或者CSS表达式来定位HTML文档中的内容。

XPath是用于选择XML文档中结点的一种语言，它也可以被用于HTML文档中。CSS是一种将样式应用于HTML文档的语言，它定义选择器将这些样式与特定的HTML元素相关联。

Scrapy选择器是基于lxml库的，这也意味着它们的速度和解析精度非常相似。

## 如何使用选择器

### 构建选择器

Scrapy选择器是Selector通过传递文本或者TextResponse对象构造的类的实例。它会根据输入类型自动选择最佳的解析规则（XML／HTML）。

```python
from scrapy.selector import Selector
from scrapy.http import HtmlResponse
```

从文本构建：

```python
body = '<html><body><span>good</span></body></html>'
Selector(text=body).xpath('//span/text()').extract()
```
> *[u'good']*

从response构建：

```python
response = HtmlResponse(url='http://example.com', body=body)
Selector(response=response).xpath('//span/text()').extract()
```
> *[u'good']*

为了方便，response对象在selector属性上有一个选择器，我们可以用下面这种简便的写法：

```python
response.selector.xpath('//span/text()').extract()
```
> *[u'good']*

### 使用选择器

我们接下来将使用Scrapy shell和一个[example页面](https://doc.scrapy.org/en/latest/_static/selectors-sample1.html)

下面是该页面的源代码：
```html
<html>
 <head>
  <base href='http://example.com/' />
  <title>Example website</title>
 </head>
 <body>
  <div id='images'>
   <a href='image1.html'>Name: My image 1 <br /><img src='image1_thumb.jpg' /></a>
   <a href='image2.html'>Name: My image 2 <br /><img src='image2_thumb.jpg' /></a>
   <a href='image3.html'>Name: My image 3 <br /><img src='image3_thumb.jpg' /></a>
   <a href='image4.html'>Name: My image 4 <br /><img src='image4_thumb.jpg' /></a>
   <a href='image5.html'>Name: My image 5 <br /><img src='image5_thumb.jpg' /></a>
  </div>
 </body>
</html>
```

首先，我们打开shell：
```shell
scrapy shell https://doc.scrapy.org/en/latest/_static/selectors-sample1.html
```

接着，在shell加载后，我们就得到了可用的shell变量response，选择器在selector属性中。

由于我们正在处理HTML，选择器将自动使用HTML解析器。

现在我们来构建一个选择网页中title标签内的文本的XPath和CSS语句：

```shell
response.xpath('//title/text()')
```
> *[< Selector xpath='//title/text()' data=u'Example website'>]*

```shell
response.css('title::text')
```
> *[< Selector xpath=u'descendant-or-self::title/text()' data=u'Example website'>]*

我们可以看到.xpath()和.css()方法返回的事一个SelectorList对象，它是一些新的selectors的列表。这个API可以帮助我们快速提取嵌套的数据。

要提取实际的文本数据，必须调用.extract()方法，如下所示：
```shell
response.xpath('//title/text()').extract()
```
> *[u'Example website']*

如果未找到元素，返回*None*：
```shell
response.xpath('//div[@id="not-exists"]/text()').extract_first() is None
```
> *True*

我们还可以设置失败返回值
```shell
response.xpath('//div[@id="not-exists"]/text()').extract_first(default='not-found')
```
> *'not-found'*

参考：[Scrapy官方文档](https://doc.scrapy.org/en/latest/topics/selectors.html?highlight=xpath#scrapy.selector.SelectorList)
