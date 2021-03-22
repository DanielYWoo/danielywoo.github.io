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

其实Serializability在70年代早期已经被广泛接受，我能看到比较早的给出严谨定义的是在1979年麻省理工的Christos H. Papadimitriou在一篇论文中。[[1]](#参考)

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

这篇论文里作者还做了个类比

>Linearizability can be viewed as a special case of strict serializability where transactions are restricted to consist of a single operation applied to a single object. 


>Nevertheless, this single-operation restriction has far-reaching practical
and formal consequences, giving linearizable computations a different flavor from
their serializable counterparts. An immediate practical consequence is that concurrency control mechanisms appropriate for serializability are typically inappropriate for linearizability because they introduce unnecessary overhead and
place unnecessary restrictions on concurrency. For example, the queue implementation given below in Section 4 is much more efficient and much more
concurrent than an analogous implementation using conventional serializabilityoriented techniques such as two-phase locking or multi-version timestamping.
One important formal difference between linearizability and serializability is
that neither serializability nor strict serializability is a local property. For
example, in history Hs shown above, if we interpret A and B as transactions
instead of processes, then it is easily seen that both Hs ] p and Hs ] q are strictly
serializable but He is not. (Because A and B overlap at each object, they are
unrelated by transaction precedence in either subhistory.) Moreover, since A and
B each dequeues an item enqueued by the other, H8 is not even serializable. A
practical consequence of this observation is that implementors of objects in
serializable systems must rely on global conventions to ensure that all objects’
concurrency control mechanisms are compatible with one another. For example,
it is well known that two-phase locking is incompatible with multiversion
timestamping [46].
Another important formal difference is that serializability places more rigorous
restrictions on concurrency. Serializability is inherently a blocking property:
under certain circumstances, a transaction may be unable to complete a pending
operation without violating serializability, even if the operation is total. Such a
transaction must be rolled back and restarted, implying that additional mechanisms must be provided for that purpose. For example, consider the following
history involving two register objects: x and y, and two transactions: A and B.
x Read( ) A (History H,)
y Read( ) B
x Ok(O) A
Y Ok(O) B
x Write(l) B
y Write(l) A
Here, A and B respectively read x and y and then attempt to write new values to
y and x. It is easy to see that both pending invocations cannot be completed
without violating serializability. Although different concurrency control mechanisms would resolve this conflict in different ways, such deadlocks are not an
artifact of any particular mechanism; they are inherent to the notion of serializability itself. By contrast, we have seen that linearizability never forces processes
executing total operations to wait for one another.
Perhaps the major practical distinction between serializability and linearizability is that the two notions are appropriate for different problem domains.
Serializability is appropriate for systems such as databases in which it must be
easy for application programmers to preserve complex application-specific invariants spanning multiple objects. A general-purpose serialization protocol, such as
two-phase locking, enables programmers to reason about transactions as if they
were sequential programs (setting aside questions of deadlock or performance).
Linearizability, by contrast, is intended for applications such as multiprocessor
operating systems in which concurrency is of primary interest, and where programmers are willing to apply special-purpose synchronization protocols, and to
reason explicitly about the effects of concurrency. 

# External Consistency

In 1981, David Gifford第一次定义了External Consistency.

> The actual time order in which transactions complete defines a unique serial schedule. This serial schedule is called the external schedule. A system is said to provide external consistency if it guarantees that the schedule it will use to process a set of transactions is equivalent to its external schedule.

看到这里你会问，External Consistency如果是表示系统行为要按照事务结束时间顺序的唯一Serial Schedule，那么这个Serial Schedule又是个啥？

Gifford在论文中提到

> Serial consistency makes it appear to a transaction that that there is no other simultaneous activity in transactional storage. 

意思就是看似串行化（并不一定要串行化）咯。

假设有两个transactions:
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

但是如果T1提交之后，T2才开始执行，那么就只有Sa这个执行schedule叫做external schedule。换句话说，external consistency（external schedule）必须要是符合实际事务发生时间顺序的serial schedule。

# 参考

1. PAPADIMITRIOU, C. H. "The serializability of concurrent database updates". *J. ACM 26, 4 (Oct. 1979), 631-653.*
2. MAURICE P. HERLIHY and JEANNETTE M. WING "Linearizability: A Correctness Condition for Concurrent Objects" *ACM Transactions on Programming Languages and Systems, Vol. 12, No. 3, July 1990, Pages 463-492.*
3. David K. Gifford. "Information Storage in a Decentralized Computer System". *CSL-81-8 June 1981; Revised March 1982*


