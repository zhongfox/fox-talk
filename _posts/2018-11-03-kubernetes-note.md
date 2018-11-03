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

`kube-apiserver` 的启动参数中用户相关的认证配置:

#### 1.3.1 基本认证：basic-auth

`--basic-auth-file=/etc/kubernetes/basic_auth.csv`

格式为password，username，uid

```
 # cat /etc/kubernetes/basic_auth.csv
FM7fhRM4NUg2j0fssXf1OMolALtPpMka,admin,admin,system:masters
```

认证方式: 在http请求的header中添加一个Authorization，value是Basic base64编码后的用户名密码信息


#### 1.3.2 Token认证：token-auth

`--token-auth-file=/etc/kubernetes/known_tokens.csv`

格式为token，username，uid

```
# cat /etc/kubernetes/known_tokens.csv
FM7fhRM4NUg2j0fssXf1OMolALtPpMka,admin,admin,system:masters
DBMmLEwHROlO4Ront5lw7GcaKMkEqtud,kubelet,kubelet,system:masters
```
认证方式: header中添加Authorization，value是Bearer token

#### 1.3.3 CA证书认证

TODO

### 1.4 用户组

* 外部用户组
* ServiceAccount 用户组

常用SA 用户组:

* `system:serviceaccounts:mynamespace` 命名空间 mynamespace 下所有的SA
* `system:serviceaccounts` 系统中所有的SA

--

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

## 4. 控制器模型

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

## 5. Admission Controller

一个 API 对象被提交给 APIServer 之后，总有一些“初始化”性质的工作需要在它们被 Kubernetes 项目正式处理之前进行。比如，自动为所有 Pod 加上某些标签（Labels）

### 5.1 Dynamic Admission Control

也叫作：Initializer

Initializer，作为一个 Pod 部署在 Kubernetes, 是一个事先编写好的“自定义控制器”（Custom Controller）


### 5.2 Istio Envoy 示例

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


