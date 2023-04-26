# k8s节点资源不足时OutOfcpu错误

## 环境
k8s: 1.10.2
docker: 17.03
## 问题
当指定nodeName并且节点资源不足时，会创建大量pod，并显示`outofcpu/outofmem`
类似下面:
```bash
prometheus-slave01-68bd9bc854-slw92 0/2 OutOfcpu 0 1m
prometheus-slave01-68bd9bc854-svxbq 0/2 OutOfcpu 0 20s
prometheus-slave01-68bd9bc854-sw25t 0/2 OutOfcpu 0 1m
```
## 相关issue
https://github.com/kubernetes/kubernetes/issues/38806
## 解析
设置nodeName会跳过调度，没有对容量做检测
分配到节点上显示资源不足，状态变为outofcpu/outofmem，k8s判断replicaset没有检测到期望pod的状态，会重新再起一个pod，而原pod不会主动删除，致使创建大量pod
## 附件
测试yaml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: my-nginx
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: my-nginx
    spec:
      nodeName: tj1-jm-cc-stag05.kscn
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 200
          requests:
            cpu: 100
```
