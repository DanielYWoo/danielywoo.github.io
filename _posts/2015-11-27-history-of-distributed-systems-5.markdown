---
layout: post
title:  "分布式系统一致性的发展历史 (五)"
date:   2015-11-27 19:18:45
published: false
---

# Quiescent Consistency, 1991 – 2010

Quiescent Consistency是最近才出现的一种一致性模型, 之前了解他的人并不多.  最早是1994年James Aspnes, Maurice Herlihy和Nir Shavit在1991年提出了quiescence的概念, 但是在1994年才正式发表:[Counting Networks, Journal of the Association for Computing Machinery, Vol 41, No 5, pp 1020-1048], 文中指出在counting networks中当输入量和输出平衡量的时候, 这是一个相对静态稳定的状态, 叫做quiescence. (这里的network不是特指计算机网络, 这篇论文的后两位作者也是The Art of Multiprocessor Programming的作者).

用counting networks实现一个全局计数器或者关系数据库的主键生成, 多个并发的输入进入时我们必须要保证输出的唯一性, 但是顺序不是那么重要. 再比如一个pool, 他必须有一个任务队列, 这个任务队列其实不需要是严格的Linearizable级别的有序, 只要它能满足这两个条件就可以正常工作:  enqueue总是成功, dequeue只要非空就成功. 至于顺序这里不重要.

在1995年Shavit和Touitou的一篇论文中引用了这个概念但是称之为ε-linearizability:

>    a variant of linearizability that captures the notion of almostness by allowing a certain fraction of concurrent operations to be out-of-order.

引用于[Elimination Trees and the Construction of Pools and Stacks, 1995]

限于篇幅我们不会详细介绍balancing networks, sorting networks, 我们直接用”almost stack”

一旦输入暂停一下, 输入和输出平衡之后, 进入的这个quiescence状态可以被看做一个隔断, 后面再产生的任何输出一定大于这个隔断之前的所有输出. 再比如一个缓存服务, 如果给定的key的value存在, 但是缓存服务返回不存在, 稍后的请求才返回缓存的数据, 如果这种情况有一个上限, 那么只是影响部分请求的性能, 但是如果这种情况可以消除并发竞争, 并且稍后当并发很大的时候带来的性能提升是可以抵消少部分假的cache miss的. 只要我们有一个不确定程度的上限, 我们就可以定义个比Linearizabiltiy更松的一致性模型.

Quiescence你可以把它想象为一个中场休息, 一个memory fence, 一个隔板, 不允许跨越隔板的reorder, 但是两块隔板之内的事件顺序可以随便排列. bingo, 你想到了什么? 增加并发了对吧?

其实意思就是说把Linearizability放松, 允许某些有重叠的操作随意顺序, 这已经是Quiescent Consistency的不严格定义了. 后来以色列科学家Yehuda Afek在2010年介绍Quasi Consistency的时候把这Quiescent Consistency的严格定义为:

>    Quiescent consistency provides high-performance at the expense of weaker constraints satisfied by the system. This property has two conditions:
    1. Operations should appear in some sequential order (legal for each object). 2. Operations whose occurrence is separated by a quiescent state should appear in the order of their occurrence. An object is in a quiescent state if currently
    there is no pending or executing operation on that object.

引用于[Quasi Linearizability – Relaxed Consistency For Improved Concurrency]

一个线程池, 最核心的部分其实是它的任务队列.

举个例子: TODO: confirm with Michal

A: — [add x] ————[remove x]–

B: ——– [add y] –[remove y]——-

This is sequential consistency and can be reordered into: add x, remove x, add y, remove y, or add x, add y, remove x, remove y.

This is also quiescent consistency but can only be reordered into: add x, add y remove x, remove y.

Actually this is also linearizable.


This is to show quiescence consistency is stronger than sequential consistency:

A: — [add x] —————————–[remove x]–

B: ————— [add y] –[remove y]—————–

 

This is sequential consistency, B sees memory latency from A. But this is not quiescently consistent because the 4 operations are separated by quiescence and cannot be reordered, the only order is : add x, add y, remove y, remove x which violates the object spec – FIFO.

This is not linearizable.

TODO: http://stackoverflow.com/questions/19209274/example-of-execution-which-is-sequentially-consistent-but-not-quiescently-consis

gossip, SWIM


2010年的时候是不是你已经大学毕业了? 我们处于一个高速发展必须不断学习的行业是不是很酷?



