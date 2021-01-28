---
layout: post
title:  "Messaging Reliability and Ordering"
date:   2020-11-24 13:11:45
published: false
---

## Overview

In a distributed system, message reliability is the basis of data consistency. On top of that, one of the major aspect of consistency is the message ordering. e.g, your deposit account changes from $0 to $1000, then $500, the downstream message consumers must be able to identify the event in the transaction order. In this article, we will briefly cover the message reliability, then define what ordering means, then explain what kind of orderings we need. and how to implement them.

## Reliability

Reliability means the message is delivered at least once, can survive some producer/consumer/broker crashes.

At-least-once is the foundation to build exactly-once semantics in messaging, to do that, the producer, consumer and the broker must be able to handle duplicated messages or process them idempotently, that is equivalent to exactly-once.

But, things could go wrong at producer, broker and consumer. Let's go through them one by one and see how to implement reliability correctly.

### Producer Reliability

#### Messages and Database Transaction
If the producer sends message in the database transaction, in case rollback the message could be sent without the actual data updated in database. If the producer sends message after database transaction, the message could fail. 

Below shows sending a message within database transaction. If the database is rolled back at step 5, the message is still sent; if the messaging broker ACK is timed out at step 4 (some temporary network issue), we have to roll back the whole transaction.

![send_in_tx](/Users/i532165/projects/danielywoo.github.io/images/2020-11-24/send_in_tx.png)

Below shows sending a message after transaction. If the sending at step 4/5 fails, the downstream system will never receive the message.

![send_out_tx](/Users/i532165/projects/danielywoo.github.io/images/2020-11-24/send_out_tx.png)

To solve this problem, ebay first published a paper about the outbox pattern and now it has been widely used.

In this pattern, we have an outbox table to hold messages to be sent. The message is committed along with the database transaction, this ensures the message is persisted with business data atomically. Then an async job scan the pending messages and publish them to the broker, if succeeded, update the message as "sent" and archive it later. There are alternatives, like adding a "sent" flag into your existing business tables.

Below is the process of outbox pattern. (step 3/4 is an optional optimization to improve latency)

![outbox_pattern](/Users/i532165/projects/danielywoo.github.io/images/2020-11-24/outbox_pattern.png)

When implementing the outbox pattern, developers often misunderstand "sent" at step 3 and 7, and still lose messages in some scenarios. The major reason is producer and broker normally use local buffers, we will talk about this in the next two sections: Producer Buffer and Broker Buffer.

Because the job has schedule interval, the message will be sent in batch in intervals, that could cause high latency. So, we can also optimize a little bit to lower the latency as the diagram below, we immediately trigger the job at step 3. But, this optimization is not applicable to strict total ordering, we will discuss it later in this article.

![outbox_low_latency](/Users/i532165/projects/danielywoo.github.io/images/2020-11-24/outbox_low_latency.png)

Note, when implementing the outbox pattern, one of the difficulty is to have the service instances coordinate to run the job to find pending or failed messages and publish them. This is often done by the help of zookeeper or etcd, to elect a leader for it. Or maybe you can have multiple outbox tables and hence a leader for each outobx to avoid contention. But this will break message ordering and we will talk about this later.

Another problem is when you have the low latency optimization at step 3 in the diagram above. The async sending triggers the job for that message without scanning database to make it fast. But you need to avoid step 3 and step 4 occurs at the same time for the same message. To avoid this, using some sort of external locking is too expensive, we can make the job just scan older pending messages to avoid sending duplicated messages.

#### Producer Buffer

Most MQ framework has in-memory buffer or TCP socket buffer at the producer side to improve batch or pipeline performance. But the buffer is volatile, in case of broken connection or process crash, the messages in buffer could be lost.

RabbitMQ API does not have a local in-memory buffer, it uses the socket buffer solely. When the TCP connection to RabbitMQ broker is broken, producer socket buffer is gone and you lose messages. RabbitMQ has connection auto recovery, but it does not help, it also drops any data in the TCP buffer. Even you have perfect network, when the producer crashes, all messages in the socket buffer is gone.

kafka producer API is async, it has a memory buffer. Kafka also has socket buffer which can be tuned by socket.send.buffer.bytes (SO_SNDBUF). In case of a network failure, messages in the socket buffer could be lost; in case of a producer crash, all messages in the two buffers are gone.

Due to the volatility of buffers, a message is deems as sent only when the ACK is received. Both RabbitMQ and Kafka provides callbacks(ACK). 

You probably need to limit the buffer size ("buffer.memory") or shorten the wait time ("linger.ms") to avoid your producer runs out of memory, when you generate events too fast or you have too many busy producers in one JVM process. Of course, if you want best reliability and you flush or acknowledge every single message, your producer buffer probably will never be full.

#### Broker Buffer

Even we have ACK between broker and producer, it's still possible to lose message at the broker side. Brokers like RabbitMQ or Kafka have in-memory buffers and periodically call fsync() jjust like database. When the message is in buffer but not persistented to disk, a node crash will cause message lost.

#### Producer-Broker ACK in RabbitMQ
RabbitMQ uses the term "Publisher Confirm", From the official doc: 

Once a channel is in confirm mode, both the broker and the client count messages (counting starts at 1 on the first confirm.select). The broker then confirms messages as it handles them by sending a basic.ack on the same channel. The delivery-tag field contains the sequence number of the confirmed message

When programing in Java, it looks like this
```
channel.confirmSelect(); // enable confirm
channel.basicPublish(exchange, queue, properties, body);

// option 1, blocking
channel.waitForConfirmsOrDie(5000); 

// option 2, non-blocking
channel.addConfirmListener((sequenceNumber, multiple) -> {
    // code when message is confirmed
}, (sequenceNumber, multiple) -> {
    // code when message is nack-ed
});
```

At the broker side, the ACK message is returned only after the message is persisted to disk. If you have mirror queues, that means all mirrored queues have this message persisted.

AMQP protocol provides transaction to ensure message published reliably but I strongly suggest not using it because it will decrease the throughput 250x.

And I strongly suggest setting "mandatory=true" for reliable messages. Without this option, if a message is not routable, it will be discarded silently. Enabling it will cause a "basic.return" (negative confirm) to acknowledge the producer.

#### Producer-Broker ACK in Kafka
Kafka uses the term "callback".   Here I just show you the API for callback.
```
Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback);
```
In Kafka Java client, send() accepts a callback and returns a Future, when send() returns it does not mean it is "sent", it means it's in buffer now. Then Sender.run() loops to check the responses by NetworkClient.poll(). If the any response is completed (the message has left in-memorry buffer or socket buffer, ACK polled) , the it executes your callback in the Sender thread. At this moment, your message is "sent".

You can also use flush() in Kafka client, it will drain the memory buffer and socket buffer for all topics, and wait for all batches to complete with ACK from brokers, and all callbacks are executed, then it returns. flush() and callback share the same code path but flush() is synchronous and affect all topics, callback is per message and asynchronous, so use callback whenever possible.

#### Producer-Broker Flow Control
AMQP has built-in flow control and back-pressure capability which makes your code quite simple on handling flow control. So, RabbitMQ will reduce the speed of connections which are publishing too quickly for queues to keep up. You don't write any code for this.

Kafka on the other hand does not need flow control at the producer side, because  you can scale the cluster by adding more partitions and nodes, it's rare to see Kafka as a bottleneck in real world.

### Broker Reliability

#### Durability with Cache

At the broker side, each node has three tiers of cache. OS cache, RAID cache and disk cache. 

The OS page cache can be controlled by fsync(), broker calls fsync() periodically, or when ACK'ed (RabbitMQ confirm, or Kafka ACK). Some brokers like Apache Qpid has O_DIRECT option on fopen() to disable the page cache. But this is only in case you have RAID cache, otherwise your system will be extremely slow although you have data safety.

The RAID cache is wired on the RAID card, it cannot be controlled by the operating system. If you don't have BBU you probably need to set the RAID cache policy to write-through, to improve write performance you'd better have BBU and set the cache policy to write-back (write-behind). 

The disk cache is inside the disk, can be disabled from the RAID card. Some drives have non-volatile disk cache, in that case you can enable it.


With safe cache policy configured, it's still possible that your server crashes and . So message brokers like RabbitMQ and Kafka have cluster support. RabbitMQ has mirrored queues to span across the cluster, Kafka has partitions across the cluster.

#### Durability with Replication

RabbitMQ has a concept of mirrored queues for replication. Each mirrored queue consists of one master and one or more mirrors.  All operations for a given queue are first applied on the queue's master node and then propagated to mirrors. This involves enqueueing publishes, delivering messages to consumers etc. Producer ACK (confirm) is sent back from the leader node after all nodes have the message persisted, that's a very strong guarantee although very slow, even a node crashes and never come back, you still have exactly the same persisted messages.

By default if a queue's master node fails, the oldest mirror will be promoted to be the new master. In some circumstances this mirror can be [unsynchronised](https://www.rabbitmq.com/ha.html#unsynchronised-mirrors), which will cause data loss. If you always use confirm (producer ack) and mirrored queues, since we receive confirm (ack) only after messages are persisted on all nodes, we don't need to worry about the data sychronization across nodes. For mirrored queues without publisher confirm, you probably encouter the unsynchronized queues. In that case, if you want consistency, you need to set ha-promote-on-failure to when-synced, and try to recover the failed node manually. If you want availability, you can set it to always, then try to re-send the messages could be lost.

Kafka's partitions replication is similar to mirror queues, but the ACK modes are much more flexible, the message is ACK'ed and persisted in 3 modes:

acks=0: the producer will not wait for any acknowledgment from the server at all. The record will be immediately added to the socket buffer and considered sent.

acks=1: The leader will write the record to its local log but will respond without awaiting full acknowledgement from all followers. In this case should the leader fail immediately after acknowledging the record but before the followers have replicated it then the record will be lost.

acks=all: This means the leader will wait for the full set of in-sync replicas to acknowledge the record. This guarantees that the record will not be lost as long as at least one in-sync replica remains alive. This is the strongest available guarantee.

So, for reliable messaging you probably need acks=all in case a server crashes and cannot get back timely.

Setting acks=all is really slow, if you want better performance, you can set acks=1, then disable "unclean election". Then in case of a leader crash, if there is no in-sync replica the partition will be unavailable until the leader recovers. (see unclean.leader.election.enable). It's a trade off between availability and performance. Actually, I would like to choose both availability and performance, with best-effort consistency. That is, set acks=1 and enable "unclean election". if we have property monitoring (e.g. UncleanLeaderElectionsPerSec in Prometheus) we can detect unclean election, then in case of "unclean election" we can revert the status of the recent messages in the outbox to "PENDING" and send them again to avoid messages lost. But this will break the ordering guarantee in each partition. We will talk about order later in this article.

### Consumer Reliability
At the consumer side, we need the message processed with the database transaction atomically, in another word, if the message ACK fails, don't commit transaction, if the message is ACK'ed, we must execute the business code and commit the transaction. 

If we ACK the message before the transaction, we could consume the message without triggering the business code to modify database, obviously this is unacceptable.

![consumer_before_tx](/Users/i532165/projects/danielywoo.github.io/images/2020-11-24/consumer_before_tx.png)

If we ACK the message in the transaction, if we fail to commit the transaction at step 5 in the diagram below, since the message is ACK'ed, we will never have another chance to consume it, the data in database will never be updated.

![consumer_in_tx](/Users/i532165/projects/danielywoo.github.io/images/2020-11-24/consumer_in_tx.png)

The better way to handle this is to ACK the message after transaction. If the transaction is committed but message is not ACK'ed, we will consume it later again. If only your message processing is idempotent, you achieve at-least-once. Or you have capability to deduplicated the message, you achieve exactly-once. See the diagram below:

![consumer_after_tx](/Users/i532165/projects/danielywoo.github.io/images/2020-11-24/consumer_after_tx.png)

#### Consumer ACK mode

In AMQP, there is an auto ack mode, you must disable it and manually ACK the message. Spring has a "auto ack" feature, but it is totally different from the concept of "auto ack" in AMQP. For more details please refer to Spring doc [AcknowledgeMode](https://docs.spring.io/spring-amqp/api/org/springframework/amqp/core/AcknowledgeMode.html).

In Kafka, you can commit synchronously or asynchronously.

```
void commitSync();
void commitAsync();
void commitAsync(OffsetCommitCallback callback);
```

It's always recommended to use async, because consumer ACK is not very crucial. If consumer ACK is lost, you will consume that message again. If only you can deduplicate it or your processing is idempotent, it's just fine.

A speical case is when you have to consume some messages, process them and then publish some of the results back to Kafka, you can use Kafka transaction. For more details, refer to [Kafka Transaction](https://www.confluent.io/blog/transactions-apache-kafka/).

#### Consumer Buffer
For RabbitMQ, pre-fetch size is equivalent to the buffer size. AMQP has built-in flow control, if in-flight messages reach the pre-fetch size, the consumer stops consuming messages. You don't need to write any code for this, AMQP rocks, right?

For Kafka, because it's designed as batch-pull pattern, there is no built-in flow control, we need to limit the consumer buffer size and stop pulling if the buffer is full on our own, otherwise we will encounter OOM, then your consumer will be kicked out of the consumer group and re-join later. Sometimes the re-join and re-balance process could take 5 minutes, that is a severe problem, right? Even worse, stopping pulling also could cause consumer kicked out from the consumer group. Too bad!

Without bulit-in flow control, you need to write some sort of flow control on your own. e,g. when the buffer is almost full, you need to dynamically increase the pull interval, increase max.poll.internval.ms, or decrease max.poll.records from the consumer side. If too difficult to implement, just use a very smaller consumer buffer.


## Message Ordering

Many people talk about message ordering but often different people have different understanding of ordering.



explain two concepts
broker delivery order
business event order

explain 3 perspectives
from producer perspective
  sync
  cross service async HTTP/RPC, Messaging local buffer

from broker perspective
  rabbmitQ vs kafka partition

from consumer perspective
  rabbmitMQ number of consumers, prefetch size, ack mode
  kafka per partition order, single consumer guanrantee
  consumer timeout handling, suicide

explain consistency levels: 
external or linearizability
sequential
causal

real case (just business event order)
1. sequential, linearizability, external, not needed
2. causal, diagram here, probability
3. almost in order

todo: perf test, flush with every send, ack with every send, compared with RabbitMQ publisher confirm.