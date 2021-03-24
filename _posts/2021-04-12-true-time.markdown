---
layout: post
title:  "分布式系统中的时间"
date:   2015-10-28 13:11:45
published: false
---

TODO: in progress

1. Leslie Lamport 1978 paper

2. Google Spanner

3. Cassandra concurrent writes
write(x, 1), write(x, 2), write(x, 3)
R=3, W=3, N=3

cassandra resolves conflicts with client timestamp, without the client timestamp the results could be
Node1:1, Node2:2, Node3:3

With client timestamp, we don't have linearizabilty but at least the 3 nodes are consistent. Timestamp is also used for replaying from hint writes. If client clock drifts too much, you will see writes out of real-time order. If we can limit client clock drifts to 200ms, it's good enough for non-critical writes (user nickname update, reply to a forum thread, etc).

if two writes occurs at the same time with the same timestamp, if value1 > value2, then value1 wins.





Spanner的设计目标是实现支持全球跨地区超过百万节点的规模的高可用数据库。通常一份数据会被复制到3到5个数据中心，每个数据分片都会被多个Paxos状态机维持。在这套强悍的基础设施之上他还实现了两个在分布式系统中非常难的特性，其实都是和一致性相关的：

特性1: External Consistency级别的一致性。

特性2: 按照时间戳的全局一致快照读。 

他的这个能力来自于TrueTime系统。这个系统通过GPS和原子钟提供了一套非常简单的API，保证节点之间的时间误差非常小。它主要是三个接口：

| Method       | Returns                                                   |
| ------------ | --------------------------------------------------------- |
| TT.now()     | TTinterval: [earliest, latest]                            |
| TT.after(t)  | true if t has definitely passed, TT.now().earlest > t     |
| TT.before(t) | true if t has definitely not arrived, TT.now().latest < t |

TT.now()返回的是一个时间范围，因为这个API本身需要时间计算，客户端发起和接收结果也有传输延迟，因此TT.now()的返回的是当前时间的可能范围。这个范围的一半叫做ϵ，通常ϵ不超过7ms。

这是因为我们可以根据TrueTime的时间戳确定事务的顺序所以可以实现特性1。一旦所有数据都有历史和时间戳，那么即便是数据在快速变化，我们仍然可以按照时间戳来查询到一组数据某个历史的快照实现特性2。

Spanner有四种事务类型，有三种都是lock-free的，这三种其实底层实现都是一样的，都是利用了“特性2”，只是Read-Only Transaction是Spanner自己决定当前时间戳，后面两个是调用方指定时间戳。

| Operation                                | Concurrency Control |
| ---------------------------------------- | ------------------- |
| Read-Write Transaction                   | pessimistic         |
| Read-Only Transaction                    | lock-free           |
| Snapshot Read, client-provided timestamp | lock-free           |
| Snapshot Read, client-provided bound     | lock-free           |

第一种Read-Write是指包含至少一个写操作的事务。它的实现包含了 2-phase locking 和 2-phase commit. 这个过程比较复杂，我们只需要关注两个操作。

Commit Start阶段：当coordinator leader收到对事务T<sub>i</sub>提交的事件e<sub>i</sub>的时候，它会为这个事务分配一个时间戳s<sub>i</sub> = TT.now().latest

Commit Wait阶段：coordinator leader会阻止任何客户端看到刚才写入的数据一会，因为Coordinator和Client的时间存在误差，要等到 TT.now().earlest > s<sub>i</sub>之后才能对外暴露（）.

 for a write Ti assigns a commit timestamp si no less than the value of TT.now().latest, computed after e server i . Note that the participant leaders do not matter here; Section 4.2.1 describes how they are involved in the implementation of the next rule.

: The coordinator leader ensures that clients cannot see any data committed by Ti until si < TT.now().earliest.

<img src="/Users/i532165/projects/danielywoo.github.io/images/2021-03-23/spanner-read-write-tx.png" max-height="500px">



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