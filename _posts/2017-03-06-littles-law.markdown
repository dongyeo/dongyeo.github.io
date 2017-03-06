---
layout:     post
title:      "Little's Law"
date:       2017-03-06 21:00:00 +0800
author:     "DongYeo"
header-img: "img/post-bg-03.jpg"
tags: ["分布式"]
---


在阅读[Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)这篇论文的时候，在“Reducing Latency” 这个章节，发现一个新的名词，**Little's Law**

遂wiki了一下，发现是一个经济学的法则，但在软件负载相关的地方也有应用。

## 什么是Little's Law

  利特尔法则由麻省理工大学斯隆商学院（MIT Sloan School of Management）的教授John Little﹐于1961年所提出与证明。它是一个有关提前期与在制品关系的简单数学公式，这一法则为精益生产的改善方向指明了道路。

  如何有效地缩短生产周期呢？利特尔法则已经很明显地指出了方向。一个方向是提高产能，从而降低生产节拍。另一个方向就是压缩存货数量。然而，提高往往意味着增加很大的投入。另外，生产能力的提升虽然可以缩短生产周期，但是，生产能力的提升总有个限度，我们无法容忍生产能力远远超过市场的需求。一般来说，每个公司在一定时期内的生产能力是大致不变的，而从长期来看，各公司也会力图使自己公司的产能与市场需求相吻合。因此，最有效地缩短生产周期的方法就是压缩在制品数量。

　利特尔法则不仅适用于整个系统，而且也适用于系统的任何一部分。

## 公式

```
Little's Law tells us that the average number of customers in the store L, is the effective arrival rate λ, times the average time that a customer spends in the store W, or simply:

L=λ*W,
```
说人话，就是一个稳定的客流的商店里，平均店中停留的客户数L等于客户进入率λ乘以每个客户在商店停留的是W。

## 在软件编程中的例子：

直接复制一下[mbalib里特尔法则](http://wiki.mbalib.com/wiki/%E5%88%A9%E7%89%B9%E5%B0%94%E6%B3%95%E5%88%99)中的例子

假定我们所开发的并发服务器，并发的访问速率是：1000客户/分钟，每个客户在该服务器上将花费平均0.5分钟，根据little's law规则，在任何时刻，服务器将承担1000×0.5＝500个客户量的业务处理。假定过了一段时间，由于客户群的增大，并发的访问速率提升为2000客户/分钟。在这样的情况下，我们该如何改进我们系统的性能？

根据little's law规则，有两种方案：

1. 提高服务器并发处理的业务量，即提高到2000×0.5＝1000

2. 第二：减少服务器平均处理客户请求的时间，即减少到：2000×0.25＝500


## 参考

1. [Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)
2. [Little's Law](https://en.wikipedia.org/wiki/Little's_law)
3. [mbalib里特尔法则](http://wiki.mbalib.com/wiki/%E5%88%A9%E7%89%B9%E5%B0%94%E6%B3%95%E5%88%99)
