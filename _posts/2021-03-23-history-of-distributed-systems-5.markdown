---
layout: post
title:  "分布式系统一致性的发展历史 (五)"
Subtitle: "Serializability Consistency, External Consistency"
date:   2021-03-23 18:00:00
published: false
---

## Overview

之前讲的都是单一操作, 很适合KV类型的数据结构，比如缓存和大多数的NoSQL数据库。但是接下来我们要看一下多操作的情况，比如对多个对象的读写操作。与之相对应的是更像传统数据库的运行方式，比如传统数据库的transaction可以操作多个对象，然后通过MVCC实现隔离。实际上这篇文章里讲到的都是数据库transaction类型的一致性问题。

## Serializability Consistency

其实Serializability的概念在70年代早期已经被广泛接受，我能看到比较早的给出严谨定义的是在1979年麻省理工的Christos H. Papadimitriou在一篇论文中。[[1]](#参考)

> A sequence of atomic user updates/retrievals is called serializable essentially if its overall effect is as though the users took turns, in some order, each executing their entire transaction indivisibly.

在Herlihy和Wing讲述Linearizability的论文中也提及了Serializability的定义[[2]](#参考)

> A history is serializable if it is equivalent to one in which transactions appear to execute sequentially, i.e., without interleaving.

从这两个定义来看，其实很多人说的Serializability表示没有并行，纯串行执行事务是不严谨的， 其实他是允许并行的，只是他的执行顺序从执行方来看，<b>等同于</b>串行执行。比如两个事务T1和T2，如果他们是并发执行，但是如果他们没有操作同样的数据，那么从其他客户端看来，T1和T2就像是串行执行的。我们有其他的方式来实现Serializability, 只是实际实现中无并发串行执行是最简单实现的，所以很多人会误认为Serializability就是串行化执行。

为了更具体的解释Serializaiblity，我们需要解释一下Serial Schedule和Non-Serial Schedule的概念。如果你读过本系列文章里的论文，你对于Schedule这个词应该不陌生，但是Serial Schedule可能你是第一次听到。

### Serial Schedule
Gifford在讲述External Consistency的论文[[3]](#参考)中举了个例子来解释Serial Consistency。假设有两个transactions T1和T2:
```
Tl                 T2
(all) Read Y       (a2l) Read X
(a12) Write X      (a22) Read Y
(a13) Write Y      (a23) Write X
```
T1和T2里面六个action执行的顺序叫做一个schedule。而一个serial schedule是指一次只执行一个事务的时候的schedule。比如，T1和T2有两种serial schedule：

```
Sa: {all, a12, a13, a2l, a22, a23}
Sb: {a2l, a22, a23, all, a12, a13}. 
```

但是如果T1提交之后，T2才开始执行，或者T2提交后T1才开始执行，两个事务没有任何时间重叠，这就是Serial Schedule。这种情况显然符合Serializability Consistency。

### Non-Serial Schedule

两个事务如果时间上有重叠的执行顺序，我们称之为Non-Serial Schedule。这种情况比较复杂，需要分析来看，有可能符合Serializability Consistency，也有可能不符合。具体来说如果符合Serializability的话有两种情况：Conflict Equivalent Serializability和View Equivalent Serializability.

#### Conflict Equivalent Serializability

首先先定义Conflicting operations，如果同时满足以下三个条件我们称之为Conflicting Operations

1. 两个相邻操作属于不同事务
2. 两个操作操作同一个数据（比如某表同一行）
3. 两个操作至少有一个是写操作（两个读操作不算）

比如最常见的情况：T1执行 W(X) ，在T1提交前T2执行 R(X) 。如果T1和T2可以通过交换那些non-conflicting operations最后把这个Schedule变成一个Serial Schedule，那么我们称这个Schedule是Conflict Serializable, 两个schedule互相为Conflict Equivalent Schedules.

下图中，左边是个non-serial schedule，T1/T2的执行是有重叠的。我用红色线标出了所有的conflicting operations。因为T1 R(X) 和 T2  R(Y)没有conflict，所以我可以互换他们变成下图中间的顺序，然后因为T1 R(X) 和 T2 W(Y) 也没有conflict，我可以继续互换他们两个操作，最后得到下图最右边的schedule，而这个schedule是没有重叠的serial schedule。因此，我们说最左边的non-serial schedule和最右边的serial schedule是Conflict Equivalent Schedules, 最左边的原始schedule是符合Conflict Equivalent Serializability的。

<img src="../images/2021-03-23/conflict-equivalent.png" max-height="500px">

这种情况的本质就是不改变causal(conflict)关系的情况下，把没有causal关系的操作保持单个进程内的programming order进行排序，得到一个完全事务串行化的schedule。因为没有改变causal关系，所以前后两个schedule等同，那么我们认为原始的schedule中的conflict操作都被等同于分割到两个完全串行化的事务了，所以我们认为这也算是一种符合Serializability Consistency的情况。

#### View Equivalent Serializability

判断两个Schedule是否是View Equivalent需要看三个条件：

我们引入新符号IR(X)来表示一个事务中第一次对X的读取, 引入新符号FW(X)表示一个事务中最后一次对X的写入，"<"表示“早于”的关系，">"表示“晚于”的关系。

1. **Initial Read:** 对任何一个变量X，在两个schedule 里都是 T1 IR(X) < T2 IR(X)，那么这两个schedule是满足第一个条件的。

2. **Final Write:** 对任何一个变量X，在两个schedule 里都是T1 FW(X) > T2 FW(X)，那么这两个schedule是满足第二个条件的。

3. **Update Read:** 对任何一个变量X，在两个schedule里都是T1 W(X) < T2 R(X), 那么这两个schedule是满足第三个条件的。

举个例子，下图中两个schedule里，我分别用绿色，蓝色，紫色来代表条件1/2/3。我们把左边的schedule保持T1和T2内部的programming order不变的情况下重排称右边的schedule，我们发现六条线一个也没少，所以二者是view equivalent的。因为右边的schedule是serial schedule，因此左边的原始schedule符合Serializability Consistency。

<img src="../images/2021-03-23/view-equivalent.png" max-height="500px">

#### 小结
如下图所示，Serial Schedule和部分Non-Serial Schedule是符合Serializability Consistency的。

<img src="../images/2021-03-23/serializability-all.png" max-height="500px">

## Strict Serializability Consistency

在这篇讲述Linearizability的经典论文中[[2]](#参考)Herlihy和Wing也提及了Strict Serializability。

>A (partial) precedence order can be defined on non-overlapping pairs of transactions in the obvious way. A history is strictly serializable if the transactions’ order in the sequential history is compatible with their precedence order.

前半句意思是说我们可以对没有重叠的事务定义一个偏序关系，这里说的"obvious way"其实就是指T1结束早于T2开始，这其实就是Linearizability体现的实时性。后半句意思是schedule要满足这个实时性。

假如有一个schedule里的T1和T2没有重叠，这个schedule是个serial schedule，那么如果我们按照实际时间观察，发现T1结束早于T2开始，那么我们T2早于T1的顺序的schedule虽然是Serializable但是不是Strictly Serializable的。如果这个Schedule是T1早于T2的，那么这就是个Strictly Serializable 的Schedule，符合Strict Serializability Consistency。

这篇论文里作者还做了个类比，Linearizability可以看做Strict Serializability只操作单个资源的时候的一个特例。

>Linearizability can be viewed as a special case of strict serializability where transactions are restricted to consist of a single operation applied to a single object. 

为什么我们这个系列的文章先讲了Linearizability并且他有单独的一个模型呢？这是因为多操作和单操作会给实现带来巨大的差异，多操作要复杂的多，通常要通过MVCC和Strict Lock等方式来实现，而单操作的Paxos/Raft要简单得多。作者在原文中提到了两个主要差异，这里就不翻译了：


>One important formal difference between linearizability and serializability is that neither serializability nor strict serializability is a local property. 
>
>Another important formal difference is that serializability places more rigorous restrictions on concurrency. 

> Serializability is appropriate for systems such as databases in which it must be easy for application programmers to preserve complex application-specific invariants spanning multiple objects. A general-purpose serialization protocol, such as two-phase locking, enables programmers to reason about transactions as if they were sequential programs (setting aside questions of deadlock or performance).

> Linearizability, by contrast, is intended for applications such as multiprocessor operating systems in which concurrency is of primary interest, and where programmers are willing to apply special-purpose synchronization protocols, and to reason explicitly about the effects of concurrency. 

# External Consistency

In 1981, David Gifford第一次定义了External Consistency.

> The actual time order in which transactions complete defines a unique serial schedule. This serial schedule is called the external schedule. A system is said to provide external consistency if it guarantees that the schedule it will use to process a set of transactions is equivalent to its external schedule.

之前我们已经解释了什么是Serial Schedule，这个定义看起来怎么感觉和Strict Serializability一样呢？又要实时性，又要多对象操作的串行化。

External Consistency在单节点系统上实现并不难，但是在高并发的分布式系统中实现则非常困难。Google的Spanner是最具代表性的实现了External Consistency的超大规模分布式系统。在James Corbett 和大神 Jeff Dean的论文[[4]](#参考)中描述了Spanner是如何通过TrueTime来实现的。

Spanner的设计目标是实现支持全球跨地区超过百万节点的规模的高可用数据库。通常一份数据会被复制到3到5个数据中心，每个数据分片都会被多个Paxos状态机维持。在这套强悍的基础设施之上他还实现了两个在分布式系统中非常难的特性，其实都是和一致性相关的：

特性1: External Consistency级别的一致性。

特性2: 按照时间戳的全局一致快照读。 

他的这个能力来自于TrueTime系统。这个系统通过GPS和原子钟提供了一套非常简单的API，保证节点之间的时间误差非常小。它主要是三个接口：

| Method | Returns |
| ------ | ------- |
| TT.now() | TTinterval: [earliest, latest] |
| TT.after(t) | true if t has definitely passed, TT.now().earlest > t |
|TT.before(t)| true if t has definitely not arrived, TT.now().latest < t |

TT.now()返回的是一个时间范围，因为这个API本身需要时间计算，客户端发起和接收结果也有传输延迟，因此TT.now()的返回的是当前时间的可能范围。这个范围的一半叫做ϵ，通常ϵ不超过7ms。

这是因为我们可以根据TrueTime的时间戳确定事务的顺序所以可以实现特性1。一旦所有数据都有历史和时间戳，那么即便是数据在快速变化，我们仍然可以按照时间戳来查询到一组数据某个历史的快照实现特性2。

Spanner有四种事务类型，有三种都是lock-free的，这三种其实底层实现都是一样的，都是利用了“特性2”，只是Read-Only Transaction是Spanner自己决定当前时间戳，后面两个是调用方指定时间戳。

| Operation              | Concurrency Control |
| ---------------------- | ------------------- |
| Read-Write Transaction | pessimistic         |
| Read-Only Transaction|lock-free |
|Snapshot Read, client-provided timestamp |lock-free|
|Snapshot Read, client-provided bound |lock-free|

第一种Read-Write是指包含至少一个写操作的事务。它的实现包含了 2-phase locking 和 2-phase commit. 这个过程比较复杂，我们只需要关注两个操作。

Commit Start阶段：当coordinator leader收到对事务T<sub>i</sub>提交的事件e<sub>i</sub>的时候，它会为这个事务分配一个时间戳s<sub>i</sub> = TT.now().latest

Commit Wait阶段：coordinator leader会阻止任何客户端看到刚才写入的数据一会，因为Coordinator和Client的时间存在误差，要等到 TT.now().earlest > s<sub>i</sub>之后才能对外暴露（）.

 for a write Ti assigns a commit timestamp si no less than the value of TT.now().latest, computed after e server i . Note that the participant leaders do not matter here; Section 4.2.1 describes how they are involved in the implementation of the next rule.

: The coordinator leader ensures that clients cannot see any data committed by Ti until si < TT.now().earliest.

<img src="../images/2021-03-23/spanner-read-write-tx.png" max-height="500px">



后三种read-only情况的实现，这个比较简单, 分两个阶段

1) 分配或者接受客户端指定的一个时间戳 s<sub>read</sub> 。

2) 按照 s<sub>read</sub> 读取数据，这个过程无锁，写入可以继续。

Snapshot reads can execute at any replica that is sufficiently up to date (i.e. sread is less than or equal to a tsafe).

if a transaction *T1* commits before another transaction *T2* starts, then *T1*’s commit timestamp is smaller than *T2*’s.

Denote the absolute time of an event e by the function t<sub>abs</sub>(e). In more formal terms, TrueTime guarantees that for an invocation tt = TT.now(), tt.earliest ≤ t<sub>abs</sub>(e<sub>now</sub>) ≤ tt.latest, where e<sub>now</sub> is the invocation event.











## TrueTime's benefits for Cloud Spanner

TrueTime is a highly available, distributed clock that is provided to applications on all Google servers[1](https://cloud.google.com/spanner/docs/true-time-external-consistency#1). TrueTime enables applications to generate monotonically increasing timestamps: an application can compute a timestamp T that is guaranteed to be greater than any timestamp T' if T' finished being generated before T started being generated. This guarantee holds across all servers and all timestamps.



在2 How TrueTime translates to distributed systems concepts 

> Spanner’s TT-based approach and the causality tracking approach of asynchronous distributed systems sit in two extreme opposite ends of the spectrum. The literature on asynchronous distributed systems ignores wallclock information completely (i.e., it assumes an infinite uncertainty interval), and orders events by just tracking logical causality relations between them based on applying these two rules transitively: 1) if events e and f are in the same site and e occurred before f, then e happened-before f, and 2) if e is a sending of a message m and f is the receipt of m, then e happened-before f. Events e and f are concurrent, if both e happened-before f and f happenedbefore e are false. This causality information is maintained, typically using vector clocks (VCs), at each site with respect to all the other sites. As the number of sites (spanservers) in Spanner can be on the order of tens of thousands, the causality information maintained as such is highly prohibitive to store in the multiversion database, and very hard to query as well. In the other end of the spectrum, Spanner’s TT-based approach discards the tracking of causality information completely. Instead it goes for an engineering solution of using highly-precise dedicated clock sources to reduce the size of the uncertainty intervals to be negligible and order events using wallclock time—provided that the uncertainty intervals of the events are non-overlapping. This wallclock ordering in TT is in one sense stronger than the causal happened-before relation in traditional distributed systems since it does not require any communication to take place between the two events to be ordered; sufficient progression of the wallclock between the two events is enough for ordering them. However, when the uncertainty intervals are overlapping TT cannot order events, and that is why in order to ensure external consistency it has to explicitly wait-out these uncertainty intervals.

# 参考

1. PAPADIMITRIOU, C. H. "The serializability of concurrent database updates". *J. ACM 26, 4 (Oct. 1979), 631-653.*
2. MAURICE P. HERLIHY and JEANNETTE M. WING "Linearizability: A Correctness Condition for Concurrent Objects" *ACM Transactions on Programming Languages and Systems, Vol. 12, No. 3, July 1990, Pages 463-492.*
3. David K. Gifford. "Information Storage in a Decentralized Computer System". *CSL-81-8 June 1981; Revised March 1982*
4. James C. Corbett, Jeffrey Dean "Spanner: Google’s Globally-Distributed Database" *Proceedings of OSDI 2012*
5. Murat Demirbas,  Sandeep Kulkarni. "Beyond TrueTime: Using AugmentedTime for Improving Spanner" *


