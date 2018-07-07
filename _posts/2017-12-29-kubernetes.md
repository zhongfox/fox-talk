---
layout: post
tags : [docker, kubernetes, linux]
title: Kubernetes 笔记

---


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

