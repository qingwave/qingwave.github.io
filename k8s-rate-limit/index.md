# kubernetes apiserver限流方案


## 背景
为了防止突发流量影响apiserver可用性，k8s支持多种限流配置，包括：
- MaxInFlightLimit，server级别整体限流
- Client限流
- EventRateLimit, 限制event
- APF，更细力度的限制配置

### MaxInFlightLimit
MaxInFlightLimit限流，apiserver默认可设置最大并发量（集群级别，区分只读与修改操作），通过参数`--max-requests-inflight`和 `--max-mutating-requests-inflight`， 可以简单实现限流。

### Client限流
例如client-go默认的qps为5，但是只支持客户端限流，集群管理员无法控制用户行为。

### EventRateLimit
EventRateLimit在1.13之后支持，只限制event请求，集成在apiserver内部webhoook中，可配置某个用户、namespace、server等event操作限制，通过webhook形式实现。

具体原理可以参考[提案](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#eventratelimit  
)，每个eventratelimit 配置使用一个单独的令牌桶限速器，每次event操作，遍历每个匹配的限速器检查是否能获取令牌，如果可以允许请求，否则返回`429`。

**优点**
- 实现简单，允许一定量的并发
- 可支持server/namespace/user等级别的限流

**缺点**
- 仅支持event，通过webhook实现只能拦截修改类请求
- 所有namespace的限流相同，没有优先级

### API 优先级和公平性
apiserver默认的限流方式太过简单，一个错误的客户端发送大量请求可能造成其他客户端请求异常，也不支持突发流量。

API 优先级和公平性（APF）是MaxInFlightLimit限流的一种替代方案，设计文档见[提案](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1040-priority-and-fairness)。

API 优先级和公平性（1.15以上，alpha版本）， 以更细粒度（byUser，byNamespace）对请求进行分类和隔离。 支持突发流量，通过使用公平排队技术从队列中分发请求从而避免饥饿。

APF限流通过两种资源，`PriorityLevelConfigurations`定义隔离类型和可处理的并发预算量，还可以调整排队行为。 `FlowSchemas`用于对每个入站请求进行分类，并与一个 `PriorityLevelConfigurations`相匹配。

可对用户或用户组或全局进行某些资源某些请求的限制，如限制default namespace写services put/patch请求。

**优点**
- 考虑情况较全面，支持优先级，白名单等
- 可支持server/namespace/user/resource等细粒度级别的限流

**缺点**
- 配置复杂，不直观，需要对APF原理深入了解
- 功能较新，缺少生产环境验证

**APF测试**
开启APF，需要在apiserver配置`--feature-gates=APIPriorityAndFairness=true --runtime-config=flowcontrol.apiserver.k8s.io/v1alpha1=true`

开启后，获取默认的FlowSchemas

```bash
$ kubectl get flowschemas.flowcontrol.apiserver.k8s.io 
NAME                           PRIORITYLEVEL     MATCHINGPRECEDENCE   DISTINGUISHERMETHOD   AGE    MISSINGPL
system-leader-election         leader-election   100                  ByUser                152m   False
workload-leader-election       leader-election   200                  ByUser                152m   False
system-nodes                   system            500                  ByUser                152m   False
kube-controller-manager        workload-high     800                  ByNamespace           152m   False
kube-scheduler                 workload-high     800                  ByNamespace           152m   False
kube-system-service-accounts   workload-high     900                  ByNamespace           152m   False
health-for-strangers           exempt            1000                 <none>                151m   False
service-accounts               workload-low      9000                 ByUser                152m   False
global-default                 global-default    9900                 ByUser                152m   False
catch-all                      catch-all         10000                ByUser                152m   False
```

FlowShema配置
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1alpha1
kind: FlowSchema
metadata:
  name: health-for-strangers
spec:
  matchingPrecedence: 1000 #匹配优先级，1~1000，越小优先级越高
  priorityLevelConfiguration: #关联的PriorityLevelConfigurations
    name: exempt #排除rules，即不限制当前flowshema的rules
  rules: #请求规则
  - nonResourceRules: #非资源
    - nonResourceURLs:
      - "/healthz"
      - "/livez"
      - "/readyz"
      verbs:
      - "*"
    subjects: #对应的用户或用户组
    - kind: Group
      group:
        name: system:unauthenticated
```

PriorityLevelConfiguration配置
```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1alpha1
kind: PriorityLevelConfiguration
metadata:
  name: leader-election
spec:
  limited: #限制策略
    assuredConcurrencyShares: 10 
    limitResponse: #如何处理被限制的请求
      queuing: #类型为Queue时，列队的设置
        handSize: 4 #队列
        queueLengthLimit: 50 #队列长度
        queues: 16 #队列数
      type: Queue #Queue或者Reject，Reject直接返回429，Queue将请求加入队列
  type: Limited #类型，Limited或Exempt， Exempt即不限制
```

## 总结
以上是k8s相关的限流策略，通过多种策略来保证集群的稳定性。

目前MaxInFlightLimit可以轻松开启，但是限制策略不精细，而APF功能较新，实现较复杂，在充分验证后，可通过APF对全集群进行限流。

