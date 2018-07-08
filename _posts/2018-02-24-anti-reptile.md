---
layout: post
title: Scrapy突破反爬虫
date: 2018-02-24 20:15:19
tags: [Scrapy, 反爬虫]
categories: 爬虫
mathjax: true
---

* content
{:toc}

在我们编写爬虫程序的过程中，经常会遇到网站的反爬措施，本文对Scrapy中常用的突破反爬虫方法进行介绍。




## 随机更换User-Agent

### 什么是User-Agent？

User-Agent中文名为用户代理，简称UA，它是一个特殊字符串头，使得服务器能够识别客户使用的操作系统及版本、CPU类型、浏览器及版本、浏览器渲染引擎、浏览器语言、浏览器插件等。

### 如何更换User-Agent？

在settings.py中搜索 `DOWNLOADER_MIDDLEWARES`，默认是被注释的，代码如下：

```python
#DOWNLOADER_MIDDLEWARES = {
#    'ArticleSpider.middlewares.ArticlespiderDownloaderMiddleware': 543,
#}
```

其中 `Articlespider` 是我的项目名，现在我们把注释取消掉，接下来就需要来写我们自定义的 `Middleware` 了，这里可以参考Scrapy的[官方示例](https://doc.scrapy.org/en/latest/topics/downloader-middleware.html#writing-your-own-downloader-middleware)。

这里有一点需要注意，如果我们要添加自己的 `Middleware`，应该把scrapy自带的 `UserAgentMiddleware` 置为 `None`,如下所示：

```python
DOWNLOADER_MIDDLEWARES = {
    'ArticleSpider.middlewares.RandomUserAgentMiddleware': 543,
    'scrapy.downloadermiddlewares.useragent.UserAgentMiddleware': None,
}
```

这是因为它们后面的数字代表的是执行顺序，如果scrapy本身的 `UserAgentMiddleware` 后执行的话，那么请求的User-Agent仍会被更新为`Scrapy`，即我们的 `Middleware` 没有生效，因此我们最好禁止掉scrapy自带的 `UserAgentMiddleware`。

那么如何编写我们自己的 `UserAgentMiddleware`呢？我们需要在 `middlewares.py` 中添加下面代码：

```python
class RandomUserAgentMiddleware(object):
    # 随机更换User-Agent
    def __init__(self, crawler):
        super(RandomUserAgentMiddleware, self).__init__()
        self.ua = UserAgent()

    @classmethod
    def from_crawler(cls, crawler):
        return cls(crawler)

    def process_request(self, request, spider):
        request.headers.setdefault('User-Agent', self.ua.random)
```

这里用到了一个第三方库`fake-useragent`，github地址在[这里](https://github.com/hellysmile/fake-useragent)，它的功能是帮助我们维护一个User-Agent池，不需要我们人工在settings中配置。

到这里，我们通过 `DownloadMiddleware` 随机更换User-Agent已经可以成功运行了，不过，为了使User-Agent的类型是可配置的，我们可以在 `settings` 中添加一个字段 `RANDOM_UA_TYPE`，它表示我们希望选择的User-Agent类型，包括random、ie、chrome、firefox等等，具体用法可参考上述的github地址。

此时我们就需要修改一下刚刚写好的 `RandomUserAgentMiddleware` 类：

```python
class RandomUserAgentMiddleware(object):
    # 随机更换User-Agent
    def __init__(self, crawler):
        super(RandomUserAgentMiddleware, self).__init__()
        self.ua = UserAgent()
        self.ua_type = crawler.settings.get('RANDOM_UA_TYPE')

    @classmethod
    def from_crawler(cls, crawler):
        return cls(crawler)

    def process_request(self, request, spider):
        def get_ua():
            return getattr(self.ua, self.ua_type)
        request.headers.setdefault('User-Agent', get_ua())
```

这样我们就完成了通过 `DownloadMiddleware` 随机更换User-Agent。

## 实现IP代理池

### 获取免费IP

这里我们选择[西刺免费代理](http://www.xicidaili.com/nn)，首先我们把网页上的ip全部抓取下来并存入数据库，代码如下：

```python
def crawl_ips():
    #爬取西刺的免费ip代理
    headers = {"User-Agent":"Mozilla/5.0 (Windows NT 6.1; WOW64; rv:52.0) Gecko/20100101 Firefox/52.0"}
    for i in range(2731):
        re = requests.get("http://www.xicidaili.com/nn/{0}".format(i), headers=headers)

        selector = Selector(text=re.text)
        all_trs = selector.css("#ip_list tr")


        ip_list = []
        for tr in all_trs[1:]:
            speed = 999
            speed_str = tr.css(".bar::attr(title)").extract()[0]
            if speed_str:
                speed = float(speed_str.split("秒")[0])
            all_texts = tr.css("td::text").extract()

            ip = all_texts[0]
            port = all_texts[1]
            proxy_type = all_texts[5] if all_texts[5] in ['HTTP', 'HTTPS'] else 'HTTP'

            ip_list.append((ip, port, speed, proxy_type))

        insert_sql = '''
            insert ignore into proxy_ip(ip, port, speed, proxy_type)
            VALUES ('%s', '%s', %s, '%s')
        '''

        for ip_info in ip_list:
            cursor.execute(insert_sql % (ip_info[0], ip_info[1], ip_info[2], ip_info[3]))
            conn.commit()
```

采集结果如下所示：
![mysql_content](http://ouy59qaqh.bkt.clouddn.com/proxy_ip.jpg)

接下来我们可以从数据库中随机获取IP，检查器可用性，从而维护我们的IP池，然后通过类似添加User-Agent Middleware的方法添加我们的代理IP Middleware，这样就可以实现代理IP访问。但经过测试，网上的免费IP可用性太差，建议选择付费IP，服务商还会提供API接口，不需要我们自己维护IP池。

### 编写代理IP中间件

在 `middlewares.py` 文件中编写 `RandomProxyMiddleware` 类：

```python
class RandomProxyMiddleware(object):
    # 随机更换IP
    def get_ip(self):
        ret = requests.get(url='http://xxx/proxy/getProxies')
        data = json.loads(ret.text)
        while data['code'] != 0:
            time.sleep(1)
            ret = requests.get(url='http://xxx/proxy/getProxies')
            data = json.loads(ret.text)
        return data['proxy']['ip'] + ':' + str(data['proxy']['port'])

    def process_request(self, request, spider):
        request.meta['proxy'] = {
            'http': self.get_ip(),
            'https': self.get_ip()
        }
```

其中，`http://xxx/proxy/getProxies` 代表我的代理ip接口，接下来，只需要在我们的 `settings/py` 文件中开启代理IP中间件：

```python
'ArticleSpider.middlewares.RandomProxyMiddleware': 555,
```

这样，就可以实现代理IP进行抓取数据。

### scrapy-crawlera实现代理IP

这是一个收费的代理IP，它的使用方法非常简单，只需要在它的[官网](https://scrapinghub.com/crawlera)上注册一个 `apikey`，就可以直接在scrapy中配置，具体使用方法可参考github上的[说明文档](https://github.com/scrapy-plugins/scrapy-crawlera/blob/master/docs/index.rst)。

## Scrapy中的一些配置

### 禁用Cookie

* settings.py中配置 `COOKIES_ENABLED = False`。

### 配置下载延迟

这里用到了Scrapy的自动限速（$AutoThrottle$）扩展，该扩展能根据Scrapy服务器及爬取的网站的负载自动限制爬取速度。

下面是控制AutoThrottle扩展的设置：

> * AUTOTHROTTLE_ENABLED
    默认: False；启用AutoThrottle扩展。
> * AUTOTHROTTLE_START_DELAY
    默认: 5.0；初始下载延迟(单位:秒)。
> * AUTOTHROTTLE_MAX_DELAY
    默认: 60.0；在高延迟情况下最大的下载延迟(单位秒)。
> * AUTOTHROTTLE_DEBUG
    默认: False；启用AutoThrottle调试(debug)模式，展示每个接收到的response。 可以通过此来查看限速参数是如何实时被调整的。
    
*具体该扩展的实现等可以参考Scrapy[文档](https://doc.scrapy.org/en/latest/topics/autothrottle.html)。*

### 不同的spider设置自定义settings

有时，不同网站的爬虫可能需要不同的配置，例如知乎爬虫需要启用Cookie，从而维持登陆状态，而其他不需要登陆的网站则最好禁用掉Cookie，这就需要我们对一些spider自定义settings。这时，我们可以在知乎的spider中添加下面代码：

```python
custom_settings = {
    'COOKIES_ENABLED': True
}
```

这样就实现了知乎爬虫的自定义settings。
