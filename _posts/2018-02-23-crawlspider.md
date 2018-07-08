---
layout: post
title: CrawlSpider源码分析
date: 2018-02-23 15:10:26
tags: [Scrapy, 源码分析]
categories: 爬虫
mathjax: true
---

* content
{:toc}

CrawlSpider是爬取那些具有一定规则网站的常用的爬虫，它基于Spider并有一些独特属性，本文对其源码进行简单分析。




## CrawlSpider介绍

CrawlSpider基于Spider，但是可以说是为全站爬取而生。
CrawlSpider是爬取那些具有一定规则网站的常用的爬虫，它基于Spider并有一些独特属性。

* `rules`： 是Rule对象的集合，用于匹配目标网站并排除干扰
* `parse_start_url`：用于爬取起始响应，必须要返回Item，Request中的一个。

因为rules是Rule对象的集合，所以这里也要介绍一下Rule。它有几个参数：`link_extractor`、`callback`、`cb_kwargs`、`follow`、`process_links`、`process_request`，其中 `link_extractor` 既可以自己定义，也可以使用已有 `LinkExtractor` 类，主要参数为：

* `allow`：满足括号中正则表达式的值会被提取，如果为空，则全部匹配。
* `deny`：与这个正则表达式(或正则表达式列表)不匹配的URL一定不提取。
* `allow_domains`：会被提取的链接的domains。
* `deny_domains`：一定不会被提取链接的domains。
* `restrict_xpaths`：使用xpath表达式，和allow共同作用过滤链接。还有一个类似的 `restrict_css`。

## CrawlSpider及Spider的源码

```python
"""
This modules implements the CrawlSpider which is the recommended spider to use
for scraping typical web sites that requires crawling pages.

See documentation in docs/topics/spiders.rst
"""

import copy
import six

from scrapy.http import Request, HtmlResponse
from scrapy.utils.spider import iterate_spider_output
from scrapy.spiders import Spider


def identity(x):
    return x


class Rule(object):

    def __init__(self, link_extractor, callback=None, cb_kwargs=None, follow=None, process_links=None, process_request=identity):
        self.link_extractor = link_extractor
        self.callback = callback
        self.cb_kwargs = cb_kwargs or {}
        self.process_links = process_links
        self.process_request = process_request
        if follow is None:
            self.follow = False if callback else True
        else:
            self.follow = follow


class CrawlSpider(Spider):

    rules = ()

    def __init__(self, *a, **kw):
        super(CrawlSpider, self).__init__(*a, **kw)
        self._compile_rules()

    def parse(self, response):
        return self._parse_response(response, self.parse_start_url, cb_kwargs={}, follow=True)

    def parse_start_url(self, response):
        return []

    def process_results(self, response, results):
        return results

    def _build_request(self, rule, link):
        r = Request(url=link.url, callback=self._response_downloaded)
        r.meta.update(rule=rule, link_text=link.text)
        return r

    def _requests_to_follow(self, response):
        if not isinstance(response, HtmlResponse):
            return
        seen = set()
        for n, rule in enumerate(self._rules):
            links = [lnk for lnk in rule.link_extractor.extract_links(response)
                     if lnk not in seen]
            if links and rule.process_links:
                links = rule.process_links(links)
            for link in links:
                seen.add(link)
                r = self._build_request(n, link)
                yield rule.process_request(r)

    def _response_downloaded(self, response):
        rule = self._rules[response.meta['rule']]
        return self._parse_response(response, rule.callback, rule.cb_kwargs, rule.follow)

    def _parse_response(self, response, callback, cb_kwargs, follow=True):
        if callback:
            cb_res = callback(response, **cb_kwargs) or ()
            cb_res = self.process_results(response, cb_res)
            for requests_or_item in iterate_spider_output(cb_res):
                yield requests_or_item

        if follow and self._follow_links:
            for request_or_item in self._requests_to_follow(response):
                yield request_or_item

    def _compile_rules(self):
        def get_method(method):
            if callable(method):
                return method
            elif isinstance(method, six.string_types):
                return getattr(self, method, None)

        self._rules = [copy.copy(r) for r in self.rules]
        for rule in self._rules:
            rule.callback = get_method(rule.callback)
            rule.process_links = get_method(rule.process_links)
            rule.process_request = get_method(rule.process_request)

    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        spider = super(CrawlSpider, cls).from_crawler(crawler, *args, **kwargs)
        spider._follow_links = crawler.settings.getbool(
            'CRAWLSPIDER_FOLLOW_LINKS', True)
        return spider

    def set_crawler(self, crawler):
        super(CrawlSpider, self).set_crawler(crawler)
        self._follow_links = crawler.settings.getbool('CRAWLSPIDER_FOLLOW_LINKS', True)

```

```python
"""
Base class for Scrapy spiders

See documentation in docs/topics/spiders.rst
"""
import logging
import warnings

from scrapy import signals
from scrapy.http import Request
from scrapy.utils.trackref import object_ref
from scrapy.utils.url import url_is_from_spider
from scrapy.utils.deprecate import create_deprecated_class
from scrapy.exceptions import ScrapyDeprecationWarning
from scrapy.utils.deprecate import method_is_overridden


class Spider(object_ref):
    """Base class for scrapy spiders. All spiders must inherit from this
    class.
    """

    name = None
    custom_settings = None

    def __init__(self, name=None, **kwargs):
        if name is not None:
            self.name = name
        elif not getattr(self, 'name', None):
            raise ValueError("%s must have a name" % type(self).__name__)
        self.__dict__.update(kwargs)
        if not hasattr(self, 'start_urls'):
            self.start_urls = []

    @property
    def logger(self):
        logger = logging.getLogger(self.name)
        return logging.LoggerAdapter(logger, {'spider': self})

    def log(self, message, level=logging.DEBUG, **kw):
        """Log the given message at the given log level

        This helper wraps a log call to the logger within the spider, but you
        can use it directly (e.g. Spider.logger.info('msg')) or use any other
        Python logger too.
        """
        self.logger.log(level, message, **kw)

    @classmethod
    def from_crawler(cls, crawler, *args, **kwargs):
        spider = cls(*args, **kwargs)
        spider._set_crawler(crawler)
        return spider

    def set_crawler(self, crawler):
        warnings.warn("set_crawler is deprecated, instantiate and bound the "
                      "spider to this crawler with from_crawler method "
                      "instead.",
                      category=ScrapyDeprecationWarning, stacklevel=2)
        assert not hasattr(self, 'crawler'), "Spider already bounded to a " \
                                             "crawler"
        self._set_crawler(crawler)

    def _set_crawler(self, crawler):
        self.crawler = crawler
        self.settings = crawler.settings
        crawler.signals.connect(self.close, signals.spider_closed)

    def start_requests(self):
        cls = self.__class__
        if method_is_overridden(cls, Spider, 'make_requests_from_url'):
            warnings.warn(
                "Spider.make_requests_from_url method is deprecated; it "
                "won't be called in future Scrapy releases. Please "
                "override Spider.start_requests method instead (see %s.%s)." % (
                    cls.__module__, cls.__name__
                ),
            )
            for url in self.start_urls:
                yield self.make_requests_from_url(url)
        else:
            for url in self.start_urls:
                yield Request(url, dont_filter=True)

    def make_requests_from_url(self, url):
        """ This method is deprecated. """
        return Request(url, dont_filter=True)

    def parse(self, response):
        raise NotImplementedError('{}.parse callback is not defined'.format(self.__class__.__name__))

    @classmethod
    def update_settings(cls, settings):
        settings.setdict(cls.custom_settings or {}, priority='spider')

    @classmethod
    def handles_request(cls, request):
        return url_is_from_spider(request.url, cls)

    @staticmethod
    def close(spider, reason):
        closed = getattr(spider, 'closed', None)
        if callable(closed):
            return closed(reason)

    def __str__(self):
        return "<%s %r at 0x%0x>" % (type(self).__name__, self.name, id(self))

    __repr__ = __str__


BaseSpider = create_deprecated_class('BaseSpider', Spider)


class ObsoleteClass(object):
    def __init__(self, message):
        self.message = message

    def __getattr__(self, name):
        raise AttributeError(self.message)

spiders = ObsoleteClass(
    '"from scrapy.spider import spiders" no longer works - use '
    '"from scrapy.spiderloader import SpiderLoader" and instantiate '
    'it with your project settings"'
)

# Top-level imports
from scrapy.spiders.crawl import CrawlSpider, Rule
from scrapy.spiders.feed import XMLFeedSpider, CSVFeedSpider
from scrapy.spiders.sitemap import SitemapSpider

```

## CrawlSpider源码分析
### CrawlSpider如何工作的？
因为CrawlSpider继承了Spider，所以具有Spider的所有函数。

首先由start_requests对start_urls中的每一个url发起请求(make_requests_from_url)$\{Spider源码：69～87行\}$，这个请求会被parse接收。

在Spider里面的parse需要我们定义，但CrawlSpider要通过parse去解析响应(self._parse_response)$\{CrawlSpider源码：42～43行\}$，_parse_response根据有无callback，follow和self.follow_links执行不同的操作$\{CrawlSpider源码：74～83行\}$。
其中_requests_to_follow又会获取link_extractor(这个是我们传入的LinkExtractor)解析页面得到的link(link_extractor.extract_links(response))$\{CrawlSpider源码:61～62行\}$，对url进行加工(process_links,需要自定义)$\{CrawlSpider源码:63～64行\}$，对符合的link发起Request$\{CrawlSpider源码:51～54行,67行\}$。使用process_request(需要自定义)处理响应$\{CrawlSpider源码:68行\}$。

### CrawlSpider如何获取rules？
CrawlSpider类会在init方法中调用_compile_rules方法，然后在其中浅拷贝rules中的各个Rule获取要用于回调(callback)，要进行处理的链接(process_links)和要进行的处理请求(process_request)$\{CrawlSpider源码:85～96行\}$。
