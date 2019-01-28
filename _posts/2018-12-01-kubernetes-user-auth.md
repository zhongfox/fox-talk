---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes 用户与认证

---

## 1. 用户

Kubernetes 用户可分为ServiceAccount 和外部用户

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
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-bcnvr (ro) # Kubernetes 官方的 Client 包（k8s.io/client-go）可以自动加载这个目录下的文件
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

## 3 API Server 认证

### 3.1 CA证书认证

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

### 3.2 基本认证：basic-auth

`--basic-auth-file=/etc/kubernetes/basic_auth.csv`

格式为`password，username，uid, 可选的组名`, 如果有多个组，则列必须是双引号

```
 # cat /etc/kubernetes/basic_auth.csv
FM7fhRM4NUg2j0fssXf1OMolALtPpMka,admin,admin,system:masters
```

认证方式: 在http请求的header中添加一个Authorization，value是Basic base64编码后的用户名密码信息

`Authorization: BasicBASE64ENCODED(USER:PASSWORD) `

### 3.3 Token认证：token-auth

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

### 3.4 OpenID Connect Tokens 认证

TODO
