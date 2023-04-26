# 捕获Kubernetes中Pod驱逐事件


最近在工作中需要捕获Kubernetes的Pod驱逐事件，再做额外的操作。第一个想法是能不能监听（watch）驱逐对象（Eviction Resource），很遗憾Eviction并没有watch接口，只是Pod下的一个子资源，和Scale、Status类似。等等，既然是子资源那能不能通过Webhook获取。

## 实现

峰回路转，在[kubernetes#pr76910](https://github.com/kubernetes/kubernetes/pull/76910)中已经实现对pod/eviction子资源的支持。

简单验证一下

### 生成项目
通过kubebuilder快速生成项目
```
kubebuilder init --component-config --domain qinng.io --repo github.com/qingwave/k8s
-eviction-operator
```

### 编写Webhook

由于pod/eviction不是自定义资源，无法通过kubebuilder直接生成，可按照如下逻辑生成ValidatingAdmissionWebhook 
```golang
package webhook

import (
	"context"
	"fmt"
	"net/http"

	"github.com/go-logr/logr"
	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
	policyv1 "k8s.io/api/policy/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/webhook"
	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

const (
	WebhookName  = "Eviction"
	EvictionKind = "Eviction"
)

// rbac注解与webhook注解
// +kubebuilder:rbac:groups="",resources=pods,verbs=get;list
// +kubebuilder:webhook:path=/validate-v1-pod-eviction,admissionReviewVersions=v1;v1beta1,sideEffects=NoneOnDryRun,matchPolicy=Equivalent,mutating=false,failurePolicy=fail,groups="",resources=pods/eviction,verbs=create,versions=v1,name=veviction.kb.io

// EvictionValidator validates Pods Eviction event
type EvictionValidator struct {
	scheme  *runtime.Scheme
	client  client.Client
	log     logr.Logger
	decoder *admission.Decoder
}

// EvictionValidator 解析Eviction, 格式化返回
func (v *EvictionValidator) Handle(ctx context.Context, req admission.Request) admission.Response {
	logger := v.log.WithValues("eviction", fmt.Sprintf("%s/%s", req.Namespace, req.Name))

	logger.Info("start handle eviction")

	if req.Operation != admissionv1.Create {
		logger.Info(fmt.Sprintf("skip none create request, verb: %s", req.Operation))
		return admission.Allowed("")
	}

	if req.DryRun != nil && *req.DryRun {
		logger.Info("skip dry run request")
		return admission.Allowed("")
	}

	if req.Kind.Kind != EvictionKind {
		logger.Info(fmt.Sprintf("expected request %s but got %s", EvictionKind, req.Kind))
		return admission.Errored(http.StatusBadRequest, fmt.Errorf("unexpected kind %v", req.Kind))
	}

	eviction, err := v.getEviction(req)
	if err != nil {
		logger.Error(err, "failed to decode eviction")
		return admission.Errored(http.StatusBadRequest, err)
	}

	logger.Info(fmt.Sprintf("reveice new obj, obj: %+#v", *eviction))

	if eviction.DeleteOptions != nil && len(eviction.DeleteOptions.DryRun) > 0 {
		logger.Info("skip eviction dry run request")
		return admission.Allowed("")
	}

	if err := v.handleEviction(eviction); err != nil {
		return admission.Errored(http.StatusInternalServerError, err)
	}

	logger.Info("handle eviction success")

	return admission.Allowed("")
}

// EvictionValidator implements admission.DecoderInjector.
// A decoder will be automatically injected.

// InjectDecoder injects the decoder.
func (v *EvictionValidator) InjectDecoder(d *admission.Decoder) error {
	v.decoder = d
	return nil
}

// 解析Eviction
func (v *EvictionValidator) getEviction(req admission.Request) (*policyv1.Eviction, error) {
	obj := &unstructured.Unstructured{}
	if err := v.decoder.Decode(req, obj); err != nil {
		return nil, err
	}

	eviction := &policyv1.Eviction{}
	if err := v.scheme.Convert(obj, eviction, nil); err != nil {
		return nil, err
	}

	return eviction, nil
}

// 处理驱逐事件
func (v *EvictionValidator) handleEviction(eviction *policyv1.Eviction) error {
	podNamespacedName := types.NamespacedName{Namespace: eviction.Namespace, Name: eviction.Name}
	pod := &corev1.Pod{}

	if err := v.client.Get(context.TODO(), podNamespacedName, pod); err != nil {
		return err
	}

	v.log.Info(fmt.Sprintf("get eviction pod: %#v", pod))

	return nil
}

// 注册Webhook
func NewEvictionWebhook(mgr ctrl.Manager) error {
	w := &EvictionValidator{
		scheme: mgr.GetScheme(),
		client: mgr.GetClient(),
		log:    mgr.GetLogger().WithName(WebhookName),
	}

	mgr.GetWebhookServer().Register("/validate-v1-pod-eviction", &webhook.Admission{
		Handler: w,
	})

	return nil
}
```

特别注意的是，Eviction包括v1、v1beat1两个版本，解析时需要可以全部转换为v1方便处理
```golang
package webhook

import (
	policyv1 "k8s.io/api/policy/v1"
	policyv1beta1 "k8s.io/api/policy/v1beta1"
	"k8s.io/apimachinery/pkg/conversion"
	"k8s.io/apimachinery/pkg/runtime"
)

func RegisterConversion(s *runtime.Scheme) error {
	return s.AddConversionFunc((*policyv1beta1.Eviction)(nil), (*policyv1.Eviction)(nil), func(a, b interface{}, scope conversion.Scope) error {
		return ConvertV1beta1EvictionToV1Eviction(a.(*policyv1beta1.Eviction), b.(*policyv1.Eviction), scope)
	})
}

// 注册转换函数v1beta1->v1
func ConvertV1beta1EvictionToV1Eviction(in *policyv1beta1.Eviction, out *policyv1.Eviction, s conversion.Scope) error {
	out.ObjectMeta = in.ObjectMeta
	out.DeleteOptions = in.DeleteOptions.DeepCopy()

	return nil
}
```

至此，大体框架完成，需要部署建议使用cert-manager来注入证书，不需要自己手动再生成。

### 测试

通过[kubectl-evict](https://github.com/ueokande/kubectl-evict)驱逐pod，在operator日志中显示已捕获事件：
```log
1.6572752110781207e+09  DEBUG   controller-runtime.webhook.webhooks     received request        {"webhook": "/validate-v1-pod-eviction", "UID": "4f98f064-97c1-4ada-a9d8-0946afb11eba", "kind": "policy/v1beta1, Kind=Eviction", "resource": {"group":"","version":"v1","resource":"pods"}}
1.6572752110781898e+09  INFO    Eviction        start handle eviction   {"eviction": "default/nginx-6799fc88d8-drkc4"}
1.6572752110787241e+09  INFO    Eviction        reveice new obj, obj: v1.Eviction{TypeMeta:v1.TypeMeta{Kind:"", APIVersion:""}, ObjectMeta:v1.ObjectMeta{Name:"nginx-6799fc88d8-drkc4", GenerateName:"", Namespace:"default", SelfLink:"", UID:"", ResourceVersion:"", Generation:0, CreationTimestamp:time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC), DeletionTimestamp:<nil>, DeletionGracePeriodSeconds:(*int64)(nil), Labels:map[string]string(nil), Annotations:map[string]string(nil), OwnerReferences:[]v1.OwnerReference(nil), Finalizers:[]string(nil), ZZZ_DeprecatedClusterName:"", ManagedFields:[]v1.ManagedFieldsEntry{v1.ManagedFieldsEntry{Manager:"kubectl-evict", Operation:"Update", APIVersion:"policy/v1beta1", Time:time.Date(2022, time.July, 8, 10, 13, 31, 0, time.Local), FieldsType:"FieldsV1", FieldsV1:(*v1.FieldsV1)(0xc000479a10), Subresource:""}}}, DeleteOptions:(*v1.DeleteOptions)(0xc00060dc20)}    {"eviction": "default/nginx-6799fc88d8-drkc4"}
1.6572752111794329e+09  INFO    Eviction        get eviction pod: &v1.Pod{TypeMeta:v1.TypeMeta{Kind:"Pod", APIVersion:"v1"}, ObjectMeta:v1.ObjectMeta{Name:"nginx-6799fc88d8-drkc4"...
```

## 总结
通过Webhook可以实现对驱逐事件的捕捉，但也有一些地方需要注意
- 如果处理逻辑比较复杂，尽量通过Webhook生成其他资源如CRD，Controller监听CRD再来处理其他的处理，防止Webhook处理超时，而且Controller遇到异常会再次重试
- 目前Webhook对于Eviction子资源，无法通过`objectSelector`选择特定的Pod，除非调用者在Eviction对象中包含了Pod的Labels


> Explore more in [https://qingwave.github.io](https://qingwave.github.io)

