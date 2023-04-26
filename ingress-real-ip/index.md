# Ingress获取真实IP


一般情况下，经过ingress的请求会携带header`X-Real-IP`，用户可根据header解析出真实访问IP。

特殊情况，用户请求可能经过多个nginx才达到ingress, 通过上述方法得到的并不是用户的真实IP。
> request -> nginx -> ... -> ingress-nginx -> backend

## 方案1 use-forwarded-headers
nginx-ingress官方的建议是开启[use-forwarded-headers](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#forwarded-for-header), 配置如下：

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
data:
  compute-full-forwarded-for: 'true'
  use-forwarded-headers: 'true'
```

## 方案2 real_ip_header
这种方式确实可以起作用，但是有用户反馈开启后访问ingres后端服务一直报`308`，检查了ingress的代码开启`use-forwarded-headers`后会同时开启`ssl-redirect`导致308。

那么我们只需要开启nginx配置中的相关real-ip的配置，如下在`http-snippet`添加`real_ip_header X-Forwarded-For;`

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
data:
  http-snippet: |
    real_ip_header X-Forwarded-For;
```

## golang中获取真实ip

```go
func RemoteIP(r *http.Request) string {
  // ingress 行为，将真实ip放到header `X-Original-Forwarded-For`, 普通nginx可去掉此条
	ip := strings.TrimSpace(strings.Split(r.Header.Get("X-Original-Forwarded-For"), ",")[0])
	if ip != "" {
		return ip
	}

	ip = strings.TrimSpace(strings.Split(r.Header.Get("X-Forwarded-For"), ",")[0])
	if ip != "" {
		return ip
	}

	ip = strings.TrimSpace(r.Header.Get("X-Real-Ip"))
	if ip != "" {
		return ip
	}

	if ip, _, err := net.SplitHostPort(strings.TrimSpace(r.RemoteAddr)); err == nil {
		return ip
	}

	return ""
}
```

## 注意
nginx-ingress configmap中的配置会是全局生效的，上线前需要严格测试。
