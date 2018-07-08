---
layout: post
title: JavaScript基础
date: 2018-06-14 19:01:25
tags: [JavaScript]
categories: JavaScript
mathjax: true
---

* content
{:toc}

本文对 *JavaScript* 的基础语法进行介绍，包括一些误区及DOM、BOM操作等，为入门笔记。




## JavaScript数据类型
### undefined与null
undefined类型只有一个值，即特殊的undefined。
*说明：一般而言，不存在需要显式地把一个变量设置为undefined值的情况。*

null值表示一个空对象指针。如果定义的变量准备在将来用于保存对象，那么最好将该变量初始化为null而不是其他值。

### Number与isNaN
Number：表示整数和浮点数
NaN：即非数值（Not a Number）是一个特殊的数值
isNaN：检测传递的参数是否是非数值，返回值为boolean类型。

*说明：isNaN()对接收的数值，先尝试转换为数值，再检测是否为非数值。*
> * 任何涉及NaN的操作（例如NaN/10）都会返回NaN。
> * NaN与任何值都不相等，包括NaN本身。

### 数值转换
有三个函数可以把非数值转换为数值：`Number()`、`parseInt()` 和 `parseFloat()`。
其中 `Number()` 可以用于任何数据类型，而 `parseInt()` 和 `parseFloat()` 则专门用于把字符串转换成数值。

* parseInt()：会忽略字符串前面的空格，直至找到第一个非空格字符。
> parseInt()：转换空字符串返回NaN。
> parseInt()这个函数提供第二个参数：转换时使用的基数（即多少进制）。

* parseFloat()：从第一个字符开始解析每个字符，直至遇见一个无效的浮点数字符为止。

*注：除了第一个小数点有效外，parseFloat()与parseInt()的第二个区别在于它始终会忽略前导的零。*

### String与Boolean
* 除了 `0` 之外的所有数字，转换为布尔型都为**true**；
* 除 `''` 之外所有的字符，转换为布尔型都为**true**；
* `null` 和 `undefined` 转换为布尔型都为**false**。