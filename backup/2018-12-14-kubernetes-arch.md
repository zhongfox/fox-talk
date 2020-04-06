---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes 架构与设计

---

## 1. 容器相关标准和术语

### 1.1 OCI

Open Container Initiative）组织，旨在围绕容器格式和运行时制定一个开放的工业化标准, 目前主要有两个标准文档：容器运行时标准 （runtime spec）和 容器镜像标准（image spec）

* image spec: OCI 容器镜像主要包括几块内容:
  * 以 layer 保存的文件系统
  * config 文件：保存了文件系统的层级信息（每个层级的 hash 值，以及历史信息），以及容器运行时需要的一些信息（比如环境变量、工作目录、命令参数、mount 列表）
  * manifest 文件：镜像的 config 文件索引，有哪些 layer，额外的 annotation 信息
  * index 文件：可选的文件

* runtime spec: 主要是指定容器的运行状态，和 runtime 需要提供的命令

<img src="/assets/images/k8s/runtime.jpg" />

### 1.2 runc

> runc is a CLI tool for spawning and running containers according to the OCI specification

docker 捐赠给 OCI 的一个符合标准的 runtime 实现，目前 docker 引擎内部也是基于 runc 构建的

runC的前身实际上是Docker的libcontainer项目演化而来。runC实际上就是libcontainer配上了一个CLI

### 1.3 CNI

### 1.4 CRI

---

## 2. 架构总览

<img src="/assets/images/k8s/k8s-cluster-architecture.jpg" />

Master 组件:

* API Server: 提供Rest API
* scheduler: 绑定pod到node
* Controller Manager: 执行 controller loops
* etcd: 键值对的数据库

Node 组件:

* Kubelet

* Kubeproxy: 实现了两种proxyMode：userspace和iptables。其中userspace mode是v1.0及之前版本的默认模式，从v1.1版本中开始增加了iptables mode，在v1.2版本中正式替代userspace模式成为默认模式

  * 通过kube-proxy来实现service的代理服务: client-> (iptables: nodePort 和serviceIP 都转到kube-proxy的端口 ) -> kube-proxy -> pod
  * 完全利用内核iptables来实现service的代理和LB, kube-proxy 只负责维护iptables


* kubends

---

## 3. 控制器模型

```
$ cd kubernetes/pkg/controller/
$ ls -d */
deployment/             job/                    podautoscaler/
cloud/                  disruption/             namespace/
replicaset/             serviceaccount/         volume/
cronjob/                garbagecollector/       nodelifecycle/          replication/            statefulset/            daemon/
...
```

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

* 实际状态往往来自于 Kubernetes 集群本身: kubelet 通过心跳汇报的容器状态和节点状态，或者监控系统中保存的应用监控数据，或者控制器主动收集的它自己感兴趣的信息
* 期望状态，一般来自于用户提交的 YAML 文件

Deployment 示例:

Deployment 以及其他类似的控制器, 定义主要分为2部分:

1. 控制器定义 (除去template的内容)
2. 被控制对象定义(template 的内容)

“控制器模式” vs “事件驱动”

* 事件往往是一次性的，如果操作失败比较难处理
* 控制器是循环一直在尝试的，更符合kubernetes申明式API，最终达到与申明一致

---

## 4. Admission Controller

一个 API 对象被提交给 APIServer 之后，总有一些“初始化”性质的工作需要在它们被 Kubernetes 项目正式处理之前进行。比如，自动为所有 Pod 加上某些标签（Labels）

### 4.1 Dynamic Admission Control

也叫作：Initializer

Initializer，作为一个 Pod 部署在 Kubernetes, 是一个事先编写好的“自定义控制器”（Custom Controller）


### 4.2 Istio Envoy 示例

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
