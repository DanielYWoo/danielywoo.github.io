---
layout: post
title:  "数据库和缓存的一致性"
date:   2020-10-30 16:00:00
published: false
---

### 前言

此文为之前发布的[Cache Consistency with Database](https://danielw.cn/cache-consistency-with-database)的中文版.

Cache consistency and coherency 是计算机科学中非常困难的两个话题，本文我们只讲述一种情况，就是我们使用数据库和缓存的时候，缓存和数据库内容的一致性。比如Redis+MySQL的模式，这可能是目前最为普遍的一种设计方式。

### 概念

在我们开始之前，我们现需要重温一下各种常用的缓存模式。

#### 缓存模式

##### When Write

* Write Through: 同步写入数据库，然后缓存。这种模式下，数据写入是安全的不会丢失挥发的。但是这种模式比较慢，因为下层的数据库通常比缓存慢很多。
* Write Behind (or write back): 先写缓存，然后立即返回成功，后台异步写入数据库。这种模式写肯定非常快，对数据库的压力也比较平缓，而且异步写入的时候可以合并多次写入变成一次写入，进一步减少对数据库的压力。但是这种模式下一旦缓存服务器重启，还没来得及flush进入数据库的数据有可能丢失。RAID card 为例，为了避免服务器掉电丢失缓存中的数据，我们通常会使用BBU备用电池来确保没有落盘的数据不会因为掉电而丢失。
* Write invalidate: 类似于write-through, 先写入数据库，然后清除缓存，而不是更新缓存。这种模式部分解决了了并发写入的时候，数据库和缓存之间的不一致, 并且不需要引入分布式锁，实现比较简单。缺点就是你的hit rate会比较低，因为你总是invalidate缓存, 这经常会导致下一个read读不到缓存。这种模式在Facebook的这边论文有所提及 [Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf).

##### When Read

* Read Through: 当读不到缓存，从数据库加载并更新到缓存。这种模式通常和写的三种模式配合使用。这种模式的缺点是你需要预热数据，否则每个key第一次读取的时候都会很慢。在市场活动的峰值流量到来之前，通常你要预热缓存, 防止冷缓存击穿数据库。

##### When not Read or Write

没有读写的时候，也不能闲着，我们需要evict和refresh。

* Refresh Ahead: 可以预测热数据并自动从数据库刷新高速缓存，从不block read，通常用于小型只读数据集，例如，邮政编码列表缓存，由于它很小且是只读的，因此可以定期刷新整个缓存。对于可写的大数据集合，如果你可以预测最常读取哪些键，那么你也可以使用这个方式来预热缓存。最后一种情况是，如果缓存数据是被外部系统更新的，而且你还得不到通知，你可能只能用这个模式来更新缓存了。

在大多数情况下，我们将read-through和write-through/write-behind/write-invalidate一起配合使用。Refresh-ahead可以单独使用，也可以作为read-through的补充优化。

从缓存的维护职责来看（调用者维护或专用层维护），有两种实现模式。

* cache-facade: 缓存层是一个类库或服务委托写到数据库，而您仅与缓存层对话。然后数据库对您的应用程序是透明的。缓存层可以处理一致性和故障转移。例如，许多数据库都有自己的缓存，调用者不必关心缓存的更新和过期，这是Cache Facade的一个很好的例子。这种模式最大的优点是, 如果facade是资源本身提供的，那么他的一致性比较容易控制。您还可以编写一些进程内DAO层来读取/写入具有嵌入式缓存层的实体，从调用者的角度来看，这个很小的层也是一个Cache Facade, 比如Spring的cache实现，但是这时候缓存一致性就无法保证了。

* cache-aside: 您的应用程序需要自己来保持缓存的一致性，这意味着您的应用程序代码更加复杂，但是这提供了更大的灵活性。比如，cache aside更容易实现对象缓存，如果是cache facade模式的数据库缓存，一般来说他只能cahce rows，如果要把数据库记录作为Java POJO或者Kotlin的Data Class来缓存就很困难了。而cache aside可以把缓存和数据库完全独立出来，所以实现起来很容易. 当然，cache facade也可以缓存POJO，比如通过Spring这样的类库把POJO序列化到Redis，但是实现起来没这么简单。但是cache aside模式下通常缓存一致性比较难保持。

无论采用哪种模式，您都必须面对和解决并发一致性，而这在分布式系统中通常很困难，并且常常被遗漏。由于无论是cache facade 还是 cache aside, 我们都需要解决一致性并且实现方式相同，因此在本文中，我将只以 cache aside 模式讨论此主题.

#### Consistency

首先，我们先定义什么是缓存一致性。关于分布式系统一致性问题的基本讨论，请参考[分布式系统一致性的发展历史](https://danielw.cn/history-of-distributed-systems-1).

对于缓存一致性 一种是底层资源和缓存之间的一致性，the cache-database consistency, 还有一种是使用缓存的应用节点之间的一致性： client-view consistency.


##### Cache-database Consistency

这是缓存和资源(通常是数据库)之间的一致性。由于它们是两个独立的系统，因此更改任何数据时总会出现不一致的时间窗口。如果第一个操作成功而第二个操作失败，则会产生许多问题。对于write-through，您首先更改数据库，然后缓存和数据库会有个很小的窗口不一致。对于write-behind，您首先要更改缓存，因此数据库会有一个短暂时间和缓存不一致。(不一致的窗口大小对于write-behind模式很重要，因为不一致的时间窗口意味着在高速缓存系统发生故障时数据丢失的可能性, 而过小的窗口又无法体现cache behind的性能优势)。基本上，它们之间总是存在不一致的地方，我们所能做的就是最小化不一致的时间窗口。

通常，在非分布式系统（如MySQL中的查询缓存）中的cache facade模式更易于实现，因为对磁盘的写入和缓存都是本地的。但是MySQL的查询缓存性能不佳，有两个原因。首先，很难识别受影响的查询，因为MySQL支持复杂的查询（您可以join很多表或使用它做很多复杂的事情,比如子查询）。假设您有一个包含100行的表，并且您有100条查询来查询每行。如果更新一行，则将evict所有其他99个缓存查询，此处缓存的好处很小。另一个原因是MySQL缓存需要提供MVCC和linearizability级别的一致性，这会使cache evict更加频繁。由于这两个原因，MySQL必须选择一种粗粒度的方法来到期并evict cache。这就是为什么我们经常使用Redis作为cache aside模式来牺牲一致性以获得更好的性能。像Cassandra这样的NoSQL数据库没有这样的问题，因为它没有提供如此强的一致性保证，并且支持更简单和可预测的查询。 Cassandra具有memtable作为write-behind缓存层，因此写入速度非常快, 这是特别好的一个cache facade模式的实现。同时为了避免由于write-behind造成的数据丢失，它具有WAL和in-memory副本以确保数据安全。因此，您不需要额外的Redis缓存层(cache aside)即可与Cassandra一起使用。

##### Client-view Consistency 

It means that each client has a consistent view of the cached data. It is important for correct application behaviors in many cases. If the data is updated from version 1 to 2. It 3, … to 5, any client should see the same total order but none of them sees something like 1-2-5-3-4. This is actually sequential consistency model in a distributed system. (for more details, you can google sequential consistency or read a series of my article [History of consistency models in distributed systems](https://danielw.cn/history-of-distributed-systems-1) (Chinese version only for now, I will write an English version later).

Sometimes you don't care about the full history, only care about the latest update is visible, in that case if a client is able to see 1-2-3-4-5 but decide to drop 2-3 and get a view of 1-4-5, that's fine either.

Sequential consistency does not enforce any latency requirement, if a client sees 1-2-3-4-5 but it takes long time to see 2 after 1, it's OK. However, sometimes we want each client always sees the most recent update *immediately*, that is linearizability consistency model and it's a very strict consistency model, often difficult to implement.

In this article, we discuss how to implement sequential consistency among client views and try to minimize the latency. Now, you have the basic concept about consistency problems in cache systems, warm-up is done­'s show some consistency problems.

### Consistency Issues

#### Client/Network Failure in Write-through

The diagram below is a write-through pattern. T1 tries to update X meanwhile T2 reads X. What if T1 crashes or its network is broken before step 3? T2 will always see stale data until the cache expires. This complies with sequential consistency model, in some cases the latency is not a big problem if only the delay is acceptable.

<img src="/images/2020-02-02/client_network_failure_sample.png" width="500px">

The real problem in this case is cache eviction. If the cache eviction is based on LRU and the data is frequently read, the cache-database inconsistency time window will be large, even infinite, that means T2 will never see the new value, this does not satisfy any consistency model among the client views and will cause severe problems in your application. To avoid this, force a fixed expiration time based on the timestamp when the key is first cached (e.g. expireAfterWrite in Caffeine).


#### Concurrency in Read-through

Suppose we don't use a distributed lock to coordinate T1 and T2, X does not exist in cache yet. The diagram below shows both T1 and T2 encounter a cache miss. After step 3, if something like a JVM full gc happens in T1, the updates to database will be deferred. Meanwhile T2 updates cache and write X to the latest value 2, eventually T1 recovers from gc and writes its stale value 1 to cache. If T2 reads X again, it sees an old value and might be confused. Both sequential and linearizability consistency are not satisfied.


<img src="/images/2020-02-02/concurrency_in_read_through.png" width="500px">

Using a distributed lock can solve this but it's too expensive. A simple solution is to prevent T1 write stale data at step 7 by CAS. Most modern cache system supports CAS write (e.g. Redis lua), we can use CAS write over a version column like this:

```
WRITE X.VALUE = 1 IF X.VERSION=<the version you want to write> OR X NOT CACHED
```

With CAS, at step 7 T1 will fail and T1 is be able to query cache again to get the latest X.

A very special case is that if T1 pauses very long, long enough that the value of X written at step 6 expires, in that case T1 is still able to write the stale data to cache, but this is extremely rare to happen, because T1 has to pause very long, maybe 15 minutes which is unlikely to happen. So, this is just a possibility in theory. If you want to solve this, consider using a timestamp when writing to cache, and the cache system can reject the write if it's too old. e.g, the expiration is set to 5 minutes, if the write with a timestamp older than 5 minutes, reject and report error so the client can be aware of this and retry. However, any timestamp based solution is vulnerable to clock drift and you must have correct NTP setup.

#### Concurrency in Write-through

Suppose we don't use a distributed lock to coordinate T1 and T2, both T1 and T2 try to update X.

<img src="/images/2020-02-02/concurrency_in_write_through.png" width="500px">

After step 2, ideally T1 should update cache to 1, but if something like a JVM full gc happens in T1 meanwhile T2 updates cache and write X to the latest value 2, then T1 will write its stale value 1 to X in the cache. This is similar to the concurrency problem mentioned before, but this happens more likely, it doesn’t require two concurrent cache misses which is rare.

To solve this kind of problem without a distributed lock, you can use write-invalidate pattern with read-through. At step 4/5, we just invalidate the cache key and the next read should recreate the cached data. This way, T1/T2 both will see X as 2 in the next read, if another T3 reads X between step 4 and 5, it sees a cache miss and will try to load the cache from database, and it sees X as 2.  Now we achieve linearizability consistency level. The drawback is obvious, in a write-then-read scenario, you will see a low hit ratio.

You can also use a CAS write to ensure order. Each time you update database you retrieve the version back (you can make it by Oracle sequence, simulating sequence in MySQL, distributed incremental key, lock the row to retrieve result), then only update cache if the version you have is higher than the version in cache to prevent step 5 to happen. Unless T1 pauses for a long time and X expires, which is rare, it should work in most cases. This solution is a little complicated and use it only when you really need it.

#### Concurrency with Write-Invalidate and Read-through

In the last section we talked about how write-invalidate solves problems caused by write-through. But write-invalidate also has problems when it is used with read-through, which is very common pattern used in many systems. Suppose we don't use a distributed lock to coordinate T1 and T2, both T1 tries to read X and T2 tries to update X.

<img src="/images/2020-02-02/concurrency_in_write_invalidate_and_read_through.png" width="500px">

If T1 is overloaded and it is slow for some reason, step 5 could be deferred and write a stale value to cache.

The CAS write solution does not work with write-invalidate pattern, because once the cached key is deleted at step 4 you have nothing to compare with CAS.

Some people use something like a <b>write-deferred-invalidate</b> solution, that is, schedule the invalidation 500ms later asynchronously and return immediately after step 3. The idea is we hope we can predict how lag T1 is and make the invalidation after step 5.

<img src="/images/2020-02-02/concurrency_in_write_defer_invalidate_and_read_through.png" width="500px">

This solution also helps to hide database master/slave latency when you have a cluster of read-only slaves. If T1 updates the master database, T2 reads from a slave database instance, T2 will not see the latest change made by T1, so T2 could populate stale cache, and the stale cache will be removed by T1 after 500 ms.

This solution has many drawbacks. Fist, in case of updating an exist value in cache, the new value always becomes visible with 500ms latency.  Moreover, it depends on correct settings of the delay, which is often impossible to predict because it varies with loads, hardware change etc. I don't recommend <b>write-deferred-invalidate</b>, since predicting latency is just a gambling. If you are very confident in your system response time and you need slave database instances, you probably want this.

### Other solutions

#### Double Deletion

This pattern is a write-through variant, originates from some engineers who wants to invalidate the cache first, then write to database. It's a 3-step solution: 1) invalidate the cache. 2) write database 3) schedule a deferred cache invalidation. I don't understand why they want to invalidate the cache before writing to database, this only causes more trouble in consistency. And the 3-step solution is very expensive. Actually if you take off the first step, it's exactly <b>write-deferred-invalidate</b> solution we just talked about in the last section. I would not recommend this.

#### MySQL binlog to Cache

This does not make sense at all. This is a solution of Alibaba engineers. They have a listener to receive MySQL binlog and populate cached data in Redis or other sort of cache. This way you don't need to write cache in your application code anymore, the cache is populated automatically by the listener. And you have slave database instance lag, so you don't need deferred cache invalidation. Sounds cool, but this solution cannot handle cache in fine-grain, if you only want to cache 1% of your data, you still have to process 100% of the binlog. I would not recommend this.

### Cache Failures

Read-through doesn’t introduce any problem if the updates to cache fail, except increasing database load. If the update to cache fails with write-through, you will not be able to see latest value until another successful write or cache expires. When you combine all these cache patterns to work together, things become complicated. If you encounter cache related issues you can always contact us and we can help you solve those problems case by case.

### Conclusions

It's often impossible to implement linearizability consistency model with distributed cache and database systems considering all kinds of errors and failures. Every cache pattern has its limitation and in some cases you cannot get sequential consistency, or sometimes you get unexpected latency between cache and database. With all the solutions I showed in this article, there are always corner cases that you might encounter with high concurrency. So, there is no silver bullet for this, know the limitation and define your consistency requirement before you choose a solution.
