---
author: "李昌"
title: "概览Redis篇一：单线程模型"
date: "2022-07-09"
tags: ["redis", "cache"]
categories: ["cache"]
ShowToc: true
TocOpen: true
---

> 极客时间《Redis 核心技术与实战》学习笔记

- [概览Redis篇一：单线程模型](https://yangchnet.github.io/Dessert/posts/cache/%E6%A6%82%E8%A7%88redis%E7%AF%87%E4%B8%80%E5%8D%95%E7%BA%BF%E7%A8%8B%E6%A8%A1%E5%9E%8B/)
- [概览Redis篇二：AOF日志](https://yangchnet.github.io/Dessert/posts/cache/%E6%A6%82%E8%A7%88redis%E7%AF%87%E4%BA%8Caof%E6%97%A5%E5%BF%97/)
- [概览Redis篇三：RDB快照](https://yangchnet.github.io/Dessert/posts/cache/%E6%A6%82%E8%A7%88redis%E7%AF%87%E4%B8%89rdb%E5%BF%AB%E7%85%A7/)
- [概览Redis篇四：主从](https://yangchnet.github.io/Dessert/posts/cache/%E6%A6%82%E8%A7%88redis%E7%AF%87%E5%9B%9B%E4%B8%BB%E4%BB%8E/)
- [概览Redis篇五：哨兵](https://yangchnet.github.io/Dessert/posts/cache/%E6%A6%82%E8%A7%88redis%E7%AF%87%E4%BA%94%E5%93%A8%E5%85%B5/)
- [概览Redis篇六：切片集群](https://yangchnet.github.io/Dessert/posts/cache/%E6%A6%82%E8%A7%88redis%E7%AF%87%E5%85%AD%E5%88%87%E7%89%87%E9%9B%86%E7%BE%A4/)

## Redis单线程模型

Redis 是单线程，主要是指 Redis 的网络 IO 和键值对读写是由一个线程来完成的，这也是 Redis 对外提供键值存储服务的主要流程。但 Redis 的其他功能，比如持久化、异步删除、集群数据同步等，其实是由额外的线程执行的。


## 为什么Redis采用单线程

通常的程序设计都采用多线程来提高性能，获得更快的响应速度。

但采用多线程模型将不可避免地面对以下问题：
1. 多个线程对同一资源的数据竞争
2. 因解决数据竞争而导致的性能损耗
3. 增加系统复杂度，降低系统代码的易调试性和可维护性

## 为啥Redis单线程还这么快

主要是两点：
1. Redis大部分操作在内存上完成，并采用了高效的数据结构
2. Redis采用了多路复用机制

