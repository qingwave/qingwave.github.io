# 如何做一个优雅的Pod


没有人不想优雅的活着，在这喧闹的生活中过得优雅从容并不容易。但在k8s的世界中，如何做个优雅的Pod还是有套路可循的。
## Pod的生命周期
在优雅之前，我们先谈谈Pod的一生，大体分为以下几个阶段
1. 创建，通过kubectl或者api创建pod, apiserver收到请求后存储到etcd
2. 调度，scheduler检测到pod创建后，通过预选优选为pod选取合适的人家(node)
3. 启动，kubelet检测到有pod调度到当前节点，开始启动pod
4. 终止，不同的pod有不同的谢幕方式，有的正常运行结束没有restart就completed，有的被kill就入土为安了，有的被驱逐换种方式重新开始

今天我们主要讨论3-4阶段，前面部分更多是deployment/daemonset这些pod的父母所决定的。

## 优雅的启动
### init container
通常pod有一些初始化操作，创建文件夹，初始化磁盘，检查某些依赖服务是不是正常，这些操作放在代码中会污染代码，写在启动命令中不方便管理，出问题也不方便排查，更优雅的方式是使用k8s的[init container][1]。

**理解 Init 容器**
Pod 可以包含多个容器，应用运行在这些容器里面，同时 Pod 也可以有一个或多个先于应用容器启动的 Init 容器。

Init 容器与普通的容器非常像，除了如下两点：
- 它们总是运行到完成。
- 每个都必须在下一个启动之前成功完成。
如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果 Pod 对应的 restartPolicy 值为 Never，它不会重新启动。

如果为一个 Pod 指定了多个 Init 容器，这些容器会按顺序逐个运行。每个 Init 容器必须运行成功，下一个才能够运行。当所有的 Init 容器运行完成时，Kubernetes 才会为 Pod 初始化应用容器并像平常一样运行。

**Init 容器能做什么？**
因为 Init 容器具有与应用容器分离的单独镜像，其启动相关代码具有如下优势：
- Init 容器可以包含一些安装过程中应用容器中不存在的实用工具或个性化代码。例如，没有必要仅为了在安装过程中使用类似 sed、 awk、 python 或 dig 这样的工具而去FROM 一个镜像来生成一个新的镜像。
- Init 容器可以安全地运行这些工具，避免这些工具导致应用镜像的安全性降低。
应用镜像的创建者和部署者可以各自独立工作，而没有必要联合构建一个单独的应用镜像。
Init 容器能以不同于Pod内应用容器的文件系统视图运行。因此，Init容器可具有访问 Secrets 的权限，而应用容器不能够访问。
- 由于 Init 容器必须在应用容器启动之前运行完成，因此 Init 容器提供了一种机制来阻塞或延迟应用容器的启动，直到满足了一组先决条件。一旦前置条件满足，Pod内的所有的应用容器会并行启动。
  
**示例**
下面的例子定义了一个具有 2 个 Init 容器的简单 Pod。 第一个等待 myservice 启动，第二个等待 mydb 启动。 一旦这两个 Init容器 都启动完成，Pod 将启动spec区域中的应用容器。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```

### readinessProbe
pod启动后，如果直接加入endpoint，有可能服务还没初始化完成，端口没有就绪，这时候接收流量肯定无法正常处理。如果能判断pod是否ready就好了，当当当，readiness来了，可以通过http，tcp以及执行命令的方式来检查服务情况，检查成功后再将pod状态设置为ready,ready后才会加入到endpoint中。

下为一个readiness探测，5秒执行一次命令，执行成功则pod变为ready
```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
  ```

> **注**
  - http, tcp探针是kubelet执行的，所以无法探测容器中localhost的端口，也无法解析service
  - exec则在容器内执行的

### ReadinessGates
ReadinessProbe机制可能无法满足某些复杂应用对容器内服务可用状态的判断，所以kubernetes从1.11版本开始引入了`Pod Ready++`特性对Readiness探测机制进行扩展，在1.14版本时达到GA稳定版本，称其为`Pod Readiness Gates`。

通过Pod Readiness Gates机制，用户可以将自定义的ReadinessProbe探测方式设置在Pod上，辅助kubernetes设置Pod何时达到服务可用状态Ready，为了使自定义的ReadinessProbe生效，用户需要提供一个外部的控制器Controller来设置相应的Condition状态。Pod的Readiness Gates在pod定义中的ReadinessGates字段进行设置，

如下示例设置了一个类型为`www.example.com/feature-1`的新Readiness Gates：
```yaml
Kind: Pod
spec:
  readinessGates:
    - conditionType: "www.example.com/feature-1"
status:
  conditions:
    - type: Ready  # kubernetes系统内置的名为Ready的Condition
      status: "True"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
    - type: "www.example.com/feature-1"   # 用户定义的Condition
      status: "False"
      lastProbeTime: null
      lastTransitionTime: 2018-01-01T00:00:00Z
  containerStatuses:
    - containerID: docker://abcd...
      ready: true
```
新增的自定义Condition的状态status将由用户自定义的外部控制器设置，默认值为False，kubernetes将在判断全部readinessGates条件都为True时，才设置pod为服务可用状态（Ready或True）。

### poststart
另外也可以通过`poststart`设置hook操作，做一些额外工作。k8s在容器创建后立即发送 postStart 事件。然而，postStart 处理函数的调用不保证早于容器的入口点（entrypoint） 的执行。postStart 处理函数与容器的代码是异步执行的，但 Kubernetes 的容器管理逻辑会一直阻塞等待 postStart 处理函数执行完毕。只有 postStart 处理函数执行完毕，容器的状态才会变成`RUNNING`。

## 优雅的运行
### livenessProbe
同readinessProbe探针，livenessProbe是用来检查pod运行状态是否正常，如果探测失败，pod被kill掉，重启启动pod。

### restartpolicy
如果pod运行时意外退出(程序故障)，kubelet会根据restart policy来判断是否重启pod，可能的值为 Always、OnFailure 和 Never。默认为 Always，如果容器退出会再再启动，pod启动次数加1。

## 优雅的结束
首先谈下pod的删除流程：
1. 用户发送命令删除 Pod，使用的是默认的宽限期（grace period 30秒）
2. apiserver中的 Pod 会随着宽限期规定的时间进行更新，过了这个时间 Pod 就会被认为已"dead"
3. 当使用客户端命令查询 Pod 状态时，Pod 显示为 “Terminating”
4. （和第 3 步同步进行）当 Kubelet 看到 Pod 由于步骤 2 中设置的时间而被标记为 terminating 状态时，它就开始执行关闭 Pod 流程
  - 如果 Pod 定义了 preStop 钩子，就在 Pod 内部调用它。如果宽限期结束了，但是 preStop 钩子还在运行，那么就用小的（2 秒）扩展宽限期调用步骤 2。
  - 给 Pod 内的进程发送 `TERM` 信号(即`kill`, `kill -15`)。请注意，并不是所有 Pod 中的容器都会同时收到 TERM 信号，如果它们关闭的顺序很重要，则每个容器可能都需要一个 preStop 钩子。
5. （和第 3 步同步进行）从服务的`endpoint`列表中删除 Pod，Pod 也不再被视为副本控制器的运行状态的 Pod 集的一部分。因为负载均衡器（如服务代理）会将其从轮换中删除，所以缓慢关闭的 Pod 无法继续为流量提供服务。
6. 当宽限期到期时，仍在 Pod 中运行的所有进程都会被`SIGKILL`(即`kill -9`)信号杀死。

### 捕捉SIGTERM
如果pod没有捕捉`SIGTERM`信号就直接退出，有些请求还没处理完，这势必影响服务质量，所以需要优雅退出，很多库都提供了类似的功能，当接受到退出信号时，清理空闲链接，等待当前请求处理完后再退出。如果善后工作较长，比较适当增加`terminationGracePeriodSeconds`的时间。

### prestop
另外也可以通过`prestop`设置hook操作，做一些额外的清理工作，
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```
命令 preStop 负责优雅地终止 nginx 服务。当因为失效而导致容器终止时，这一处理方式很有用。

> **注**
  Kubernetes 只有在 Pod 结束（Terminated） 的时候才会发送 preStop 事件，这意味着在 Pod 完成（Completed） 时 preStop 的事件处理逻辑不会被触发。

## 总结
优雅就不要怕麻烦，来我们总结下优雅的秘诀：
1. 需要初始化的操作使用initcontainer来做
2. 就绪检查，探活检查少不了,必要时也可以配置ReadinessGates
3. 优雅退出要处理`SIGTERM`
4. 需要时也可以设置下poststart, prestop
5. 其他的，设置limit/reqeust也是必须的

## 引用
- https://kubernetes.io/zh/docs/concepts/workloads/pods/init-containers/
- https://kubernetes.io/zh/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/
