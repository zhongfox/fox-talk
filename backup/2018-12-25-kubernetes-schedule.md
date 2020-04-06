---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes 资源管理与调度

---

## 1. 资源模型

### 1.1 对象依赖的资源分类:

* 可压缩资源（compressible resources）: 它本身无状态，申请资源很快，也能快速正常回收, 当可压缩资源不足时，Pod 只会“饥饿”，但不会退出, 如 CPU
* 不可压缩资源（compressible resources）。它是有状态的（内存里面保存的数据），申请资源很慢（需要计算和分配内存块的空间），并且回收可能失败（被占用的内存一般不可回收）, 当不可压缩资源不足时，Pod 就会因为 OOM（Out-Of-Memory）被内核杀掉

内存和CPU限制:

* 指定 Container’s request, 没有指定Container’s limit: limit 使用 default memory limit for the namespace
* 指定 Container’s limit, 没指定Container’s request: request 和 limit 相同

设置单位:

* CPU:
  * 个数设置:  cpu=1 指的就是，这个 Pod 的 CPU 限额是 1 个 CPU
  * 分数设置:  cpu=500m, 就是 500 millicpu，也就是 0.5 个 CPU
* Memory: 支持单位:
  * Ei、Pi、Ti、Gi、Mi、Ki
  * E、P、T、G、M、K


* 当 Pod 仅设置了 limits 没有设置 requests 的时候，Kubernetes 会自动为它设置与 limits 相同的 requests 值

### 1.2 limit range

资源评级QOS:

QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时需要用到的

* Guaranteed: Pod 里的每一个 Container 都同时设置了requests 和 limits，并且 requests 和 limits 值相等的时候 (包括只设置limits省略requests, 会自动设置requests)
* Burstable: 当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别
* BestEffort: 如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort

驱逐策略: TODO

