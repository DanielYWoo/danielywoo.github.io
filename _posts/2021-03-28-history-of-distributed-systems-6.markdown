---
layout: post
title:  "分布式系统一致性的发展历史 (六)"
date:   2015-11-27 19:18:45
published: false
---

尽管我想按照历史顺序来介绍分布式系统历史上出现的各种模型，但是实际上分布式系统的发展不是线性的，本文作为此系列的终结篇，会介绍一下CAP和FLP。

## CAP Theorem

CRDT
lattice consistency
https://aphyr.com/posts/294-jepsen-cassandra
PN-counter
G-sets 2P-sets, OR-sets





## FLP Impossibility





TODO: in progress

 CAP缺乏严格的理论场景, 作为一个theorem他的描述太不严格. 甚至有人说CAP is CRAP. 什么是C? Consistency是指数据库的ACID里的C么? 不是, 其实C是指分布式系统的Linearizability级别的一致性. 但是这一点一直没有说清楚, 直到2012年, Lynch和Gilbert发表了一篇论文[Perspectives on the CAP Theorem]重新阐释了CAP, 才更加确切的定义了C. 文中做一个分布式web service的例子, 其中Consistency的定义依赖于服务的特性, 服务可以分为四类:

  1. trivial serivce: 不需要节点交互的服务, 比如转换摄氏度和华氏度. 讨论这个没有意义.
  2. weekly consistent service: 
  3. simple service: 比如一个分布式读写寄存器(consensus的模型), 这是C其实就是linearizaiblity
  4. complicated service: 
     其实C这里是指linarizability.


 Availability: 表示一个请求总是可以得到返回结果, 尽管可能返回不一致的结果. 但是多久返回?

 Partition-tolerance: 这没得选, 这是一个前提假设. 在一个异步网络中, 网络分区是不可避免的事实存在的. 实际上在一个异步网络之中, 就算是没有丢包, 网络是正常的, 也有可能由于一个包的延迟造成一些结点认为某个结点p1无法访问, 这也是一种网络分区的情况.

 所以, CAP更好的说法应该是, 在一个不可靠的网络中, 不可能有一个分布式读写寄存器同时具有C和A.




 这篇文章消除了很多人的困惑. 文中说明了两点. 第一, C其实就是safety, A就是liveness, P是不可避免的unreliable network(异步网络). CAP其实是告诉我们在一个不可靠的分布式系统中safety和liveness只能确保其中一个. 这其实和FLP理论不谋而合, FLP是告诉我们在异步网络中如果要保证safety(total correct)你就无法保证liveness. 无论是CAP还是FLP都是在告诉我们这个最基础的道理: 分布式系统中safety和liveness需要取舍, 只不过是1985年的FLP是从consensus问题入手来阐述这个取舍关系. consensus问题中的validity和agreement是safety的体现, termination是liveness的体现. CAP只是更加宽泛, 除了consensus问题也适用于其他类型的问题.



在我的"[分布式系统一致性的发展历史](https://danielw.cn/history-of-distributed-systems-1)"系列文章中基本上每篇文章都是讲了至少一个一致性模型的发展, 但是本篇文章里没有介绍任何新的一致性模型, 

1998年加州伯克利教授Eric Brewer发布了一篇论文Harvest, yield, and scalable tolerant systems[1], 首次提出了CAP猜想. 但是真正让CAP广为人知的是两年后PODC 2000会议上Brewer有一个主题叫做Towards Robust Distributed Systems, 在他讲ACID vs BASE的时候把CAP猜想从学术领域进入了工程领域, 引起了大家的关注和思考:



2002年, Lynch和她的学生Seth Gilbert一起发表了对于CAP的formalization和证明 [2].

首先论文里先建立了一个Atomic Data Objects的模型, 这个模型里明确指出C是指Linearizability, A是指non-failing node必须返回结果(可能达不到Linearizability, 但是会返回结果而不是无限循环), P是指网络断开为两个以上的分区, 在分布式系统中这其实不是一个Option, 这是一个事实. 另外, 模型处于异步网络中 (关于异步网络的定义请阅读 TODO link), 没有全局时钟, 消息延迟没有上限. 下面开始证明

 

定理1

It is impossible in the asynchronous network model to implement a read/write data object that guarantees the following properties

  availability

  atomic consistency

In all fair executions (including those in which messages are lost)

 

假设存在算法A在这个模型里满足CAP, 所有的节点被分区为两个非空离散集G1和G2, 发生了分区导致二者之间的消息全部丢失. 这种情况下如果G1里有一个对atomic data object v0的写更新操作, 我们称之为a1, 为了满足Availability这个写操作a1必然要成功, 然后G2如果要读这个v0, 我们称之为a2, 由于期间G1/G2之间没有通信, 如果要满足G2的Availability, a2只能返回一个旧的值, 这就不满足Consistency. 更正式的证明可以参考论文 [2], 用的一个非常简单的反证法, 你大概5分钟就可以看懂.

由此, CAP从conjecture变成了theorem.

 

但是现实世界中网络分区是非常复杂的, 有单向不通的, 还有的场景需要考虑客户端的连通性, 工程领域还是有一些误解. 12年后, 2012年5月Brewer教授又发表了一篇文章 [3] 回顾了他的一些思考, 其中我们可以看到之前PODC 2000的slides上他在ACID和BASE下写了一句"I *think* it's a spectrum"其实已经埋下了伏笔, 在这篇文章里他再次强调了CAP不是简单的三选二, 其实我们很多时候都是A和C之间的平衡, 比如在确保A的情况下最大化的确保C (AP but best-effort consistency). Eric在这篇文章中讲到: The modern CAP goal should be to maximize combinations of consistency and availability that make sense for the specific application.

 

文中也提及了Latency的问题, 原始的CAP并不是建立在Lynch定义的异步网络模型上的, 实际上就没怎么提及网络延迟. 当一个节点运行算法遇到消息延迟的时候这个节点要么返回可能是过期的数据牺牲C, 要么终止执行返回错误代码. 即便你用paxos或者其他算法在这种情况下也只是延迟决定, 一个可以terminate的算法最终你需要做一个决定, 这个决定Brewer称之为Partition Decision.

 

Partition Decision在工程实现中是非常难的, 因为异步网络模型里网络中断和超时是比较难判断的, 假设你的算法判断基于比较短的超时, 那么某些节点可能误判导致另外一些节点进入了分区状态, 造成不必要的数据迁移(比如ElasticSearch的shard rebalance)或者服务降级(比如zookeeper少数派分区失去可用性). 如果基于比较长的超时, 那么每个节点可能对分区故障不敏感, 可能会导致系统更慢的从故障中恢复. 比如elastic search有一个参数, 用来设定在其他节点发现一个节点无法通讯时, 延迟多久再rebalance shards. 如果你设定非常短, 那么一旦有网络拥堵或者某个节点过载就会立即rebalance, 然后重新可以支持写入, 又会产生不必要的频繁rebalance, 副作用就是这段时间内部分shard的写入会失败, 丢失了A.

 

同年2月, 除了Brewer这篇文章之外, Nacy Lynch和Seth Gilbert也发表了一篇论文Perspectives on the CAP Theorem [4]再次对CAP做了思考和扩展, 两位作者认为其实CAP可以更加泛化, 并提出了一些工程方案.

 

论文里对CAP扩展核心的一句话是:

The CAP Theorem is simply one example of the fundamental fact that you cannot achieve both safety

and liveness in an unreliable distributed system.

 

这里safety对应着原来CAP的C, 但是不止于CAP所说的一致性, 还包含了算法正确性, 任何不破坏invariants的行为都是safe的. 举个例子, 假设异步网络中存在算法A满足linearizability一致性, 但是在分区情况下计算的结果不按照program order执行, 所有节点都得到了一样的错误的结果, 这虽然满足一致性但是已经不满足正确性了, 这也算是失去了safety.

这里的liveness对应着A, 但是不止于CAP所说的可用性, 还包含了making progress的要求. 比如, 每次消息发送给某个节点, 这个节点都可以把状态向目标状态更进一步, 有所进展. 原来CAP的availability只是说web service收到请求一定要返回个结果, 这是一个最常见的liveness的场景, 但是不是代表了所有的liveness的场景.

unreliable distributed system对应着P, 但是不止于CAP所说的分区忍耐性. 除此之外还有Byzantine故障等.

所以, CAP是一个Lynch总结的这句话一个最常见特例, 其实之前很多人在工程实践中都会遇到这个问题但是Brewer教授做的非常了不起的事情就是第一个把他抽象为一个conjecture, 而Lynch和Seth把他规范化, 证明和扩展.

 

遗憾的是, 社区在并没有非常深入理解CAP的情况下, 把很多NoSQL会标称自己是AP的还是CP的, 但是实际上这是不太科学, 比较武断的. 这篇论文里也提及到了一些保证一致性的同时Best-effort Availability方法, 比如chubby和zookeeper通过quorum来尽可能实现可用性. 另外也提到了一些保证可用性同时best-effort consistency的方法, 比如Akamai CDN. 不过我觉得更好的例子是Cassandra的写复制因子和轻量事务. 2015年的时候著名布道师Martin Kelppmann发表了一篇文章Please stop calling databases CP or AP, 有兴趣的同学可以看一下. 我个人也是觉得CAP提出的时候虽然Brewer教授从未称之为Theorem并且表达非常模糊缺乏模型, 但是由于他比FLP易懂(也易误解), 在工程领域正好赶上NoSQL起步的那几年, 传播非常快, 误解也非常多.

 

我个人认为, FLP有着非常严谨的模型和理论支撑, 它通过一个更窄的模型(Crash stop failure model, one crashe failure at most)更加强有力的揭示了liveness和safety的关系, 而且它早在1985年就发表了, 但是它却并没有像CAP那样引起工程界的重视. 我想大概是因为工程总是滞后于科学, 当时分布式系统还处于中世纪黑暗时期, 而2000年后NoSQL萌芽的时候, 一个更加容易被工程师理解(误解)的CAP横空出世, 被工程界广为人知. 尽管CAP当时提出的时候确实缺乏规范化的模型和理论支持, 并且所表达的场景太模糊, 但是它确实对工程领域产生了很大的影响和贡献. 即使他并没有跳出更早的FLP所展现出来的liveness和safety的本质矛盾.

 

另外对于CAP和FLP尽管表达了一些共性的问题看似有一些相似, 但是具体还是不同的scope和模型, 有兴趣可以参考一篇不错的文章 History of the Impossibles - CAP and FLP[6].

 

 

 

[1] Fox and E.A. Brewer, "Harvest, Yield and Scalable Tolerant Systems," *Proc. 7th Workshop Hot Topics in Operating Systems* (HotOS 99)

 

[2] Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services - Seth Gilbert and Nancy Lynch, in ACM SIGACT News

 

[3] CAP Twelve Years Later: How the“Rules”Have Changed

 

[4] Perspectives on the CAP Theorem -

 

[5] https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html

 

[6] History of the Impossibles - CAP and FLP

