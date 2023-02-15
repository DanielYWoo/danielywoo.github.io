---
layout: post
title:  "Pratical Design of Health Check Probes in Kubernetes"
date:   2023-02-13 10:00:00
published: true
---

## Overview

Kubernetes relies on health check probes to monitor the status of each pod and manage the lifecycles of them. The kubelet initiates three types of probes, as detailed in the [official documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes). However, there is a lack of practical guidance on designing and implementing these probes in real world, resulting in incorrect usage by many users. For example, some Spring Boot applications only use the default `/manage/health` endpoint for both liveness and readiness, which can lead to severe problems. This article aims to provide practical guidance on how to correctly design and implement the three probes to ensure the smooth functioning of your Kubernetes environment.

>Note that there are many different methods to implement a probe, such as the command-line etc. In this article, we mainly focus on probes implemented via REST APIs.

## Liveness Probe

Liveness means your container is alive and can make some progress. For instance, your container is up and running, and can handle incoming requests or messages. It may run very slow, but it can still make some progress. Many developers make mistakes regarding the liveness probe. We will discuss some real-world scenarios below.

### Case 1: Fail on Thread Deadlock/Livelock

If there is a deadlock or livelock in your application thread pool, and you don't have a reasonable timeout to release the threads, some of the business data will not continue to be processed. The only way to solve this is to restart the container. In such case, you should fail your liveness probe and let Kubernetes restart your unhealthy container.

### Case 2: Fail on Dead Loop

Sometimes, you might have a bug and your container run into a dead loop. Similar to thread deadlock, your business logic cannot continue, and you have to restart the container.

### Case 3: Fail on Runtime Errors

If a container encounters a segment fault or OOM, all the APIs, including the liveness probe, could fail. Kubernetes will not be able to reach the liveness probe, and eventually restart your container.

### Case 4: Ignore Dependency Errors

A microservice could have many dependencies, such as a database, a Redis cluster, a RabbitMQ cluster, or another microservice. A Spring Boot application can aggregate all those dependencies into the default health check endpoint `manage/health`. If any component fails, the whole endpiont reports `DOWN` with `503`. e.g.,

```
{
    "status": "DOWN",
    "components": {
        "mysql": {
            "status": "UP", "details": {...}
        },
        "rabbitmq": {
            "status": "DOWN", "details": {...}
        },
        "redis": {
            "status": "UP", "details": {...}
        },
        ...
    }
}
```
At first glance, it may seems attractive to use this endpoint as the liveness probe. But, do not use this approach! This probe is implemented too sensitive. If, for instance, the Redis cluster is down, a restart of the container won't help at all. Additionally, a microservice may use Redis as cache, when Redis is down, it may slow down but still work. If you use this endpoint as a liveness probe, Kubernetes will restart all the containers constantly and could bring down your whole application. Even if 80% of your APIs have hard dependencies on Redis and fail, you still should not fail the liveness probe. In such scenarios, the remaining 20% APIs can still function without Redis, and your microservice should survive by implementing thread pool bulkhead with short timeouts to ensure a failed Redis cluster will not exhaust all your worker threads. Event better, you can use the circuit breaker pattern to access the Redis cluster and your work threads won't be wasted on the failed Redis. Once the Redis cluster recovers, your microservice will be healthy again.

Similiarly, if your microservice depends on other microservices, don't consider them in your liveness probe.

### Implementation Best Practices

In reality, detecting cases 1, 2, and 3 mentioned above is an incredibly difficult task. Failure to accurately identify these cases may lead to false positives, causing your container to be unnecessarily and periodically restarted, ultimately resulting in unfinished jobs and incomplete data. Careful implementation is, therefore, necessary.

It is recommended that you always report 200 in your liveness probe, unless you can accurately identify the specific problem, are confident that a restart can solve the problem, and your service can survive false positives and be restarted multiple times without introducing more problems.

Runtime errors should not be dealt with, as Kubernetes handles them appropriately in most cases. 

## Readiness Probe

The readiness probe is used to check if a container is ready to accept traffic. A Pod is considered ready when all of its containers are healthy and not overloaded, ready to serve. When any container fails its readiness probe, the whole Pod will be deemed as unready and will be temporarily removed from the Kubernetes routing rules, so no more traffic comes to this Pod, until the probe succeeds again.

> Note, any container's readiness probe failure will prevent traffic to the *whole pod* temporarily. While a liveness probe failure only affects the failed container, and only the failed container will be restarted.

There are several scenarios described below where failing the readiness probe is appropriate.

### Case 1: Starting

When the containers in a pod start begin, some containers may take a few seconds to bootstrap, while certain containers could take a minute or more to be fully functional. The pod cannot accept incoming requests until all containers are ready. In such case, you can fail the readiness probe.

Another similar scenario is that some containers may have some asynchronous initialization process, by the time the API component is ready for incoming requests, other components such as databases are still undergoing the initialization process. The readiness probe API must return `DOWN` and `503` before other components are fully initialized.

Regardless, during the startup process, it's necessary to fail the readiness probe to prevent Kubernetes from directing traffic to a pod that is not fully opertional.


### Case 2: Overloaded

In case a pod is overloaded, for example, due to the exhaustion of the thread pool, full incoming request queue, or busy event listener, we need to "cool down" the overloaded pod by taking off the traffic for a while. The readiness probe can be useful in detecting that, but it's not that easy as many people expected, sometimes it requires other mechanisms to ensure your application healthy.

For instance, when the database connection pool is exhausted in a pod, it is challenging to determine whether the database is overloaded or the pods are. Let's consider three different scenarios:

- If the database is overloaded and all connections are busy, the readiness probes will fail, and Kubernetes will take the traffic off most of the pods, which will reduce the load on the database and help it recover. This is a good scenario.
- If the database is fine but a single pod is overloaded, Kubernetes will take that pod out of the traffic and bring it back later, which is also a good scenario.
- If the database is fine but many or all pods are about to be overloaded, Horizontal Pod Autoscaler (HPA) should kick in and add more pods to mitigate the peak load. If Kubernetes reaches the `maxReplica` and more traffic are coming, the pods are about to be overloaded,  the Istio service-level rate limit should kick in. Without rate limiting, the increasing load will overload all the pods, causing Kubernetes to take off traffic from all pods, which would be a disaster.

Similarly, this kind of implementation also works with other dependencies such as Cassandra, Redis, ElasticSearch, and RabbitMQ.

> Note, If a single pod is overloaded, adding more pods will not help the overloaded pod. Actually, in most cases HPA won't add more pods at all due to the HPA algorithm does not care about unbalanced load. The algorithm is `desiredReplicas = ceil(currentMetircValue / desiredMetricValue * currentReplicas)`, the metric values are "average values" across all pods. Thus, if a single pod is overloaded while the overall average load is still good, HPA won't kick in. The HPA only helps when pods have an even load and the load on the whole service increases.

### Implementation Best Practices

When configuring a readiness probe, two factors need to be checked: 
- whether a dependency is initialized and ready to serve, such as a database connection pool.
- and whether a component is overloaded once all dependencies are up and running. For instance, if the database connection pool is exhausted.

If other errors are encountered, failing the readiness probes is unnecessary in most cases. For example, if a service pod fails to connect to a database and reports a connection reset or timeout (the connection pool is not exhausted), the problem may be due to a crashed database or network issues. Failing the readiness probe in such cases may not solve the problem of the database or network, and could potentially cause additional issues.

Differentiating between an overloaded pod and dependencies can be challenging. Luckily, in most cases, it is acceptable not to differentiate them in a readiness probe if you have rate limit and HPA.

Even if a dependency component is overloaded, failing the readiness probe may not be necessary if the component is not critical to the application. For example, if 90% of the APIs rely on a database, and only 10% depend on Redis, failing the readiness probe when Redis is overloaded could lead to traffic being removed from all pods, even though most of the APIs are still functional. Thus, it is important to consider the criticality of each dependency component before failing a readiness probe, don't be too sensitive to a non-critical dependency component.

## Startup Probe

Originally, there were two types of probes in Kubernetes: readiness and liveness. However, people have encountered issues with slow-start containers. When a container takes a long time to start, Kubernetes does the first check on the liveness probe after `initialDelaySeconds`. If the check fails, Kubernetes attempts `failureThreshold` times with an interval of `periodSeconds`. If the liveness probe still fails, Kubernetes assumes that the container is not alive and restarts it. Unfortunately, the container will likely fail again, resulting in an endless cycle of restarting.

You may want to increase `failureThreshold` and `periodSeconds` to avoid the endless restarting, but it can cause longer detection and recovery times in case of a thread deadlock. 

You may want to make the `initialDelaySeconds` longer to allow sufficient time for the container to start. However, it can be challenging to determine the appropriate delay since your application can run on various hardware. For instance, increasing `initialDelaySeconds` to 60 seconds to avoid this problem in one environment may cause unnecessary slow startup when deploying the service to a more advanced hardware that only requires 20 seconds to start. In such a scenario, Kubernetes waits for 60 seconds for the first liveness check, causing the pod to be idle for 40 seconds, and it still takes 60 seconds to serve.

To address this issue, Kubernetes introduced the startup probe in 1.16, which defers all other probes until a pod completes its startup process. For slow-starting pods, the startup probe can poll at short intervals with a high failure threshold until it is satisfied, at which point the other probes can begin.

If a container's components take a long time to be ready except for the API component, the container can simply report 200 in the liveness probe, and the startup probe is not needed. Because the API component will be ready and report 200 very soon, Kubernetes will not restart the container endlessly, it will patiently wait until all the readiness probes indicate that the containers are all "ready" then take traffic to the pod.

The startup probe can be implemented in the same way as the liveness probe. Once the startup probe confirms that the container is initialized, the liveness probe will immediately report that the container is alive, leaving no room for Kubernetes to mistakenly restart the container.

## Conclusion

The article provides practical guidance on implementing liveness and readiness probes in Kubernetes. In short:

- Avoid using the Spring Boot default health check API for Kubernetes probes.
- In most cases you can just always return 200 in the liveness probe.
- Failing readiness probes are appropriate when containers are undergoing the initialization process or overloaded.
