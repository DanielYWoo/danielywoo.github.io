---
layout: post
title:  "HPA with Custom/External Metrics in Kubernetes"
subtitle: "Step by step PoC from scratch with Docker-desktop"
date:   2025-09-30 16:00:00
published: true
---

# Overview

Setting up even a minimal Kubernetes environment with monitoring, collecting custom and external metrics, and experimenting with Horizontal Pod Autoscaling (HPA) is not trivial.
This article walks you through the process end to end using a lightweight local setup so you can understand how everything fits together—from core concepts to advanced usage.

If you're new to this: think of what follows as building a tiny (but realistic) “metrics → decisions → scaling” pipeline. You will:
1. Learn how time‑series data is structured (so later PromQL doesn’t look like wizardry).
2. See how Prometheus stores and queries your application metrics.
3. Expose a custom metric and watch Kubernetes scale on it.
4. Simulate an “external pressure” metric (a queue) and scale based on that too.
5. Read the raw metric APIs the HPA itself uses—no black boxes.

The goal: after finishing, you should be able to explain (with confidence) how a number scraped from an HTTP endpoint eventually causes new pods to appear.

Please download the source code from [here](https://github.com/DanielYWoo/hpa-demo) before you get stareted.

# Concepts

Take the time to internalize these concepts; shallow understanding of metrics is a common root cause of failed monitoring systems and weak SLOs.

## TSDB data model

Why do we start here? Because everything else (PromQL queries, adapter rules, HPA math) assumes you’re comfortable with how a time series is identified. If you know how a metric name + tags becomes a unique stream of numbers over time, you’re 70% of the way to sanity.

### Fundamental Concepts:
This is an example from InfluxDB's official documentation.
```
Series      Measurement    Tag set                               Field key
series 1    census         location = 1, scientist = langstroth  butterflies
series 2    census         location = 2, scientist = langstroth  butterflies
series 3    census         location = 1, scientist = perpetua    butterflies
series 4    census         location = 2, scientist = perpetua    butterflies
series 5    census         location = 1, scientist = langstroth  honeybees
series 6    census         location = 2, scientist = langstroth  honeybees
series 7    census         location = 1, scientist = perpetua    honeybees
series 8    census         location = 2, scientist = perpetua    honeybees
```

- The `measurement` describes what is being measured (e.g. `census`). It can be filtered by selecting a `tag set`.
- A `tag set` is a combination of tag key/value pairs representing a specific dimension slice.
- The `field key` is the subject whose numeric value you store (e.g. `butterflies`). A `field value` is the measurement of that field key. In many practical cases you only track one field per measurement so you can ignore it sometimes.

### Point and Series:
- A `series` is the `measurement` on a `field key` at a `tag set` over time. (also called Range Vector)
- A `point` is the `field set` (all field key/value pairs) at a given time and tag set. (also called Instance Vector)

You can think of `measurement` as the table name, `tag set` as a filter to get the data into the table, and
`series` and `point` as rows and columns of a table. That analogy may help you better understand them.

Beginner mental model cheat‑sheet:
- Need “a single evolving value”? You’re thinking “series”.
- Need “a snapshot of multiple related values at one instant”? You’re thinking “point”.
- Need “subset by city, host, user, region”? You’re choosing a tag set.

e.g., the "table" of the measurement "census" looks like
```
Given Tag Set = "location = 1, scientist = langstroth"
Timestamp       butterflies          honeybees           other fields
t1              12                   23                  ...
t2              14                   21                  ...
t3              17                   45                  ...
...             ...                  ...                 ...
```

e.g., a series of the measurement "census", vertical row-like data:
```
Given Tag Set = "location = 1, scientist = langstroth", Field Key = "butterflies"
Timestamp       Census
t1              12
t2              14
t3              17
```

e.g., a point of the measurement "census", horizontal column-like data:
```
Given Timestamp = t1, Tag Set = "location = 1, scientist = langstroth"
butterflies    honeybees   other field keys ...
12             23          other field values ...
```

## Prometheus Concepts

Prometheus glues together: scraping (pull model), storage (embedded TSDB), and a query language (PromQL) that lets you reshape streams of numbers. The only trick: high cardinality (too many label combinations) explodes storage and burns CPU. Neat power—handle with care.

### Fundamental Concepts from TSDB

Prometheus has its own internal TSDB and narrows the use cases to only monitoring, so some terms are renamed and have narrower meanings:
- A `measurement` in a TSDB is narrowed to the concept `metric` in Prometheus, and only 4 types of measurements are supported in Prometheus. We will talk about them later.
- A `tag` key/value pair in a TSDB is renamed to a `label` key/value pair in Prometheus. (This is probably an unnecessary renaming, e.g., tags are still called tags in OpenTSDB. In TDEngine, both terms are used interchangeably.)

Labels enable Prometheus's dimensional data model: any given combination of labels (that is a tag set in a TSDB) for a metric name identifies a particular dimensional instantiation of that metric over time; that's a series in Prometheus. This is basically the same concept from a TSDB. For example: the HTTP throughput (measurement/metric) on `verb=POST, URL=/api/tracks` (tag set/labels)

- Adding a new label value will create some new combinations of labels (tag sets), hence some new series will be created.
- Deleting a label value will delete some existing combinations of labels (tag sets), hence some existing series will be deleted.
- Adding a new label like `hostname`, or deleting one will delete all series and create a whole new set of series.
- The PromQL query language allows filtering and aggregation based on labels (tag sets).

Given a metric name and a set of labels, time series are frequently identified using the OpenTSDB notation: `MetricName{Label1=someValue,Label2=someValue, ...}`,
for example, `http_requests_total{method="POST", url="/jobApplication/*"}`

### Metric Types

If you only remember one thing: choose the simplest type that answers your question. Counters for “how many”, gauges for “what’s the current value”, histograms for latency distributions, summaries only if you really know why.

Prometheus as a monitoring tool offers 4 metric types (that's why we don't call them measurements, which are more general).
>Currently it's only conceptual; the Prometheus server does not yet make use of the type information and flattens all data into untyped time series. This may change in the future.

#### Gauge
A gauge represents a single numerical value that can go up or down (e.g. current CPU load).

Good for: in‑flight requests, queue depth, temperature, cache size. Bad for: values you actually want a rate over (that’s a counter + rate()).

#### Counter
A counter is a cumulative metric that represents a single monotonically increasing counter whose value can only increase or be reset to zero on restart.
For example, you can use a counter to represent the number of requests served. In most cases, we convert them into rates which makes more sense (e.g., the
number of requests served will be turned into TPS with PromQL and exposed to K8S metrics API. We will talk about that later.

Rule of thumb: expose it as a counter; convert to “per second” later with `rate()` or `irate()`. Don’t pre‑compute “requests_per_second” in your code—you lose flexibility.

#### Histogram
A histogram samples observations (like response latency) and counts them in configurable buckets. It provides a distribution over buckets so you can calculate percentiles (e.g., P95 latencies).

When choosing buckets: too few → coarse, misleading percentiles; too many → resource waste. Start with an exponential progression (e.g., 5ms,10,20,40,80,160…).

A histogram is not a single metric; with a base metric name of `<basename>`, a histogram exposes several metrics:
- cumulative counters for the observation buckets, exposed as `<basename>_bucket{le="<upper inclusive bound>"}` as multiple time series during a scrape.
- the total sum of all observed values, such as the total response time, exposed as a counter metric `<basename>_sum`
- the count of events that have been observed, exposed as a counter metric `<basename>_count` (short version of `<basename>_bucket{le="+Inf"}`)

> Note, in most cases you don't have negative observations so the `<basename>_sum` increases monotonically. There are some complex workarounds but they are beyond this article.

e.g., the `<basename>_bucket` has 5 buckets from 20ms to 100ms, hence 5 time series.
```
     ^
100ms |   98        +97       +91
80ms  |   66        +71       +68
60ms  |   54        +55       +59
40ms  |   34        +35       +38
20ms  |   23        +21       +22
      ------------------------------>
          t1       t2       t3

```
Besides the quantiles/percentiles, you can also use `<basename>_sum` and `<basename>_count` to calculate the average metric value time series. e.g.,
 `rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])`


#### Summary
Similar to a histogram, more accurate but slow. There are two important limits that make it less useful. Given the example below,
```
http_request_duration_seconds_sum{api="add_product" instance="host1.domain.com"} 8953.332
http_request_duration_seconds_count{api="add_product" instance="host1.domain.com"} 27892
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0"}
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0.5"} 0.232227334
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0.95"} 1.528948804
http_request_duration_seconds{api="add_product" instance="host1.domain.com" quantile="0.99"} 2.829188272
```
- It's impossible to aggregate summaries across multiple series. For example, you cannot ignore `instance` and only aggregate on `api`.
- The quantiles must be predefined and cannot change. For example, if we want to add a 0.90 quantile to the metric below, we need to rebuild the metric.

### PromQL

PromQL looks intimidating at first (“why are there angle brackets, pipes, and rate functions everywhere?”), but it’s mostly: select series → shape/aggregate → optionally divide two results. Practice small queries early: list a metric, then sum it, then rate it, then group it.

Learning ladder suggestion:
1. Raw selector: `http_server_requests_seconds_count`.
2. Filter by label: `{method="GET"}`.
3. Turn counter into rate: `rate(...[5m])`.
4. Group: `sum by (status) (...)`.
5. Combine: error rate = `sum(rate(errors[5m])) / sum(rate(requests[5m]))`.

- PromQL: https://prometheus.io/docs/prometheus/latest/querying/basics/
  - expression types: instant vector, range vector, scalar sample and histogram sample
  - literals: string and float
  - time series selector
  - operator and function, group with `without/by`
  - subquery

# Understanding HPA

Below is a high‑level view of how resource, custom, and external metrics flow through the system into scaling decisions. The layout follows the referenced diagram style; the HPA Controller and HPA objects are shown together at the bottom, and the reconcile loop is implied inside the controller box.

Note, we can use other tools to replace Prometheus and Prometheus Adapter in the diagram below but those two tools are easy to set up so we use them to demo in this article.

If HPA ever felt magical, here’s the demystification: it just fetches numbers from APIs every ~15s, does middle‑school proportional math, and updates a replica count—tempered by guardrails (cooldowns / stabilization). No AI, no predictive models (by default). You can always reconstruct its decision.

```
+------------------+     +----------------------+   +---------------------------+
|    cAdvisor      |     |       App Pods       |   |    External Data Source   |
|   CPU & memory   |     |  Micrometer endpoint |   |    e.g., Kafka Admin API  |
+---------+--------+     +----------+-----------+   +-------------+-------------+
          v                         v                             v
+-------------------+    +------------------------------------------------------+
|  metrics-server   |    |                    Prometheus Server                 |
|  short retention  |    |                      internal TSDB                   |
+---------+---------+    +------------------------+-----------------------------+
          |                                       | PromQL queries
          |                                       v
          |                       +---------------------------------+
          |                       |       Prometheus Adapter        |
          |                       |    expose data via K8S APIs     |
          |                       +---+------------------------+----+
     metrics.k8s.io        custom.metrics.k8s.io           external.metrics.k8s.io
          |                           |                        |
          +---------------------------+------------------------+
                                      | (API aggregation layer)
            +-------------------------v---------------------+
            |                kube-apiserver                 |
            |    - metrics.k8s.io (direct)                  |
            |    - custom.metrics.k8s.io (adapter)          |
            |    - external.metrics.k8s.io (adapter)        |
            +-------------------------+---------------------+
                                      | watch / list / GET metric values
                                      v
                 +--------------------------------------+
                 |           HPA Controller             |
                 |      (periodic reconcile loop)       |
                 |   - Reads HPA specs                  |
                 |   - Fetches metrics (resource/       |
                 |     custom/external)                 |
                 |   - Computes desired replicas        |
                 |   - Applies scaling policies         |
                 +--------------------+-----------------+
                                      |
                          scales (Deployment/StatefulSet)
```

Key points:
- Resource metrics (CPU/memory) bypass Prometheus; they flow kubelet → metrics-server → metrics.k8s.io.
- Custom & external metrics require: exporter → Prometheus scrape → Prometheus Adapter rule → aggregated APIService (custom/external groups).
- The adapter does name translation (regex), aggregation (rate/avg/sum), and exposes synthetic metric resources.
- HPA Controller loops (default ~15s), normalizes current vs target, applies stabilization windows & scaling policies, and updates the scale subresource.
- External metrics are cluster (not pod) scoped; label selectors (e.g., AWS Event Queue Metrics with a queue_name label) narrow series.

Formula (scale up/down for Value / AverageValue metrics):
  desiredReplicas = ceil(currentReplicas * currentMetricValue / targetMetricValue)

For utilization (CPU %):
  desiredReplicas = ceil( sum(currentUsage) / ( targetUtilization% * podRequest ) )

Scaling intuition: if your current value is 4x the target, you’ll (attempt to) scale to roughly 4x the replicas. If it’s half, the controller nudges down (respecting stabilization windows). That’s it—no rocket science.

# Environment Setup

We’re intentionally keeping this “kitchen counter lab” minimal so you can run it on a laptop and still see all the moving parts. Each component exists for a reason:
- Ingress: friendly hostnames instead of random NodePorts.
- metrics-server: baseline CPU/memory so HPA works out of the box.
- Prometheus: durable, queryable metric brain.
- Prometheus Adapter: translator from PromQL world to Kubernetes metric APIs.
- Sample app + fake exporter: predictable, tweakable signals.

In the following steps, each time you install a new components, please go back to the diagram above to refresh your memory and don't get lost.

## K8s and Ingress
We use Docker Desktop on macOS for convenience; a full multi-node cluster is unnecessary.
The demo is a Spring Boot application using Micrometer to expose metrics via the actuator endpoint `http://localhost:8080/actuator/prometheus`.
Add the following hostnames to `/etc/hosts`:
```
127.0.0.1	prometheus.local
127.0.0.1	hpa-demo.local
127.0.0.1	monitor.local
```

Install jq and hey
```bash
> brew install jq
> brew install hey
```

Install the ingress controller if you are using Docker Desktop
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'
```

Install metrics-server. Run `kubectl top pods`; if you see CPU and memory metrics, `metrics-server` is working.
By default, metrics-server will be installed. If not, install it yourself.
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/args/-",
    "value": "--kubelet-insecure-tls"
  }
]'
```

## Compile and build the image
Build the image locally so the cluster can pull it without a registry:
```bash
./gradlew clean build
docker build -t daniel/hpa-demo .
```

# HPA with Basic Resource Metrics

Start simple: before fancy custom metrics, prove the default (CPU) scaling path works. This validates cluster primitives (metrics-server, API aggregation, controller loop) without Prometheus yet.

## Deploy the service
```bash
> kubectl apply -f deploy_resource_metrics.yaml
```

You should see one pod and an HPA with minimal CPU usage.
```bash
> kubectl get all
NAME                           READY   STATUS    RESTARTS   AGE
pod/hpa-demo-6d5884df9-tlgps   1/1     Running   0          4d4h

NAME                                           REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/hpa-demo   Deployment/hpa-demo   1%/50%    1         10        1          4d4h
```

Add `hpa-demo.local` to `/etc/hosts` (127.0.0.1) and test the service:
```bash
> curl -sv http://hpa-demo.local/test_increase_counter
* Connected to localhost (::1) port 80
...
OK
```

## Test HPA with basic resource metrics
Now, let's put some fake CPU load on a pod for 10 minutes:
```bash
> curl -sv http://hpa-demo.local/fakeCPULoad?minutes=10
```

You should observe elevated CPU in the HPA and additional pods being created:

What you’re seeing: metrics-server reports rising CPU usage → HPA compares actual vs target 50% → desired replicas > current → scale up event → Deployment creates new pods.

```bash
> kubectl get all
NAME                           READY   STATUS    RESTARTS   AGE     IP            NODE                        NOMINATED NODE   READINESS GATES
pod/hpa-demo-6d5884df9-c76zg   1/1     Running   0          20s     100.64.0.44   ip-10-180-0-127.internal    <none>           <none>
pod/hpa-demo-6d5884df9-nwjq7   1/1     Running   0          20s     100.64.1.7    ip-10-180-15-182.internal   <none>           <none>
pod/hpa-demo-6d5884df9-szjkf   1/1     Running   0          4h39m   100.64.1.2    ip-10-180-15-182.internal   <none>           <none>
pod/hpa-demo-6d5884df9-vtk4j   1/1     Running   0          19s     100.64.0.45   ip-10-180-0-127.internal    <none>           <none>

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                       SELECTOR
deployment.apps/hpa-demo   4/4     4            4           16h   main         daniel/hpa-demo:latest   app=hpa-demo

NAME                                           REFERENCE             TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/hpa-demo   Deployment/hpa-demo   441%/50%   1         10        1          16h
```

# HPA with Custom Metrics

Now we graduate from “CPU percent” (a proxy) to “business-ish” signals. Custom metrics let you scale on things your users actually feel (request throughput, active sessions, processing rates) rather than internal resource saturation.
We need an aggregated API server (adapter) that serves endpoints like `/apis/custom.metrics.k8s.io/v1beta1/` so the Kubernetes API server can delegate metric queries via an `APIService` registration:

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.custom.metrics.k8s.io
spec:
  insecureSkipTLSVerify: true
  group: custom.metrics.k8s.io
  service:
    name: my-custom-metrics-server
    namespace: my-custom-metrics-server
  version: v1beta1
```
And the APIs should expose metrics like `/namespaces/my-ns/pods/my-pod/{my-custom-metric-name}`. Then the HPA controller will read the HPA configuration and try to
find this API by namespace, pod and the "my-custom-metric-name" to get the metric value, finally calculate how many pods should be there.
The most popular adapter is the Prometheus Adapter; we will see how it works later.

Mental model: Adapter = (Pattern Match + Rename + PromQL Template) → Synthetic Kubernetes Metric Endpoint.

Common pitfalls (watch out now; saves debugging later):
- Metric name mismatch (regex not catching `_total`).
- Counter used without `rate()` causing ever‑growing “pressure”.
- Too granular labels (every user ID) bloating series count.
- Forgetting scrape annotations → “Where is my metric?!”

## Install Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace prometheus
# node exporter does not work on Docker-Desktop and we don't use it in the test
helm install prometheus prometheus-community/prometheus \
  --version 27.39.0 \
  -n prometheus \
  --set prometheus-node-exporter.enabled=false
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus
  namespace: prometheus
spec:
  ingressClassName: nginx
  rules:
    - host: prometheus.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-server
                port:
                  number: 80
EOF
```
Add the DNS entry to `/etc/hosts`, then open `http://prometheus.local` in your browser and you can see the Prometheus UI.
Wait for a few minutes; you can list all metrics with this:
```bash
curl -sv http://prometheus.local/api/v1/label/__name__/values |jq
```
With a clean new installation of Prometheus, you see no metrics for our microservice.

## Collect custom metrics

Just add a few annotations to our service to tell Prometheus to collect the metrics.
```yaml
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

Run `kubectl apply -f deploy_custom_metrics_collect.yaml` and wait for a while, you should see some metrics of our service
```bash
# no metric value
curl -sv 'http://prometheus.local/api/v1/query?query=my_test_counter_total' |jq
# request once
curl -sv http://hpa-demo.local/test_increase_counter
# wait for a while, now you see your metric value as 1
curl -sv 'http://prometheus.local/api/v1/query?query=my_test_counter_total' |jq
```
>In the `/actuator/prometheus` endpoint, besides our custom metric generated in our code `DemoConfig`, Spring Boot Actuator also exposes a lot of useful metrics such as `tomcat_threads_busy_threads` and `tomcat_servlet_error_total`.

> Note, when Prometheus is deployed on Kubernetes, it uses the environment variables to guess the K8S api-server URL and queries the pod list from K8S to discover all pods and other resources to monitor, so you don't need to add them manually to the configuration of Prometheus.

You can also experiment with PromQL via the UI page. The target of the pod can be found: `http://prometheus.local/targets?search=hpa-demo`.
Click the query button and enter `my_test`; auto-completion will show the metric `my_test_counter_total`.

>Tip: If a metric doesn’t appear after one scrape interval (e.g., 15s), trigger it (hit the endpoint), then re‑query. Idle counters often don’t show up until first increment.

## Install Prometheus Adapter

Although Prometheus knows our custom metrics, we still need to register Prometheus with a custom metric adapter behind the API server so HPA can query it to get the custom metrics. To install the Prometheus Adapter:
```bash
# custom metrics do not exist
> kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
Error from server (ServiceUnavailable): the server is currently unable to handle the request

# install the adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter -n prometheus --set-string prometheus.url=http://prometheus-server --set-string prometheus.port=80

# request once
curl -sv http://hpa-demo.local/test_increase_counter
# wait for a while; you will see the custom API registered with the API server. You can query our test metric "my_test_counter" now
> kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 |jq '.resources[] | select(.name =="pods/my_test_counter")'
{
  "name": "pods/my_test_counter",
  "singularName": "",
  "namespaced": true,
  "kind": "MetricValueList",
  "verbs": [
    "get"
  ]
}
```
There are two kinds of custom metric objects: `Pod` and `Object`. For Pod custom metrics, you can query the metric value with the pattern `/apis/custom.metrics.k8s.io/v1beta1/namespaces/<namespace>/pods/<pod name>/<pod metric name>`, e.g.,
```bash
❯ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/hpa-demo-75b48b9f74-zcgx4/my_test_counter |jq
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "hpa-demo-75b48b9f74-zcgx4",
        "apiVersion": "/v1"
      },
      "metricName": "my_test_counter",
      "timestamp": "2023-02-17T14:45:55Z",
      "value": "0",
      "selector": null
    }
  ]
}
```

You can make a few requests by `curl -sv http://hpa-demo.local/test_increase_counter` and wait 30 seconds for Prometheus to pull the metrics, then check the metric value via api-server; you may see the value is something like a `10m` which means a rate at `0.01`.
> Note, the counter value is converted to a rate value by the Prometheus Adapter, and rate values are often with the suffix `m` which means 1/1000, e.g., `12m` means 0.012 TPS. Check the `metricsQuery` in the next example.

There are some default rules in Prometheus adapter to make this happen.

- Discovery, which specifies how the adapter should find all Prometheus metrics for this rule. See `seriesQuery` and `seriesFilters`.
- Association, which specifies how the adapter should determine which Kubernetes resources a particular metric is associated with. See `resources`.
- Naming, which specifies how the adapter should expose the metric in the custom metrics API. See `name` and `as`.
- Querying, which specifies how a request for a particular metric on one or more Kubernetes objects should be turned into a query to Prometheus.

You can get all the rules by `kubectl get cm prometheus-adapter -n prometheus -o yaml`. In our case, the test metric `my_test_counter_total` is matched by the default rule below, and converted into the rate `my_test_counter` under the custom metrics resource.
```yaml
rules:
  - seriesQuery: '{namespace!="",__name__!~"^container_.*"}'
    seriesFilters:
    - isNot: .*_seconds_total
    resources:
      template: <<.Resource>>
    name:
      matches: ^(.*)_total$  <--- this matches the metric name, which is my_test_counter_total in our case
      as: "my_test_counter_rate"
    metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[5m])) by (<<.GroupBy>>)
```
Because the `rate()` function converts the counter into a value like "how much it increased over 5 minutes",
so the value is very small like 0.01 and when you run `kubectl get hpa` you could see `10m` instead of `0.01`.
Later in our test, we will use hey to generate enough load to make the metric value higher.
> The `.Series` and `.LabelMatchers` are from Go template syntax; `<<` and `>>` are to avoid conflicts with PromQL.
> By default, any custom metrics exposed by Actuator will be available in K8S custom metrics.

## Test HPA with custom metrics
Now we have a metric `my_test_counter` registered to the K8S API server, we can configure HPA to autoscale pods based on it.
We will create an HPA targeting the metric value `my_test_counter` at 1 (or 1000m), which is 1 TPS.
Then we use `hey` to generate a load of 5 TPS to the deployment, so HPA will spin up 4–5 pods.
```bash
# install the HPA with custom metrics my_test_counter
kubectl apply -f deploy_custom_metrics_collect_hpa.yaml
# check the HPA
> kubectl get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/hpa-demo-699586ccf5-jtv9x   1/1     Running   0          2m54s

NAME                                           REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/hpa-demo   Deployment/hpa-demo   0/1       1         10        1          72s

> hey -c 1 -n 10000 -q 5 http://hpa-demo.local/test_increase_counter
# wait for a while, you will see more pods are created by HPA, and the TARGETS field in HPA increases
> kubectl get all
NAME                            READY   STATUS    RESTARTS   AGE
pod/hpa-demo-699586ccf5-4gm4n   1/1     Running   0          103s
pod/hpa-demo-699586ccf5-lmknn   1/1     Running   0          4m42s
pod/hpa-demo-699586ccf5-qg8w9   1/1     Running   0          56m
pod/hpa-demo-699586ccf5-tpcgh   1/1     Running   0          3m43s
```

>Custom metrics can only be used in HPA as `averageValue`, which is a raw absolute value. The ratio value `AverageUtilization` is not supported.

Why? The API doesn’t define “utilization” semantics for arbitrary metrics—you decide what absolute target makes sense (e.g., 1 request/sec per pod). Keep targets stable and tweak iteratively.

# HPA with External Metrics

We use a dummy Python script to simulate an external monitoring service exposing a metric `kafka_queue_length` via a REST API.
The interesting thing here is, external metrics are not collected per pod like the custom metrics, for example we could collect the queue length from Kafka, not the microservice consuming events from Kafka.
In most cases you will have a tag to tell which microservice you want to scale. e.g., there could be many Kafka topics consumed by many microservices,
and there should be tag in the metric like `topic_lag` to indicate which microservice is consuming this topic, if there are too many events piled up in that queue with high latency,
you expect HPA to pick up the microservice by that tag to scale up and reduce the latency.

## Deploy a fake monitoring system
Deploy the Python fake monitoring system to simulate a Kafka metric source provider (maybe kafka-exporter, or DynaTrace, whatever tool has the metrics).
```bash
kubectl apply -f deploy_external_metrics_collect.yaml
> kubectl get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/fake-kafka-monitor-677dccb675-h6gjn   1/1     Running   0          37s
pod/fake-kafka-monitor-677dccb675-p884m   1/1     Running   0          17m
```

Add the DNS entry to your `/etc/hosts` so you can access it directly via ingress.
```bash
> curl -sv http://monitor.local/metrics

# HELP kafka_queue_length Fake Kafka queue length
# TYPE kafka_queue_length gauge
kafka_queue_length{queue_name="hpa-demo"} 10
kafka_queue_length{queue_name="foo-svc"} 1
kafka_queue_length{queue_name="bar-svc"} 2
```

## Collect external metrics
Here we simulate the monitoring system exposes the metric `kafka_queue_length` for 3 services, each queue is for one microservice,
now we will pick up the queue `hpa-demo` to scale out our microservice if that queue has too many pending events/jobs to run.
Because external metrics are not collected from the microservice pods, they are collected from a central service like Kafka,
there is no need for `pod` or `deployment` tag/label, and we can collect from the service regardless the Load Balancing algorithm,
Prometheus can hit any Kafka pod and the queue length metrics are always the same, so aggregation from Kafka pods is not needed.

We will add a new scrape job like this
```yaml
    - job_name: fake-kafka-monitor
      static_configs:
        - targets: ['fake-kafka-monitor-service.default.svc.cluster.local:8080']
      metrics_path: /metrics
```
Run the helm upgrade command below to add the scrape job to Prometheus.
```bash
helm upgrade prometheus prometheus-community/prometheus -n prometheus -f values_scrape_external_metrics.yaml
```

Then check the generated config file, you should see the new job added to Prometheus
```bash
> kubectl exec -ti prometheus-server-645f64b476-7xz45 -c prometheus-server -n prometheus -- cat /etc/config/prometheus.yml |grep kafka
- job_name: fake-kafka-monitor
    - fake-kafka-monitor-service.default.svc.cluster.local:8080
```

Wait for a few minutes and open the Prometheus UI, you should see the metric `kafka_queue_length` as well.
Note, there is no tag like `pod` or `namespace`, but a tag `queue-name` to differentiate the queues.
```
kafka_queue_length{instance="fake-kafka-monitor-service.default.svc.cluster.local:8080", job="fake-kafka-monitor", queue_name="hpa-demo"}	10
kafka_queue_length{instance="fake-kafka-monitor-service.default.svc.cluster.local:8080", job="fake-kafka-monitor", queue_name="foo-svc"}	1
kafka_queue_length{instance="fake-kafka-monitor-service.default.svc.cluster.local:8080", job="fake-kafka-monitor", queue_name="bar-svc"}	2
```
Unlike the resource and custom metrics, the default rules in Prometheus Adapter are not sufficient here.
An `external metric` is not associated with a pod or any other K8S object, so the `resources` field is empty, we need to create a rule.

## Expose external metrics to HPA
You need to add our new external metric to the adapter with the command line below
```bash
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter -n prometheus -f values_adapter_external_metrics.yaml
```

Now you can verify the external metric is registered to the api-server and exposed with values of the 3 queues.
> Note, because the external metric is not bound to a namespace, the namespace `default` is just a placeholder to follow the K8S API spec; you can use anything here.
```bash
❯ kubectl get --raw /apis/external.metrics.k8s.io/v1beta1/namespaces/default/kafka_queue_length | jq
{
  "kind": "ExternalMetricValueList",
  "apiVersion": "external.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "metricName": "kafka_queue_length",
      "metricLabels": {
        "queue_name": "hpa-demo"
      },
      "timestamp": "2025-09-29T09:43:58Z",
      "value": "10"
    },
    {
      "metricName": "kafka_queue_length",
      "metricLabels": {
        "queue_name": "foo-svc"
      },
      "timestamp": "2025-09-29T09:43:58Z",
      "value": "1"
    },
    {
      "metricName": "kafka_queue_length",
      "metricLabels": {
        "queue_name": "bar-svc"
      },
      "timestamp": "2025-09-29T09:43:58Z",
      "value": "2"
    }
  ]
}
```

## Test HPA with external metrics

We will update the HPA to target 2 as in the code snippet below. If the queue length is 2 then HPA will stabilize the number of pods. 
Because the current metric value is set to 10 in our fake monitoring system, the HPA controller will increase the number of the pods.
```yaml
  metrics:
    - type: External
      external:
        metric:
          name: kafka_queue_length
          selector:
            matchLabels:
              queue_name: hpa-demo
        target:
          # desiredReplicas = ceil( currentReplicas * ( currentMetricValue / targetValue ) ) when scaling up,
          type: Value
          value: 2
```
Run this command and wait for a while, you will see pod increasing.
```bash
kubectl apply -f deploy_external_metrics_hpa.yaml
```
Because the queue length is fixed at 10, it simulates the scenario that even though more pods are scheduled the work is still queued,
so HPA schedules more and more pods until it hits the limit of 10.

Now we simulate the queue length is reduced. We simulate the scenario where events are processed and only one event in queue,
because our HPA targets for 2, so the number of pods will decrease.
```bash
kubectl patch configmap fake-kafka-config --type merge -p '{"data":{"queue_length":"1"}}'

```
Now you will see the fake external monitor reports queue size as 1 for "hpa-demo".
```bash
> curl -sv http://monitor.local/metrics

# HELP kafka_queue_length Fake Kafka queue length
# TYPE kafka_queue_length gauge
kafka_queue_length{queue_name="hpa-demo"} 1
kafka_queue_length{queue_name="foo-svc"} 1
kafka_queue_length{queue_name="bar-svc"} 2
```

Wait for a while and you will see the metrics in Prometheus UI
```
kafka_queue_length{instance="fake-kafka-monitor-service.default.svc.cluster.local:8080", job="fake-kafka-monitor", queue_name="hpa-demo"}	1
kafka_queue_length{instance="fake-kafka-monitor-service.default.svc.cluster.local:8080", job="fake-kafka-monitor", queue_name="foo-svc"}	1
kafka_queue_length{instance="fake-kafka-monitor-service.default.svc.cluster.local:8080", job="fake-kafka-monitor", queue_name="bar-svc"}	2
```

Prometheus Adapter is also aware of this change via the K8S API
```bash
❯ kubectl get --raw /apis/external.metrics.k8s.io/v1beta1/namespaces/default/kafka_queue_length | jq
{
  "kind": "ExternalMetricValueList",
  "apiVersion": "external.metrics.k8s.io/v1beta1",
  "metadata": {},
  "items": [
    {
      "metricName": "kafka_queue_length",
      "metricLabels": {
        "queue_name": "hpa-demo"
      },
      "timestamp": "2025-09-29T10:25:47Z",
      "value": "1"
    },
  ...
```

You need to wait a bit longer to see the HPA attempt to scale down (you must wait at least 5 minutes by default; check the official documentation for `stabilizationWindowSeconds`).

Why the delay? HPA intentionally resists rapid oscillation: scaling down too fast can trash cache warmup, connection pools, and latency. The stabilization window asks, “Have we been consistently below target long enough to relax?”

```bash
> kubectl get hpa hpa-demo -o yaml
# You will see desiredReplicas changing every 5 minutes
...
    currentMetrics:
    - external:
        current:
          value: "1"
        metric:
          name: kafka_queue_length
          selector:
            matchLabels:
              queue_name: hpa-demo
      type: External
    currentReplicas: 10
    desiredReplicas: 5
...
```

# Summary

In this walkthrough we built a minimal end‑to‑end pipeline that collects service and external signals, adapts them into Kubernetes custom/external metrics, and drives a stable, explainable HPA scaling loop.

Conclusion: With a clear separation between collection, adaptation, and policy (HPA), you can evolve each layer independently—add new metrics, refine queries, or tune scaling behavior—without re‑architecting the system.

If you can now explain to a teammate how a counter increment becomes a new pod, you’ve won. Keep iterating—production autoscaling is a game of feedback quality and patience.


