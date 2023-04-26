# Kubernetes中Sidecar生命周期管理


## 背景
在多个容器的Pod中，通常业务容器需要依赖sidecar。启动时sidecar需要先启动，退出时sidecar需要在业务容器退出后再退出。k8s目前对于sidecar的生命周期比较有争议，见[issue](https://github.com/kubernetes/enhancements/issues/753)、[sidecarcontainers](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/0753-sidecarcontainers.md)。

Kubernetes Pod 内有两种容器: 初始化容器(init container)和应用容器(app container)。

其中初始化容器的执行先于应用容器，按顺序启动，执行成功启动下一个：
```go
    if container := podContainerChanges.NextInitContainerToStart; container != nil {
        // Start the next init container.
        if err := start("init container", containerStartSpec(container)); err != nil {
            return
        }

        // Successfully started the container; clear the entry in the failure
        klog.V(4).Infof("Completed init container %q for pod %q", container.Name, format.Pod(pod))
    }
```

而对于应用容器，无法保证容器ready顺序，启动代码如下:
```go
    // Step 7: start containers in podContainerChanges.ContainersToStart.
    for _, idx := range podContainerChanges.ContainersToStart {
        // start函数向docker发请求启动容器，这里没有检测函数返回而且不确定ENTRYPOINT是否成功
        start("container", containerStartSpec(&pod.Spec.Containers[idx]))
    }
```

在删除时，同样无法保证删除顺序，代码如下
```go
    for _, container := range runningPod.Containers {
        go func(container *kubecontainer.Container) {
            killContainerResult := kubecontainer.NewSyncResult(kubecontainer.KillContainer, container.Name)
            // 每一个容器起goroutine执行删除
            if err := m.killContainer(pod, container.ID, container.Name, "", gracePeriodOverride); err != nil {
               ...
            }
            containerResults <- killContainerResult
        }(container)
    }
```

## 启动顺序
k8s原生方式，对于pod中一个容器依赖另一个容器，目前需要业务进程判断依赖服务是否启动或者sleep 10s，这种方式可以工作，但不太优雅。需要业务更改启动脚本。

那么，有没有其他的解决办法？

### 源码分析
在启动时，start函数调用startContainer来创建容器，主要代码如下：
```go
func (m *kubeGenericRuntimeManager) startContainer(podSandboxID string, podSandboxConfig *runtimeapi.PodSandboxConfig, spec *startSpec, pod *v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, podIP string, podIPs []string) (string, error) {
    container := spec.container

    // Step 1: 拉镜像.
    imageRef, msg, err := m.imagePuller.EnsureImageExists(pod, container, pullSecrets, podSandboxConfig)
    if err != nil {
        ...
     }

    // Step 2: 调用cri创建容器
    // For a new container, the RestartCount should be 0
    containerID, err := m.runtimeService.CreateContainer(podSandboxID, containerConfig, podSandboxConfig)
   ...

    // Step 3: 启动容器
    err = m.runtimeService.StartContainer(containerID)

    // Step 4: 执行 post start hook.
    if container.Lifecycle != nil && container.Lifecycle.PostStart != nil {
        kubeContainerID := kubecontainer.ContainerID{
            Type: m.runtimeName,
            ID:   containerID,
        }
        // 调用Run来执行hook
        msg, handlerErr := m.runner.Run(kubeContainerID, pod, container, container.Lifecycle.PostStart)
        ...
    }

    return "", nil
}
```

步骤如下：
1. 拉取镜像
2. 创建容器
3. 启动容器
4. 执行hook

一个Pod中容器的启动是有顺序的，排在前面容器的先启动。同时第一个容器执行完ENTRYPOINT和PostStart之后（异步执行，无法确定顺序），k8s才会创建第二个容器（这样的话就可以保证第一个容器创建多长时间后再启动第二个容器）

如果我们PostStart阶段去检测容器是否ready，那么只有在ready后才去执行下一个容器。

![sidecar-lifecycle](/img/blogImg/sidecar-lifecycle.png)

### 测试

配置如下，sidecar模拟需要依赖的容器，main为业务容器
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-start
spec:
  containers:
  - name: sidecar
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "sleep 20"]
  - name: main
    image: busybox
    command: ["/bin/sh", "-c", "sleep 3600"]
```

得到结果如下，可以看到sidecar启动21s后才开始启动main容器，满足需求
```bash
Events:
  Type    Reason     Age   From                                          Message
  ----    ------     ----  ----                                          -------
  Normal  Scheduled  54s   default-scheduler                             Successfully assigned default/test-start to tj1-staging-k8s-slave95-202008.kscn
  Normal  Pulling    53s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Pulling image "busybox"
  Normal  Pulled     44s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Successfully pulled image "busybox"
  Normal  Created    44s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Created container sidecar
  Normal  Started    44s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Started container sidecar
  Normal  Pulling    23s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Pulling image "busybox"
  Normal  Pulled     19s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Successfully pulled image "busybox"
  Normal  Created    18s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Created container main
  Normal  Started    18s   kubelet, tj1-staging-k8s-slave95-202008.kscn  Started container main
```

此方案可能存在的缺点：
1. 如果sidecar启动失败或者hook失败，其他容器会立即启动

## 退出顺序
容器启动顺序比较好解决，退出顺序则是按照相反的顺序，业务容器先退出，之后sidecar再退出。

目前，在kubelet删除pod步骤如下;
1. 遍历容器，每个容器起一个goroutine删除
2. 删除时，先执行pre stop hook，得到gracePeriod=DeletionGracePeriodSeconds-period(stophook)
3. 再调用cri删除接口m.runtimeService.StopContainer(containerID.ID, gracePeriod)

如果在sidecar的pre stop hook检测业务容器状态，那么可以延迟退出。

### 测试
业务容器main退出时，创建文件；sidecar通过post-stop检测到文件后，执行退出
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-stop
spec:
  containers:
  - name: sidecar
    image: busybox
    command: 
    - "/bin/sh"
    - "-c"
    - |
      trap "touch /lifecycle/sidecar-terminated" 15
      until [ -f "/lifecycle/sidecar-terminated" ];do
        date
        sleep 1
      done
      sleep 5
      cat /lifecycle/main-terminated
      t=$(date)
      echo "sidecar exit at $t"
    lifecycle:
      preStop:
        exec:
          command:
          - "/bin/sh"
          - "-c"
          - |
            until [ -f "/lifecycle/main-terminated" ];do
              sleep 1
            done
            t=$(date)
            echo "main exit at $t" > /lifecycle/main-terminated
    volumeMounts:
    - name: lifecycle
      mountPath: /lifecycle
  - name: main
    image: busybox
    command: 
    - "/bin/sh"
    - "-c"
    - |
      trap "touch /lifecycle/main-terminated" 15
      until [ -f "/lifecycle/main-terminated" ];do
        date
        sleep 1
      done
    volumeMounts:
    - name: lifecycle
      mountPath: /lifecycle
  volumes:
  - name: lifecycle
    emptyDir: {}
```

在日志中看到，main容器先结束，sidecar检测到main-terminated文件后，执行完post-stop-hook，sidecar主进程开始退出
```bash
$ kubectl  logs -f test-stop main
...
Tue Sep  8 03:14:20 UTC 2020
Tue Sep  8 03:14:21 UTC 2020
Tue Sep  8 03:14:22 UTC 2020

$ kubectl  logs -f test-stop sidecar
Tue Sep  8 03:14:22 UTC 2020
Tue Sep  8 03:14:23 UTC 2020
# post stop hook 检测到main容器退出，记录日志
main exit at Tue Sep  8 03:14:23 UTC 2020
# sidecar主进程退出
sidecar exit at Tue Sep  8 03:14:29 UTC 2020
```

通过测试，使用postStopHook可以达到sidecar延迟退出的目的，但这种方式也有一些缺点
1. 配置复杂，多个sidecar都需要配置postStop监听业务容器状态
2. 业务容器需要有可观察性（提供特定形式的健康检测）
3. poststop执行异常，会等到最大优雅退出时间（默认30s）后才终止

## 总结
目前对于sidecar生命周期的支持方案对比如下：

| 方案              | 启动顺序 | 退出顺序 | job sidecar | 是否需要用户修改代码 | 是否需要修改k8s代码   | 缺点                                                 | 备注                 |
|-----------------|------|------|-------------|------------|---------------|----------------------------------------------------|--------------------|
| 用户控制            | 支持   | 不支持  | 不支持         | 需要         | 不需要           | 需要用户更改启动脚本;退出支持难度大，需要同时修改业务容器与sidecar启动脚本；大部分情况不支持 | 启动时需要检测sidecar服务状态 |
| Lifecycle Hooks | 支持   | 支持   | 不支持         | 不需要        | 不需要           | 配置hook复杂度高;在hook执行异常情况下不能确保顺序                      |                    |
| 富容器             | 支持   | 部分支持 | 部分支持        | 不需要        | 需要（更改镜像或启动命令） | 所有功能集成在一个容器中，对于外部sidecar如istio envoy等，不可控;          |                    |
| 修改源码            | 支持   | 支持   | 支持          | 不需要        | 需要            | 需要满足各种情况，实现难度较大                                    |社区有计划支持|

在k8s提供此类功能前，目前没有完善的方案。Lifecycle Hooks不需要更改用户启动代码以及k8s相关代码，相对于其他方式不失为一种解决思路。

