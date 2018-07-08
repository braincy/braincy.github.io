---
layout: post
title: 布隆过滤器原理及使用
date: 2018-01-31 19:20:16
tags: [去重, 大数据]
categories: 爬虫
mathjax: true
---

* content
{:toc}

本文对 *布隆过滤器* 的原理进行介绍，并举例说明了PyBloom的使用方法。



## 什么是布隆过滤器

1970年，由布隆提出来的一个用于判断元素是否在集合中的高效的算法，集合中的元素可以增加，但是要删除一个元素比较困难，同时还有少量的误报率。

在数据量比较小的时候，我们可以使用 *Hash* 来判断元素是否命中，但是当元素增加起来后，*Hash* 算法需要的空间就会急速增长，查找时间也会增加。布隆过滤器主要用在样本集合量大但是很少有删除元素，不要求 $100\%$ 正确率的场景下。例如：`网页黑名单`、`垃圾邮件过滤`、`爬虫URL去重` 等。

## 布隆过滤器原理

### 爬虫URL去重

#### 初始条件

* 设数据集合 $A={a_1,a_2,....,a_n}$，含 $n$ 个 $url$ 记为 $a_i(i\in[1,n])$，作为待操作的集合。

* `Bloom Filter` 用一个长度为 $m$ 的位向量 $bitarray$ 表示的集合中的元素，位向量初始值全为 $0$。

* $k$ 个优秀且各自独立的哈希函数 $H_1,H_2,....,H_k$，且输出域应 $\geq m$。

#### 加入url的处理

* 首先经过 $k$ 个散列函数产生 $k$ 个随机数 $h_1,h_2,......h_k$，接着对 $m$ 取模得到 $h_i^{'}$，使向量 $bitarray$ 的相应位置 $h_1^{'},h_2^{'},......h_k^{'}$ 均置为 $1$。集合中其他 $url$ 也通过类似的操作，将向量 $bitarray$ 的若干位置为 $1$。

#### 检查是否重复

* 首先将该元素经过上步中类似操作，获得 $k$ 个随机数 $h_1^{'},h_2^{'},......h_k^{'}$ ，然后查看向量 $bitarray$ 的相应位置上的值，若全为 $1$，则该元素已经在之前的集合中；若至少有一个 $0$ 存在，表明，此元素不在之前的集合中，为新元素。

### 执行示意图

![原理图](/img/bloom_filter.jpg)

### 算法特点
* 对于已经在集合中的元素，通过上述中的查找方法，一定可以判定该元素在集合中。
* 对于不在集合中的元素，可能会被误判在集合中。

## 布隆过滤器的选择与质量评估

### 确定布隆过滤器的长度 $m$

设样本个数为 $n$，允许的错误率为 $p$，则，$$m=-\frac{n\cdot lnp}{(ln2)^2}$$

### 确定哈希函数的个数 $k$

根据已求得的 $m$，可得，$$k=ln2\cdot \frac{m}{n}$$

### 计算真实失误率

根据向上取整的 $m、n、k$，可求得，$$p_{true}=(1-e^{-\frac{nk}{m}})^{k}$$

## Python实现布隆过滤器

### 安装PyBloom

Python中有多个实现 `BloomFilter` 的包详情可以自己搜索Pypi，本文中主要介绍 PyBloom，可以通过 pip 进行安装。
```shell
pip install pybloom
```
也可以直接去作者的github上下载[源码](https://github.com/jaybaird/python-bloomfilter)编译安装。
```shell
python setup.py install
```

### PyBloom源码解析

pybloom主要包括两个类：`BloomFilter`和`ScalableBloomFilter`。

`BloomFilter` 是一个定容的过滤器，$error_{rate}$ 是指最大的误报率是 `0.1%`，而 `ScalableBloomFilter`是一个不定容量的布隆过滤器，它可以不断添加元素。`add` 方法是添加元素，如果元素已经在布隆过滤器中，就返回 `True`，如果不在返回 `Fasle` 并将该元素添加到过滤器中。判断一个元素是否在过滤器中，只需要使用 `in` 运算符即可。

#### ScalableBloomFilter类

在`ScalableBloomFilter`的 `add` 方法中可以看到：

![ScalableBloomFilter-add](/img/sca_bloom_filter.jpg)

其本质依旧是创建了一个`BloomFilter`类。

#### BloomFilter类

在`BloomFilter`的 `__init__` 函数中：

![BloomFilter-init](/img/bloom_filter_init.jpg)

可以看到它引用了Python的bitarray库来实现布隆过滤器。

在`BloomFilter`的 `add` 方法中：

![BloomFilter-add](/img/bloom_add.jpg)

可以看到，我们可以通过设置 $skip_{check}$ 的值来手动选择是否过滤当前元素，否则就根据算出的 $k$ 个 `Hash` 函数的值所对应的位是否都为 $1$ 来确定元素是否存在，存在则返回 `True`，否则返回 `False`。

### PyBloom的使用

#### 使用BloomFilter

```python
from pybloom import BloomFilter
bf = BloomFilter(capacity=10000, error_rate=0.001)
bf.add('test-bf')
print 'test-bf' in bf
```
> *True*

#### 使用ScalableBloomFilter

```python
from pybloom import ScalableBloomFilter
sbf = ScalableBloomFilter(mode=ScalableBloomFilter.SMALL_SET_GROWTH)
sbf.add('test-sbf')
print 'sbf' in sbf
```
> *False*
