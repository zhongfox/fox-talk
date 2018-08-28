---
layout: post
tags : [docker, kubernetes, linux, helm]
title: 云计算

---

## Kubernetes 笔记

brew cask install minikube

minikube start

---

kubectl action resource 
kubectl action resource --help

资源如: node, container


kubectl cluster-info

kubectl get nodes

kubectl get deployments


----

内存和CPU限制:

* 指定 Container’s request, 没有指定Container’s limit: limit 使用 default memory limit for the namespace
* 指定 Container’s limit, 没指定Container’s request: request 和 limit 相同



---

crt to client-certificate-data: 进行base64编码

`cat flowtest.crt | base64`


查看证书: `openssl x509 -in ./flowtest.crt -text`


数字证书中主题(Subject)中字段的含义

公用名称 (Common Name) 简称：CN 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端证书则为证书申请者的姓名；
单位名称 (Organization Name) ：简称：O 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端单位证书则为证书申请者所在单位名称；


---

在Kubernetes中，授权有

* ABAC（基于属性的访问控制）
* RBAC（基于角色的访问控制）
* Webhook
* Node
* AlwaysDeny（一直拒绝）
* AlwaysAllow（一直允许）

## RBAC

### 角色和集群角色

在RBAC API中，角色包含代表权限集合的规则。在这里，权限只有被授予，而没有被拒绝的设置

* Role: 被用来授予访问单一命令空间中的资源
* ClusterRole: 定义集群范围的角色 

### 角色绑定和集群角色绑定

角色绑定用于将角色与一个或一组用户进行绑定，从而实现将对用户进行授权的目的。主体分为用户、组和服务帐户

角色绑定也分为角色普通角色绑定和集群角色绑定

* RoleBinding: 角色绑定只能引用同一个命名空间下的角色
*

---

## Helm


Helm 服务端正常安装完成后，Tiller默认被部署在kubernetes集群的kube-system命名空间下:

```
kubectl get pod -n kube-system -l app=helm
NAME                             READY     STATUS    RESTARTS   AGE
tiller-deploy-546cf9696c-wq5nl   1/1       Running   0          57m
```

---

## Spinnaker

* Instance: Pod

* Server Group: Replica Set

  `labels: ${SERVER-GROUP}: true`

   Docker Registry accounts associated with the Kubernetes Account

   `imagePullSecrets: name: ${DOCKER-REGISTRY}`

* Clusters

  Clusters are a logical grouping,

  one should not attempt to let Spinnaker’s orchestration (Red/Black, Highlander) manage Server Groups handled by Kubernetes’ orchestration (Rolling Update), since do not, and are not intended to work together

* Load Balancer: Service

  `selector: load-balancer-${LOAD-BALANCER}: true`

  M:N relationship between pods and services

### Deployment strategy

* 蓝绿(红黑):

  * 起始状态: 部署版本1的应用(绿)
  * 部署版本2的应用(蓝)
  * 将流量从版本1切换到版本2
  * 如版本2测试正常，就删除版本1正在使用的资源（例如实例），从此正式用版本2(绿)

  需要两份资源

  可以快速切回
  
  可能会出现需要同时处理“微服务架构应用”和“传统架构应用”的情况，如果在蓝绿部署中协调不好这两者，还是有可能会导致服务停止。
  需要提前考虑数据库与应用部署同步迁移 /回滚的问题。
  蓝绿部署需要有基础设施支持。
  在非隔离基础架构（ VM 、 Docker 等）上执行蓝绿部署，蓝色环境和绿色环境有被摧毁的风险

* 蓝绿滚动:

  切换流量比例逐步增加, 每次增加后使用validate


  更加节约资源

* 金丝雀

  从负载均衡列表中移除掉“金丝雀”服务器。
  升级“金丝雀”应用（排掉原有流量并进行部署）。
  对应用进行自动化测试。
  将“金丝雀”服务器重新添加到负载均衡列表中（连通性和健康检查）。
  如果“金丝雀”在线使用测试成功，升级剩余的其他服务器。（否则就回滚）

  不消耗额外资源


<https://www.jianshu.com/p/022685baba7d>
