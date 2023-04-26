# ingress nginx benchmark


Ingress是目前Kubernetes集群流量接入的重要入口，了解其性能指标有助于用户选用合适的网络方案。
<!--more-->

## 测试方案
通过wrk压测后端nginx服务，对比ingress-nginx, 原生nginx，以及直连后端性能的差异，如下图:
![](/img/blogImg/ingress-benchmark1.png)

- 方案1，经过ingress
- 方案2，经过nginx
- 方案3，直连ip

### 硬件环境
- CPU： 2x  Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz, 32 cores
- Network： 10-Gigabit
- Memory： 128 GB

### 测试工具
- wrk, 4.1.0, 在k8s master测试，减少网络影响
- ingress-nginx, 0.30.0, https://github.com/kubernetes/ingress-nginx
- nginx, 1.13.5 
- k8s, v1.14.9 
- centos, 7.3.1611(Linux 4.9.2)

### 测试方法

ingress-nginx主要工作是转发请求到后端pod, 我们着重对其RPS（每秒请求量）进行测试

通过以下命令
```yaml
wrk -t4 -c1000 -d120s --latency http://my.nginx.svc/1kb.bin
```

## 测试结果

### 不同cpu下的性能

对比不同ingress-nginx启动不同worker数量的性能差异，以下测试ingress-nginx开启了keepalive等特性

| CPU | RPS    |
|-----|--------|
| 1   | 5534   |
| 2   | 11203  |
| 4   | 22890  |
| 8   | 47025  |
| 16  | 93644  |
| 24  | 125990 |
| 32  | 153473 |

![](/img/blogImg/ingress-benchmark2.png)

如图所示，不同cpu下，ingress的rps与cpu成正比，cpu在16核之后增长趋势放缓。

### 不同方案的性能对比

|方案|RPS|备注|
|-----|--------|---------|
|ingress-nginx(原始)|69171||
|ingress-nginx(配置优化)|153473|调整worker，access-log, keepalive等|
|nginx	|336769|开启keepalive, 关闭log|
|直连ip	|340748|测试中的pod ip为真实ip|

通过实验可以看到，使用nginx代理和直连ip，rps相差不大；原始ingress-nginx rps很低，优化后rps提升一倍，但对比nginx还是有较大的性能差异。

## 结论

默认ingress-nginx性能较差，配置优化后也只有15w RPS，对比原生nginx（33W) 差距较大。经过分析主要瓶颈在于ingress-nginx的lua过滤脚本，具体原因需要进一步分析。

## 参考
1. https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#upstream-keepalive-connections
2. https://www.nginx.com/blog/testing-performance-nginx-ingress-controller-kubernetes/

## 配置文件

本测试所有配置见[qingwave/ingress-nginx-benchmark](https://github.com/qingwave/ingress-nginx-benchmark)
