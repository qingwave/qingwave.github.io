# 可能是史上最全的Kubernetes证书解析


为了避免广告法，题目还是加个可能吧。

想要安全就必须复杂起来，证书是少不了的。在Kubernetes中提供了非常丰富的证书类型，满足各种不同场景的需求，今天我们就来看一看Kubernetes中的证书。

## k8s证书分类
在说证书之前，先想想作为集群的入口apiserver需要提供那些服务，与那些组件通信，通信的两方可能需要配置证书。
与apiserver通信的组件大体可以分为以下几类：
- client(kubectl，restapi等)：普通用户与apiserver之间的通信，对各类资源进行操作
- kubelet，kubeproxy：master与node之间的通信
- etcd：k8s的存储库
- webhook：这里指apiserver提供的admission-webhook，在数据持久化前调用webhook
- aggregation layer：扩展apiserver, 需要将自定义的api注册到k8s中，相比CRD性能更新
- pod: 在pod中调用apiserver(一般调用为10.254.0.1:433)

居然有这么多种，除了在pod中通过serviceacount认证（当然pod需要认证apiserver的证书），其他几种都需要配置证书。

其他集群内组件与apiserver通信的，kubelet/etcd/kube-proxy对应的也可以配置证书。

## apiserver证书
简单列举下apiserver证书相关的启动参数
```
--cert-dir string                           The directory where the TLS certs are located. If --tls-cert-file and --tls-private-key-file are provided, this flag will be ignored. (default "/var/run/kubernetes")
--client-ca-file string                     If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.
--etcd-certfile string                      SSL certification file used to secure etcd communication.
--etcd-keyfile string                       SSL key file used to secure etcd communication.
--kubelet-certificate-authority string      Path to a cert file for the certificate authority.
--kubelet-client-certificate string         Path to a client cert file for TLS.
--kubelet-client-key string                 Path to a client key file for TLS.
--proxy-client-cert-file string             Client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins. It is expected that this cert includes a signature from the CA in the --requestheader-client-ca-file flag. That CA is published in the 'extension-apiserver-authentication' configmap in the kube-system namespace. Components recieving calls from kube-aggregator should use that CA to perform their half of the mutual TLS verification.
--proxy-client-key-file string              Private key for the client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins.
--requestheader-allowed-names stringSlice   List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers. If empty, any client certificate validated by the authorities in --requestheader-client-ca-file is allowed.
--requestheader-client-ca-file string       Root certificate bundle to use to verify client certificates on incoming requests before trusting usernames in headers specified by --requestheader-username-headers
--service-account-key-file stringArray      File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. If unspecified, --tls-private-key-file is used. The specified file can contain multiple keys, and the flag can be specified multiple times with different files.
--ssh-keyfile string                        If non-empty, use secure SSH proxy to the nodes, using this user keyfile
--tls-ca-file string                        If set, this certificate authority will used for secure access from Admission Controllers. This must be a valid PEM-encoded CA bundle. Alternatively, the certificate authority can be appended to the certificate provided by --tls-cert-file.
--tls-cert-file string                      File containing the default x509 Certificate for HTTPS. (CA cert, if any, concatenated after server cert). If HTTPS serving is enabled, and --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to /var/run/kubernetes.
--tls-private-key-file string               File containing the default x509 private key matching --tls-cert-file.
--tls-sni-cert-key namedCertKey             A pair of x509 certificate and private key file paths, optionally suffixed with a list of domain patterns which are fully qualified domain names, possibly with prefixed wildcard segments. If no domain patterns are provided, the names of the certificate are extracted. Non-wildcard matches trump over wildcard matches, explicit domain patterns trump over extracted names. For multiple key/certificate pairs, use the --tls-sni-cert-key multiple times. Examples: "example.crt,example.key" or "foo.crt,foo.key:*.foo.com,foo.com". (default [])
--oidc-ca-file string                       If set, the OpenID server's certificate will be verified by one of the authorities in the oidc-ca-file, otherwise the host's root CA set will be used.
--tls-sni-cert-key namedCertKey             A pair of x509 certificate and private key file paths, optionally suffixed with a list of domain patterns which are fully qualified domain names, possibly with prefixed wildcard segments. If no domain patterns are provided, the names of the certificate are extracted. Non-wildcard matches trump over wildcard matches, explicit domain patterns trump over extracted names. For multiple key/certificate pairs, use the --tls-sni-cert-key multiple times. Examples: "example.crt,example.key" or "foo.crt,foo.key:*.foo.com,foo.com". (default [])
```
不要害怕，咱们一个个看。

### tls证书
首先，apiserver本身是一个http服务器，需要tls证书
```
--tls-cert-file string
    File containing the default x509 Certificate for HTTPS. (CA cert, if any, concatenated after server cert). If HTTPS serving is enabled, and --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to the directory specified by --cert-dir.

--tls-private-key-file string
    File containing the default x509 private key matching --tls-cert-file.
其他client验证apiserver时可以通过签署这两个证书的CA，我们称为`tls-ca`
```

### client证书
apiserver提供了tls证书，同样也需要验证client的配置，但是client太多了(kubectl,各种restapi调用的), 这些client需要统一用一个CA签发，我们称为`client-ca`。
```
--client-ca-file string
    If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.
```

需要注意的是，在apiserver认证中，通过`CN`和`O`来识别用户，开启RBAC的用户要配置`CN`和`O`做一些授权：
- CN：Common Name，kube-apiserver 从证书中提取作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
- O：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)

如kube-proxy的证书申请, User为`system:kube-proxy`, Group为`k8s`
```json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

### requestheader证书
apiserver可以使用HTTP请求头中的指定字段来进行认证，相关配置如下:
```
--requestheader-allowed-names stringSlice
    List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers. If empty, any client certificate validated by the authorities in --requestheader-client-ca-file is allowed.
--requestheader-client-ca-file string
    Root certificate bundle to use to verify client certificates on incoming requests before trusting usernames in headers specified by --requestheader-username-headers. WARNING: generally do not depend on authorization being already done for incoming requests.
--requestheader-extra-headers-prefix strings        
    List of request header prefixes to inspect. X-Remote-Extra- is suggested.
--requestheader-group-headers strings               
    List of request headers to inspect for groups. X-Remote-Group is suggested.
--requestheader-username-headers strings            
    List of request headers to inspect for usernames. X-Remote-User is common.
```
收到请求时，apiserver会首先认证`requsetheader-ca`，验证成功并且`CN`在`requestheader-allowed-names`（默认全部需求）中，然后通过Http header中的`X-Remote-User, X-Remote-Group`去得到用户；如果匹配不成功回去验证`client-ca`。

如上，`requestheader`证书与`client-ca`不能是同一个。

### proxy证书
k8s提供了丰富的扩展机制，CRD与[API Aggregation][https://kubernetes.io/zh/docs/tasks/access-kubernetes-api/configure-aggregation-layer/]。
对于API Aggregation(例如metrics-server提供了metrics.k8s.io api), apiserver接受到请求后经过一系列验证过滤，会将请求转发到扩展API，这里apisever作为代理服务器，需要配置配置证书。
```
--proxy-client-cert-file string             
    Client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins. It is expected that this cert includes a signature from the CA in the --requestheader-client-ca-file flag. That CA is published in the 'extension-apiserver-authentication' configmap in the kube-system namespace. Components recieving calls from kube-aggregator should use that CA to perform their half of the mutual TLS verification.
--proxy-client-key-file string              
    Private key for the client certificate used to prove the identity of the aggregator or kube-apiserver when it must call out during a request. This includes proxying requests to a user api-server and calling out to webhook admission plugins.
```
需要注意的是对证书需要通过`requestheader-ca`签发，扩展api会通过requestheader证书去验证，具体流程后面会写一篇，下图为官方提供的流程
![aggregation-api](https://d33wubrfki0l68.cloudfront.net/3c5428678a95c3715894011d8dd4812d2cf229b9/e745c/images/docs/aggregation-api-auth-flow.png)

### kubelet证书
对于kubelet，apiserver单独提供了证书配置选项，同时kubelet组件也提供了反向设置的相关选项:
```
# API Server
--kubelet-certificate-authority string
    Path to a cert file for the certificate authority.
--kubelet-client-certificate string
    Path to a client cert file for TLS.
--kubelet-client-key string
    Path to a client key file for TLS.

# kubelet
--client-ca-file string
    If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.
--tls-cert-file string 
    File containing x509 Certificate used for serving HTTPS (with intermediate certs, if any, concatenated after server cert). If --tls-cert-file and --tls-private-key-file are not provided, a self-signed certificate and key are generated for the public address and saved to the directory passed to --cert-dir.
--tls-private-key-file string
    File containing x509 private key matching --tls-cert-file.
```
kubelet也是即作为server也作为client, 需要提供tls证书和client-ca, 我们称这个CA为`kubelet-ca`, 可以是单独的CA。

### etcd证书
这个也不用多说，用来连接etcd，由`etcd-ca`签发
```
--etcd-certfile string                      SSL certification file used to secure etcd communication.
--etcd-keyfile string                       SSL key file used to secure etcd communication.
```

### serviceaccount证书
在k8s中，通过`JWT`认证`serviecaccount`，同样有两个证书配置:
```
# apiserver
--service-account-key-file stringArray # 用于验证sa
    File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. The specified file can contain multiple keys, and the flag can be specified multiple times with different files. If unspecified, --tls-private-key-file is used. Must be specified when --service-account-signing-key is provided
--service-account-signing-key-file string
    Path to the file that contains the current private key of the service account token issuer. The issuer will sign issued ID tokens with this private key. (Requires the 'TokenRequest' feature gate.)

# controller-manager
–service-account-private-key-file #用于签署sa
```
这两个配置描述了对`serviceaccount`进行签名验证时所使用的证书；可以是单独的生成，我们称为`sa-key`。

## 其他证书
其他还有`oidc`证书，用于OpenID认证；`ssh`证书，用来连接node，目前以及废弃。

etcd与kubelet证书上面已经提过了，需要双方都配置。

k8s中也支持证书申请，用户可以创建`CertificateSigningRequest`来申请证书，需要在controller-manager配置下面的证书，用于签发证书称为`sing-ca`，多用于webhook的证书配置。
```
--cluster-signing-cert-file string          Filename containing a PEM-encoded X509 CA certificate used to issue cluster-scoped certificates (default "/etc/kubernetes/ca/ca.pem")
--cluster-signing-key-file string           Filename containing a PEM-encoded RSA or ECDSA private key used to sign cluster-scoped certificates (default "/etc/kubernetes/ca/ca.key")
```

## 总结
k8s提供了强大的功能，需要考虑到各个场景的安全问题，上面我们梳理了遍目前常用的证书
- tls-ca
- client-ca
- requestheader-ca
- proxy-ca
- kubelet-ca
- etcd-ca
- sa-key
- sign-ca

上面除了`proxy-ca`必须使用`requestheader-ca`签发，其他所有的都可以是单独的CA，可以根据安全性评估是使用一个CA还是多个CA，我们建议下面的CA尽量是独立的
- client-ca
- requestheader-ca
- etcd-ca
- kubelet-ca
- sign-ca

终于理完了，可以起床啦。
