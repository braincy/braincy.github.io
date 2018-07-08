---
layout: post
title: 多变量线性回归
date: 2018-01-25 14:41:58
tags: [线性回归]
categories: 机器学习
mathjax: true
---

* content
{:toc}

本文对 *多变量线性回归* 问题及常用解决方法进行介绍。




## 梯度下降算法

与单变量线性回归类似，在多变量线性回归中，同样需要构建一个代价函数，这个代价函数是所有建模误差的平方和，即:

$$J(\theta\_{0},\theta\_{1}...\theta\_{n})=\frac{1}{2m}\sum\_{i=1}^{m}(h\_\theta(x^{(i)})-y^{(i)})^2$$

其中：$$h\_\theta(x)=\theta^TX=\theta\_0x\_0+\theta\_1x\_1+\theta\_2x\_2+...+\theta\_nx\_n$$

现在要找出使得代价函数最小的一系列参数。 多变量线性回归的批量梯度下降算法为:

$$\theta\_j:=\theta\_j-\alpha\frac{\partial}{\partial\theta\_j}\frac{1}{2m}J(\theta\_{0},\theta\_{1}...\theta\_{n})$$

即：$$\theta\_j:=\theta\_j-\alpha\frac{\partial}{\partial\theta\_j}\frac{1}{2m}\sum\_{i=1}^{m}(h\_\theta(x^{(i)})-y^{(i)})^2$$

求导后得到：$$\theta\_j:=\theta\_j-\alpha\frac{1}{m}\sum\_{i=1}^{m}((h\_\theta(x^{(i)})-y^{(i)})\cdot x\_{j}^{(i)})$$

接下来需要随机选择一系列的参数值$\theta\_j$，计算所有的预测结果后，再给所有的参数$\theta\_j$一个新的值，如此循环直至收敛。

## 正规方程

正规方程是通过求解下面的方程来找出使得代价函数最小的参数的：

$$\frac{\partial}{\partial\theta\_j}J(\theta\_j)=0$$

假设我们的训练集特征矩阵为 $X$（包含了 $x\_0=1$），并且我们的训练集结果为向量$y$，则利用正规方程解出向量：

$$\theta=(X^TX)^{-1}X^Ty$$

上标 $T$ 代表矩阵转置，上标 $-1$ 代表矩阵的逆。

在Octave中，正规方程写作：

```shell
pinv(X'*X)*X'*y
```

*注：对于那些不可逆矩阵（通常是因为特征之间不独立，也有可能特征数量大于训练集数量），正规方程方法是不能用的。*

## 梯度下降与正规方程的比较

![比较](http://ouy59qaqh.bkt.clouddn.com/linear-regression-compare.jpg)

总结一下，只要特征变量的数目并不大，正规方程是一个很好的计算参数 $\theta$ 的替代方法。具体来说，只要特征变量数量小于一万，通常使用正规方程法，而不使用梯度下降法。
