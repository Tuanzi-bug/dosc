---
title: "Redis-八股题"
slug: "redis"
description: "redis-八股题"
date: "2024-12-11"
math: true
license: 
hidden: false
draft: false 
categories: 
    - database
tags:
    - "redis"
    - "数据库"
---

![下载](./assets/%E4%B8%8B%E8%BD%BD.webp)

## 数据结构

### 1. Redis的数据类型以及实现的原理

Redis 中**5 种基础数据类型**：String（字符串）、List（列表）、Set（集合）、Hash（散列）、Zset（有序集合）。

![基本数据类型和数据结构对应关系](./assets/%E5%9F%BA%E6%9C%AC%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B.webp)

这些数据结构均为值的底层实现方式，在 Redis 中，核心在于键与值之间的组织。只要理解了这一核心要点，那么对于其他数据结构的操作，其实就是先找到值，然后在集合中进行相关操作。

Redis 使用了一个哈希表来保存所有键值对，一个哈希表，其实就是一个数组，数组的每个元素称为一个哈希桶。哈希桶中的 entry 元素中保存了`*key`和`*value`指针，分别指向了实际的键和值。值保存的并不是值本身，而是指向具体值的指针。

![全局哈希表](./assets/全局哈希表.webp)

### 2. 哈希表存在的冲突问题

当你往哈希表中写入更多数据时，可能两个 key 的哈希值和哈希桶计算对应关系时，正好落在了同一个哈希桶中。这就是哈希表存在的冲突问题。

Redis 解决哈希冲突的方式，就是链式哈希，就是指**同一个哈希桶中的多个元素用一个链表来保存，它们之间依次用指针连接**。

### 3. 为什么需要rehash？

哈希冲突链上的元素只能通过指针逐一查找再操作。如果哈希表里写入的数据越来越多，哈希冲突可能也会越来越多，这就会导致某些哈希冲突链过长，进而导致这个链上的元素查找耗时长，效率降低。

所以需要rehash操作，也就是增加现有的哈希桶数量，让逐渐增多的 entry 元素能在更多的桶之间分散保存，减少单个桶中的元素数量，从而减少单个桶中的冲突。

### 4. rehash是怎么做的，rehash会带来什么问题？怎么解决？

Redis 默认有两个全局哈希表：哈希表 1 和哈希表 2。开始插入数据时用哈希表 1，此时哈希表 2 未分配空间。数据增多时执行 rehash，分三步：

* 给哈希表 2 分配更大空间，如哈希表 1 的两倍；
* 把哈希表 1 数据重新映射并拷贝到哈希表 2；
* 释放哈希表 1 空间，然后切换到哈希表 2 保存更多数据，哈希表 1 留作下次 rehash 扩容备用。

（就是把数据向另一个更大的表拷贝了一下，之后就是两个表相互这样倒腾。）

存在一个问题：如果一次性把哈希表 1 中的数据都迁移完，会造成 Redis 线程阻塞，无法服务其他请求。此时，Redis 就无法快速访问数据了。

解决这个问题，采用**渐进式 rehash**。

在第二步拷贝数据时，Redis 正常处理客户端请求。每处理一个请求，就从哈希表 1 的第一个索引位置开始，顺带把此索引位置的所有 entries 拷贝到哈希表 2 。处理下一个请求时，再顺带拷贝哈希表 1 下一个索引位置的 entries。

**渐进式 rehash**巧妙地把一次性大量拷贝的开销，分摊到了多次处理请求的过程中，避免了耗时操作，保证了数据的快速访问。

![渐进式rehash](./assets/渐进式rehash.webp)

> 如果有新的请求进来，在rehash过程中：
>
> 对于查询、删除、更新等操作，Redis 会先在当前正在使用的哈希表（一般称为旧表）中进行查找。如果没有找到，再到正在 rehash 的新哈希表中查找。
>
> 对于插入操作，如果当前正在 rehash，新元素可能会被插入到新哈希表中，以加快 rehash 的完成。

### 5. 整数数组和压缩列表在查找时间复杂度方面并没有很大的优势，那为什么 Redis 还会把它们作为底层数据结构呢？

* 整数数组和压缩列表在存储相对紧凑的数据时，能够更有效地利用内存空间。相比于一些更复杂的数据结构，它们减少了内存开销和内存碎片的产生。
* 在数据量较小且操作相对简单的场景下，整数数组和压缩列表的实现较为简单，操作的性能开销相对较低，可以更好地利用 CPU 的缓存机制，提高数据访问的效率。
* 即使当进行随机访问时，虽然不是顺序访问，但由于数组元素的内存地址相邻，第一次访问某个元素可能未命中高速缓存，但相邻元素被加载进高速缓存的概率较大。

## 线程模型

### 1. redis 为什么是单线程的？



### 2. 单线程的redis 为什么这么快呢？

## 存储

### 1. AOF 日志介绍





### 2. AOF有什么风险





### 3. AOF 写回策略





### 4. 日志文件太大怎么办？





### 5. AOF重写会阻塞吗？

