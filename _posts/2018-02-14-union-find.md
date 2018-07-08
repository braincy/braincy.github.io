---
layout: post
title: 并查集介绍
date: 2018-02-14 02:11:56
tags: [并查集]
categories: 算法
mathjax: true
---

* content
{:toc}

本文对 *并查集* 的应用场景及实现方法进行介绍。




## 并查集介绍
### 并查集的应用
并查集可以解决的问题可以称为连接问题，那么什么是连接问题，下面来举例说明。

例如判断网络中节点间的连接状态，这里的网络是个抽象的概念，它包括用户之间形成的网络、道路交通、实际的网络等等。其中用户之间形成的网络是指我们在社交网站上会与一些人相互关注、相互联系，这样就形成了一个网络，通过并查集我们可以判断两个人能否通过各自相关的人产生联系。

### 并查集的操作
对于一组数据，主要支持两个操作：

* $union(p, q)$：并操作，将 $p$ 和 $q$ 两个元素合并（连接）在一起。
* $find(p)$：查操作，找到 $p$ 元素所在的组。

通过这两个操作我们就可以回答一个问题：

* $isConnected(p, q)$：判断 $p$，$q$ 两元素是否相连接。

## 并查集实现（优化后）

优化的总体思想就是将并查集生成的树形结构扁平化，从而减小查找、合并时间。

### 变量声明

* $rank[]$：$rank[i]$ 表示以 $i$ 为根的集合所表示的树的层数
* $parent$：$parent[i]$ 表示第 $i$ 个元素所指向的父节点

### 基于rank的优化
#### 算法思想
在每次执行 $union$ 操作的时候，选择将层数少的集合合并到层数多的集合上，从而最大程度上减慢树的高度的增加。
#### 代码实现
```C++
// 合并元素p和元素q所属的集合
// O(h)复杂度, h为树的高度
void unionElements(int p, int q){

    int pRoot = find(p);
    int qRoot = find(q);
    
    // 两元素已连接则无需合并
    if(pRoot == qRoot)
        return;

    // 根据两个元素所在树的元素层数不同判断合并方向
    // 将层数少的集合合并到层数多的集合上
    if(rank[pRoot] < rank[qRoot]){
        parent[pRoot] = qRoot;
    }
    else if(rank[qRoot] < rank[pRoot]){
        parent[qRoot] = pRoot;
    }
    else{ // rank[pRoot] == rank[qRoot]
        parent[pRoot] = qRoot;
        rank[qRoot] += 1;   // 此时, 我维护rank的值
    }
}
```
### 路径压缩
#### 算法思想
在每次执行 $find$ 操作的时候，如果未找到根结点，则将当前结点及其子结点向上移动，从而减小树的高度。
#### 代码实现--1
```C++
// 查找过程, 查找元素p所对应的集合编号
// O(h)复杂度, h为树的高度
int find(int p){
    assert( p >= 0 && p < count );

    while(p != parent[p]){
        parent[p] = parent[parent[p]]; // 每次向上跳一步
        p = parent[p];
    }
    return p;
}
```
#### 代码实现--2（递归版，理论最优）
```C++
// 查找过程, 查找元素p所对应的集合编号
// O(h)复杂度, h为树的高度
int find(int p){
    assert( p >= 0 && p < count );

    if( p != parent[p] )
        parent[p] = find(parent[p]); // 直接与根结点相连
    return parent[p];
}
```
