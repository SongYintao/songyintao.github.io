## APIServer

## 访问控制

Kubernetes API 的每个请求都会经过多阶段的访问控制之后才会被接受，这包括认证、授权以及准入控制（Admission Control）等。

![img](https://feisky.gitbooks.io/kubernetes/content/components/images/access_control.png)

### 认证

开启 TLS 时，所有的请求都需要首先认证。Kubernetes 支持多种认证机制，并支持同时开启多个认证插件（只要有一个认证通过即可）。如果认证成功，则用户的 `username` 会传入授权模块做进一步授权验证；而对于认证失败的请求则返回 HTTP 401。

> **Kubernetes 不直接管理用户**
>
> 虽然 Kubernetes 认证和授权用到了 username，但 Kubernetes 并不直接管理用户，不能创建 `user` 对象，也不存储 username。

更多认证模块的使用方法可以参考 [Kubernetes 认证插件](https://feisky.gitbooks.io/kubernetes/content/plugins/auth.html# 认证)。

### 授权

认证之后的请求就到了授权模块。跟认证类似，Kubernetes 也支持多种授权机制，并支持同时开启多个授权插件（只要有一个验证通过即可）。如果授权成功，则用户的请求会发送到准入控制模块做进一步的请求验证；而对于授权失败的请求则返回 HTTP 403.

更多授权模块的使用方法可以参考 [Kubernetes 授权插件](https://feisky.gitbooks.io/kubernetes/content/plugins/auth.html# 授权)。

### 准入控制

准入控制（Admission Control）用来对请求做进一步的验证或添加默认参数。不同于授权和认证只关心请求的用户和操作，准入控制还处理请求的内容，并且仅对创建、更新、删除或连接（如代理）等有效，而对读操作无效。准入控制也支持同时开启多个插件，它们依次调用，只有全部插件都通过的请求才可以放过进入系统。

更多准入控制模块的使用方法可以参考 [Kubernetes 准入控制](https://feisky.gitbooks.io/kubernetes/content/plugins/admission.html)。

## 启动 apiserver 示例

```sh
kube-apiserver --feature-gates=AllAlpha=true --runtime-config=api/all=true \
    --requestheader-allowed-names=front-proxy-client \
    --client-ca-file=/etc/kubernetes/pki/ca.crt \
    --allow-privileged=true \
    --experimental-bootstrap-token-auth=true \
    --storage-backend=etcd3 \
    --requestheader-username-headers=X-Remote-User \
    --requestheader-extra-headers-prefix=X-Remote-Extra- \
    --service-account-key-file=/etc/kubernetes/pki/sa.pub \
    --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
    --tls-private-key-file=/etc/kubernetes/pki/apiserver.key \
    --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
    --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
    --insecure-port=8080 \
    --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds \
    --requestheader-group-headers=X-Remote-Group \
    --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \
    --secure-port=6443 \
    --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname \
    --service-cluster-ip-range=10.96.0.0/12 \
    --authorization-mode=RBAC \
    --advertise-address=192.168.0.20 --etcd-servers=http://127.0.0.1:2379
```

## 工作原理

kube-apiserver 提供了 Kubernetes 的 REST API，实现了认证、授权、准入控制等安全校验功能，同时也负责集群状态的存储操作（通过 etcd）。

![img](https://feisky.gitbooks.io/kubernetes/content/components/images/kube-apiserver.png)



以 `/apis/batch/v2alpha1/jobs` 为例，GET 请求的处理过程如下图所示：

![img](https://feisky.gitbooks.io/kubernetes/content/components/assets/API-server-flow.png)

POST 请求的处理过程为：

![img](https://feisky.gitbooks.io/kubernetes/content/components/assets/API-server-storage-flow.png)

（图片来自 [OpenShift Blog](https://blog.openshift.com/kubernetes-deep-dive-api-server-part-1/)）

## API 访问

有多种方式可以访问 Kubernetes 提供的 REST API：

- [kubectl](https://feisky.gitbooks.io/kubernetes/content/components/kubectl.html) 命令行工具
- SDK，支持多种语言
  - [Go](https://github.com/kubernetes/client-go)
  - [Python](https://github.com/kubernetes-incubator/client-python)
  - [Javascript](https://github.com/kubernetes-client/javascript)
  - [Java](https://github.com/kubernetes-client/java)
  - [CSharp](https://github.com/kubernetes-client/csharp)
  - 其他 [OpenAPI](https://www.openapis.org/) 支持的语言，可以通过 [gen](https://github.com/kubernetes-client/gen) 工具生成相应的 client

### kubectl

```sh
kubectl get --raw /api/v1/namespaces
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods
```

### kubectl proxy

```sh
$ kubectl proxy --port=8080 &

$ curl http://localhost:8080/api/
{
  "versions": [
    "v1"
  ]
}
```

### curl

```sh
# In Pods with service account.
$ TOKEN=$(cat /run/secrets/kubernetes.io/serviceaccount/token)
$ CACERT=/run/secrets/kubernetes.io/serviceaccount/ca.crt
$ curl --cacert $CACERT --header "Authorization: Bearer $TOKEN"  https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
# Outside of Pods.
$ APISERVER=$(kubectl config view | grep server | cut -f 2- -d ":" | tr -d " ")
$ TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token'| cut -f2 -d':'| tr -d '\t')
$ curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```





##  API Aggregation api聚合层

API Aggregation 允许在不修改 Kubernetes 核心代码的同时扩展 Kubernetes API，即将第三方服务注册到 Kubernetes API 中，这样就可以通过 Kubernetes API 来访问外部服务。

另外一种扩展 Kubernetes API 的方法是使用 [CustomResourceDefinition (CRD)](https://feisky.gitbooks.io/kubernetes/content/concepts/customresourcedefinition.html)。



## 何时使用 Aggregation

| 满足以下条件时使用 API Aggregation                           | 满足以下条件时使用独立 API                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Your API is [Declarative](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#declarative-apis). | Your API does not fit the [Declarative](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#declarative-apis) model. |
| You want your new types to be readable and writable using `kubectl`. | `kubectl` support is not required                            |
| You want to view your new types in a Kubernetes UI, such as dashboard, alongside built-in types. | Kubernetes UI support is not required.                       |
| You are developing a new API.                                | You already have a program that serves your API and works well. |
| You are willing to accept the format restriction that Kubernetes puts on REST resource paths, such as API Groups and Namespaces. (See the [API Overview](https://kubernetes.io/docs/concepts/overview/kubernetes-api/).) | You need to have specific REST paths to be compatible with an already defined REST API. |
| Your resources are naturally scoped to a cluster or to namespaces of a cluster. | Cluster or namespace scoped resources are a poor fit; you need control over the specifics of resource paths. |
| You want to reuse [Kubernetes API support features](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#common-features). | You don’t need those features.                               |

## 开启 API Aggregation

kube-apiserver 增加以下配置

```sh
--requestheader-client-ca-file=<path to aggregator CA cert>
--requestheader-allowed-names=aggregator
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--proxy-client-cert-file=<path to aggregator proxy cert>
--proxy-client-key-file=<path to aggregator proxy key>
```

如果 `kube-proxy` 没有在 Master 上面运行，还需要配置

```sh
--enable-aggregator-routing=true
```

## 创建扩展 API

1. 确保开启 APIService API（默认开启，可用 `kubectl get apiservice` 命令验证）
2. 创建 RBAC 规则
3. 创建一个 namespace，用来运行扩展的 API 服务
4. 创建 CA 和证书，用于 https
5. 创建一个存储证书的 secret
6. 创建一个部署扩展 API 服务的 deployment，并使用上一步的 secret 配置证书，开启 https 服务
7. 创建一个 ClusterRole 和 ClusterRoleBinding
8. 创建一个非 namespace 的 apiservice，注意设置 `spec.caBundle`
9. 运行 `kubectl get `，正常应该返回 `No resources found.`

可以使用 [apiserver-builder](https://github.com/kubernetes-incubator/apiserver-builder) 工具自动化上面的步骤。

```sh
# 初始化项目
$ cd GOPATH/src/github.com/my-org/my-project
$ apiserver-boot init repo --domain <your-domain>
$ apiserver-boot init glide

# 创建资源
$ apiserver-boot create group version resource --group <group> --version <version> --kind <Kind>

# 编译
$ apiserver-boot build executables
$ apiserver-boot build docs

# 本地运行
$ apiserver-boot run local

# 集群运行
$ apiserver-boot run in-cluster --name nameofservicetorun --namespace default --image gcr.io/myrepo/myimage:mytag
$ kubectl create -f sample/<type>.yaml
```

## 示例

见 [sample-apiserver](https://github.com/kubernetes/sample-apiserver) 和 [apiserver-builder/example](https://github.com/kubernetes-incubator/apiserver-builder/tree/master/example)。





# CustomResourceDefinition

CustomResourceDefinition（CRD）是 v1.7 新增的无需改变代码就可以扩展 Kubernetes API 的机制，用来管理自定义对象。它实际上是 ThirdPartyResources（TPR）的升级版本，而 TPR 已经在 v1.8 中弃用。

## API 版本对照表

| Kubernetes 版本 | CRD API 版本                 |
| --------------- | ---------------------------- |
| v1.8+           | apiextensions.k8s.io/v1beta1 |

## CRD 示例

下面的例子会创建一个 `/apis/stable.example.com/v1/namespaces//crontabs/…` 的自定义 API：

```sh
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # versions to use for REST API: /apis/<group>/<version>
  versions:
  - name: v1beta1
    # Each version can be enabled/disabled by Served flag.
    served: true
    # One and only one version must be marked as the storage version.
    storage: true
  - name: v1
    served: true
    storage: false
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

API 创建好后，就可以创建具体的 CronTab 对象了

```sh
$ cat my-cronjob.yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * /5"
  image: my-awesome-cron-image

$ kubectl create -f my-crontab.yaml
crontab "my-new-cron-object" created

$ kubectl get crontab
NAME                 KIND
my-new-cron-object   CronTab.v1.stable.example.com
$ kubectl get crontab my-new-cron-object -o yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  creationTimestamp: 2017-07-03T19:00:56Z
  name: my-new-cron-object
  namespace: default
  resourceVersion: "20630"
  selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
  uid: 5c82083e-5fbd-11e7-a204-42010a8c0002
spec:
  cronSpec: '* * * * /5'
  image: my-awesome-cron-image
```

## Finalizer

Finalizer 用于实现控制器的异步预删除钩子，可以通过 `metadata.finalizers` 来指定 Finalizer。

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - finalizer.stable.example.com
```

Finalizer 指定后，客户端删除对象的操作只会设置 `metadata.deletionTimestamp` 而不是直接删除。这会触发正在监听 CRD 的控制器，控制器执行一些删除前的清理操作，从列表中删除自己的 finalizer，然后再重新发起一个删除操作。此时，被删除的对象才会真正删除。

## Validation

v1.8 开始新增了实验性的基于 [OpenAPI v3 schema](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#schemaObject) 的验证（Validation）机制，可以用来提前验证用户提交的资源是否符合规范。使用该功能需要配置 kube-apiserver 的 `--feature-gates=CustomResourceValidation=true`。

比如下面的 CRD 要求

- `spec.cronSpec` 必须是匹配正则表达式的字符串
- `spec.replicas` 必须是从 1 到 10 的整数

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  validation:
   # openAPIV3Schema is the schema for validating custom objects.
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
```

这样，在创建下面的 CronTab 时

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * *"
  image: my-awesome-cron-image
  replicas: 15
```

会报验证失败的错误：

```sh
The CronTab "my-new-cron-object" is invalid: []: Invalid value: map[string]interface {}{"apiVersion":"stable.example.com/v1", "kind":"CronTab", "metadata":map[string]interface {}{"name":"my-new-cron-object", "namespace":"default", "deletionTimestamp":interface {}(nil), "deletionGracePeriodSeconds":(*int64)(nil), "creationTimestamp":"2017-09-05T05:20:07Z", "uid":"e14d79e7-91f9-11e7-a598-f0761cb232d1", "selfLink":"","clusterName":""}, "spec":map[string]interface {}{"cronSpec":"* * * *", "image":"my-awesome-cron-image", "replicas":15}}:
validation failure list:
spec.cronSpec in body should match '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
spec.replicas in body should be less than or equal to 10
```

## Subresources

v1.10 开始 CRD 还支持 `/status` 和 `/scale` 等两个子资源（Beta），并且从 v1.11 开始默认开启。

> v1.10 版本使用前需要在 `kube-apiserver` 开启 `--feature-gates=CustomResourceSubresources=true`。

```yaml
# resourcedefinition.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  # subresources describes the subresources for custom resources.
  subresources:
    # status enables the status subresource.
    status: {}
    # scale enables the scale subresource.
    scale:
      # specReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Spec.Replicas.
      specReplicasPath: .spec.replicas
      # statusReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Replicas.
      statusReplicasPath: .status.replicas
      # labelSelectorPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Selector.
      labelSelectorPath: .status.labelSelector
$ kubectl create -f resourcedefinition.yaml
$ kubectl create -f- <<EOF
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 3
EOF

$ kubectl scale --replicas=5 crontabs/my-new-cron-object
crontabs "my-new-cron-object" scaled

$ kubectl get crontabs my-new-cron-object -o jsonpath='{.spec.replicas}'
5
```

## Categories

Categories 用来将 CRD 对象分组，这样就可以使用 `kubectl get ` 来查询属于该组的所有对象。

```yaml
# resourcedefinition.yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
    # categories is a list of grouped resources the custom resource belongs to.
    categories:
    - all
# my-crontab.yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
$ kubectl create -f resourcedefinition.yaml
$ kubectl create -f my-crontab.yaml
$ kubectl get all
NAME                          AGE
crontabs/my-new-cron-object   3s
```

## CRD 控制器

在使用 CRD 扩展 Kubernetes API 时，通常还需要实现一个新建资源的控制器，监听新资源的变化情况，并作进一步的处理。

https://github.com/kubernetes/sample-controller 提供了一个 CRD 控制器的示例，包括

- 如何注册资源 `Foo`
- 如何创建、删除和查询 `Foo` 对象
- 如何监听 `Foo` 资源对象的变化情况

## Kubebuilder

从上面的实例中可以看到从头构建一个 CRD 控制器并不容易，需要对 Kubernetes 的 API 有深入了解，并且RBAC 集成、镜像构建、持续集成和部署等都需要很大工作量。

[kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) 正是为解决这个问题而生，为 CRD 控制器提供了一个简单易用的框架，并可直接生成镜像构建、持续集成、持续部署等所需的资源文件。

### 安装

```sh
# Install kubebuilder
VERSION=1.0.1
wget https://github.com/kubernetes-sigs/kubebuilder/releases/download/v${VERSION}/kubebuilder_${VERSION}_linux_amd64.tar.gz
tar zxvf kubebuilder_${VERSION}_linux_amd64.tar.gz
sudo mv kubebuilder_${VERSION}_linux_amd64 /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin

# Install dep kustomize
go get -u github.com/golang/dep/cmd/dep
go get github.com/kubernetes-sigs/kustomize
```

### 使用方法

#### 初始化项目

```sh
mkdir -p $GOPATH/src/demo
cd $GOPATH/src/demo
kubebuilder init --domain k8s.io --license apache2 --owner "The Kubernetes Authors"
```

#### 创建 API

```sh
kubebuilder create api --group ships --version v1beta1 --kind Sloop
```

然后按照实际需要修改 `pkg/apis/ship/v1beta1/sloop_types.go` 和 `pkg/controller/sloop/sloop_controller.go` 增加业务逻辑。

#### 本地运行测试

```sh
make install
make run
```

> 如果碰到错误 `ValidationError(CustomResourceDefinition.status): missing required field "storedVersions" in io.k8s.apiextensions-apiserver.pkg.apis.apiextensions.v1beta1.CustomResourceDefinitionStatus]`，可以手动修改 `config/crds/ships_v1beta1_sloop.yaml`: ```yaml status: acceptedNames: kind: "" plural: "" conditions: [] storedVersions: []
>
> 然后运行 `kubectl apply -f config/crds` 创建 CRD。

然后就可以用 `ships.k8s.io/v1beta1` 来创建 Kind 为 `Sloop` 的资源了，比如

```sh
kubectl apply -f config/samples/ships_v1beta1_sloop.yaml
```

#### 构建镜像并部署控制器

```sh
# 替换 IMG 为你自己的
export IMG=feisky/demo-crd:v1
make docker-build
make docker-push
make deploy
```

> kustomize 已经不再支持通配符，因而上述 `make deploy` 可能会碰到 `Load from path ../rbac/*.yaml failed` 错误，解决方法是手动修改 `config/default/kustomization.yaml`:
>
> resources:
>
> - ../rbac/rbac_role.yaml
> - ../rbac/rbac_role_binding.yaml
> - ../manager/manager.yaml
>
> 然后执行 `kustomize build config/default | kubectl apply -f -` 部署，默认部署到 `demo-system` namespace 中。

#### 文档和测试

```sh
# run unit tests
make test

# generate docs
kubebuilder docs
```

## 参考文档

- [Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#validation)
- [CustomResourceDefinition API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#customresourcedefinition-v1beta1-apiextensions-k8s-io)

## 插件扩展机制



https://feisky.gitbooks.io/kubernetes/content/plugins/admission.html    



## kube-proxy细节分析

其实kube-proxy的代码本身并不复杂，只是有个细节容易被大家忽略，大家可能都知道它有轮询的复杂均衡策略，是通过iptables实现的，那它是怎样控制平均转发的呢？iptables有个random的模块支持，那怎样控制权重呢？
看代码，一步一步分析

    {
        tablesNeedServicesChain := []utiliptables.Table{utiliptables.TableFilter, utiliptables.TableNAT}
        for _, table := range tablesNeedServicesChain {
            if _, err := proxier.iptables.EnsureChain(table, kubeServicesChain); err != nil {
                glog.Errorf("Failed to ensure that %s chain %s exists: %v", table, kubeServicesChain, err)
                return
            }
        }
    
        tableChainsNeedJumpServices := []struct {
            table utiliptables.Table
            chain utiliptables.Chain
        }{
            {utiliptables.TableFilter, utiliptables.ChainInput},
            {utiliptables.TableFilter, utiliptables.ChainOutput},
            {utiliptables.TableNAT, utiliptables.ChainOutput},
            {utiliptables.TableNAT, utiliptables.ChainPrerouting},
        }
        comment := "kubernetes service portals"
        args := []string{"-m", "comment", "--comment", comment, "-j", string(kubeServicesChain)}
        for _, tc := range tableChainsNeedJumpServices {
            if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, tc.table, tc.chain, args...); err != nil {
                glog.Errorf("Failed to ensure that %s chain %s jumps to %s: %v", tc.table, tc.chain, kubeServicesChain, err)
                return
            }
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
首先是建立filter表的INPUT/OUTPUT和nat表的OUTPUT/PREROUTE规则全部跳转到service链
效果如下：

-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES

-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
1
2
3
4
这样出去的流量都会被service的链截获了

当然如果有些流量需要通过SNAT出去

    {
        if _, err := proxier.iptables.EnsureChain(utiliptables.TableNAT, kubePostroutingChain); err != nil {
            glog.Errorf("Failed to ensure that %s chain %s exists: %v", utiliptables.TableNAT, kubePostroutingChain, err)
            return
        }
    
        comment := "kubernetes postrouting rules"
        args := []string{"-m", "comment", "--comment", comment, "-j", string(kubePostroutingChain)}
        if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, utiliptables.TableNAT, utiliptables.ChainPostrouting, args...); err != nil {
            glog.Errorf("Failed to ensure that %s chain %s jumps to %s: %v", utiliptables.TableNAT, utiliptables.ChainPostrouting, kubePostroutingChain, err)
            return
        }
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
效果如下：

-A POSTROUTING -m comment --comment "kubernetes postrouting rules" -j KUBE-POSTROUTING
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
1
2
现在开始建立kubernetes proxy的各个链

    writeLine(proxier.filterChains, "*filter")
    writeLine(proxier.natChains, "*nat")
    
    // Make sure we keep stats for the top-level chains, if they existed
    // (which most should have because we created them above).
    if chain, ok := existingFilterChains[kubeServicesChain]; ok {
        writeLine(proxier.filterChains, chain)
    } else {
        writeLine(proxier.filterChains, utiliptables.MakeChainLine(kubeServicesChain))
    }
    if chain, ok := existingNATChains[kubeServicesChain]; ok {
        writeLine(proxier.natChains, chain)
    } else {
        writeLine(proxier.natChains, utiliptables.MakeChainLine(kubeServicesChain))
    }
    if chain, ok := existingNATChains[kubeNodePortsChain]; ok {
        writeLine(proxier.natChains, chain)
    } else {
        writeLine(proxier.natChains, utiliptables.MakeChainLine(kubeNodePortsChain))
    }
    if chain, ok := existingNATChains[kubePostroutingChain]; ok {
        writeLine(proxier.natChains, chain)
    } else {
        writeLine(proxier.natChains, utiliptables.MakeChainLine(kubePostroutingChain))
    }
    if chain, ok := existingNATChains[KubeMarkMasqChain]; ok {
        writeLine(proxier.natChains, chain)
    } else {
        writeLine(proxier.natChains, utiliptables.MakeChainLine(KubeMarkMasqChain))
    }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
这个里面创建KUBE-SERVICES、KUBE-NODEPORTS、KUBE-POSTROUTING、KUBE-MARK-MASQ

通过kubernetes创建的service会分配一个clusterIP，这些clusterIP是在iptables上面实现的

        args := []string{
            "-A", string(kubeServicesChain),
            "-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcNameString),
            "-m", protocol, "-p", protocol,
            "-d", fmt.Sprintf("%s/32", svcInfo.clusterIP.String()),
            "--dport", fmt.Sprintf("%d", svcInfo.port),
        }
        if proxier.masqueradeAll {
            writeLine(proxier.natRules, append(args, "-j", string(KubeMarkMasqChain))...)
        }
        if len(proxier.clusterCIDR) > 0 {
            writeLine(proxier.natRules, append(args, "! -s", proxier.clusterCIDR, "-j", string(KubeMarkMasqChain))...)
        }
        writeLine(proxier.natRules, append(args, "-j", string(svcChain))...)
1
2
3
4
5
6
7
8
9
10
11
12
13
14
上面就是截获clusterIP的流量做DNAT，这里面需要补充的就是如果一个服务后面有多个endpoint的，

for i, endpointChain := range endpointChains {
            // Balancing rules in the per-service chain.
            args := []string{
                "-A", string(svcChain),
                "-m", "comment", "--comment", svcNameString,
            }
            if i < (n - 1) {
                // Each rule is a probabilistic match.
                args = append(args,
                    "-m", "statistic",
                    "--mode", "random",
                    "--probability", fmt.Sprintf("%0.5f", 1.0/float64(n-i)))
            }
            // The final (or only if n == 1) rule is a guaranteed match.
            args = append(args, "-j", string(endpointChain))
            writeLine(proxier.natRules, args...)

            // Rules in the per-endpoint chain.
            args = []string{
                "-A", string(endpointChain),
                "-m", "comment", "--comment", svcNameString,
            }
            // Handle traffic that loops back to the originator with SNAT.
            writeLine(proxier.natRules, append(args,
                "-s", fmt.Sprintf("%s/32", strings.Split(endpoints[i].endpoint, ":")[0]),
                "-j", string(KubeMarkMasqChain))...)
            // Update client-affinity lists.
            if svcInfo.sessionAffinityType == api.ServiceAffinityClientIP {
                args = append(args, "-m", "recent", "--name", string(endpointChain), "--set")
            }
            // DNAT to final destination.
            args = append(args, "-m", protocol, "-p", protocol, "-j", "DNAT", "--to-destination", endpoints[i].endpoint)
            writeLine(proxier.natRules, args...)
        }
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
上面通过循环的方式创建后端endpoint的转发，概率是通过probability后的1.0/float64(n-i)计算出来的，譬如有两个的场景，那么将会是一个0.5和1也就是第一个是50%概率第二个是100%概率，如果是三个的话类似，33%、50%、100%。下面是10个endpoint的例子。

kubectl get svc --all-namespaces
NAMESPACE      NAME                    CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
admin          docker2048              10.13.52.135    11.11.1.1     80/TCP                       1d


[root@master-62 ~]# 
[root@master-62 ~]# iptables-save |grep 10.13.52.135
-A KUBE-SERVICES -d 10.13.52.135/32 -p tcp -m comment --comment "admin/docker2048:docker2048-1 cluster IP" -m tcp --dport 80 -j KUBE-SVC-MHWEDWK6NM5OGU2T
[root@master-62 ~]# 
[root@master-62 ~]# 
[root@master-62 ~]# iptables-save |grep KUBE-SVC-MHWEDWK6NM5OGU2T
:KUBE-SVC-MHWEDWK6NM5OGU2T - [0:0]
-A KUBE-SERVICES -d 10.13.52.135/32 -p tcp -m comment --comment "admin/docker2048:docker2048-1 cluster IP" -m tcp --dport 80 -j KUBE-SVC-MHWEDWK6NM5OGU2T
-A KUBE-SERVICES -d 11.11.1.1/32 -p tcp -m comment --comment "admin/docker2048:docker2048-1 external IP" -m tcp --dport 80 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-MHWEDWK6NM5OGU2T
-A KUBE-SERVICES -d 11.11.1.1/32 -p tcp -m comment --comment "admin/docker2048:docker2048-1 external IP" -m tcp --dport 80 -m addrtype --dst-type LOCAL -j KUBE-SVC-MHWEDWK6NM5OGU2T
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.10000000009 -j KUBE-SEP-VC767CJYOTCBCN3B
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.11110999994 -j KUBE-SEP-HQELSIUR5HSCB2VN
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.12500000000 -j KUBE-SEP-X2UDSU7Q4UA4IKY7
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.14286000002 -j KUBE-SEP-DQ3TZIZIDTXU77P7
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.16667000018 -j KUBE-SEP-A3JWOZYQIIDDEKNM
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.20000000019 -j KUBE-SEP-6EZ2MUBOPU2WH44E
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.25000000000 -j KUBE-SEP-4KG3GD3BQ5TCAUPR
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-6EXLETYC4LYB5NLM
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-VLQQMEFA6Y5RZLE7
-A KUBE-SVC-MHWEDWK6NM5OGU2T -m comment --comment "admin/docker2048:docker2048-1" -j KUBE-SEP-CXDZACZ7ESWWLYJM


## 解析kubernetes Aggregated API Servers

kubernetes的 Aggregated API是什么呢？它是允许k8s的开发人员编写一个自己的服务，可以把这个服务注册到k8s的api里面，这样，就像k8s自己的api一样**，你的服务只要运行在k8s集群里面，k8s 的Aggregate通过service名称就可以转发到你写的service里面去了。**
这个设计理念：

- 第一是**增加了api的扩展性**，这样k8s的开发人员就可以编写自己的API服务器来公开他们想要的API。集群管理员应该能够使用这些服务，而不需要对核心库存储库进行任何更改。
- 第二是**丰富了APIs**，核心kubernetes团队阻止了很多新的API提案。通过允许开发人员将他们的API作为单独的服务器公开，并使集群管理员能够在不对核心库存储库进行任何更改的情况下使用它们，这样就无须社区繁杂的审查了
- 第三是**开发分阶段实验性API的地方，新的API可以在单独的聚集服务器中开发**，当它稳定之后，那么把它们封装起来安装到其他集群就很容易了。
- 第四是**确保新API遵循kubernetes约定**：如果没有这里提出的机制，社区成员可能会被迫推出自己的东西，这可能会或可能不遵循kubernetes约定。

一句话阐述就是：

Aggregator for Kubernetes-style API servers: dynamic registration, discovery summarization, secure proxy
**动态注册、发现汇总、和安全代理**。好了基本概念说清楚了，下面就说说实现。

如果你已经已经阅读我上一篇blog就知道，proxy的巨大作用。

下面看看这个聚合api的神奇之处。
先看怎么使用，然后再看源代码

```yaml
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1alpha1.custom-metrics.metrics.k8s.io
spec:
  insecureSkipTLSVerify: true
  group: custom-metrics.metrics.k8s.io
  groupPriorityMinimum: 1000
  versionPriority: 15
  service:
    name: api
    namespace: custom-metrics
  version: v1alpha1
```

上面定义了资源类型为APIService，service名称为api，空间为custom-metrics的一个资源聚合接口。
下面带大家从源代码的角度来看你
**pkg/apiserver/apiservice_controller.go**
和k8s其它controller一样，watch变化分发到add、update和delete方法这套原理在此就不赘述了.

```go
apiServiceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    c.addAPIService,
        UpdateFunc: c.updateAPIService,
        DeleteFunc: c.deleteAPIService,
    })
```



```go
serviceInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    c.addService,
    UpdateFunc: c.updateService,
    DeleteFunc: c.deleteService,
})
```

主要监听两种资源apiService和service，分别看看

```go
func (s *APIAggregator) AddAPIService(apiService *apiregistration.APIService) error {
    // if the proxyHandler already exists, it needs to be updated. The aggregation bits do not
    // since they are wired against listers because they require multiple resources to respond
    if proxyHandler, exists := s.proxyHandlers[apiService.Name]; exists {
        proxyHandler.updateAPIService(apiService)
        if s.openAPIAggregationController != nil {
            s.openAPIAggregationController.UpdateAPIService(proxyHandler, apiService)
        }
        return nil
    }
```



```go
proxyPath := "/apis/" + apiService.Spec.Group + "/" + apiService.Spec.Version
// v1. is a special case for the legacy API.  It proxies to a wider set of endpoints.
if apiService.Name == legacyAPIServiceName {
    proxyPath = "/api"
}

// register the proxy handler
proxyHandler := &proxyHandler{
    contextMapper:   s.contextMapper,
    localDelegate:   s.delegateHandler,
    proxyClientCert: s.proxyClientCert,
    proxyClientKey:  s.proxyClientKey,
    proxyTransport:  s.proxyTransport,
    serviceResolver: s.serviceResolver,
}
proxyHandler.updateAPIService(apiService)
if s.openAPIAggregationController != nil {
    s.openAPIAggregationController.AddAPIService(proxyHandler, apiService)
}
s.proxyHandlers[apiService.Name] = proxyHandler
s.GenericAPIServer.Handler.NonGoRestfulMux.Handle(proxyPath, proxyHandler)
s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandlePrefix(proxyPath+"/", proxyHandler)

// if we're dealing with the legacy group, we're done here
if apiService.Name == legacyAPIServiceName {
    return nil
}

// if we've already registered the path with the handler, we don't want to do it again.
if s.handledGroups.Has(apiService.Spec.Group) {
    return nil
}

// it's time to register the group aggregation endpoint
groupPath := "/apis/" + apiService.Spec.Group
groupDiscoveryHandler := &apiGroupHandler{
    codecs:        Codecs,
    groupName:     apiService.Spec.Group,
    lister:        s.lister,
    delegate:      s.delegateHandler,
    contextMapper: s.contextMapper,
}
// aggregation is protected
s.GenericAPIServer.Handler.NonGoRestfulMux.Handle(groupPath, groupDiscoveryHandler)
s.GenericAPIServer.Handler.NonGoRestfulMux.UnlistedHandle(groupPath+"/", groupDiscoveryHandler)
s.handledGroups.Insert(apiService.Spec.Group)
return nil
}
```


上面path是

```
proxyPath := "/apis/" + apiService.Spec.Group + "/" + apiService.Spec.Version
```


结合上面的例子就是/apis/custom-metrics.metrics.k8s.io/v1alpha1.
而处理方法请求的handle就是

结合上面的例子就是/apis/custom-metrics.metrics.k8s.io/v1alpha1.
而处理方法请求的handle就是

```go
proxyHandler := &proxyHandler{
        contextMapper:   s.contextMapper,
        localDelegate:   s.delegateHandler,
        proxyClientCert: s.proxyClientCert,
        proxyClientKey:  s.proxyClientKey,
        proxyTransport:  s.proxyTransport,
        serviceResolver: s.serviceResolver,
    }
    proxyHandler.updateAPIService(apiService)
```

上面的updateAPIService就是更新这个proxy的后端service



```go
func (r *proxyHandler) updateAPIService(apiService *apiregistrationapi.APIService) {
    if apiService.Spec.Service == nil {
        r.handlingInfo.Store(proxyHandlingInfo{local: true})
        return
    }
newInfo := proxyHandlingInfo{
    restConfig: &restclient.Config{
        TLSClientConfig: restclient.TLSClientConfig{
            Insecure:   apiService.Spec.InsecureSkipTLSVerify,
            ServerName: apiService.Spec.Service.Name + "." + apiService.Spec.Service.Namespace + ".svc",
            CertData:   r.proxyClientCert,
            KeyData:    r.proxyClientKey,
            CAData:     apiService.Spec.CABundle,
        },
    },
    serviceName:      apiService.Spec.Service.Name,
    serviceNamespace: apiService.Spec.Service.Namespace,
}
newInfo.proxyRoundTripper, newInfo.transportBuildingError = restclient.TransportFor(newInfo.restConfig)
if newInfo.transportBuildingError == nil && r.proxyTransport.Dial != nil {
    switch transport := newInfo.proxyRoundTripper.(type) {
    case *http.Transport:
        transport.Dial = r.proxyTransport.Dial
    default:
        newInfo.transportBuildingError = fmt.Errorf("unable to set dialer for %s/%s as rest transport is of type %T", apiService.Spec.Service.Namespace, apiService.Spec.Service.Name, newInfo.proxyRoundTripper)
        glog.Warning(newInfo.transportBuildingError.Error())
    }
}
r.handlingInfo.Store(newInfo)
}
```


这个restConfig就是调用service的客户端参数，其中

ServerName: apiService.Spec.Service.Name + "." + apiService.Spec.Service.Namespace + ".svc",
就是具体的service。
而上面**watch service的变化就是为了动态更新这个apiservice后端handler所用的service**。

