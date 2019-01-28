---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes API 和控制器模型

---

## 1. API

### 1.1 声明式API

* 命令式: kubectl create,  kubectl replace
* 声明式: kubectl apply

声明式API的特点:

* 可对原有 API 对象的 PATCH 操作, 具备 Merge 能力
* 一次能处理多个写操作
* 对“实际状态”和“期望状态”的调谐（Reconcile）过程, 所以说，声明式 API，才是 Kubernetes 项目编排能力“赖以生存”的核心所在

### 1.2. API 资源对象

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

1. Filter: Authentication, Authorization, 超时, 审计
2. Handler: Mux, Routes
3. 查找该API资源定义: Group->Version->Kind
4. Convert: YAML -> Super Version 对象(该 API 资源类型所有版本的字段全集)
5. Admission()
6. Validation()
7. 序列化保存到etcd

#### Admission 准入控制器

在kubernetes中，一些高级特性正常运行的前提条件为，将一些准入模块处于enable状态.

在对集群进行请求时，每个准入控制插件都按顺序运行，只有全部插件都通过的请求才会进入系统，如果序列中的任何插件拒绝请求，则整个请求将被拒绝，并返回错误信息。

在某些情况下，为了适用于应用系统的配置，准入逻辑可能会改变目标对象。此外，准入逻辑也会改变请求操作的一部分相关资源

在kubernetes apiserver中有一个flag：admission_control，他的值为一串用逗号连接起、有序的准入模块列表，设置后，就可在对象操作前执行一定顺序的准入模块调用.

开启: `kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...`

关闭: `kube-apiserver --disable-admission-plugins=PodNodeSelector,AlwaysDeny ...`

见: <https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/>

#### Dynamic Admission Control

admission controller 对于大多数用户来说还不够灵活：

* 需要把它编译到kube-apiserver 二进制文件中
* 只能在启动时进行配置

> Admission webhooks are HTTP callbacks that receive admission requests and do something with them

有2种类型的admission:

* validating admission Webhook 
* mutating admission webhook

要求: api server 必须开启 MutatingAdmissionWebhook, ValidatingAdmissionWebhook

运行时进行admission webhooks 配置:

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: <name of this configuration object>
webhooks:
- name: <webhook name, e.g., pod-policy.example.io>
  rules:
  - apiGroups:
    - ""
    apiVersions:
    - v1
    operations:
    - CREATE
    resources:
    - pods
  clientConfig:
    service:
      namespace: <namespace of the front-end service>
      name: <name of the front-end service>
    caBundle: <pem encoded ca cert that signs the server cert used by the webhook>
```

当 api server 接收到的请求满足rule配置(group/verion/resource/operation), 将会发送特定的`admissionReview`请求到`clientConfig.service`, 然后admission webhook server 返回`admissionResponse`

#### Initializers

* 存储在每一个对象元数据中的一系列的预初始化的任务， （例如，”AddMyCorporatePolicySidecar”)
* initializer controllers: 用户自定义的 controller，用来执行那些初始化任务。任务的名称跟执行该任务的控制器是相关联的

initializer controllers 的操作流程:

* 配置 initializerConfiguration 对象, 设置规则需要处理的资源集合
* 当一个资源post后, 如果满足initializerConfiguration规则, 会将其`spec.initializers[].name` 追加到其`metadata.initializers.pending`中
* 一个 initializer controller 应该通过查询参数?includeUninitialized=true 来 list 和 watch 没有被初始化的对象, 对`metadata.initializers.pending[0]`匹配的对象, 进行具体操作, 最后需要将自己的name从该资源列表中删除

Objects which have a non-empty initializer list are considered uninitialized, and are not visible in the API unless specifically requested by using the query parameter, ?includeUninitialized=true

开启Initializers 功能:

* `kube-apiserver --enable-admission-plugins=...Initializers...`
* `--runtime-config=admissionregistration.k8s.io/v1alpha1` 这个是为了支持 initializerConfiguration 配置对象吗

顺序????

---

## 2. CRD

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

