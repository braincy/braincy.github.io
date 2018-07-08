---
layout: post
title: Scrapy-redis源码分析
date: 2018-02-27 11:08:36
tags: [Scrapy, Redis, 分布式, 源码分析]
categories: 爬虫
mathjax: true
---

* content
{:toc}

*Scrapy-Redis* 则是一个基于Redis的Scrapy分布式组件，本文对 *Scrapy-Redis* 的源码进行分析。




## Scrapy与Scrapy-redis比较

Scrapy是一个比较好用的Python爬虫框架，你只需要编写几个组件就可以实现网页数据的爬取。但是当我们要爬取的页面非常多的时候，单个主机的处理能力就不能满足我们的需求了（无论是处理速度还是网络请求的并发数），这时候分布式爬虫的优势就显现出来。

而Scrapy-Redis则是一个基于Redis的Scrapy分布式组件。它利用Redis对用于爬取的请求(Request)进行存储和调度(Schedule)，并对爬取产生的项目(items)存储以供后续处理使用。scrapy-redi重写了scrapy一些比较关键的代码，将scrapy变成一个可以在多个主机上同时运行的分布式爬虫。

原生的Scrapy的架构是这样子的：

![scrapy](/img/scrapy_2.png)

而Scrapy-redis的架构是下面这样的：

![scrapy-redis](/img/scrapy-redis.png)

可以看到，Scrapy-redis就是将Scrapy的大部分中间状态保存到Redis当中，从而使多台机器上的爬虫可以实现通信，实现分布式爬虫。

## Scrapy-redis源码

### connection.py

负责根据settings中配置实例化redis连接。被dupefilter和scheduler调用，总之涉及到redis存取的都要使用到这个模块。

```python
import six

from scrapy.utils.misc import load_object

from . import defaults


# Shortcut maps 'setting name' -> 'parmater name'.
SETTINGS_PARAMS_MAP = {
    'REDIS_URL': 'url',
    'REDIS_HOST': 'host',
    'REDIS_PORT': 'port',
    'REDIS_ENCODING': 'encoding',
}


def get_redis_from_settings(settings):
    """Returns a redis client instance from given Scrapy settings object.

    This function uses ``get_client`` to instantiate the client and uses
    ``defaults.REDIS_PARAMS`` global as defaults values for the parameters. You
    can override them using the ``REDIS_PARAMS`` setting.

    Parameters
    ----------
    settings : Settings
        A scrapy settings object. See the supported settings below.

    Returns
    -------
    server
        Redis client instance.

    Other Parameters
    ----------------
    REDIS_URL : str, optional
        Server connection URL.
    REDIS_HOST : str, optional
        Server host.
    REDIS_PORT : str, optional
        Server port.
    REDIS_ENCODING : str, optional
        Data encoding.
    REDIS_PARAMS : dict, optional
        Additional client parameters.

    """
    params = defaults.REDIS_PARAMS.copy()
    params.update(settings.getdict('REDIS_PARAMS'))
    # XXX: Deprecate REDIS_* settings.
    for source, dest in SETTINGS_PARAMS_MAP.items():
        val = settings.get(source)
        if val:
            params[dest] = val

    # Allow ``redis_cls`` to be a path to a class.
    if isinstance(params.get('redis_cls'), six.string_types):
        params['redis_cls'] = load_object(params['redis_cls'])

    return get_redis(**params)


# Backwards compatible alias.
from_settings = get_redis_from_settings


def get_redis(**kwargs):
    """Returns a redis client instance.

    Parameters
    ----------
    redis_cls : class, optional
        Defaults to ``redis.StrictRedis``.
    url : str, optional
        If given, ``redis_cls.from_url`` is used to instantiate the class.
    **kwargs
        Extra parameters to be passed to the ``redis_cls`` class.

    Returns
    -------
    server
        Redis client instance.

    """
    redis_cls = kwargs.pop('redis_cls', defaults.REDIS_CLS)
    url = kwargs.pop('url', None)
    if url:
        return redis_cls.from_url(url, **kwargs)
    else:
        return redis_cls(**kwargs)

```

connect文件引入了redis模块，这个是redis-python库的接口，用于通过python访问redis数据库，可见，这个文件主要是实现连接redis数据库的功能（返回的是redis库的Redis对象或者StrictRedis对象，这俩都是可以直接用来进行数据操作的对象）。这些连接接口在其他文件中经常被用到。其中，我们可以看到，要想连接到redis数据库，和其他数据库差不多，需要一个ip地址、端口号、用户名密码（可选）和一个整形的数据库编号，同时我们还可以在scrapy工程的settings文件中配置套接字的超时时间、等待时间等。

### picklecompat.py

```python
"""A pickle wrapper module with protocol=-1 by default."""

try:
    import cPickle as pickle  # PY2
except ImportError:
    import pickle


def loads(s):
    return pickle.loads(s)


def dumps(obj):
    return pickle.dumps(obj, protocol=-1)
```

这里实现了loads和dumps两个函数，其实就是实现了一个serializer，因为redis数据库不能存储复杂对象（value部分只能是字符串，字符串列表，字符串集合和hash，key部分只能是字符串），所以我们存啥都要先串行化成文本才行。这里使用的就是python的pickle模块，一个兼容py2和py3的串行化工具。这个serializer主要用于一会的scheduler存reuqest对象，至于为什么不实用json格式，我也不是很懂，item pipeline的串行化默认用的就是json。

### dupefilter.py

负责执行requst的去重，实现的很有技巧性，使用redis的set数据结构。但是注意scheduler并不使用其中用于在这个模块中实现的dupefilter键做request的调度，而是使用queue.py模块中实现的queue。当request不重复时，将其存入到queue中，调度时将其弹出。

```python
import logging
import time

from scrapy.dupefilters import BaseDupeFilter
from scrapy.utils.request import request_fingerprint

from . import defaults
from .connection import get_redis_from_settings
from ScrapyRedisTest.utils.bloomfilter import conn, PyBloomFilter


logger = logging.getLogger(__name__)


# TODO: Rename class to RedisDupeFilter.
class RFPDupeFilter(BaseDupeFilter):
    """Redis-based request duplicates filter.

    This class can also be used with default Scrapy's scheduler.

    """

    logger = logger

    def __init__(self, server, key, debug=False):
        """Initialize the duplicates filter.

        Parameters
        ----------
        server : redis.StrictRedis
            The redis server instance.
        key : str
            Redis key Where to store fingerprints.
        debug : bool, optional
            Whether to log filtered requests.

        """
        self.server = server
        self.key = key
        self.debug = debug
        self.logdupes = True

        self.bf = PyBloomFilter(conn=conn, key=key)

    @classmethod
    def from_settings(cls, settings):
        """Returns an instance from given settings.

        This uses by default the key ``dupefilter:<timestamp>``. When using the
        ``scrapy_redis.scheduler.Scheduler`` class, this method is not used as
        it needs to pass the spider name in the key.

        Parameters
        ----------
        settings : scrapy.settings.Settings

        Returns
        -------
        RFPDupeFilter
            A RFPDupeFilter instance.


        """
        server = get_redis_from_settings(settings)
        # XXX: This creates one-time key. needed to support to use this
        # class as standalone dupefilter with scrapy's default scheduler
        # if scrapy passes spider on open() method this wouldn't be needed
        # TODO: Use SCRAPY_JOB env as default and fallback to timestamp.
        key = defaults.DUPEFILTER_KEY % {'timestamp': int(time.time())}
        debug = settings.getbool('DUPEFILTER_DEBUG')
        return cls(server, key=key, debug=debug)

    @classmethod
    def from_crawler(cls, crawler):
        """Returns instance from crawler.

        Parameters
        ----------
        crawler : scrapy.crawler.Crawler

        Returns
        -------
        RFPDupeFilter
            Instance of RFPDupeFilter.

        """
        return cls.from_settings(crawler.settings)

    def request_seen(self, request):
        """Returns True if request was already seen.

        Parameters
        ----------
        request : scrapy.http.Request

        Returns
        -------
        bool

        """
        fp = self.request_fingerprint(request)

        if self.bf.is_exist(fp):
            return True
        else:
            self.bf.add(fp)
            return False
        # This returns the number of values added, zero if already exists.
        # added = self.server.sadd(self.key, fp)
        # return added == 0

    def request_fingerprint(self, request):
        """Returns a fingerprint for a given request.

        Parameters
        ----------
        request : scrapy.http.Request

        Returns
        -------
        str

        """
        return request_fingerprint(request)

    def close(self, reason=''):
        """Delete data on close. Called by Scrapy's scheduler.

        Parameters
        ----------
        reason : str, optional

        """
        self.clear()

    def clear(self):
        """Clears fingerprints data."""
        self.server.delete(self.key)

    def log(self, request, spider):
        """Logs given request.

        Parameters
        ----------
        request : scrapy.http.Request
        spider : scrapy.spiders.Spider

        """
        if self.debug:
            msg = "Filtered duplicate request: %(request)s"
            self.logger.debug(msg, {'request': request}, extra={'spider': spider})
        elif self.logdupes:
            msg = ("Filtered duplicate request %(request)s"
                   " - no more duplicates will be shown"
                   " (see DUPEFILTER_DEBUG to show all duplicates)")
            self.logger.debug(msg, {'request': request}, extra={'spider': spider})
            self.logdupes = False
```

这个文件看起来比较复杂，重写了scrapy本身已经实现的request判重功能。因为本身scrapy单机跑的话，只需要读取内存中的request队列或者持久化的request队列（scrapy默认的持久化似乎是json格式的文件，不是数据库）就能判断这次要发出的request url是否已经请求过或者正在调度（本地读就行了）。而分布式跑的话，就需要各个主机上的scheduler都连接同一个数据库的同一个request池来判断这次的请求是否是重复的了。

在这个文件中，通过继承BaseDupeFilter重写他的方法，实现了基于redis的判重。根据源代码来看，scrapy-redis使用了scrapy本身的一个fingerprint接request_fingerprint，这个接口很有趣，根据scrapy文档所说，他通过hash来判断两个url是否相同（相同的url会生成相同的hash结果），但是当两个url的地址相同，get型参数相同但是顺序不同时，也会生成相同的hash结果。所以scrapy-redis依旧使用url的fingerprint来判断request请求是否已经出现过。这个类通过连接redis，使用一个key来向redis的一个set中插入fingerprint（这个key对于同一种spider是相同的，redis是一个key-value的数据库，如果key是相同的，访问到的值就是相同的，这里使用spider名字+DupeFilter的key就是为了在不同主机上的不同爬虫实例，只要属于同一种spider，就会访问到同一个set，而这个set就是他们的url判重池），如果返回值为0，说明该set中该fingerprint已经存在（因为集合是没有重复值的），则返回True，如果返回值为1，说明添加了一个fingerprint到set中，则说明这个request没有重复，于是返回False，还顺便把新fingerprint加入到数据库中了。 DupeFilter判重会在scheduler类中用到，每一个request在进入调度之前都要进行判重，如果重复就不需要参加调度，直接舍弃就好了，不然就是白白浪费资源。

### queue.py

其作用如dupefilter.py所述，但是这里实现了三种方式的queue：FIFO的SpiderQueue，SpiderPriorityQueue，以及LIFI的SpiderStack，默认使用的是第二种。

```python
from scrapy.utils.reqser import request_to_dict, request_from_dict

from . import picklecompat


class Base(object):
    """Per-spider base queue class"""

    def __init__(self, server, spider, key, serializer=None):
        """Initialize per-spider redis queue.

        Parameters
        ----------
        server : StrictRedis
            Redis client instance.
        spider : Spider
            Scrapy spider instance.
        key: str
            Redis key where to put and get messages.
        serializer : object
            Serializer object with ``loads`` and ``dumps`` methods.

        """
        if serializer is None:
            # Backward compatibility.
            # TODO: deprecate pickle.
            serializer = picklecompat
        if not hasattr(serializer, 'loads'):
            raise TypeError("serializer does not implement 'loads' function: %r"
                            % serializer)
        if not hasattr(serializer, 'dumps'):
            raise TypeError("serializer '%s' does not implement 'dumps' function: %r"
                            % serializer)

        self.server = server
        self.spider = spider
        self.key = key % {'spider': spider.name}
        self.serializer = serializer

    def _encode_request(self, request):
        """Encode a request object"""
        obj = request_to_dict(request, self.spider)
        return self.serializer.dumps(obj)

    def _decode_request(self, encoded_request):
        """Decode an request previously encoded"""
        obj = self.serializer.loads(encoded_request)
        return request_from_dict(obj, self.spider)

    def __len__(self):
        """Return the length of the queue"""
        raise NotImplementedError

    def push(self, request):
        """Push a request"""
        raise NotImplementedError

    def pop(self, timeout=0):
        """Pop a request"""
        raise NotImplementedError

    def clear(self):
        """Clear queue/stack"""
        self.server.delete(self.key)


class FifoQueue(Base):
    """Per-spider FIFO queue"""

    def __len__(self):
        """Return the length of the queue"""
        return self.server.llen(self.key)

    def push(self, request):
        """Push a request"""
        self.server.lpush(self.key, self._encode_request(request))

    def pop(self, timeout=0):
        """Pop a request"""
        if timeout > 0:
            data = self.server.brpop(self.key, timeout)
            if isinstance(data, tuple):
                data = data[1]
        else:
            data = self.server.rpop(self.key)
        if data:
            return self._decode_request(data)


class PriorityQueue(Base):
    """Per-spider priority queue abstraction using redis' sorted set"""

    def __len__(self):
        """Return the length of the queue"""
        return self.server.zcard(self.key)

    def push(self, request):
        """Push a request"""
        data = self._encode_request(request)
        score = -request.priority
        # We don't use zadd method as the order of arguments change depending on
        # whether the class is Redis or StrictRedis, and the option of using
        # kwargs only accepts strings, not bytes.
        self.server.execute_command('ZADD', self.key, score, data)

    def pop(self, timeout=0):
        """
        Pop a request
        timeout not support in this queue class
        """
        # use atomic range/remove using multi/exec
        pipe = self.server.pipeline()
        pipe.multi()
        pipe.zrange(self.key, 0, 0).zremrangebyrank(self.key, 0, 0)
        results, count = pipe.execute()
        if results:
            return self._decode_request(results[0])


class LifoQueue(Base):
    """Per-spider LIFO queue."""

    def __len__(self):
        """Return the length of the stack"""
        return self.server.llen(self.key)

    def push(self, request):
        """Push a request"""
        self.server.lpush(self.key, self._encode_request(request))

    def pop(self, timeout=0):
        """Pop a request"""
        if timeout > 0:
            data = self.server.blpop(self.key, timeout)
            if isinstance(data, tuple):
                data = data[1]
        else:
            data = self.server.lpop(self.key)

        if data:
            return self._decode_request(data)


# TODO: Deprecate the use of these names.
SpiderQueue = FifoQueue
SpiderStack = LifoQueue
SpiderPriorityQueue = PriorityQueue
```

该文件实现了几个容器类，可以看这些容器和redis交互频繁，同时使用了我们上边picklecompat中定义的serializer。这个文件实现的几个容器大体相同，只不过一个是队列，一个是栈，一个是优先级队列，这三个容器到时候会被scheduler对象实例化，来实现request的调度。比如我们使用SpiderQueue作为调度队列的类型，到时候request的调度方法就是先进先出，而使用SpiderStack就是先进后出了。

我们可以仔细看看SpiderQueue的实现，他的push函数就和其他容器的一样，只不过push进去的request请求先被scrapy的接口request_to_dict变成了一个dict对象（因为request对象实在是比较复杂，有方法有属性不好串行化），之后使用picklecompat中的serializer串行化为字符串，然后使用一个特定的key存入redis中（该key在同一种spider中是相同的）。而调用pop时，其实就是从redis用那个特定的key去读其值（一个list），从list中读取最早进去的那个，于是就先进先出了。

这些容器类都会作为scheduler调度request的容器，scheduler在每个主机上都会实例化一个，并且和spider一一对应，所以分布式运行时会有一个spider的多个实例和一个scheduler的多个实例存在于不同的主机上，但是，因为scheduler都是用相同的容器，而这些容器都连接同一个redis服务器，又都使用spider名加queue来作为key读写数据，所以不同主机上的不同爬虫实例公用一个request调度池，实现了分布式爬虫之间的统一调度。

### pipelines.py

它将Item存储在redis中以实现分布式处理。另外可以发现，同样是编写pipelines，在这里的编码实现不同于文章中所分析的情况，由于在这里需要读取配置，所以就用到了from_crawler()函数。

```python
from scrapy.utils.misc import load_object
from scrapy.utils.serialize import ScrapyJSONEncoder
from twisted.internet.threads import deferToThread

from . import connection, defaults


default_serialize = ScrapyJSONEncoder().encode


class RedisPipeline(object):
    """Pushes serialized item into a redis list/queue

    Settings
    --------
    REDIS_ITEMS_KEY : str
        Redis key where to store items.
    REDIS_ITEMS_SERIALIZER : str
        Object path to serializer function.

    """

    def __init__(self, server,
                 key=defaults.PIPELINE_KEY,
                 serialize_func=default_serialize):
        """Initialize pipeline.

        Parameters
        ----------
        server : StrictRedis
            Redis client instance.
        key : str
            Redis key where to store items.
        serialize_func : callable
            Items serializer function.

        """
        self.server = server
        self.key = key
        self.serialize = serialize_func

    @classmethod
    def from_settings(cls, settings):
        params = {
            'server': connection.from_settings(settings),
        }
        if settings.get('REDIS_ITEMS_KEY'):
            params['key'] = settings['REDIS_ITEMS_KEY']
        if settings.get('REDIS_ITEMS_SERIALIZER'):
            params['serialize_func'] = load_object(
                settings['REDIS_ITEMS_SERIALIZER']
            )

        return cls(**params)

    @classmethod
    def from_crawler(cls, crawler):
        return cls.from_settings(crawler.settings)

    def process_item(self, item, spider):
        return deferToThread(self._process_item, item, spider)

    def _process_item(self, item, spider):
        key = self.item_key(item, spider)
        data = self.serialize(item)
        self.server.rpush(key, data)
        return item

    def item_key(self, item, spider):
        """Returns redis key based on given spider.

        Override this function to use a different key depending on the item
        and/or spider.

        """
        return self.key % {'spider': spider.name}
```

pipeline文件实现了一个item pipieline类，和scrapy的item pipeline是同一个对象，通过从settings中拿到我们配置的REDIS_ITEMS_KEY作为key，把item串行化之后存入redis数据库对应的value中（这个value可以看出出是个list，我们的每个item是这个list中的一个结点），这个pipeline把提取出的item存起来，主要是为了方便我们延后处理数据。

### scheduler.py

此扩展是对scrapy中自带的scheduler的替代（在settings的SCHEDULER变量中指出），正是利用此扩展实现crawler的分布式调度。其利用的数据结构来自于queue中实现的数据结构。

scrapy-redis所实现的两种分布式：`爬虫分布式` 以及 `item处理分布式` 就是由模块scheduler和模块pipelines实现。上述其它模块作为为二者辅助的功能模块。

```python
import importlib
import six

from scrapy.utils.misc import load_object

from . import connection, defaults


# TODO: add SCRAPY_JOB support.
class Scheduler(object):
    """Redis-based scheduler

    Settings
    --------
    SCHEDULER_PERSIST : bool (default: False)
        Whether to persist or clear redis queue.
    SCHEDULER_FLUSH_ON_START : bool (default: False)
        Whether to flush redis queue on start.
    SCHEDULER_IDLE_BEFORE_CLOSE : int (default: 0)
        How many seconds to wait before closing if no message is received.
    SCHEDULER_QUEUE_KEY : str
        Scheduler redis key.
    SCHEDULER_QUEUE_CLASS : str
        Scheduler queue class.
    SCHEDULER_DUPEFILTER_KEY : str
        Scheduler dupefilter redis key.
    SCHEDULER_DUPEFILTER_CLASS : str
        Scheduler dupefilter class.
    SCHEDULER_SERIALIZER : str
        Scheduler serializer.

    """

    def __init__(self, server,
                 persist=False,
                 flush_on_start=False,
                 queue_key=defaults.SCHEDULER_QUEUE_KEY,
                 queue_cls=defaults.SCHEDULER_QUEUE_CLASS,
                 dupefilter_key=defaults.SCHEDULER_DUPEFILTER_KEY,
                 dupefilter_cls=defaults.SCHEDULER_DUPEFILTER_CLASS,
                 idle_before_close=0,
                 serializer=None):
        """Initialize scheduler.

        Parameters
        ----------
        server : Redis
            The redis server instance.
        persist : bool
            Whether to flush requests when closing. Default is False.
        flush_on_start : bool
            Whether to flush requests on start. Default is False.
        queue_key : str
            Requests queue key.
        queue_cls : str
            Importable path to the queue class.
        dupefilter_key : str
            Duplicates filter key.
        dupefilter_cls : str
            Importable path to the dupefilter class.
        idle_before_close : int
            Timeout before giving up.

        """
        if idle_before_close < 0:
            raise TypeError("idle_before_close cannot be negative")

        self.server = server
        self.persist = persist
        self.flush_on_start = flush_on_start
        self.queue_key = queue_key
        self.queue_cls = queue_cls
        self.dupefilter_cls = dupefilter_cls
        self.dupefilter_key = dupefilter_key
        self.idle_before_close = idle_before_close
        self.serializer = serializer
        self.stats = None

    def __len__(self):
        return len(self.queue)

    @classmethod
    def from_settings(cls, settings):
        kwargs = {
            'persist': settings.getbool('SCHEDULER_PERSIST'),
            'flush_on_start': settings.getbool('SCHEDULER_FLUSH_ON_START'),
            'idle_before_close': settings.getint('SCHEDULER_IDLE_BEFORE_CLOSE'),
        }

        # If these values are missing, it means we want to use the defaults.
        optional = {
            # TODO: Use custom prefixes for this settings to note that are
            # specific to scrapy-redis.
            'queue_key': 'SCHEDULER_QUEUE_KEY',
            'queue_cls': 'SCHEDULER_QUEUE_CLASS',
            'dupefilter_key': 'SCHEDULER_DUPEFILTER_KEY',
            # We use the default setting name to keep compatibility.
            'dupefilter_cls': 'DUPEFILTER_CLASS',
            'serializer': 'SCHEDULER_SERIALIZER',
        }
        for name, setting_name in optional.items():
            val = settings.get(setting_name)
            if val:
                kwargs[name] = val

        # Support serializer as a path to a module.
        if isinstance(kwargs.get('serializer'), six.string_types):
            kwargs['serializer'] = importlib.import_module(kwargs['serializer'])

        server = connection.from_settings(settings)
        # Ensure the connection is working.
        server.ping()

        return cls(server=server, **kwargs)

    @classmethod
    def from_crawler(cls, crawler):
        instance = cls.from_settings(crawler.settings)
        # FIXME: for now, stats are only supported from this constructor
        instance.stats = crawler.stats
        return instance

    def open(self, spider):
        self.spider = spider

        try:
            self.queue = load_object(self.queue_cls)(
                server=self.server,
                spider=spider,
                key=self.queue_key % {'spider': spider.name},
                serializer=self.serializer,
            )
        except TypeError as e:
            raise ValueError("Failed to instantiate queue class '%s': %s",
                             self.queue_cls, e)

        try:
            self.df = load_object(self.dupefilter_cls)(
                server=self.server,
                key=self.dupefilter_key % {'spider': spider.name},
                debug=spider.settings.getbool('DUPEFILTER_DEBUG'),
            )
        except TypeError as e:
            raise ValueError("Failed to instantiate dupefilter class '%s': %s",
                             self.dupefilter_cls, e)

        if self.flush_on_start:
            self.flush()
        # notice if there are requests already in the queue to resume the crawl
        if len(self.queue):
            spider.log("Resuming crawl (%d requests scheduled)" % len(self.queue))

    def close(self, reason):
        if not self.persist:
            self.flush()

    def flush(self):
        self.df.clear()
        self.queue.clear()

    def enqueue_request(self, request):
        if not request.dont_filter and self.df.request_seen(request):
            self.df.log(request, self.spider)
            return False
        if self.stats:
            self.stats.inc_value('scheduler/enqueued/redis', spider=self.spider)
        self.queue.push(request)
        return True

    def next_request(self):
        block_pop_timeout = self.idle_before_close
        request = self.queue.pop(block_pop_timeout)
        if request and self.stats:
            self.stats.inc_value('scheduler/dequeued/redis', spider=self.spider)
        return request

    def has_pending_requests(self):
        return len(self) > 0
```

这个文件重写了scheduler类，用来代替scrapy.core.scheduler的原有调度器。其实对原有调度器的逻辑没有很大的改变，主要是使用了redis作为数据存储的媒介，以达到各个爬虫之间的统一调度。

scheduler负责调度各个spider的request请求，scheduler初始化时，通过settings文件读取queue和dupefilters的类型（一般就用上边默认的），配置queue和dupefilters使用的key（一般就是spider name加上queue或者dupefilters，这样对于同一种spider的不同实例，就会使用相同的数据块了）。每当一个request要被调度时，enqueue_request被调用，scheduler使用dupefilters来判断这个url是否重复，如果不重复，就添加到queue的容器中（先进先出，先进后出和优先级都可以，可以在settings中配置）。当调度完成时，next_request被调用，scheduler就通过queue容器的接口，取出一个request，把他发送给相应的spider，让spider进行爬取工作。

### spider.py

设计的这个spider从redis中读取要爬的url，然后执行爬取，若爬取过程中返回更多的url，那么继续进行直至所有的request完成。之后继续从redis中读取url，循环这个过程。

```python
from scrapy import signals
from scrapy.exceptions import DontCloseSpider
from scrapy.spiders import Spider, CrawlSpider

from . import connection, defaults
from .utils import bytes_to_str


class RedisMixin(object):
    """Mixin class to implement reading urls from a redis queue."""
    redis_key = None
    redis_batch_size = None
    redis_encoding = None

    # Redis client placeholder.
    server = None

    def start_requests(self):
        """Returns a batch of start requests from redis."""
        return self.next_requests()

    def setup_redis(self, crawler=None):
        """Setup redis connection and idle signal.

        This should be called after the spider has set its crawler object.
        """
        if self.server is not None:
            return

        if crawler is None:
            # We allow optional crawler argument to keep backwards
            # compatibility.
            # XXX: Raise a deprecation warning.
            crawler = getattr(self, 'crawler', None)

        if crawler is None:
            raise ValueError("crawler is required")

        settings = crawler.settings

        if self.redis_key is None:
            self.redis_key = settings.get(
                'REDIS_START_URLS_KEY', defaults.START_URLS_KEY,
            )

        self.redis_key = self.redis_key % {'name': self.name}

        if not self.redis_key.strip():
            raise ValueError("redis_key must not be empty")

        if self.redis_batch_size is None:
            # TODO: Deprecate this setting (REDIS_START_URLS_BATCH_SIZE).
            self.redis_batch_size = settings.getint(
                'REDIS_START_URLS_BATCH_SIZE',
                settings.getint('CONCURRENT_REQUESTS'),
            )

        try:
            self.redis_batch_size = int(self.redis_batch_size)
        except (TypeError, ValueError):
            raise ValueError("redis_batch_size must be an integer")

        if self.redis_encoding is None:
            self.redis_encoding = settings.get('REDIS_ENCODING', defaults.REDIS_ENCODING)

        self.logger.info("Reading start URLs from redis key '%(redis_key)s' "
                         "(batch size: %(redis_batch_size)s, encoding: %(redis_encoding)s",
                         self.__dict__)

        self.server = connection.from_settings(crawler.settings)
        # The idle signal is called when the spider has no requests left,
        # that's when we will schedule new requests from redis queue
        crawler.signals.connect(self.spider_idle, signal=signals.spider_idle)

    def next_requests(self):
        """Returns a request to be scheduled or none."""
        use_set = self.settings.getbool('REDIS_START_URLS_AS_SET', defaults.START_URLS_AS_SET)
        fetch_one = self.server.spop if use_set else self.server.lpop
        # XXX: Do we need to use a timeout here?
        found = 0
        # TODO: Use redis pipeline execution.
        while found < self.redis_batch_size:
            data = fetch_one(self.redis_key)
            if not data:
                # Queue empty.
                break
            req = self.make_request_from_data(data)
            if req:
                yield req
                found += 1
            else:
                self.logger.debug("Request not made from data: %r", data)

        if found:
            self.logger.debug("Read %s requests from '%s'", found, self.redis_key)

    def make_request_from_data(self, data):
        """Returns a Request instance from data coming from Redis.

        By default, ``data`` is an encoded URL. You can override this method to
        provide your own message decoding.

        Parameters
        ----------
        data : bytes
            Message from redis.

        """
        url = bytes_to_str(data, self.redis_encoding)
        return self.make_requests_from_url(url)

    def schedule_next_requests(self):
        """Schedules a request if available"""
        # TODO: While there is capacity, schedule a batch of redis requests.
        for req in self.next_requests():
            self.crawler.engine.crawl(req, spider=self)

    def spider_idle(self):
        """Schedules a request if available, otherwise waits."""
        # XXX: Handle a sentinel to close the spider.
        self.schedule_next_requests()
        raise DontCloseSpider


class RedisSpider(RedisMixin, Spider):
    """Spider that reads urls from redis queue when idle.

    Attributes
    ----------
    redis_key : str (default: REDIS_START_URLS_KEY)
        Redis key where to fetch start URLs from..
    redis_batch_size : int (default: CONCURRENT_REQUESTS)
        Number of messages to fetch from redis on each attempt.
    redis_encoding : str (default: REDIS_ENCODING)
        Encoding to use when decoding messages from redis queue.

    Settings
    --------
    REDIS_START_URLS_KEY : str (default: "<spider.name>:start_urls")
        Default Redis key where to fetch start URLs from..
    REDIS_START_URLS_BATCH_SIZE : int (deprecated by CONCURRENT_REQUESTS)
        Default number of messages to fetch from redis on each attempt.
    REDIS_START_URLS_AS_SET : bool (default: False)
        Use SET operations to retrieve messages from the redis queue. If False,
        the messages are retrieve using the LPOP command.
    REDIS_ENCODING : str (default: "utf-8")
        Default encoding to use when decoding messages from redis queue.

    """

    @classmethod
    def from_crawler(self, crawler, *args, **kwargs):
        obj = super(RedisSpider, self).from_crawler(crawler, *args, **kwargs)
        obj.setup_redis(crawler)
        return obj


class RedisCrawlSpider(RedisMixin, CrawlSpider):
    """Spider that reads urls from redis queue when idle.

    Attributes
    ----------
    redis_key : str (default: REDIS_START_URLS_KEY)
        Redis key where to fetch start URLs from..
    redis_batch_size : int (default: CONCURRENT_REQUESTS)
        Number of messages to fetch from redis on each attempt.
    redis_encoding : str (default: REDIS_ENCODING)
        Encoding to use when decoding messages from redis queue.

    Settings
    --------
    REDIS_START_URLS_KEY : str (default: "<spider.name>:start_urls")
        Default Redis key where to fetch start URLs from..
    REDIS_START_URLS_BATCH_SIZE : int (deprecated by CONCURRENT_REQUESTS)
        Default number of messages to fetch from redis on each attempt.
    REDIS_START_URLS_AS_SET : bool (default: True)
        Use SET operations to retrieve messages from the redis queue.
    REDIS_ENCODING : str (default: "utf-8")
        Default encoding to use when decoding messages from redis queue.

    """

    @classmethod
    def from_crawler(self, crawler, *args, **kwargs):
        obj = super(RedisCrawlSpider, self).from_crawler(crawler, *args, **kwargs)
        obj.setup_redis(crawler)
        return obj
```

spider的改动也不是很大，主要是通过connect接口，给spider绑定了spider_idle信号，spider初始化时，通过setup_redis函数初始化好和redis的连接，之后通过next_requests函数从redis中取出strat url，使用的key是settings中REDIS_START_URLS_AS_SET定义的（注意了这里的初始化url池和我们上边的queue的url池不是一个东西，queue的池是用于调度的，初始化url池是存放入口url的，他们都存在redis中，但是使用不同的key来区分，就当成是不同的表吧），spider使用少量的start url，可以发展出很多新的url，这些url会进入scheduler进行判重和调度。直到spider跑到调度池内没有url的时候，会触发spider_idle信号，从而触发spider的next_requests函数，再次从redis的start url池中读取一些url。

## 组件之间的关系

![scrapy-redis-struct](/img/scrapy-redis_struct.png)

转自：[标点符](https://www.biaodianfu.com/)的[《使用Scrapy-redis实现分布式爬取》](https://www.biaodianfu.com/scrapy-redis.html)
