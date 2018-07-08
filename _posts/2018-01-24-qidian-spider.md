---
layout: post
title: Scrapy爬取起点小说
date: 2018-01-24 20:09:50
tags: [Scrapy]
categories: 爬虫
mathjax: true
---

* content
{:toc}

本文写了一个简单实例使用Scrapy完成对起点小说的爬取。




## 目标

本次爬虫的目标是爬取起点中文网免费分区完结的小说。

抓取的字段包括小说的标题、作者、分类（大类／小类）、字数、章节数、简介、摘要、点击数、推荐数等等。

## 实现过程

### 寻找抓取入口

```python
    start_urls = [
        'https://www.qidian.com/free/all?size=1&action=1&orderId=&vip=hidden&month=3&style=1&pageSize=20&siteid=1&pubflag=0&hiddenField=1&page='+str(page) for page in range(1, {N}) # N为自定义要爬取的页码范围
    ]
```

### 定义抓取字段

> *item.py*

```python
class NovelItem(scrapy.Item):
    _id = scrapy.Field()
    url = scrapy.Field()
    intro = scrapy.Field()
    score = scrapy.Field()
    title = scrapy.Field()
    cover = scrapy.Field()
    status = scrapy.Field()
    author = scrapy.Field()
    book_id = scrapy.Field()
    category = scrapy.Field()
    word_num = scrapy.Field()
    abstract = scrapy.Field()
    click_num = scrapy.Field()
    created_at = scrapy.Field()
    chapter_num = scrapy.Field()
    sub_category = scrapy.Field()
    recommend_num = scrapy.Field()
    chapter_content = scrapy.Field()
    pass
```

### 确定抓取规则
> *qidian.py*

#### 抓取列表页的信息

```python
    def parse(self, response):
        sel = Selector(response)
        element = sel.xpath('//ul[@class="all-img-list cf"]/li')
        for el in element:
            item = NovelItem()
            item['title'] = el.xpath('.//div[@class="book-mid-info"]//h4/a/text()').extract_first()
            item['author'] = el.xpath('.//p[@class="author"]//a[@class="name"]/text()').extract_first()
            item['category'] = el.xpath('.//p[@class="author"]//a[@data-eid="qd_B60"]/text()').extract_first()
            item['sub_category'] = el.xpath('.//p[@class="author"]//a[@class="go-sub-type"]/text()').extract_first()
            item['status'] = 0 if el.xpath('.//p[@class="author"]//span/text()').extract_first() == '完本' else 1
            item['url'] = 'https:' + el.xpath('.//div[@class="book-mid-info"]//h4/a/@href').extract_first()
            item['abstract'] = el.xpath('.//p[@class="intro"]/text()').extract_first().strip()
            item['word_num'] = el.xpath('.//p[@class="update"]/span/text()').extract_first()
            item['cover'] = 'https:' + el.xpath('.//div[@class="book-img-box"]/a/img/@src').extract_first()
            yield Request(url=item['url'], meta={'item':item}, callback=self.parse_details)
```

#### 抓取详情页的信息

##### 注意点

* 小说的评分需要通过ajax接口获取（不存在则为-1）

```python
    def parse_details(self, response):
        item = response.meta.get('item', NovelItem())
        sel = Selector(response)
        prefix_url = 'https://book.qidian.com/ajax/comment/index?_csrfToken=noZtdawi6Zu8sYFtR2m2o3ujn4lyQLrauItBqnzG&bookId='
        item['book_id'] = item['url'][item['url'].rfind('/')+1:]
        item['score'] = json.loads(requests.get(prefix_url + item['book_id']).content)['data']['rate']
        item['intro'] = sel.xpath('//div[@class="book-info "]//p[@class="intro"]/text()').extract_first()
        click_num = sel.xpath('//div[@class="book-info "]/p[3]/em[2]/text()').extract_first()
        item['click_num'] = click_num + sel.xpath('//div[@class="book-info "]/p[3]/cite[2]/text()[1]').extract_first().replace('总点击','')
        item['chapter_num'] = int(sel.xpath('//li[@class="j_catalog_block"]//span[@id="J-catalogCount"]/text()').extract_first()[1:-2])
        recommend_num = sel.xpath('//div[@class="book-info "]/p[3]/em[3]/text()').extract_first()
        item['recommend_num'] = recommend_num + sel.xpath('//div[@class="book-info "]/p[3]/cite[3]/text()[1]').extract_first().replace('总推荐','')
        item['created_at'] = time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        item['chapter_content'] = {}
        chapter_el = sel.xpath('//div[@class="volume-wrap"]//div[@class="volume"]/ul/li')
        for i, el in enumerate(chapter_el):
            chapter_url = 'https:' + el.xpath('.//a/@href').extract_first()
            chapter_title = el.xpath('.//a/text()').extract_first()
            yield Request(url=chapter_url, meta={'item':item, 'chapter_title':chapter_title}, callback=self.parse_chapters)
```

#### 抓取当前小说所有章节
##### 注意点
* chapter_title中的“.”需要被替换掉，否则插入数据库会报错
* 这里使用OSS上传每个章节的文本

```python
    def parse_chapters(self, response):
        item = response.meta.get('item', NovelItem())
        chapter_title = response.meta.get('chapter_title').replace('.','')
        sel = Selector(response)
        content = "\n".join(sel.xpath('//div[@class="read-content j_readContent"]').xpath('string(.)').extract()).encode('utf-8').strip()
        oss_value = self.oss.uploadPage(content)
        print item['title'] + ': ' + chapter_title + '  ' + oss_value
        item['chapter_content'][chapter_title] = oss_value
        if len(item['chapter_content']) >= item['chapter_num']:
            print 'finished ' + item['title'] + ':  ' + item['author']
            yield item
```

## 效果截图

### 运行效果
![运行截图](http://ouy59qaqh.bkt.clouddn.com/%E8%BF%90%E8%A1%8C%E6%88%AA%E5%9B%BE.jpg)

### 存储结果
![存储结果](http://ouy59qaqh.bkt.clouddn.com/novel.jpg)

项目源码在[这里](https://github.com/braincy/Spider_Practice/tree/master/novel)
