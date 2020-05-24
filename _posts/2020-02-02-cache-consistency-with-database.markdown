---
layout: post
title:  "Cache Consistency with Database"
date:   2020-02-02 16:00:00
published: true
---

### Overview

Cache consistency and coherency is one of the most difficult problems in computer science and it's a very big topic. In this article, we only talk about layered cache like Redis on top of a database, which is commonly used nowadays. But the generality exists among all cache applications.

### Concepts

Before we start, let's go through the commonly used cache patterns by how we refresh the cache.

#### Cache Patterns

##### When Write

* Write Through: Synchronously write to database then cache. This is safe because it writes to database first, but it's slower than Write-Behind. It offers better performance for write-then-read scenario than write-invalidate.
* Write Behind (or write back): write to cache first, then asynchronously write to database. This is fast for writing, and even faster if you combine multiple writes on the same key into a single write to database. But you lose data in case the process crashes before the data is flushed to database. A RAID card is a good example of this pattern, to avoid data loss you often need a battery backup unit on a RAID card to hold the data in cache but not landed to disk yet.
* Write invalidate: similar to write-through, write to database first, but then invalidate the cache. This simplifies handling consistency between cache and database in case of concurrent updates. You don't need complex synchronization, the trade-off is hit rate is lower because you always invalidate cache and the next read will always be a miss. See [Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf).

##### When Read

* Read Through: When a read misses, load it from database and save it to cache. The major problem of this pattern is that sometimes you need to warm up your cache, if you have a hot product on sale exactly at 9:00 AM on black Friday in your website, a cold cache could cause many requests pending for that product.

##### When not Read or Write

* Refresh ahead: predict hot data and automatically refresh cache from database, never blocking reads, often used for small read-only dataset, or the date is updated outside of your system and your system can’t be notified, or you can predict what keys to be read most frequently. For example, zip code list cache, you can periodically refresh the whole cache since it's small and read-only.

In most cases, we use read-through with write-through/write-behind/write-invalidate. Refresh-ahead can be used standalone, or as an optimization to predict and warm up reads for read-through.

And there are two implementations patterns by who is responsible for cache maintenance, the caller or a dedicated layer.

* Cache-façade: The cache layer is a library or service delegates write to database and you only talk to the cache layer. Then database is transparent to your application. The cache layer can handle the consistency and failover. For example, many databases have its own cache, this is a good example of cache-façade. You can also write some in-process DAO layer to read/write entities that has an embedded cache layer, from the callers' perspective this tiny layer is also a cache-facade.

* Cache-aside: Your application maintains the cache consistency that means your application code is more complicated, but this allows more flexibility. Some people say, for a cache-façade pattern, the data structure to be cached must be the same with the database, which means you can only cache database rows. With cache-aside, you can directly cache your programming native objects (for example, POJO in Java) in memory/Redis. This isn’t true, for example, your architect team can provide a library to cache POJOs, and handle POJOs with database under the hood with ORM framework automatically. People say so just because caching rows in a façade is much easier than caching POJOs.

The cache-façade and cache-aside patterns are distinguished from the caller's perspective. No matter which pattern you go, you always have to deal with concurrency and consistency that is difficult and often omitted in a distributed system. Since it has to be solved either in cache-aside or cache-façade pattern, I won’t focus on this implementation difference in this article. I’ll talk about this topic under cache-aside pattern throughout this article.

#### Consistency

Now, let's define our consistency problems. We have two kinds of consistency here, the cache-database consistency client-view consistency.

##### Cache-database Consistency

It is the consistency between cache and database. Because they’re two independent systems, there are always inconsistency time window when you change any data. If the first operation succeeds and the second one fails, it creates many problems. For Write-through, you change database first, then the cache is inconsistent. For Write-behind, you change the cache first, so the database is inconsistent. The inconsistency matters for write-behind pattern since the inconsistent time window means the probability of data loss if cache system fails. Basically, there is always inconsistency between them, all we can do is to minimize the time window of the inconsistency. 
In general, a cache-facade pattern in a non distributed system like the query cache in MySQL is easier to implement since both the write to disk and cache is local. But MySQL's query cache is not very performant for two reasons. First, it's difficult to identify affected queres because MySQL supports complex queries (you can join tables or do a lot of complex things with it, for the same reason subquery is not cached). Suppose you have a table with 100 rows, and you have 100 queries to query each of them. If one row is updated, all other 99 cached queries are evicted and the benefit of cache here is little. Another reason is that MySQL cache needs to provide MVCC and linearizability level consistency which makes cache eviction more frequent. Due to the two reasons, MySQL has to choose a coarse-grained method to expire and evict cache. That's why we often use Redis as cache-aside pattern to trade off consistency for better performance. NoSQL databases like Cassandra does not have such problem because it does not provide such strong consistency guarantee and it supports much simpler and predictable queries. Cassandra has memtable as write-behind cache layer, so writing it extremely fast. To avoid data loss due to write-behind pattern, it has write ahead log and in-memory replica to ensure data safety. So you don't need an extra Redis cache layer to work with Cassandra.

##### Client-view Consistency 

It means that each client has a consistent view of the cached data. It is important for correct application behaviors in many cases. If the data is updated from version 1 to 2. It 3, … to 5, any client should see the same total order but none of them sees something like 1-2-5-3-4. This is actually sequential consistency model in a distributed system. (for more details, you can google sequential consistency or read a series of my article [History of consistency models in distributed systems](https://danielw.cn/history-of-distributed-systems-1) (Chiense version only for now, I will write an English version later).

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

A very special case is that if T1 pauses very long, long enough that the value of X written at step 6 expires, in that case T1 is still able to write the stale data to cache, but this is extremely rare to happen, because T1 has to pause very long, maybe 15 minutes which is unlikely to happen. So, this is just a possibility in theory. If you want to solve this, consider using a timestamp when writing to cache, and the cache system can reject the write if it's too old. e.g. the expiration is set to 5 minutes, if the write with a timestamp older than 5 minutes, reject and report error so the client can be aware of this and retry. However, any timestamp based solution is vulnerable to clock drift and you must have correct NTP setup.

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

It's often impossible to implement linearizability consistency model with distributed cache and database systems considering all kinds of errors and failures. Every cache pattern has its limitation and in some cases you cannot get sequential consistency, or sometimes you get unexpected latency between cache and database. With all the solutions I showed in this article, there are always corner cases that you might encounter with high concurrency. So, there is no silver bullet for this, konw the limitation and define your consistency requirement before you choose a solution.
