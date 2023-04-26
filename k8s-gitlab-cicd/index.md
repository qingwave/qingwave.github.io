# k8s+gitlab实现cicd

## 前言
目前Gitlab11已经支持了Kubernetes Runner, 任务可以跑在Pod中。本文介绍如何通过CICD接入Kubernetes，开始前需要以下必备条件：
- Kubernetes集群
- 配置Kubernetes Runner, 网上有很多教程，若是生产环境或是多租户k8s集群，建议通过yaml手动配置；默认通过helm安装权限比较大，而且配置不灵活

## CI过程
通常编译镜像有三种方式：
- docker in docker：与物理方式类似，需要权限多，性能较差
- kaniko：镜像编译工具，性能好

我们使用kaniko编译镜像，push到镜像仓库，过程如下：

1. 配置变量
   配置镜像相关变量，仓库的账户密码，推送的镜像名称`CI_REGISTRY_IMAGE`等
   ![gitlab-ci](/img/blogImg/gitlab-ci.png)
2. gitlab-ci配置如下
```yaml
build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    entrypoint: [""]
  script:
    - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
  after_script:
    - echo "build completed"
  only:
    - tags # 打tag才会执行，测试可去掉
```

## CD过程
CD即需要将生成的镜像更新到Kubernetes集群中，有如下几种方式：
- k8s restful api：需要对api较了解，更新过程需要调用`PATH`方法，不推荐
- kubectl: 常规方式
- helm: 如有可用的helm仓库，也可使用helm进行更新
  
我们以kubectl为例，CD配置如下：
1. 配置变量
   配置必须的集群地址，token，需要更新服务的namespace, container等
2. CD配置
   配置与物理环境类似，首先配置kubectl token、集群等，最后调用`set image`更新服务
```yaml
deploy:
  image:
    name: kubectl:1.17
    entrypoint: [""]
  before_script:
  script:
    - IMAGE=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - kubectl config set-credentials $CD_USER --token $CD_APP_AK 
    - kubectl config set-cluster $CD_CLUSTER --server https://$CD_SERVER
    - kubectl config set-context $CD_USER@$CD_CLUSTER/$CD_NAMESPACE --user $CD_USER --cluster $CD_CLUSTER --namespace $CD_NAMESPACE
    - kubectl config use-context $CD_USER@$CD_CLUSTER/$CD_NAMESPACE
    - kubectl set image -n $CD_NAMESPACE $CD_APP_TYPE/$CD_APP_NAME $CD_CONTAINER=$IMAGE
  only:
    - tags
```
3. 运行结果
  ```bash
  $ kubectl set image -n $CD_NAMESPACE $CD_APP_TYPE/$CD_APP_NAME $CD_CONTAINER=$IMAGE
  deployment.extensions/helloworld image updated
  Job succeeded
  ```

## 备注
本文所列举的CICD过程较简单，可以使用CICD完成服务的多集群部署，更新结果检查等功能。

## 参考
1. https://docs.gitlab.com/ee/ci/docker/using_kaniko.html
2. https://docs.gitlab.com/runner/executors/kubernetes.html
