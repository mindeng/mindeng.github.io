
---
title: "Bloom Filter 布隆过滤器"
date: 2022-12-04T11:11:00.000Z
lastmod: 2022-12-04T13:05:00.000Z
tags: ['algorithm']
draft: false
---


> Bloom Filter 是由 Bloom 在 1970 年提出的一种多哈希函数映射的快速查找算法。  
通常应用在一些需要快速判断某个元素是否属于集合，但是并不严格要求 100% 正确的场合。  


## HashMap 的问题

HashMap 具有 O(1) 的查找复杂度，但缺点也很明显：内存利用率不高。

如果需要查询的是海量数据，使用 HashMap 就变得不太现实。


## Bloom Filter

Bloom filter 是一种概率型数据结构，具备如下几个特点：  
  
-   高效的插入、查询  
-   极高的空间利用率  
-   可以告诉你：*某条记录****一定不存在，****或者****有可能存在***


## 实现原理

![](/uploads/images/cb93f59d-fcf0-4e3e-a4b0-0cc08403c235/Untitled.png)

如上图，bloom filter 底层结构是一个 bit 数组（也可以是多个）。  
  
-   插入：采用多个独立的 hash 函数，分别计算该记录的 hash code，并将 hash code 所指向的 bit 位设置为 1；  
-   查询：采用多个独立的 hash 函数，分别计算该记录的 hash code  
      
    -   如果其中某个 hash code 指向的 bit 位 ≠ 1，则该记录**一定不存在**；      
    -   如果每个 hash code 指向的 bit 位都为 1，则该记录**可能存在**。


## 使用场景

例如：  
  
-   浏览器用来快速确定某个 URL 是否存在本地缓存；  
-   爬虫程序用来快速确定某个 URL 是否被爬取过；  
-   搜索引擎用来快速确定某个 URL 是否被索引过。