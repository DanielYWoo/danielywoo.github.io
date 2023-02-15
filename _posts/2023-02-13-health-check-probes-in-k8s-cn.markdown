---
layout: post
title:  "Kubernetes 健康监测探针的设计和实现指南"
date:   2023-02-13 10:00:00
published: true
---

## Overview

Kubernetes 依靠健康检查probe来监控每个pod的状态并管理它们的生命周期。 kubelet启动三种类型的probe，详见[官方文档](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#container-probes)。然而，由于基于真实场景的缺乏设计和实施这些probe的实用指南，导致许多开发人员使用不当。例如，某些 Spring Boot 应用程序仅使用默认的“/manage/health” Actuator API 来实现活性探针 (liveness probe) 和就绪探针 (readiness probe)，这可能会导致严重的问题。本文旨在提供有关如何正确设计和实施三个探针的实用指南，以确保您的 Kubernetes 环境流畅运行。

>请注意，有许多不同的方法可以实现probe，例如命令行等。在本文中，我们主要关注通过 REST API 实现的probe。

## Liveness Probe

Liveness 意味着你的容器是活跃的并且可以取得一些进展。例如，您的容器已启动并正在运行，并且可以处理传入的请求或消息。尽管它可能运行得很慢，但它仍然可以取得一些进展。接下来我们将讨论一些真实场景。

### Case 1: Fail on Thread Deadlock/Livelock

如果你的应用线程池出现了死锁或者活锁，而你又没有合理的超时释放线程，那么部分业务数据将不会继续处理。解决这个问题的唯一方法是重启容器。在这种情况下，你应该让你的 liveness probe 失败，让 Kubernetes 重启不健康的容器。

### Case 2: Fail on Dead Loop

有时，您可能会遇到错误，并且您的容器会陷入死循环。类似于线程死锁，你的业务逻辑无法继续，你不得不重启容器。

### Case 3: Fail on Runtime Errors

如果容器遇到 segment fault 或 OOM，则所有 API（包括liveness probe）都可能失败。 Kubernetes 将无法到达 liveness probe，并最终重新启动您的容器。

### Case 4: Ignore Dependency Errors

微服务可能有很多依赖项，例如数据库、Redis 集群、RabbitMQ 集群或其他微服务。 Spring Boot 应用程序可以将所有这些依赖项聚合到默认的健康检查 API `manage/health` 中。如果任何组件出现故障，整个端点将报告 `DOWN` 和 `503`。例如，

```json
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

乍一看，使用此 API 作为 liveness probe 似乎很有吸引力。但是，不要使用这种方法！这样的实现过于敏感。例如，如果 Redis 集群宕机，重新启动容器根本无济于事。此外，微服务可能使用 Redis 作为缓存，当 Redis 关闭时，它可能会变慢但仍然可以工作。如果您将此端点用作 liveness probe，Kubernetes 将不断重启所有容器，并可能导致整个应用程序崩溃。即使 80% 的 API 对 Redis 有硬依赖并且失败了，你仍然不应该让 liveness probe 失败。在这种情况下，剩余 20% 的 API 仍然可以在没有 Redis 的情况下运行，并且您的微服务应该通过实现具有短 timeout 的线程池 bulkhead 来生存，以确保失败的 Redis 集群不会耗尽您的所有工作线程。更好的是，您可以使用 circuit-breaker 模式访问 Redis 集群，并且您的工作线程不会浪费在发生故障的 Redis 上。一旦 Redis 集群恢复，您的微服务就会恢复健康。

同样，如果您的微服务依赖于其他微服务，请不要在您的 liveness probe 中考虑它们。

### Implementation Best Practices

实际上，检测前面讲的案例 1、2 和 3 是一项极其困难的任务。未能准确识别这些情况可能会导致误报，导致您的容器不必要地不断重启，最终导致未完成的作业和不完整的数据。因此，谨慎实施是必要的。

建议您始终在 liveness 探测中报告 200，除非您可以准确识别具体问题，确信重启可以解决问题，并且您的服务可以在误报中幸存下来并多次重启而不会发生更大故障。

不应处理运行时错误，因为在大多数情况下 Kubernetes 会适当地处理它们。

## Readiness Probe

Readiness probe 用于检查容器是否准备好接受流量。当 Pod 的所有容器都健康且未过载并准备好提供服务时，就认为这个 Pod 已经ready了。当任何容器未通过就绪探测时，整个 Pod 被看作没有ready，他将暂时从 Kubernetes 路由规则中删除，因此不再有流量到达此 Pod，直到 readiess probe 再次成功。

> 请注意，任何一个容器的readiness probe 失败将暂时阻止流向 *整个pod* 的流量。不同的是 liveness probe 失败只会影响失败的容器，并且只有失败的容器才会重新启动。

在下面几种情况下，readiness probe 失败是合适的。

### Case 1: Starting

当 pod 中的容器启动时，一些容器可能需要几秒钟才能启动，而某些容器可能需要一分钟或更长时间才能完全发挥作用。在所有容器都准备好之前，pod 无法接受传入的请求。这种情况下，你可以让 readiness probe 报错。

另一个类似的场景是，一些容器可能有一些异步初始化过程，当 API 组件准备好接收传入请求时，其他组件（例如数据库）仍在进行初始化过程。这时候 readiness probe 已经可以工作了，但是他需要返回 `DOWN` 和 `503` 防止流量进入。

无论如何，在启动过程中，有必要使 readiness probe 失败，以防止 Kubernetes 将流量定向到没有完全初始化完成的 pod。


### Case 2: Overloaded

如果 pod 过载，例如，由于请求线程池耗尽、请求队列已满、或消息事件处理线程池繁忙，我们需要通过暂时关闭流量来“冷却”过载的 pod。Readiness probe 在检测这些情况很有用，但它并不像许多人预期的那么容易实现，有时它需要其他机制来确保您的应用程序健康。

例如，当 Pod 中的数据库连接池耗尽时，很难确定是数据库过载还是 Pod 过载。让我们考虑三种不同的场景：

- 如果数据库过载并且所有连接都处于繁忙状态，则 readiness probe 将失败，Kubernetes 将把大部分 pod 的流量卸载下来，这将减少数据库的负载并帮助其恢复。这是一个很好的场景。
- 如果数据库没问题，但单个 pod 过载，Kubernetes 会将该 pod 从流量中取出，稍后再将其恢复，这也是一个很好的场景。
- 如果数据库正常，但许多或所有 pod 即将过载，Horizo​​ntal Pod Autoscaler (HPA) 应该启动并添加更多 pod 以减轻峰值负载。如果 Kubernetes 达到 `maxReplica` 并且流量仍然在增长，则 Istio 的 service level 的 rate limit 应该启动。如果没有 rate limit，增加的负载将使所有 pod 过载，导致 Kubernetes 停止所有 pod 的流量，那将是一场灾难。

同样，这种实现也适用于其他依赖，例如 Cassandra、Redis、ElasticSearch 和 RabbitMQ。

> 注意，如果单个 pod 过载，添加更多 pod 对过载的 pod 没有帮助。实际上，在大多数情况下，HPA 根本不会添加更多的 pod，因为 HPA 算法不关心负载是否平衡。该算法是`desiredReplicas = ceil(currentMetircValue / desiredMetricValue * currentReplicas)`，这里的 metric value 是所有 pod 的“平均值”。因此，如果单个 pod 过载而总体平均负载仍然不错的时候，HPA 不会启动。HPA 仅在许多 pod 负载均匀上升时才会发挥作用。

### Implementation Best Practices

配置 readiness probe 时，需要检查两个因素：
- 依赖项是否已初始化并准备好提供服务，例如数据库连接池。
- 以及一旦所有依赖项都启动并运行后组件是否过载。例如，如果数据库连接池耗尽。

如果遇到其他错误，在大多数情况下不需要让 readiness probe 失败。例如，如果服务 pod 无法连接到数据库并报告连接重置或超时（连接池并未耗尽），这可能是由于数据库崩溃或网络问题导致的。在这种情况下，让 readiness probe 失败可能无法解决数据库或者网络的问题，并可能导致其他问题。

区分是 pod 过载还是依赖项过载可能具有挑战性。幸运的是，在大多数情况下，如果您有 rate limit 和 HPA，在 readiness probe 中不区分它们是可以接受的。

即使依赖组件过载，如果该组件对应用程序不重要，也可能没有必要使就绪探测失败。例如，如果 90% 的 API 依赖于数据库，而只有 10% 不重要的 API 依赖于 Redis，那么当 Redis 过载时让 readiness probe 失败可能会导致流量从所有 pod 上移除，即使大多数 API 其实仍可以正常运行。因此，在 readiness probe 里要考虑每个依赖组件的关键性， 不要对非关键组件的故障太过敏感。

## Startup Probe

最初，Kubernetes 中有两种类型的 probe：liveness and readiness。然而，后来人们遇到了容器启动慢的问题。当容器需要很长时间才能启动时，Kubernetes 会在 `initialDelaySeconds` 之后对 liveness probe 进行第一次检查。如果检查失败，Kubernetes 会再尝试 `failureThreshold` 次，间隔为 `periodSeconds`。如果 liveness probe 仍然失败，Kubernetes 会认为容器不 alive 并重新启动它。不幸的是，容器大概率会再次失败，从而导致无休止的重启循环。

您可能希望增加 `failureThreshold` 和 `periodSeconds` 以避免初始化时候的无穷启动，但在线程死锁的情况下可能会导致更长的检测和恢复时间。

您可能希望延长 `initialDelaySeconds` 以允许容器有足够的时间启动。但是，由于您的应用程序可以在各种硬件上运行，因此确定适当的延迟可能具有挑战性。例如，将 `initialDelaySeconds` 增加到 60 秒以避免在一个环境中出现此问题可能会在更快的硬件时导致不必要的缓慢启动。在更好的硬件下，可能 pod 现在只要 20 秒就 ready 了，但是Kubernetes 仍然会等待 60 秒之后才进行第一次 liveness check，这样 pod 启动完了白白空闲了 40 秒才可用。

为了解决这个问题，Kubernetes 在 1.16 中引入了 startup probe，它推迟了所有其他 probe，直到 pod 完成其启动过程。对于启动缓慢的 Pod，startup probe 可以用比较高的 `failureThreshold` 和较短的`periodSeconds`进行轮询，直到满足为止，之后再开始检测其他 probe。

如果一个容器的组件需要很长时间才能准备好，但 API 组件可以很快完成初始化，那么容器可以在 liveness probe 中写死报告 200，不需要startup probe。因为如果 API 很快初始化完了，可以在 liveness probe 中报告 200 了，那么 Kubernetes 就不会无休止地重启容器，它会耐心等待，直到所有容器的 readiness probe 都报告就绪，然后才会将流量带到 pod。

startup probe 的实现方式可以与 liveness probe 相同。这样一旦 startup probe 确认容器已初始化，liveness 探针将立即报告容器处于 alive 状态，不给 Kubernetes 留下空间错误地重启容器。

## Conclusion

本文提供了有关在 Kubernetes 中实施 liveness 和 readiness probe 的实用指南。简而言之：

- 避免使用Spring Boot默认的健康监测API。
- 在大多数情况下，您可以始终在 liveness probe 中返回 200。
- 当容器正在进行初始化过程或过载时, readiness probe 是可以报错的。
