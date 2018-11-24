---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes 笔记整理

---

## 1. 用户

用户可分为ServiceAccount 和外部用户

### 1.1 ServiceAccount

```
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
```

* Namespaced Object
* Kubernetes 会为一个 ServiceAccount 自动创建并分配一个 Secret 对象, 用来跟 APIServer 进行交互的授权文件, 文件的内容一般是证书或者密码，它以一个 Secret 对象的方式保存在 Etcd 当中
* 每个命名空间下有一个默认的SA: default

```
$ kubectl get sa -n mynamespace -o yaml
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: 2018-09-08T12:59:17Z
    name: example-sa
    namespace: mynamespace
    resourceVersion: "409327"
    ...
  secrets:
  - name: example-sa-token-vmfg6
```

ServiceAccount 对应的secret的格式:

```

$ kubectl describe secret default-token-s8rbq
Name:         default-token-s8rbq
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=default # 这里表明这个secret是和默认的SA default绑定的
              kubernetes.io/service-account.uid=ffcb12b2-917f-11e8-abde-42010aa80002
Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      <TOKEN 数据 >
```

### 1.2 Pod 与 ServiceAccount

Pod以声明使用某个 ServiceAccount:

```
apiVersion: v1
kind: Pod
metadata:
  namespace: mynamespace
  name: sa-token-test
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
  serviceAccountName: example-sa
```

该 ServiceAccount 的 token，也就是一个 Secret 对象，被 Kubernetes 自动挂载到了容器的 `/var/run/secrets/kubernetes.io/serviceaccount` 目录下:

```
$ kubectl describe pod sa-token-test -n mynamespace
Name:               sa-token-test
Namespace:          mynamespace
...
Containers:
  nginx:
    ...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from example-sa-token-vmfg6 (ro) # TODO 挂载类型
```

具体挂载文件:

```
$ kubectl exec -it sa-token-test -n mynamespace -- /bin/bash
root@sa-token-test:/# ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt namespace  token
```

如果一个 Pod 没有声明 serviceAccountName，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 Pod:

```
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-bcnvr (ro)
Volumes:
  default-token-bcnvr:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  defau
```

这个默认 ServiceAccount 并没有关联任何 Role。也就是说，此时它有访问 APIServer 的绝大多数权限 TODO ??


### 1.3 外部用户

K8s 可以从支持的认证中获取用户和组:

* 基本认证：basic-auth
* Token认证：token-auth
* OpenID Connect Tokens 认证

见 API Server 认证


### 1.4 用户组

* 外部用户组
* ServiceAccount 用户组

常用SA 用户组:

* `system:serviceaccounts:mynamespace` 命名空间 mynamespace 下所有的SA
* `system:serviceaccounts` 系统中所有的SA

---

## 2. RBAC

基于角色的权限控制：Role-Based Access Control

基本概念:

### 2.1 Role

* Namespaced Object
* 被用来授予访问**单一命令空间**中 Kubernetes API 对象的操作权限
* 在RBAC API中，角色包含代表权限集合的规则。在这里，权限只有被授予，而没有被拒绝的设置
* 操作定义: `verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]`

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

### 2.2 RoleBinding

* 定义了“被作用者”和“角色”的绑定关系
* Namespaced Object, roleRef 也只能引用当前 Namespace 里的 Role 对象

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User # Kubernetes 里的“User”，也就是“用户”，只是一个授权系统里的逻辑概念。它需要通过外部认证服务
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

### 2.3 Subject

被作用者，既可以是“人”，也可以是“机器”，也可以使你在 Kubernetes 里定义的“用户”。

subjects.kind 取值:

* User (外部用户)
* ServiceAccount (内置用户)
* Group

### 2.4 ClusterRole

用法跟 Role 完全一样。只不过，它的定义里，没有了 Namespace 字段

在 Kubernetes 中已经内置了很多个为系统保留的 ClusterRole，它们的名字都以 system: 开头, 一般来说，这些系统 ClusterRole，是绑定给 Kubernetes 系统组件对应的 ServiceAccount 使用的

四个预先定义好的 ClusterRole 来供用户直接使用:

* cluster-admin: 对应的是整个 Kubernetes 项目中的最高权限（verbs=*）
* admin
* edit
* view

### 2.5 ClusterRoleBinding

用法跟 RoleBinding 完全一样。只不过，它的定义里，没有了 Namespace 字段

---

## 3. API

### 3.1 声明式API

* 命令式: kubectl create,  kubectl replace
* 声明式: kubectl apply

声明式API的特点:

* 可对原有 API 对象的 PATCH 操作, 具备 Merge 能力
* 一次能处理多个写操作
* 对“实际状态”和“期望状态”的调谐（Reconcile）过程, 所以说，声明式 API，才是 Kubernetes 项目编排能力“赖以生存”的核心所在

### 3.2. API 资源对象

常见资源对象:

* /api/v1  核心 API 对象, Group 为空
  * /nodes
  * /pods
* /apis/batch/v1
  * /jobs
  * /watch
* /apis/batch/v2alpha1
  * /cronjobs

API 资源对象定义:

* apiVersion: `API组/版本`, 如`batch/v2alpha1`
* kind: `资源类型`, 如 `CronJob`

API Server 处理流程:

1. Filter: 授权, 超时, 审计
2. Handler: Mux, Routes
3. 查找该API资源定义: Group->Version->Kind
4. Convert: YAML -> Super Version 对象(该 API 资源类型所有版本的字段全集)
5. Admission()
6. Validation()
7. 序列化保存到etcd

### 3.3 CRD

API 插件机制: Custom Resource Definition

CRD 用来描述自定义的资源类型, 示例:

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
```

---

## 4 API Server 认证

### 4.1 CA证书认证

apiserver 启动参数配置:

* `--client-ca-file=/etc/kubernetes/cluster-ca.crt` 指定根证书地址
* `--tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem` 指定kube-apiserver证书地址
* `--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem` 指定kube-apiserver私钥地址

客户端 kubectl 访问配置:

证书认证配置:

* `kubectl config --kubeconfig=配置文件名 set-cluster 集群命名 --server=https://1.2.3.4 --certificate-authority=证书地址` 集群的根证书路径??? TODO
* `kubectl config --kubeconfig=config-demo set-cluster 集群命名 --server=https://5.6.7.8 --insecure-skip-tls-verify`


用户鉴权配置:

* `kubectl config --kubeconfig=配置文件名 set-credentials 集群命名 --client-certificate=客户端证书?? --client-key=客户端私钥` 双向认证????
* `kubectl config --kubeconfig=配置文件名 set-credentials 集群命名 --username=用户名 --password=密码`

### 4.2 基本认证：basic-auth

`--basic-auth-file=/etc/kubernetes/basic_auth.csv`

格式为`password，username，uid, 可选的组名`, 如果有多个组，则列必须是双引号

```
 # cat /etc/kubernetes/basic_auth.csv
FM7fhRM4NUg2j0fssXf1OMolALtPpMka,admin,admin,system:masters
```

认证方式: 在http请求的header中添加一个Authorization，value是Basic base64编码后的用户名密码信息

`Authorization: BasicBASE64ENCODED(USER:PASSWORD) `

### 4.3 Token认证：token-auth

`--token-auth-file=/etc/kubernetes/known_tokens.csv`

格式为`token，username，uid, 可选的组名`, 如果有多个组，则列必须是双引号

bearer token必须是，可以放在HTTP请求头中且值不需要转码和引用的一个字符串

```
# cat /etc/kubernetes/known_tokens.csv
FM7fhRM4NUg2j0fssXf1OMolALtPpMka,admin,admin,system:masters
DBMmLEwHROlO4Ront5lw7GcaKMkEqtud,kubelet,kubelet,system:masters
```
认证方式: header中添加Authorization，value是Bearer token

`Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269`

### 4.4 OpenID Connect Tokens 认证

TODO


---

## 5. 控制器模型

控制循环: 死循环, 它不断地获取“实际状态”，然后与“期望状态”作对比，并以此为依据决定下一步的操作

```
for {
  实际状态 := 获取集群中对象 X 的实际状态（Actual State）
  期望状态 := 获取集群中对象 X 的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

这个操作，通常被叫作调谐（Reconcile）。这个调谐的过程，则被称作“Reconcile Loop”（调谐循环）或者“Sync Loop”（同步循环）

实际状态往往来自于 Kubernetes 集群本身, 而期望状态，一般来自于用户提交的 YAML 文件

Deployment 示例:

Deployment 以及其他类似的控制器, 定义主要分为2部分:

1. 控制器定义 (除去template的内容)
2. 被控制对象定义(template 的内容)

---

## 6. Admission Controller

一个 API 对象被提交给 APIServer 之后，总有一些“初始化”性质的工作需要在它们被 Kubernetes 项目正式处理之前进行。比如，自动为所有 Pod 加上某些标签（Labels）

### 6.1 Dynamic Admission Control

也叫作：Initializer

Initializer，作为一个 Pod 部署在 Kubernetes, 是一个事先编写好的“自定义控制器”（Custom Controller）


### 6.2 Istio Envoy 示例

Istio中每个Pod里, 有一个以 sidecar 运行的Envoy容器, 实现方式:

1. Istio 将 Envoy 容器本身的定义，以 ConfigMap 的方式保存在 Kubernetes
2. Istio 将一个编写好的 Initializer，作为一个 Pod 部署在 Kubernetes, 这个Pod主要是运行`envoy-initializer` 它是一个自定义控制器”（Custom Controller）
3. 该控制器会执行控制器逻辑: 不断获取到的“实际状态”，就是用户新创建的 Pod。而它的“期望状态”，则是：这个 Pod 里被添加了 Envoy 容器的定义
4. Kubernetes 的 API 库，为我们提供了一个方法，使得我们可以直接使用新旧两个 Pod 对象，生成一个 TwoWayMergePatch

   ```
   // 生成 patch 数据
   patchBytes := strategicpatch.CreateTwoWayMergePatch(pod, newPod)
   // 发起 PATCH 请求，修改这个 pod 对象
   client.Patch(pod.Name, patchBytes)
   ```
5. 当在 Initializer 里完成了要做的操作后，一定要记得将这个 metadata.initializers.pending 标志清除掉

---

K8s 编排对象可以分为两大类:

1. 在线业务/Long Running Task: Deployment, StatefulSet, DaemonSet
2. 离线业务/ Batch Job: Job

## 7. 离线业务

### 7.1 Job

Job Controller 控制的对象，直接就是 Pod, (spec.template)

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2
  completions: 4
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```


Job 对 Pod 的控制:

* Pod template, 被自动加上了一个 `controller-uid=< 一个随机字符串 >`这样的 Label
* Job.Selector, 同样设置了上述label用于定位pod

Job 对Pod的调谐过程:

* 需要创建的 Pod 数目 = 最终需要的 Pod 数目 - 实际在 Running 状态 Pod 数目 - 已经成功退出的 Pod 数目
* 单时创建的 Pod 数目受到并行度(spec.parallelism)的限制

#### 7.1.1 失败控制: restartPolicy

在 Job 对象里只允许被设置为 Never 或者 OnFailure

* `restartPolicy=Never`: job失败会导致不断地尝试创建一个新 Pod, 重试上限是 Job 对象的 spec.backoffLimit(默认值是 6)

  新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10 s、20 s、40 s …后

* restartPolicy=OnFailure: Job 失败不会重新创建pod, 但会不断地尝试重启 Pod 里的容器

#### 7.1.2 失败控制: spec.activeDeadlineSeconds

设置最长运行时间, 超时后对应 Pod 都会被终止。并且可以在 Pod 的状态里看到终止的原因是 reason: DeadlineExceeded

#### 7.1.3 并行控制

* spec.parallelism: 它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行

* spec.completions: 它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数

Job 状态表:

```
$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
pi        4         0            3s
```

* DESIRED: 对应 spec.completions
* SUCCESSFUL: 成功完成的pod数量

### 7.2 CronJob

CronJob 是一个 Job 对象的控制器（定义中的jobTemplate）

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

schedule: 标准的Unix Cron格式的表达式

CronJob 状态表:

```
$ kubectl get cronjob hello
NAME      SCHEDULE      SUSPEND   ACTIVE    LAST-SCHEDULE
hello     */1 * * * *   False     0         Thu, 6 Sep 2018 14:34:00 -070
```

#### 7.2.1 并行控制:

某个 Job 还没有执行完，按照调度新 Job 将被创建时, 是否允许并行由`concurrencyPolicy` 来控制:

* concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在
* concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过
* concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job

#### 7.2.2 失败控制

如果某一次 Job 创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss 的数目达到失败上限100时，那么 CronJob 会停止再创建这个 Job

时间窗口: spec.startingDeadlineSeconds 字段指定

---

## 8. Deployment

TODO

---

## 9. Service

```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: app1
  name: app1
  namespace: default
spec:
  type: ClusterIP/NodePort/LoadBalancer
  clusterIP:            # 在type为ClusterIP时有效, 如果没有设置系统会自动提供一个
  ports:
  - name:               # 端口名称
    port: 8080          # service暴露在cluster ip上的端口, <cluster ip>:port 是提供给集群内部客户访问service的入口
    targetPort: 8080    # pod上的端口
    nodePort: 30062     # 节点port, 提供给集群外部客户访问service的入口
  selector:
    name: app1
```

### 9.1 ServiceType

* ClusterIP:

  仅使用一个集群内部的IP地址 - 这是默认值

  如果设置为`None`, 将创建一个Headless service

* NodePort: 在集群内部IP的基础上，在集群的每一个节点的端口上开放这个服务。你可以在任意<NodeIP>:NodePort地址上访问到这个服务 

  Kubernetes的master会从由启动参数配置的范围（默认是：30000-32767）中分配一个端口，然后每一个Node都会将这个端口（在每一个Node上相同的端口）代理到你的Service。这个端口会被写入你的Service的spec.ports[*].nodePort字段中

  也可以自行设定 nodePort

* LoadBalancer:

  在公有云提供的 Kubernetes 服务里，都使用了一个叫作 CloudProvider 的转接层，来跟公有云本身的 API 进行对接。所以，在 LoadBalancer 类型的 Service 被提交后，Kubernetes 就会调用 CloudProvider 在公有云上创建一个负载均衡服务，并且把被代理的 Pod 的 IP 地址配置给负载均衡服务做后端. (然后将LB ip 配置到EXTERNAL-IP ?) 

  EXTERNAL-IP 是一个LB, 端口是服务的port, 转发到node的nodeport

  ```
  NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE       SELECTOR
  node-koa-demo-service-2   LoadBalancer   172.16.255.211   119.28.109.191   4000:31750/TCP   87d       qcloud-app=node-koa-demo-service-2
  ```

  其中port 是 `服务内部port:主机nodeport`, 服务内部port同样用于EXTERNAL-IP

  另外`Service.externalIPs` 可以直接设置, Kubernetes 要求 externalIPs 必须是至少能够路由到一个 Kubernetes 的节点.

* ExternalName:

  ```
  kind: Service
  apiVersion: v1
  metadata:
    name: my-service
  spec:
    type: ExternalName
    externalName: my.database.example.com
  ```

  在`kube-dns` 里添加了一条 CNAME 记录: `Service的DNS域名 -> my.database.example.com`

### 9.2 Endpoints

被 selector 选中的 Pod，就称为 Service 的 Endpoints，可以使用 `kubectl get ep` 查看

只有处于 Running 状态，且 readinessProbe 检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉

### 9.3 Headless Service

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx

```

Headless Service 被创建后并不会被分配一个 VIP，而是会以 DNS 记录的方式暴露出它所代理的 Pod, 它所代理的所有 Pod 的 IP 地址，都会被绑定一个这样格式的 DNS 记录:

```
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

### 9.5 Service 的 iptables 实现

Service 是由 kube-proxy 组件，加上 iptables 来共同实现的

1. `KUBE-SERVICES` Service 的入口链: 拦截service VIP 流量

   kube-proxy 通过 Service 的 Informer 感知 Service 对象的添加, 然后在本机创建iptables规则:

   在链`KUBE-SERVICES`新增接管该service VIP的流量

   `-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3`

   所有目的地址是 10.0.1.175:80 (VIP+service port)的 IP 包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3

2. `KUBE-SVC-(hash)` 负载均衡链: 均分流量到POD链

   新建的 该Service对应的iptables 链, 这些规则的数目应该与 Endpoints 数目一致:

   ```
   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
   ```

   通过随机流量均分到3条新链, 每个链对应一个pod的链

3. `KUBE-SEP-(hash)` Pod DNAT链: 这些规则应该与 Endpoints 一一对应

   ```
   -A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
   -A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

   -A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
   -A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

   -A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
   -A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
   ```

   做了2件事:

   * IP 包进行了DNAT, 目的地址改为该pod的的ip和端口
   * `--set-xmark` 打上标记`0x00004000`

NodePort 的 Service 实现:

1. NodePort `KUBE-NODEPORTS` Service 的入口链

   ```
   -A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
   ```

2. `KUBE-SVC-(hash)` 负载均衡链: 同上

3. `KUBE-SEP-(hash)` Pod DNAT链: 同上

4. `KUBE-POSTROUTING` 对离开本主机去其他node的, 由Service 转发出来的 IP 包进行SNAT

   ```
   -A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
   ```

   根据`mark==0x4000` 进行匹配需要处理的包, 该标记是第三步打上的

   SNAT的目的是让具体pod的返包原路返回, 而不是由pod直接返回给node, 不过这样pod 是不知道真实的clint ip.

   如果pod 需要知道client 真实的ip, 可以设置Service `externalTrafficPolicy: local`: 这时候，一台宿主机上的 iptables 规则，会设置为只将 IP 包转发给运行在这台宿主机上的 Pod, 这也就意味着如果在一台宿主机上，没有任何一个被代理的 Pod 存在，请求会直接被 DROP 掉


### 9.6 Service 的 IPVS 实现

TODO

### 9.7 Service 与 DNS

Service 有2中访问方式:

1. VIP
2. DNS: Service 对应的域名

#### 9.7.1 DNS A 记录:

* Normal Service: 服务A 记录: `<service-name>.<namespace>.svc.cluster.local -> my-svc的VIP`
* Pod: `<pod-ip-address>.<namespace>.pod.cluster.local`, 如: `1-2-3-4.default.pod.cluster.local`
* Headless Service: 服务A 记录: `<service-name>.<namespace>.svc.cluster.local -> my-svc所有pod ip集合`
* Headless Service pod 新增 A 记录: `<pod-hostname>.my-svc.my-namespace.svc.cluster.local -> 该pod ip`

#### 9.7.2 DNS SRV 记录:

> DNS SRV是DNS记录中一种，用来指定服务地址。与常见的A记录、cname不同的是，SRV中除了记录服务器的地址，还记录了服务的端口，并且可以设置每个服务地址的优先级和权重。访问服务的时候，本地的DNS resolver从DNS服务器查询到一个地址列表，根据优先级和权重，从中选取一个地址作为本次请求的目标地址

Service 命名端口: `<port-name>.<protocol>.<service-name>.<namespace>.svc.cluster.localhost`

* Normal Service: 解析出service 对应的A记录和该端口号
* Headless Service: 返回多个值(所有pod A记录)和该端口号

---

### 10. Statefulset



---

本地代理service: TODO

`kubectl proxy`

`http://localhost:8001/api/v1/namespaces/default/services/foxdemo-develop:80/proxy/`


