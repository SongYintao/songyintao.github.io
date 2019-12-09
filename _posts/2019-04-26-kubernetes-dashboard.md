---
title: kubernetes dashboard trouble shooot
subtitle: k8s看板问题定位
layout: post
tags: [k8s,dashboard]
---

最近，在部署测试环境的dashborad。按照git上面的部署方式，发现部署总是失败的。通过查看日志，dashboard连接master节点api-server时报错。提示bad certification。

为了让小伙伴们用上这个小工具，可谓是煞费苦心，折腾了两天。

现在记录一下具体的过程：

### 1. 车祸现场

```shell
root@test-a1-60-83:~/k8s# kubectl get pods -n kube-system
NAME                                    READY     STATUS              RESTARTS   AGE
coredns-78fcdf6894-prh2s                1/1       Running             36822      122d
etcd-192.168.60.83                      1/1       Running             0          127d
kube-apiserver-192.168.60.83            1/1       Running             1          50m
kube-controller-manager-192.168.60.83   1/1       Running             2          127d
kube-proxy-2f8z5                        1/1       NodeLost            10         127d
kube-proxy-67kmp                        1/1       Running             1          127d
kube-proxy-6gmd8                        1/1       Running             1          127d
kube-proxy-8s275                        1/1       Running             1          127d
kube-proxy-b2w7j                        0/1       ContainerCreating   0          14d
kube-proxy-dqscm                        1/1       NodeLost            9          127d
kube-proxy-gkwrq                        1/1       Running             0          127d
kube-proxy-hj4t4                        1/1       Running             1          127d
kube-proxy-hv9sc                        1/1       Running             1          127d
kube-proxy-ld7jw                        1/1       Running             2          127d
kube-proxy-mbd2w                        1/1       NodeLost            4          127d
kube-proxy-q2hnt                        1/1       Running             1          127d
kube-proxy-v8sts                        1/1       Running             1          127d
kube-proxy-z567f                        1/1       Running             12         127d
kube-scheduler-192.168.60.83            1/1       Running             2          127d
kubernetes-dashboard-64c497d69d-db8m2   0/1       CrashLoopBackOff    13         42m
```

kubernetes-dashboard对应的状态是：CrashLoopBackOff。是不是很尴尬。我们继续往下看。

```shell
root@test-a1-60-83:~/k8s# kubectl logs kubernetes-dashboard-64c497d69d-db8m2 -n kube-system
2019/04/26 02:07:58 Starting overwatch
2019/04/26 02:07:58 Using apiserver-host location: https://192.168.60.83:6443
2019/04/26 02:07:58 Skipping in-cluster config
2019/04/26 02:07:58 Using random key for csrf signing
2019/04/26 02:07:58 Error while initializing connection to Kubernetes apiserver. This most likely means that the cluster is misconfigured (e.g., it has invalid apiserver certificates or service account's configuration) or the --apiserver-host param points to a server that does not exist. Reason: Get https://192.168.60.83:6443/version: x509: failed to load system roots and no roots provided
Refer to our FAQ and wiki pages for more information: https://github.com/kubernetes/dashboard/wiki/FAQ
```

给我报了这个错，呵呵哒啊。。。Reason: Get https://192.168.60.83:6443/version: x509: failed to load system roots and no roots provided。

https://github.com/kubernetes/dashboard/wiki/FAQ按照它给的地址我去看看啥原因。



既然，是到master api-server的请求，我们不妨看看api-server小兄弟的日志：

```shell
root@test-a1-60-83:~/k8s# kubectl logs kube-apiserver-192.168.60.83 -n kube-system
Flag --insecure-port has been deprecated, This flag will be removed in a future version.
I0426 01:18:07.144229       1 server.go:703] external host was not specified, using 192.168.60.83
I0426 01:18:07.144360       1 server.go:145] Version: v1.11.0
I0426 01:18:08.335899       1 plugins.go:158] Loaded 7 mutating admission controller(s) successfully in the following order: NamespaceLifecycle,LimitRanger,ServiceAccount,NodeRestriction,DefaultTolerationSeconds,DefaultStorageClass,MutatingAdmissionWebhook.
I0426 01:18:08.335921       1 plugins.go:161] Loaded 5 validating admission controller(s) successfully in the following order: LimitRanger,ServiceAccount,PersistentVolumeClaimResize,ValidatingAdmissionWebhook,ResourceQuota.
I0426 01:18:08.336742       1 plugins.go:158] Loaded 7 mutating admission controller(s) successfully in the following order: NamespaceLifecycle,LimitRanger,ServiceAccount,NodeRestriction,DefaultTolerationSeconds,DefaultStorageClass,MutatingAdmissionWebhook.
I0426 01:18:08.336754       1 plugins.go:161] Loaded 5 validating admission controller(s) successfully in the following order: LimitRanger,ServiceAccount,PersistentVolumeClaimResize,ValidatingAdmissionWebhook,ResourceQuota.
I0426 01:18:08.379603       1 master.go:234] Using reconciler: lease
W0426 01:18:09.758835       1 genericapiserver.go:319] Skipping API batch/v2alpha1 because it has no resources.
W0426 01:18:10.087031       1 genericapiserver.go:319] Skipping API rbac.authorization.k8s.io/v1alpha1 because it has no resources.
W0426 01:18:10.094188       1 genericapiserver.go:319] Skipping API scheduling.k8s.io/v1alpha1 because it has no resources.
W0426 01:18:10.115396       1 genericapiserver.go:319] Skipping API storage.k8s.io/v1alpha1 because it has no resources.
W0426 01:18:10.699343       1 genericapiserver.go:319] Skipping API admissionregistration.k8s.io/v1alpha1 because it has no resources.
[restful] 2019/04/26 01:18:10 log.go:33: [restful/swagger] listing is available at https://192.168.60.83:6443/swaggerapi
[restful] 2019/04/26 01:18:10 log.go:33: [restful/swagger] https://192.168.60.83:6443/swaggerui/ is mapped to folder /swagger-ui/
[restful] 2019/04/26 01:18:11 log.go:33: [restful/swagger] listing is available at https://192.168.60.83:6443/swaggerapi
[restful] 2019/04/26 01:18:11 log.go:33: [restful/swagger] https://192.168.60.83:6443/swaggerui/ is mapped to folder /swagger-ui/
I0426 01:18:11.932230       1 plugins.go:158] Loaded 7 mutating admission controller(s) successfully in the following order: NamespaceLifecycle,LimitRanger,ServiceAccount,NodeRestriction,DefaultTolerationSeconds,DefaultStorageClass,MutatingAdmissionWebhook.
I0426 01:18:11.932248       1 plugins.go:161] Loaded 5 validating admission controller(s) successfully in the following order: LimitRanger,ServiceAccount,PersistentVolumeClaimResize,ValidatingAdmissionWebhook,ResourceQuota.
I0426 01:18:14.709551       1 serve.go:96] Serving securely on [::]:6443
I0426 01:18:14.709607       1 available_controller.go:278] Starting AvailableConditionController
I0426 01:18:14.709616       1 cache.go:32] Waiting for caches to sync for AvailableConditionController controller
I0426 01:18:14.709643       1 naming_controller.go:284] Starting NamingConditionController
I0426 01:18:14.709699       1 establishing_controller.go:73] Starting EstablishingController
I0426 01:18:14.709649       1 customresource_discovery_controller.go:199] Starting DiscoveryController
I0426 01:18:14.709701       1 controller.go:84] Starting OpenAPI AggregationController
I0426 01:18:14.709655       1 autoregister_controller.go:136] Starting autoregister controller
I0426 01:18:14.709837       1 cache.go:32] Waiting for caches to sync for autoregister controller
I0426 01:18:14.709719       1 crdregistration_controller.go:112] Starting crd-autoregister controller
I0426 01:18:14.709904       1 controller_utils.go:1025] Waiting for caches to sync for crd-autoregister controller
I0426 01:18:14.709622       1 crd_finalizer.go:242] Starting CRDFinalizer
I0426 01:18:14.710524       1 apiservice_controller.go:90] Starting APIServiceRegistrationController
I0426 01:18:14.710569       1 cache.go:32] Waiting for caches to sync for APIServiceRegistrationController controller
I0426 01:18:14.909954       1 cache.go:39] Caches are synced for autoregister controller
I0426 01:18:14.910062       1 controller_utils.go:1032] Caches are synced for crd-autoregister controller
I0426 01:18:14.910687       1 cache.go:39] Caches are synced for APIServiceRegistrationController controller
I0426 01:18:14.915473       1 cache.go:39] Caches are synced for AvailableConditionController controller
I0426 01:18:15.891189       1 storage_scheduling.go:100] all system priority classes are created successfully or already exist.
I0426 01:18:30.766571       1 logs.go:49] http: TLS handshake error from 172.28.2.228:36864: remote error: tls: bad certificate
I0426 01:18:32.488380       1 controller.go:597] quota admission added evaluator for: { endpoints}
I0426 01:23:32.721354       1 logs.go:49] http: TLS handshake error from 172.28.2.228:37786: remote error: tls: bad certificate
I0426 01:24:16.897449       1 controller.go:597] quota admission added evaluator for: {apps deployments}
I0426 01:24:16.936581       1 controller.go:597] quota admission added evaluator for: {apps replicasets}
I0426 01:26:09.215593       1 controller.go:597] quota admission added evaluator for: { serviceaccounts}
I0426 01:26:09.275471       1 controller.go:597] quota admission added evaluator for: {rbac.authorization.k8s.io roles}
I0426 01:26:09.283566       1 controller.go:597] quota admission added evaluator for: {rbac.authorization.k8s.io rolebindings}
I0426 01:26:32.855547       1 logs.go:49] http: TLS handshake error from 172.28.8.99:33280: remote error: tls: bad certificate
I0426 01:26:33.476085       1 logs.go:49] http: TLS handshake error from 172.28.8.99:33284: remote error: tls: bad certificate
I0426 01:26:55.909992       1 logs.go:49] http: TLS handshake error from 172.28.8.99:33338: remote error: tls: bad certificate
I0426 01:27:17.890830       1 logs.go:49] http: TLS handshake error from 172.28.8.99:33390: remote error: tls: bad certificate
I0426 01:28:03.914155       1 logs.go:49] http: TLS handshake error from 172.28.8.99:33484: remote error: tls: bad certificate
I0426 01:29:36.917816       1 logs.go:49] http: TLS handshake error from 172.28.8.99:33724: remote error: tls: bad certificate
I0426 01:32:18.942086       1 logs.go:49] http: TLS handshake error from 172.28.8.99:34088: remote error: tls: bad certificate
I0426 01:37:29.927708       1 logs.go:49] http: TLS handshake error from 172.28.8.99:34808: remote error: tls: bad certificate
I0426 01:42:41.926876       1 logs.go:49] http: TLS handshake error from 172.28.8.99:35518: remote error: tls: bad certificate
I0426 01:47:42.938512       1 logs.go:49] http: TLS handshake error from 172.28.8.99:36308: remote error: tls: bad certificate
I0426 01:52:44.918971       1 logs.go:49] http: TLS handshake error from 172.28.8.99:37062: remote error: tls: bad certificate
I0426 01:57:49.901699       1 logs.go:49] http: TLS handshake error from 172.28.8.99:37852: remote error: tls: bad certificate
I0426 02:02:54.923722       1 logs.go:49] http: TLS handshake error from 172.28.8.99:38598: remote error: tls: bad certificate
I0426 02:07:58.910090       1 logs.go:49] http: TLS handshake error from 172.28.8.99:39378: remote error: tls: bad certificate
I0426 02:13:03.937215       1 logs.go:49] http: TLS handshake error from 172.28.8.99:40206: remote error: tls: bad certificate
root@test-a1-60-83:~/k8s#
```



看看最后面的几行啊， **http: TLS handshake error from 172.28.8.99:40206: remote error: tls: bad certificate**果然很清楚了，小兄弟dashboard连接大管家的certification居然是有问题的，那么我们就朝这个方向去看一下吧。



那么，我们就从底层的开始分析处理。首先，确保我们的api-server打开了ServiceAccount的admission-access的plugin，以及相关的秘钥文件地址。不看不知道啊。由于我们采用的是kubeadm安装的集群，相关的配置和我们想的不一样啊。

进入我们集群的静态pod的配置，master节点上的kubelet服务启动的时候就会先把这个Pod拉起来，不受调度器的控制。

```shell
root@test-a1-60-83:~/k8s# cd /etc/kubernetes/manifests/
root@test-a1-60-83:/etc/kubernetes/manifests# ls
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
root@test-a1-60-83:/etc/kubernetes/manifests#
```

瞅一下 **kube-apiserver.yaml** 

```shell
 containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --advertise-address=192.168.60.83
    - --allow-privileged=true
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --disable-admission-plugins=PersistentVolumeLabel
    - --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.0
    imagePullPolicy: IfNotPresent
```

`--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction`这个是正常的配置，kubeadm默认设置的`NodeRestriction`。鉴于此，我们做了修改。还需要确认一下，`--service-account-key-file=/etc/kubernetes/pki/sa.pub`这个参数有没有配置。

总的来说，kubernetes这个东西权限校验，采用了https，只要我们保证，所有的组件配置都遵循这个标准就妥了。

解决了：

```shell
Error while initializing connection to Kubernetes apiserver. 
This most likely means that the cluster is misconfigured (e.g., it has invalid 
apiserver certificates or service accounts configuration) or the --apiserver-host 
param points to a server that does not exist. Reason: 
Get https://foo.bar.com:6443/version: x509: failed to load system roots and 
no roots provided
```

发现这尼玛，又来了一个问题。

```shell
open /certs/dashboard.crt
```

对这个文件没有写操作。呵呵哒，可以有多种方式解决；

一种，将挂载的目录readOnly参数改成，false。

另外一种，由于dashboard的配置里面说，自动生成这些文件。那么，我可不可以自己手动给他create呢？当然是可以的。。。

```shell
openssl req -newkey rsa:4096 -nodes -sha256 -keyout dashboard.key -x509 -days 365 -out dashboard.crt
```

根据提示，自己输入相关信息就可以了。

```shell
Generating a 4096 bit RSA private key
......................++
....................................................................................................++
writing new private key to 'dashboard.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Hubei  
Locality Name (eg, city) [Default City]:Wuhan
Organization Name (eg, company) [Default Company Ltd]:Muxistudio
Organizational Unit Name (eg, section) []:be
Common Name (eg, your name or your server's hostname) []:iZwz9f6pgul78p7die5tlzZ
Email Address []:3480437308@qq.com
```

执行完之后，你的命令执行目录就会多出来两个文件，妥了。

手动创建secret资源：

```shell
kubectl create secret generic kubernetes-dashboard-certs --from-file=/certs -n kube-system
```

再次，覆盖之前的dashboard的配置。

```shell
$ kubectl create -f kubernetes-dashboard.yml
```

下面，我们要访问服务器上的`kubernetes-dashboard`服务就还需要做下面的一步操作：将kubernetes-dashboard的service变为外部可路由。将service的类型由默认的ClusterIP改为NodePort.
执行下面的命令，更改service的配置文件:

```shell
$ kubectl edit service kubernetes-dashboard -n kube-system
```

更改完成之后，我们查看服务的暴露的主机端口:

```shell
$ kubectl get service kubernetes-dashboard -n kube-system
```

输出:

```
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   10.109.219.222   <nodes>       443:30087/TCP   52m
```

如上，服务暴露的主机端口为30087,我们访问`https://<主机公网IP>:30087/`即可访问到dashboard服务。注意，**这里使用的是HTTPS服务，并且使用的是自签名证书，所以在我们初次进入时，浏览器会因为无法识别证书的签署机构而阻止继续访问，这时我们只需要点击高级选项，标识网站为安全即可。**

