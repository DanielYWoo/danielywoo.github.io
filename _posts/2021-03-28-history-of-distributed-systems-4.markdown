---
layout: post
title:  "分布式系统一致性的发展历史 (四)"
date:   2015-11-26 19:18:45
published: false
---

# Eventual Consistency

CRDT
lattice consistency
https://aphyr.com/posts/294-jepsen-cassandra
PN-counter
G-sets 2P-sets, OR-sets
microsoft video

# CAP Theorem



# 总结 Consistency Models without Synchronization

一致性模型一句话概括

Strict: 不存在

Linearizability: 所有进程看到的事件历史一致有序并符合时间先后顺序, 单个进程遵守program order。total order。

Sequential: 所有进程看到的事件历史一致有序但不需要符合时间先后顺序, 单个进程遵守program order。total order。

Causal: 所有进程看到的causally related事件历史一致有序, 单个进程遵守program order。partial order，不对没有causal关系的并发排序。

PRAM: 所有进程互相看到的写无序, 除非两个写来自一个进程 (所以叫pipeline)。partial order，不对跨进程的消息排序。

其他还有一些在分布式系统中不太常用的一致性模型在此就不做介绍了.


