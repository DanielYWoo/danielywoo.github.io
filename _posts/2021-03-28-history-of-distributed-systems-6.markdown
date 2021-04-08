---
layout: post
title:  "分布式系统一致性的发展历史 (六)"
date:   2021-03-28 11:00:00
published: true
---

尽管我想按照历史顺序来介绍分布式系统历史上出现的各种模型，但是实际上分布式系统的发展不是线性的，原谅我无法严格按照时间来介绍所有的重要模型。我们之前已经介绍过了所有的主要一致性模型，本文作为此系列的终结篇，不再介绍一致性模型了，我来介绍一下CAP Theorem和FLP Impossibility。

## FLP Impossibility

FLP Impossibility[[6]](#参考)是分布式系统中重要的原理之一，这篇论文是黑暗中的灯塔，告诉我们你去设计一个系统的时候，你的能力边界是什么，这点极为重要，它避免你徒劳的去设计一个不可能存在的系统。

1985年Nancy Lynch团队发表的这篇论文提出了

> No Consensus protocol is totally correct in spite of one fault. (in an asynchronous system)

FLP定义的模型比较繁琐, 限于篇幅我这里不做非常细致的描述，论文中提到了三个Lemma来证明。其中Lemma 1是证明Lemma 2/3的基础，Lemma 2表达了初始的不确定性，Lemma 3表达了从不确定的configuration开始，运行过程可以保持不确定性。所谓的total correct就是同时满足agreement, validity, termination。FLP证明了在异步网络有一个故障节点的时候termination是无法满足的, 这时候只满足两个条件，所以是partial correct。我们常见的Paxos/Raft都是partial correct。只是实际工程中我们通过增加随机性来降低无法termination的可能性，因而让这些算法尽量接近total correct，但是理论上是永远无法达到的。

FLP告诉你异步网络中liveness和safety是一对冤家. total correct意味着同时要满足safety(agreement + validity)和liveness(termination), 那是做不到的。你只能严格保证liveness和safety其中的一个。有了FLP的指导，我们能够理解设计算法的时候，要有取舍。比如Paxos/Raft必须要放松liveness才能实现linearzability, 我们可以理解理论上极端情况下Paxos/Raft可能永远无法结束。另外有一点很多人没有注意到, FLP的论文提出的0-1 consensus问题成为了以后很多问题的研究模型。

因为FLP是一个收窄的模型(fail-stop模型, 消息不会丢失, 有一个节点进入decision state就算结束), 所以我觉得FLP的证明远比CAP更有杀伤力. 但是工程领域中大家反而对CAP至少有所听说（虽然很多也不太理解），但是对于更重要的FLP了解的人却非常少。

FLP有两种证明方式, FLP论文中给的是图论证明方式, 除了看原文, 在The Art Of Multiprocessor Programming也有介绍. FLP证明有点精妙，但是有点难懂。关于FLP Impossibility的详细介绍和<b>白话文</b>证明请参考我的另外一篇文章[FLP Impossibility的证明](/FLP-proof)

## CAP Theorem

加州伯克利教授Eric Brewer曾经开办了一家叫做Inktomi的创业公司，为eBay，Walmart等客户提供代理和缓存软件服务。在设计和实现这些分布式系统的时候，Eric Brewer和他的学生产生了CAP猜想。1998年Eric Brewer和他的学生Armando Fox发布了论文 “Harvest, yield, and scalable tolerant systems”[[1]](#参考), 首次提出了CAP猜想。但是真正让CAP广为人知的是两年后PODC 2000会议上Brewer有一个主题叫做Towards Robust Distributed Systems, 在他讲ACID vs BASE的时候把CAP猜想从学术领域进入了工程领域, 引起了大家的关注和思考。在这次分享中，他提出了传统单节点数据库是符合ACID的，而很多分布式系统是BASE类型的，其实我们设计一个存储系统的时候并不是这样二选一，他猜想应该是一个spectrum， 应该还有介于二者之间的选择。因此他提出了CAP猜想，C代表一致性，A是可用性，P是网络分区容忍性。那么ACID的系统其实就是选择了C和A的单节点数据库，BASE就是选择了A和P的就是不能提供strong consistency的分布式系统，而二者之间还有选择了C和P不能满足高可用的分布式系统。简单讲，CAP只能pick two。下图是当时他展示的slides。

<img src="../images/2021-03-28/CAP.jpg" max-height="500px">

但是Brewer在PODC 2000上提出的CAP猜想并没有非常严谨的模型, 他自己其实说的也不太明白。比如Consistency指的是啥意思？Linearizability还是Sequential Consistency? Partition-Tolerance是指分区的时候, 系统能怎样正常工作? 再加上有时候<b>工程领域的很多人其实并不具备完善的理论知识</b>, 导致社区里产生了一些跑偏的讨论, 甚至有人愤怒的说说CAP is CRAP。 直到2002年，MIT的Nancy Lynch和她的学生Seth Gilbert才一起发表了对于CAP的formalization和证明 [[2]](#参考)， 至此CAP conjecture变成了CAP Theorem。

定理
>It is impossible in the asynchronous network model to implement a read/write data object that guarantees the following properties
> * availability
> * atomic consistency
> 
>in all fair executions (including those in which messages are lost)

首先论文里先建立了一个Atomic Data Objects的模型, 这个模型里明确指出:
1. Consistency (atomic consistency)是指Linearizability.
2. Availability是指non-failing node必须返回结果(可能返回的结果不一定正确，但是一定会返回结果而不是无限循环或者失败), 并且不对相应时间有上限要求。读到这里你可能会问，实际工程中10年没有响应还能算是Available么？这里只是个理论模型，实际工程中会有更严格的超时限制。
3. Partition tolerance 是指网络断开为两个以上的分区的时候, 节点之间会丢失消息。除非你设计了一个单节点的系统，否则在分布式系统中这其实不是一个Option, 这是一个不可避免的事实。 另外, 模型处于异步网络中 (没有全局时钟, 消息延迟没有上限，关于异步网络的定义请阅读[分布式系统中的网络模型和故障模型](/network-failure-models))。

下面我们看一下Nancy Lynch和Seth Gilbert的证明过程.

假设存在算法A在这个模型里满足CAP, 所有的节点被分区为两个非空离散集G1和G2, 发生了分区导致二者之间的消息全部丢失. 这种情况下如果G1里有一个对atomic data object v0的写更新操作, 我们称之为a1, 为了满足Availability, <b>这个写操作a1必然要成功</b>, 然后G2如果要读这个v0, 我们称之为a2, 由于期间G1/G2之间没有通信, 如果要满足G2的Availability, a2只能返回一个旧的值, 这就不满足Consistency. 更正式的证明可以参考原论文 [[2]](#参考)，非常简单，你大概5分钟就可以看懂.

由此, CAP从conjecture变成了theorem.

值得一提的是，从节点之间角度来看，分区会导致节点之间不一致。但是从<b>外部调用</b>的角度来看，某些分区情况下仍然可以保证CAP都满足, 这就是Paxos之类的算法有优秀的实际工程用途的原因。比如对于一大一小两个分区的情况，Paxos之类的算法仍然可以让<b>外部调用方</b>继续正常工作，虽然数据在小区节点之间已经不一致，但是外部访问可以通过quorum在大区写入和得到正确的结果。但是如果你分区分成三个各占1/3个节点的区，那么为了保证一致性这三个区的内部节点谁都无法完成quorum写入，那这时候已经没有Availability了。如果你非要Availability，这种情况也一定要写成功，那就只能做个瞎写的算法破坏之前一致的数据了（过半一致变成1/3一致）。

## 12年后
尽管我们有了CAP的理论支撑，但是在现实的工程领域中网络分区是非常复杂的, 有单向不通的, 还有的场景需要考虑客户端的连通性, 工程领域经常还是有一些误解. 12年后, 2012年5月Brewer教授又发表了一篇文章 [[3]](#参考) 回顾了他的一些思考 （其中我们可以看到之前PODC 2000的slides上他在ACID和BASE下写了一句"I *think* it's a spectrum"其实已经埋下了伏笔）。 在这篇文章里他再次强调了CAP不是简单的三选二, 其实我们很多时候都是A和C之间的平衡, 比如在确保A的情况下最大化的确保C (AP but best-effort consistency). Eric Brewer在这篇文章中讲到: The modern CAP goal should be to maximize combinations of consistency and availability that make sense for the specific application.

文中也提及了Latency的问题, 不像FLP有明确的网络模型，当初CAP提出的时候并不是明确建立在异步网络模型上的, 实际上压根就没怎么提及网络延迟上限. 当一个节点运行算法遇到消息延迟的时候（比如和其他节点通信timeout）这个节点得要做决定，要么返回可能是过期的数据牺牲Consistency, 要么终止执行返回错误代码牺牲Availability. 你用paxos或者其他算法在这种情况下（ack消息延迟造成无限循环）也只是延迟决定, 一个可以terminate的算法最终你需要做一个决定, 这个决定Brewer称之为Partition Decision.

Partition Decision在工程实现中是非常难的, 因为异步网络模型里网络中断和超时是比较难判断的, 假设你的算法判断基于比较短的超时, 那么某些节点可能误判导致另外一些节点进入了分区状态, 造成不必要的数据迁移或者服务降级. 比如elastic search有一个参数, 用来设定在其他节点发现一个节点无法通讯时, 延迟多久再rebalance shards. 如果你设定非常短, 那么一旦有网络拥堵或者某个节点过载就会立即rebalance, 这段时间内集群性能会收到rebalance的影响， 如果这个故障很快就恢复了，那么这次rebalance就很不划算，还不如让他故障稍微久一点但是我可以避免一次大的rebalance。如果你遇到了频繁的网络故障，过短的设置甚至会导致频繁rebalance。 如果基于比较长的超时, 那么每个节点可能对分区故障不敏感, 可能会导致系统更慢的从故障中恢复。

同年2月, 除了Brewer这篇文章之外, Nacy Lynch和Seth Gilbert也发表了一篇论文Perspectives on the CAP Theorem [[4]](#参考)再次对CAP做了思考和扩展, 两位作者认为其实CAP可以更加泛化, 并提出了一些工程方案.

这篇论文中作者重新阐释了CAP, 更加确切的定义了C的含义. 文中做一个分布式service的例子, 其中Consistency的定义依赖于服务的特性, 服务可以分为四类:

  1. trivial serivce: 不需要节点交互的服务, 比如转换摄氏度和华氏度. 讨论这个没有意义.
  2. weekly consistent service: 就是先保证availability再保证best-effort consistency的情况。
  3. simple service: 比如一个分布式读写寄存器(consensus的模型), 这是C其实就是linearizaiblity
  4. complicated service: 实际工程中各种组合的一致性模型，不在本文讨论范围之内。

论文中只拿第三种作为讨论的基础模型。

Consistency: 其实C这里是指上面提到的第三个情况下达到atomicity(linearizability级别)的一致性.

Availability: 表示一个请求总是可以得到返回结果, 尽管可能返回不一致的结果. 但是最慢要多久返回在这篇论文中作者觉得不需要定义上限也可以描述出来问题。实际工程中这个响应时间是要有上限的。

Partition-tolerance: 这没得选, 这是一个前提. 在一个异步网络中, 网络分区是不可避免的事实存在的. 实际上在一个异步网络之中, 就算是没有丢包, 网络是正常的, 也有可能由于一个节点过载导致处理消息产生延迟造成其它一些结点认为某个结点无法访问, 这也是一种网络分区的特殊情况. 这个场景比我们想象的要多的多。 如果说在分布式系统中一定要pick two，那么P是没得选一定要pick的，作者说P其实是个statement，而不是个option。实际上分区是个比较复杂的问题，实际上系统分成三个区，四个区会怎样？论文也没有提及，我们在这个模型里基本只考虑分为两个区的最简单的情况。

所以所谓的pick two其实在分布式系统的前提下其实就是pick one，只能有A或者C，P是没得商量的。然后在此基础上，对AP系统best-effort保证C，对CP系统best-effort保证A。这个定义就清晰的多了对吧？

## CAP扩展

Nancy Lynch在对CAP证明的这篇论文里对CAP的核心做了总结和扩展，最重要的一句话是:
>The CAP Theorem is simply one example of the fundamental fact that you cannot achieve both safety and liveness in an unreliable distributed system.

这样来看，Consistency其实可以从linearizability一致性扩展到safety。一个符合linearizability一致性的算法, 如果在某些情况下（丢包或者延迟消息）计算的结果不按照program order执行, 所有节点都得到了一样的错误的结果, 这虽然满足一致性(agreement)但是已经不满足正确性(validity)了, 这也算是失去了safety.

Availability可以扩展为liveness而不止于CAP所说的可用性, 它还包含了making progress的要求. 比如, 每次消息发送给某个节点, 这个节点都可以把状态向目标状态更进一步, 有所进展. 原来CAP的availability只是说web service收到请求一定要返回个结果, 这是一个最常见的liveness的场景, 但是不是代表了所有的liveness的场景.

Partition Tolerance是声明了模型基于异步网络，有不可避免的无上限的延迟，会导致分区。这个没啥好商量的，就是个事实而已。

CAP其实是告诉我们在一个不可靠的分布式系统中safety和liveness只能确保其中一个. 这其实和FLP理论不谋而合, FLP是告诉我们在异步网络中只要有一个故障节点，如果要同时保证safety和liveness(total correct)是不可能的. 无论是CAP还是FLP都是在告诉我们这个最基础的道理: 分布式系统中safety和liveness需要取舍, 只不过是1985年的FLP是从consensus问题入手来阐述这个取舍关系. consensus问题中的validity和agreement是safety的体现, termination是liveness的体现. CAP只是更加宽泛, 除了consensus问题也适用于其他类型的问题，但是CAP提出的时候因为是工程实践产生的conjecture，它既没有证明也没有模型，导致后面很多年工程领域对它缺乏正确的理解，因而产生很多误解。

所以, CAP是一个Lynch总结的这句话一个最常见特例, 其实之前很多人在工程实践中都会遇到这个问题但是Brewer教授做的非常了不起的事情就是第一个把他抽象为一个conjecture, 而Lynch和Seth的了不起在于把他规范化, 证明为Theorum和提出扩展. 

遗憾的是, 社区在并没有非常深入理解CAP的情况下, 把很多NoSQL会标称自己是AP的还是CP的, 但是实际上这是不太科学, 比较武断的. 这篇论文里也提及到了一些保证一致性的同时Best-effort Availability方法, 比如chubby和zookeeper通过quorum来尽可能实现可用性. 另外也提到了一些保证可用性同时best-effort consistency的方法, 比如Akamai CDN. 不过我觉得更好的例子是Cassandra的写复制因子和轻量事务. 2015年的时候著名布道师Martin Kelppmann发表了一篇文章Please stop calling databases CP or AP[[5]](#参考), 有兴趣的同学可以看一下. 我个人也是觉得CAP提出的时候虽然Brewer教授从未称之为Theorem并且表达非常模糊缺乏模型, 但是由于他比FLP易懂(也易误解), 在工程领域正好赶上NoSQL起步的那几年, 传播非常快, 误解也非常多.


## 对比CAP和FLP

另外对于CAP和FLP尽管表达了一些共性的问题看似有一些相似, 但是具体还是不同的scope和模型。


不同之处：

1. 早期的CAP缺乏严格模型, 而FLP的模型严谨。
2. CAP是针对分布式存储(Brewer最早提出的时候)或者web service(Lynch证明)的特定工程场景，FLP的concensus问题则要宽泛的多，并且他的模型有不失一般性的原子寄存器和transition function，应用很广泛。
3. 故障模型范围不同：CAP的模型里failure是指网络分区，其实也就是crash-recovery。而FLP的failure是指单个node crash，也就是crash-stop，显然FLP更故障领域更小，crash-recovery包含了crash-stop。
4. agreement定义不同。CAP是指从外部访问端视角看待一致性，即便non-faulty节点之间没有agreement，通过内部过半一致让外部访问端感受一致也可以。而FLP是从参与节点的视角看一致性，所有non-faulty节点必须agree。

考虑到这些差异性, FLP和CAP其实没太大关系。非要说有什么相似之处的话，那就是FLP和CAP都蕴涵了liveness和safety的关系。

但是，FLP通过一个用途更宽泛的模型(consensus, register and transition function)和更容易触发的条件(连crash-stop failure, one faulty process都能触发，更不用说crash-recover failure或者Byzantine了，更不用说多个faulty process了)更加强有力的揭示了liveness和safety的关系。

FLP早在1985年就发表了, 但是它却并没有像CAP那样引起工程界的重视. 我想大概是因为工程总是滞后于理论科学, 当时分布式系统还处于中世纪黑暗时期, 而2000年后NoSQL萌芽的时候, 一个更加容易被工程师理解(误解)的CAP横空出世, 被工程界广为人知. 尽管CAP当时提出的时候确实缺乏规范化的模型和理论支持, 并且所表达的场景太模糊, 但是它确实比FLP更加直观，符合我们的直觉，FLP理解起来要复杂得多。尽管CAP所表达的liveness和safety的本质矛盾并非开创性思维，其实已经在更早的FLP等前辈的论文中有所体现, 但是 CAP的直观性和易理解对当时工程领域产生了很大的贡献。

## 系列总结

至此，分布式系统的一致性系列结束。我常常觉得伸缩性不是个很难的问题，伸缩的同时保持一致性才是真正困难的问题，这是我写这个系列的原因之一。这个系列并没有提出非常多的实际工程解决方案，但是理解前辈们提出的理论基础，可以让我们在实际工程中看的更高，走的更远。如果你看完了本系列，有这样的感受，那我会为你感到非常开心。

## 参考
1. Fox and E.A. Brewer, "Harvest, Yield and Scalable Tolerant Systems," *Proc. 7th Workshop Hot Topics in Operating Systems* (HotOS 99)
2. Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services - Seth Gilbert and Nancy Lynch, in ACM SIGACT News
3. E. Brewer, "CAP twelve years later: How the "rules" have changed," *in Computer, vol. 45, no. 2, pp. 23-29, Feb. 2012, doi: 10.1109/MC.2012.37.*
4. S. Gilbert and N. Lynch, "Perspectives on the CAP Theorem," *in Computer, vol. 45, no. 2, pp. 30-36, Feb. 2012, doi: 10.1109/MC.2011.389.*
5. [Martin Kleppmann - Please Stop Calling Databases CP or AP](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)
6. Michael J. Fischer, Nancy A. Lynch, Michael S. Paterson. "Impossibility of distributed consensus with one faulty process" ****Journal of the ACM April 1985 https://doi.org/10.1145/3149.214121*


## 系列文章目录

1. [Lamport Clock, Linearizability and Sequential Consistency](/history-of-distributed-systems-1)
2. [Two Generals Paradox, 2PC and 3PC, FLP and Paxos](/history-of-distributed-systems-2)
3. [PRAM, Causal Consistency, Weak Consistency](/history-of-distributed-systems-3)
4. [Eventual Consistency](/history-of-distributed-systems-4)
5. [Serializability Consistency, External Consistency](/history-of-distributed-systems-5)
6. [CAP, FLP](/history-of-distributed-systems-6)


