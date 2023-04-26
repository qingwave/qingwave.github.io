# 优化Kubernetes横向扩缩HPA

Pod水平自动扩缩（Horizontal Pod Autoscaler, 简称HPA）可以基于 CPU/MEM 利用率自动扩缩Deployment、StatefulSet 中的 Pod 数量，同时也可以基于其他应程序提供的自定义度量指标来执行自动扩缩。默认HPA可以满足一些简单场景，对于生产环境并不一定适合，本文主要分析HPA的不足与优化方式。
<!--more-->
## HPA Resource类型不足
默认HPA提供了Resource类型，通过CPU/MEM使用率指标（由metrics-server提供原始指标）来扩缩应用。
### 使用率计算方式
在Resource类型中，使用率计算是通过`request`而不是`limit`，源码如下：
```golang
// 获取Pod resource request
func calculatePodRequests(pods []*v1.Pod, resource v1.ResourceName) (map[string]int64, error) {
	requests := make(map[string]int64, len(pods))
	for _, pod := range pods {
		podSum := int64(0)
		for _, container := range pod.Spec.Containers {
			if containerRequest, ok := container.Resources.Requests[resource]; ok {
				podSum += containerRequest.MilliValue()
			} else {
				return nil, fmt.Errorf("missing request for %s", resource)
			}
		}
		requests[pod.Name] = podSum
	}
	return requests, nil
}
// 计算使用率
func GetResourceUtilizationRatio(metrics PodMetricsInfo, requests map[string]int64, targetUtilization int32) (utilizationRatio float64, currentUtilization int32, rawAverageValue int64, err error) {
	metricsTotal := int64(0)
	requestsTotal := int64(0)
	numEntries := 0

	for podName, metric := range metrics {
		request, hasRequest := requests[podName]
		if !hasRequest {
			// we check for missing requests elsewhere, so assuming missing requests == extraneous metrics
			continue
		}

		metricsTotal += metric.Value
		requestsTotal += request
		numEntries++
	}


	currentUtilization = int32((metricsTotal * 100) / requestsTotal)

	return float64(currentUtilization) / float64(targetUtilization), currentUtilization, metricsTotal / int64(numEntries), nil
}
```
通常在Paas平台中会对资源进行超配，`limit`即用户请求资源，`request`即实际分配资源，如果按照request来计算使用率（会超过100%）是不符合预期的。相关issue见[72811](https://github.com/kubernetes/kubernetes/issues/72811)，目前还存在争论。可以修改源码，或者使用自定义指标来代替。

### 多容器Pod使用率问题
默认提供的`Resource`类型的HPA，通过上述方式计算资源使用率，核心方式如下：
```golang
metricsTotal = sum(pod.container.metricValue)
requestsTotal = sum(pod.container.Request)
currentUtilization = int32((metricsTotal * 100) / requestsTotal)
```
计算出所有`container`的资源使用量再比总的申请量，对于单容器Pod这没影响。但对于多容器Pod，比如Pod包含多个容器con1、con2(request都为1cpu)，con1使用率10%，con2使用率100%，HPA目标使用率60%，按照目前方式得到使用率为55%不会进行扩容，但实际con2已经达到资源瓶颈，势必会影响服务质量。当前系统中，多容器Pod通常都是1个主容器与多个sidecar，依赖主容器的指标更合适点。

好在*1.20*版本中已经支持了[ContainerResource](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/#container-resource-metrics)可以配置基于某个容器的资源使用率来进行扩缩，如果是之前的版本建议使用自定义指标替换。


## 性能问题
### 单线程架构
默认的`hpa-controller`是单个Goroutine执行的，随着集群规模的增多，势必会成为性能瓶颈，目前默认hpa资源同步周期会`15s`，假设每个metric请求延时为`100ms`，当前架构只能支持`150`个HPA资源（保证在15s内同步一次）
```golang
func (a *HorizontalController) Run(stopCh <-chan struct{}) {
  // ...
	// start a single worker (we may wish to start more in the future)
	go wait.Until(a.worker, time.Second, stopCh)

	<-stopCh
}
```
可以通过调整`worker`数量来横向扩展，已提交[PR](https://github.com/kubernetes/kubernetes/pull/99688)。

### 调用链路
在`hpa controller`中一次hpa资源同步，需要调用多次apiserver接口，主要链路如下
1. 通过`scaleForResourceMappings`得到scale资源
2. 调用`computeReplicasForMetrics`获取metrics value
3. 调用`Scales().Update`更新计算出的副本数

尤其在获取metrics value时，需要先调用apiserver，apiserver调用metrics-server/custom-metrics-server，当集群内存在大量hpa时可能会对apiserver性能产生一定影响。

## 其他
对于自定义指标用户需要实现`custom.metrics.k8s.io`或`external.metrics.k8s.io`，目前已经有部分开源实现见[custom-metrics-api](https://github.com/kubernetes/metrics/blob/master/IMPLEMENTATIONS.md#custom-metrics-api)。

另外，hpa核心的扩缩算法根据当前指标和期望指标来计算扩缩比例，并不适合所有场景，只使用线性增长的指标。
```
期望副本数 = ceil[当前副本数 * (当前指标 / 期望指标)]
```
[watermarkpodautoscaler](https://github.com/DataDog/https://github.com/DataDog/watermarkpodautoscaler)提供了更灵活的扩缩算法，比如平均值、水位线等，可以作为参考。

## 总结
Kubernetes提供原生的HPA只能满足一部分场景，如果要上生产环境，必须对其做一些优化，本文总结了当前HPA存在的不足，例如在性能、使用率计算方面，并提供了解决思路。

