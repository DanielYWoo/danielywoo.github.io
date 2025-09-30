---
layout: post
title:  "在 Kubernetes 中使用 HPA + Custom / External Metrics（完整中文版）"
subtitle: "从零搭建 metrics → decisions → scaling 的最小可验证链路"
date:   2025-09-30 16:00:00
published: true
---

# 概览 (Overview)

本篇带你在本地搭建一个最小但**真实可解释**的链路：应用 / 外部信号 → Prometheus 收集与查询 → Prometheus Adapter 适配 → HPA Controller 计算副本数 → Deployment 扩缩。  
目标：读完后你能自信回答：“一个 HTTP endpoint 暴露出的 counter 为什么最后会让集群新增 Pod？”

你将完成：
1. 理解 time‑series（时间序列）数据模型，弄清楚 metric name + labels 如何唯一标识一条 series（后面 PromQL 才不显得像魔法）。
2. 观察 Prometheus 如何存储和查询你的指标。
3. 暴露一个自定义 Counter，并让 HPA 基于它扩缩。
4. 模拟一个 External Metric（队列长度）并基于它扩缩。
5. 直接访问 HPA 实际调用的 Kubernetes Metrics APIs，消除“黑盒”感。

请先下载示例代码：[https://github.com/DanielYWoo/hpa-demo](https://github.com/DanielYWoo/hpa-demo)
本文AI翻译, 如果有纰漏请查看[英文原文](/hpa-with-custom-and-external-metrics-in-k8s) 

# 核心概念 (Concepts)

监控体系失败、SLO 无法落地、扩缩策略失真，往往源于对指标模型理解肤浅。本节是后面所有操作的前提。

## TSDB Data Model（通用时序数据库模型）

为什么从这里开始？因为 PromQL、Adapter 规则、HPA 计算都默认你已理解：  
“metric name + tag(label) 组合 = 一条随时间变化的数值流 (series)”。

### 基础示例 (来自 InfluxDB 文档)

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

- measurement：被测量对象（类似“逻辑表名”）。
- tag set：一组 tag 键值对，指定一个维度切片。
- field key：该 measurement 下存的具体数值字段名（很多场景只用一个 field）。
- series：measurement + field key + tag set 随时间形成的纵向数值序列。
- point：某一时间点该 tag set 下所有 field 的集合。

类比（仅帮助理解，不要过拟合）：
- measurement ≈ 表
- tag set ≈ where 条件
- series ≈ 表中同一条件下某“列”的时间演进
- point ≈ 某时刻“行”快照

示例（固定 tag set）：
```
Given Tag Set = "location = 1, scientist = langstroth"
Timestamp       butterflies   honeybees
t1              12            23
t2              14            21
t3              17            45
```

一个 series（butterflies）：
```
Timestamp   value
t1          12
t2          14
t3          17
```

某个 point（时间 t1）：
```
butterflies  honeybees
12           23
```

心智快速索引：
- 想“某个值如何随时间变化” → series
- 想“这个瞬间一组相关值” → point
- 想“按主机 / 地域 / 用户切分” → tag set

## Prometheus 概念 (Prometheus Concepts)

Prometheus = pull 模式采集 + 内嵌 TSDB + PromQL。  
最大雷区：label 组合过多 = High Cardinality → 内存 / CPU 暴炸。

### 与通用 TSDB 的对应

- measurement 在 Prometheus 中称 metric（并区分 4 种类型）。
- tag = label（换个名字而已）。
- metric name + labels = 唯一 series。
- label 决定过滤与聚合维度；新增 label value 会产生新 series。
- 典型表示：`http_requests_total{method="POST",url="/api/tracks"}`

### Metric Types（四种）

只记一个优先级口诀：能用 Counter 就别用 Histogram；能用 Gauge 就别上 Summary。

- Counter：单调递增（重启归零），后续用 `rate()` 得“速率”
- Gauge：当前值，可上可下
- Histogram：分桶统计 + `_count` + `_sum`
- Summary：客户端计算分位；非聚合场景再用

#### Gauge
代表一个瞬时值：并发数、队列长度、缓存大小。可以升降。

#### Counter
单调递增；不要自己拼 “xxx_per_second”，直接暴露 Counter，然后 `rate(counter[5m])` → QPS/TPS。

#### Histogram
一组桶（`_bucket{le=".."}`）+ `_sum` + `_count`。  
近似 P95 等：`histogram_quantile(0.95, sum(rate(<metric>_bucket[5m])) by (le))`  
均值：`rate(x_sum[5m]) / rate(x_count[5m])`

示意（累计计数随时间累加）：
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

#### Summary
内置 quantile 但：
- 不能跨实例聚合（没法 sum 后再算）
- 量化点必须预定义

### PromQL 基本套路

本质模式：选择 → 变形(rate/irate/increase/deriv) → 按标签聚合(sum / avg / max / topk) → 组合运算。

学习阶梯：
1. 选择：`http_server_requests_seconds_count`
2. 过滤：`{method="GET"}`
3. 速率：`rate(...[5m])`
4. 聚合：`sum by (status) (...)`
5. 组合：错误率 = `sum(rate(error_total[5m])) / sum(rate(request_total[5m]))`

文档：https://prometheus.io/docs/prometheus/latest/querying/basics/

# 理解 HPA (Understanding HPA)

如果 HPA 让你觉得“有点神秘”，下面这张链路图和文字会把它拆平：它只是周期性（默认 ~15s）拉几个 API，比例换算一个数字，然后写回 `scale` 副资源，并受稳定窗口与策略约束。

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

关键拆解：
- Resource Metrics (CPU/Memory)：kubelet → metrics-server → metrics.k8s.io（不经 Prometheus）
- Custom / External：Exporter → Prometheus → Adapter（PromQL 模板 + 资源关联）→ 聚合 API
- Adapter 工作：正则匹配命名 → PromQL 查询 → 输出成 Kubernetes MetricValueList
- HPA Controller：对比“当前 vs 目标”→ 比例缩放 → 受 stabilization / scaling policies 限制
- External Metric：不绑定 Pod；可以用 label selector 选择某一条队列、Topic、账户等

核心公式（Value/AverageValue 类）：
```
desiredReplicas = ceil( currentReplicas * currentMetricValue / targetMetricValue )
```

CPU 利用率（%场景）：
```
desiredReplicas = ceil( sum(currentUsage) / ( targetUtilization% * podRequest ) )
```

直觉：当前值是目标 4 倍 → 期望副本 ≈ 4 倍；当前值 0.5 倍 → 倾向缩容（受稳定窗口影响）。

# 环境搭建 (Environment Setup)

组件职责：
- ingress-nginx：Host 方式访问
- metrics-server：Resource Metrics 来源
- Prometheus：统一时间序列存储 + 查询引擎
- Prometheus Adapter：桥接 PromQL 与 Kubernetes Metrics API
- sample app + fake exporter：可控信号注入

## Kubernetes & Ingress

编辑 `/etc/hosts`：
```
127.0.0.1  prometheus.local
127.0.0.1  hpa-demo.local
127.0.0.1  monitor.local
```

工具：
```bash
brew install jq
brew install hey
```

安装 ingress-nginx：
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec":{"type":"LoadBalancer"}}'
```

安装 metrics-server：
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}
]'
```

## 构建镜像
```bash
./gradlew clean build
docker build -t daniel/hpa-demo .
```

# 使用 Resource Metrics 的 HPA（基线验证）

```bash
kubectl apply -f deploy_resource_metrics.yaml
kubectl get hpa
curl -sv http://hpa-demo.local/test_increase_counter
```

施压：
```bash
curl -sv "http://hpa-demo.local/fakeCPULoad?minutes=10"
kubectl get hpa -w
```

观察：CPU 利用率超过目标 → 副本数上升 → Deployment 创建新 Pod。

# 使用 Custom Metrics 的 HPA

我们希望使用“更贴近业务”的信号（如请求吞吐）。链路：
1. 应用暴露 `/actuator/prometheus`
2. Prometheus 抓取
3. Adapter 规则匹配 + 转换（可能 `rate()`）
4. HPA 配置引用该 metric

常见坑：
- 忘了加 scrape 注解
- Counter 未转 `rate()` 直接用 → 无限增大
- label 里包含 userId 等高基数
- 规则 regex 不匹配 `_total`

## 安装 Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace prometheus
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

列出所有 metric 名：
```bash
curl -s http://prometheus.local/api/v1/label/__name__/values | jq
```

## 采集自定义指标

Service / Pod 注解：
```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "8080"
prometheus.io/path: "/actuator/prometheus"
```

应用：
```bash
kubectl apply -f deploy_custom_metrics_collect.yaml
```

测试：
```bash
curl -s 'http://prometheus.local/api/v1/query?query=my_test_counter_total' | jq
curl -s http://hpa-demo.local/test_increase_counter
sleep 20
curl -s 'http://prometheus.local/api/v1/query?query=my_test_counter_total' | jq
```

## 安装 Prometheus Adapter

```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 || true
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  -n prometheus \
  --set-string prometheus.url=http://prometheus-server \
  --set-string prometheus.port=80
```

触发请求后：
```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq '.resources[] | select(.name=="pods/my_test_counter")'
kubectl get pod -l app=hpa-demo
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/<pod>/my_test_counter | jq
```

默认规则片段（概念说明）：
```yaml
seriesQuery: '{namespace!="",__name__!~"^container_.*"}'
name:
  matches: ^(.*)_total$
  as: my_test_counter_rate
metricsQuery: sum(rate(<<.Series>>{<<.LabelMatchers>>}[5m])) by (<<.GroupBy>>)
```
说明：
- `matches` 捕获 `_total` 结尾 Counter
- 通过 `rate()` 转成 5m 平滑速率
- 输出名用 `as` 重命名
- Go template 占位 `<<.Series>>` / `<<.LabelMatchers>>`

如果值显示 `12m` → 表示 0.012（milli）

## 基于 Custom Metric 的 HPA

目标：每 Pod 维持 1 TPS。

```bash
kubectl apply -f deploy_custom_metrics_collect_hpa.yaml
hey -c 1 -n 10000 -q 5 http://hpa-demo.local/test_increase_counter
kubectl get hpa -w
```

注意：Custom Metrics 只能用 `averageValue`（绝对值），没有 `AverageUtilization` 概念。

# 使用 External Metrics 的 HPA

External Metric 不与 Pod 绑定，典型如：Kafka backlog、队列长度、第三方 SaaS 指标。我们用一个模拟服务暴露：
```
kafka_queue_length{queue_name="hpa-demo"} 10
kafka_queue_length{queue_name="foo-svc"} 1
kafka_queue_length{queue_name="bar-svc"} 2
```

## 部署模拟外部源
```bash
kubectl apply -f deploy_external_metrics_collect.yaml
curl -s http://monitor.local/metrics
```

## Prometheus 抓取外部指标

添加 scrape job：
```yaml
- job_name: fake-kafka-monitor
  static_configs:
    - targets: ['fake-kafka-monitor-service.default.svc.cluster.local:8080']
  metrics_path: /metrics
```

更新：
```bash
helm upgrade prometheus prometheus-community/prometheus -n prometheus -f values_scrape_external_metrics.yaml
kubectl exec -ti $(kubectl get pod -n prometheus -l app=prometheus,component=server -o jsonpath='{.items[0].metadata.name}') -n prometheus -- \
  grep -n 'fake-kafka-monitor' /etc/config/prometheus.yml
```

查询 `kafka_queue_length`，确认存在。

## Adapter 暴露 External Metric

External Metric 不关联资源，规则中 `resources` 留空。

```bash
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter -n prometheus -f values_adapter_external_metrics.yaml
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1/namespaces/default/kafka_queue_length | jq
```

## 基于 External Metric 的 HPA

HPA 目标：`kafka_queue_length{queue_name="hpa-demo"} = 2`，当前为 10 → 扩容直到 maxReplicas。

```bash
kubectl apply -f deploy_external_metrics_hpa.yaml
kubectl get hpa -w
```

降低队列：
```bash
kubectl patch configmap fake-kafka-config --type merge -p '{"data":{"queue_length":"1"}}'
curl -s http://monitor.local/metrics
```

待传播（Exporter → Prometheus scrape → Adapter → HPA），缩容受 stabilization window（默认下调延迟约 5 分钟）影响。

查看：
```bash
kubectl get hpa hpa-demo -o yaml
```

# 总结 (Summary)

链路拆分为三个层次：
1. 采集层：Exporter / scrape / 指标命名
2. 适配层：PromQL 模板化、聚合、命名重写、暴露成 K8S Metric API
3. 策略层：HPA 比例计算 + 稳定策略 + 上下限

这种解耦让你可以独立演进：新增指标无需改 HPA 控制器；调整扩缩策略无需改应用；优化 PromQL 聚合无需改代码。  
如果你现在能向同事解释：“一个 Counter 递增如何最终触发 Deployment scale out”，本教程目的已达成。祝实验顺利。