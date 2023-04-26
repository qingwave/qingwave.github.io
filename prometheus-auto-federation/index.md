# Prometheus高可用自动分区方案


在[Prometheus分区实践](/prometheus-federation)中我们介绍了使用集群联邦与远程存储来扩展Prometheus以及监控数据持久化，但之前的分区方案存在一定不足，如分区配置较难维护，全局Prometheus存在性能瓶颈等，本文通过`Thanos+Kvass`实现更优雅的Prometheus扩展方案。
<!--more-->

## 自动分区
之前分区方案依赖Prometheus提供的`hashmod`方法，通过在配置中指定`hash`对象与`modules`进行散列（md5），每个分片只抓取相同job命中的对象，例如我们可以通过对`node`散列从而对`cadvisor`、`node-exporter`等job做分片。

通过这种方式可以简单的扩展Prometheus，降低其抓取压力，但是显而易见`hashmod`需要指定散列对象，每个job可能需要配置不同的对象如`node`、`pod`、`ip`等，随着采集对象增多，配置难以维护。直到看见了[Kvass](https://github.com/tkestack/kvass)，Kvass是一个Prometheus横向扩展方案，可以不依赖`hashmod`动态调整target，支持数千万series规模。

Kvass核心架构如下：
![kvass](https://github.com/tkestack/kvass/raw/master/README.assets/image-20201126031456582.png)
- `Kvass-Coordinator`: 加载配置文件并进行服务发现，获取所有target，周期性分配target到`kvass-sidecar`，以及管理分片负载与扩缩容
- `Kvass-Sidecar`: 根据`Coordinator`分发的target生成配置，以及代理Prometheus请求

通过Kvass可实现Prometheus动态横向扩展，而不依赖`hashmod`，灵活性更高。

## 全局查询
另一个问题是在集群联邦中我们需要一个全局的Prometheus来聚合分区Prometheus的数据，依赖原生的`/federate`接口，随着数据量增多，全局Prometheus必然会达到性能瓶颈。高可用Prometheus集群解决方案[Thanos](https://github.com/thanos-io/thanos)中提供了全局查询功能，通过`Thanos-Query`与`Thanos-Sidecar`可实现查询多个Prometheus的数据，并支持了去重。

Thanos组件较多，核心架构如下：
![Thanos](/img/blogImg/thanos-arch.png)
- `Thanos Query`: 实现了`Prometheus API`，将来自下游组件提供的数据进行聚合最终返回给查询数据的client (如 grafana)，类似数据库中间件
- `Thanos Sidecar`: 连接Prometheus，将其数据提供给`Thanos Query`查询，并且可将其上传到对象存储，以供长期存储
- `Thanos Store Gateway`: 将对象存储的数据暴露给`Thanos Query`去查询
- `Thanos Ruler`: 对监控数据进行评估和告警，还可以计算出新的监控数据，将这些新数据提供给`Thanos Query`查询并且可上传到对象存储，以供长期存储
- `Thanos Compact`: 将对象存储中的数据进行压缩和降低采样率，加速大时间区间监控数据查询的速度

借助于Thanos提供的`Query`与`Ruler`我们可以实现全局查询与聚合。

## 最终方案
`Kvass+Thanos`可实现Prometheus自动扩展、全局查询，再配合`Remote Wirite`实现数据支持化，通过Grafana展示监控数据
![Prometheus-HA](/img/blogImg/prometheus-ha.png)

### 测试验证
所有部署文件见[prometheus-kvass](https://github.com/qingwave/kube-monitor/tree/master/prometheus-kvass)
```bash
git clone https://github.com/qingwave/kube-monitor.git
kubectl apply -f kube-monitor/prometheus-kvass
```
结果如下：
```bash
$ kubectl get po
NAME                                 READY   STATUS    RESTARTS   AGE
kvass-coordinator-7f65c546d9-vxgxr   2/2     Running   2          29h
metrics-774949d94d-4btzh             1/1     Running   0          10s
metrics-774949d94d-558gn             1/1     Running   1          29h
metrics-774949d94d-gs8kc             1/1     Running   1          29h
metrics-774949d94d-r85rc             1/1     Running   1          29h
metrics-774949d94d-xhbk9             1/1     Running   0          10s
metrics-774949d94d-z5mwk             1/1     Running   1          29h
prometheus-rep-0-0                   3/3     Running   0          49s
prometheus-rep-0-1                   3/3     Running   0          48s
prometheus-rep-0-2                   3/3     Running   0          19s
thanos-query-b469b648f-ltxth         1/1     Running   0          60s
thanos-rule-0                        1/1     Running   2          25h
```

Deployment `metrics`有6个副本，每个生成10045 series，`kvass-coordinator`配置每个分区最大series为30000，以及Prometheus默认的指标，需要3个Prometheus分片。

每个分片包含2个target
```bash
prometheus_tsdb_head_chunks{instance="127.0.0.1:9090",job="prometheus_shards",replicate="prometheus-rep-0-0",shard="0"}	20557
```

通过`Thanos Query`可以查询到多个Prometheus分片的数据，以及聚合规则`metrics_count`
![thanos-query](/img/blogImg/thanos-query.png)

### 待优化问题
此方案可满足绝大部分场景，用户可通过自己的实际环境配合不同的组件，但也存在一些需要优化确认的问题
- `Thanos Ruler`不支持远程写接口，只能存储于Thanos提供的对象存储中
- `Thanos Query`全局查询依赖多个下游组件，可能只返回部分结果挺好使
- `Coordinator`性能需要压测验证

## 总结
`Kvass+Thanos+Remote-write`可以实现Prometheus集群的自动分区、全局查询、数据持久化等功能，满足绝大部分场景。虽然有一些问题需要验证优化，但瑕不掩瑜，能够解决原生Prometheus扩展性问题。

## 引用
- https://qingwave.github.io/prometheus-federation/
- https://github.com/tkestack/kvass
- https://github.com/thanos-io/thanos

