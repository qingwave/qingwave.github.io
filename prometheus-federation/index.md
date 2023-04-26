# Prometheus分区实践

## 背景
单个Prometheus Server可以轻松的处理数以百万的时间序列。但当机器规模过大时，需要对其进行分区，Prometheus也提供了集群联邦的功能，方便对其扩展。
<!--more-->

我们采用Prometheus来监控k8s集群，节点数400，采集的samples是280w，Prometheus官方的显示每秒可抓取10w samples。当集群规模扩大到上千节点时，单个Prometheus不足以处理大量数据，需要对其进行分区。

> 可以根据`scrape_samples_scraped{job=${JOBNAME}}`来统计各个job的samples数目
> 可以根据`count({__name__=~".*:.*"})`来统计metrics总数

## 集群联邦
在Promehtues的源码中，`federate`联邦功能在`web`中，是一个特殊的查询接口，允许一个prometheus抓取另一个prometheus的metrics

可以通过全局的prometheus抓取其他slave prometheus从而达到分区的目的
![federate](/img/blogImg/federate1.png)

使用federate进行分区通过有两种方式

### 功能分区
每个模块为一个分区，如node-exporter为一个分区，kube-state-metrics为一个分区，再使用全局的Prometheus汇总
![federate by job](/img/blogImg/federate2.png)

实现简单，但当单个job采集任务过大（如node-exporter）时，单个Prometheus slave也会成为瓶颈

### 水平扩展
针对功能分区的不足，将同一任务的不同实例的监控数据采集任务划分到不同的Prometheus实例。通过relabel设置，我们可以确保当前Prometheus Server只收集当前采集任务的一部分实例的监控指标。
![federate by modulus](/img/blogImg/federate3.png)

下为官方提供的配置

```yaml
global:
  external_labels:
    slave: 1  # This is the 2nd slave. This prevents clashes between slaves.
scrape_configs:
  - job_name: some_job
    # Add usual service discovery here, such as static_configs
    relabel_configs:
    - source_labels: [__address__]
      modulus:       4    # 4 slaves
      target_label:  __tmp_hash
      action:        hashmod
    - source_labels: [__tmp_hash]
      regex:         ^1$  # This is the 2nd slave
      action:        keep
```

并且通过当前数据中心的一个中心Prometheus Server将监控数据进行聚合到任务级别。

```yaml
- scrape_config:
  - job_name: slaves
    honor_labels: true
    metrics_path: /federate
    params:
      match[]:
        - '{__name__=~"^slave:.*"}'   # Request all slave-level time series
    static_configs:
      - targets:
        - slave0:9090
        - slave1:9090
        - slave3:9090
        - slave4:9090
```

水平扩展，即通过联邦集群的特性在任务的实例级别对Prometheus采集任务进行划分，以支持规模的扩展。

## 我们的方案

### 整体架构
- Promehtues以容器化的方式部署在k8s集群中
- 收集node-exporter、cadvisor、kubelet、kube-state-metrics、k8s核心组件、自定义metrics
- 通过实现opentsdb-adapter，对监控数据做持久化
- 通过falcon-adapter,为监控数据提供报警
![monitor](/img/blogImg/Promehtues-arch.png)

### 分区方案
- Prometheus分区包括master Prometheus 与 slave Promehtues
- 我们将监控数据分为多个层次: cluster, namespace, deployment/daemonset, pod, node
- 由于kubelet, node-exporter, cadvisor等是以node为单位采集的，所以安装node节点来划分不同job
- slave Prometheus 按照node切片采集node，pod级别数据
- kube-state-metrics暂时无法切片，可通过replicaset 设置多个，单独作为一个kube-state Prometheus，供其他slave Prometheus采集
- 其他etcd, apiserver等自定义组件可通过master Promehtues直接采集

整体架构如下
![federate-plan](/img/blogImg/federate-plan.png)

master Prometheus配置

```yaml
global:
  scrape_interval: 60s
  scrape_timeout: 30s
  evaluation_interval: 60s
  external_labels:
    cluster: {{CLUSTER}}
    production_environment: {{ENV}}

rule_files:
- cluster.yml
- namespace.yml
- deployment.yml
- daemonset.yml

scrape_configs:

- job_name: federate-slave
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
      - '{__name__=~"pod:.*|node:.*"}'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - kube-system
  relabel_configs:
  - source_labels:
    - __meta_kubernetes_pod_label_app
    action: keep
    regex: prometheus-slave.*
  - source_labels:
    - __meta_kubernetes_pod_container_port_number
    action: keep
    regex: 9090

- job_name: federate-kubestate
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
      - '{__name__=~"deployment:.*|daemonset:.*"}'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - kube-system
  relabel_configs:
  - source_labels:
    - __meta_kubernetes_pod_label_app
    action: keep
    regex: prometheus-kubestate.*
  - source_labels:
    - __meta_kubernetes_pod_container_port_number
    action: keep
    regex: 9090
```

slave Prometheus配置

```yaml
global:
  scrape_interval: 60s
  scrape_timeout: 30s
  evaluation_interval: 60s
  external_labels:
    cluster: {{CLUSTER}}
    production_environment: {{ENV}}

rule_files:
- node.yml
- pod.yml

scrape_configs:

- job_name: federate-kubestate
  honor_labels: true
  metrics_path: '/federate'
  params:
    'match[]':
      - '{__name__=~"pod:.*|node:.*"}'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names:
      - kube-system
  relabel_configs:
  - source_labels:
    - __meta_kubernetes_pod_label_app
    action: keep
    regex: prometheus-kubestate.*
  - source_labels:
    - __meta_kubernetes_pod_container_port_number
    action: keep
    regex: 9090
  metric_relabel_configs:
  - source_labels: [node]
    modulus:       {{MODULES}}
    target_label:  __tmp_hash
    action:        hashmod
  - source_labels: [__tmp_hash]
    regex:         {{SLAVEID}}
    action:        keep

- job_name: kubelet
  scheme: https
  kubernetes_sd_configs:
  - role: node
    namespaces:
      names: []
  tls_config:
    insecure_skip_verify: true
  relabel_configs:
  - source_labels: []
    regex: __meta_kubernetes_node_label_(.+)
    replacement: "$1"
    action: labelmap
  - source_labels: [__meta_kubernetes_node_label_kubernetes_io_hostname]
    modulus:       {{MODULES}}
    target_label:  __tmp_hash
    action:        hashmod
  - source_labels: [__tmp_hash]
    regex:         {{SLAVEID}}
    action:        keep

- job_name: ...
```
### 可能的问题
- 如何部署，配置复杂，现在采用shell脚本加kustomize,是否有更简单的方法
- 分区的动态扩展随着node的规模
- kube-state-metrics是否会成为瓶颈，目前的[kube-state-metrics性能测试](https://docs.google.com/document/d/1hm5XrM9dYYY085yOnmMDXu074E4RxjM7R5FS4-WOflo/edit)
- 由于分区同一个job的不同instance采集的时间有偏差，对聚合有一定影响
- 可靠性保证，如果一个或多个slave的挂了如何处理，使用k8s来保证prometheus的可用性是否可靠
