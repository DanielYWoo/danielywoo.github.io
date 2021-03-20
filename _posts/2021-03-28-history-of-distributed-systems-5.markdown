---
layout: post
title:  "分布式系统一致性的发展历史 (五)"
date:   2015-11-27 19:18:45
published: false
---

# Overview

之前讲的都是单一操作, 很适合KV类型的数据结构，比如缓存和大多数的NoSQL数据库。但是接下来我们要看一下多操作的情况，比如对多个对象的读写操作。与之相对应的是更像传统数据库的运行方式，比如传统数据库的transaction可以操作多个对象，然后通过MVCC实现隔离。实际上这篇文章里讲到的都是transaction类型的一致性问题。

# Serializability Consistency

其实Serializability在70年代早期已经被广泛接受，我能看到比较早的给出严谨定义的是在1979年麻省理工的Christos H. Papadimitriou在一篇论文中。[[1]](#参考)

> A sequence of atomic user updates/retrievals is called serializable essentially if its overall effect is as though the users took turns, in some order, each executing their entire transaction indivisibly.

在Herlihy和Wing讲述Linearizability的论文中也提及了Serializability的定义[[2]](#参考)

> A history is serializable if it is equivalent to one in which transactions appear to execute sequentially, i.e., without interleaving.

从这两个定义来看，其实很多人说的Serializability表示没有并行，纯串行执行事务是不严谨的。他有其他的方式来实现Serializability, 只是实际实现中串行执行是最简单实现的。



提及了Serializability和Linearizability的区别[[2]](#参考)



A (partial) precedence order can be defined on non-overlapping pairs of transactions in the obvious way. A history is strictly
serializable if the transactions’ order in the sequential history is compatible with
their precedence order. Strict serializability is ensured by some synchronization
mechanisms, such as two-phase locking [12], but not by others, such as multiversion timestamp schemes [41], or schemes that provide high levels of availability in the presence of network partitions [22].

Linearizability can be viewed as a special case of strict serializability where
transactions are restricted to consist of a single operation applied to a single
object. Nevertheless, this single-operation restriction has far-reaching practical
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

1. PAPADIMITRIOU, C. H. The serializability of concurrent database updates. J. ACM 26, 4 (Oct. 1979), 631-653.
2. MAURICE P. HERLIHY and JEANNETTE M. WING. ACM Transactions on Programming Languages and Systems, Vol. 12, No. 3, July 1990
3. David K. Gifford. Information Storage in a Decentralized Computer System. CSL-81-8 June 1981; Revised March 1982


