---
layout: post
tags : [分布式, web架构, nosql, elasticsearch, redis, 集群, kafka, couchbase]
title: 分布式系统浅析

---

## 分布式系统

#### 目标

* 提高系统伸缩性
* 提高性能: 分布式意味着可以使用更多的计算机资源, 提高并发能力
* 提高可扩展性

#### 分布式带来的问题

* 服务调用必须走网络
* 分离粒度越小, 服务器越多, 服务宕机的概率也就越大
* 分布式环境中数据一致性和事务难度增大

分布式需要量力而行, 切莫为了分布式而分布式

#### 分布式系统场景

* 分布式应用和服务
* 分布式静态资源 [js css 多个域名, 独立静态资源域名]
* 分布式数据和存储 [Nosql/分布式文件系统]
* 分布式计算 [deal service 消息分级]
* 分布式配置
* 分布式锁

## Why

* 技术选型
* 系统设计
* 资源评估
* 权衡

---

## 分布式系统理论

* 异常是常态: 服务器宕机, 网络异常, 磁盘故障

* 请求三态: 成功, 失败, 超时

  超时不等同于失败. 超时处理: 不断查询之前操作状态/设计幂等请求


### ACID

### CAP

<img src="/assets/images/distributed/cap.jpg" />

* C 一致性: 数据在多个副本之间是否能够保持一致的特性
* A 可用性: 在任何时刻对大规模的数据读写操作都能保证在限定的延时内返回合理结果
* P 分区容忍性: 当部分节点无法互通出现网络分区现象，但是整个系统还是可以对外提供服务


### BASE

* 基本可用（Basically Available）
* 软状态（ Soft State）
* 最终一致性（ Eventual Consistency）

---

## 一致性协议

* 两阶段提交
* 三阶段提交
* 向量时钟
* RWN协议
* Paxos协议
* Raft协议


---

## 术语

* 机器节点(Node)

* Rebalance: online-rebalancing/auto-sharding

* 分区容错(Partition Tolerance)

* 逻辑分区: key空间/Bucket/Index/Topic

* 数据分片(Partition): Slot/vBuckets/Shard/Partition

* 数据路由(Routing):

  * 哈希分片(Hash Partition) :

    1) 哈希取模(Round Robin): `H(key)=hash(key) mod lengthOfNode`: 未区分node和Partition角色, Node增删将导致hash不稳定

    2) 虚拟桶(Virtual Buckets): `Key->Partition` `Partition->Node`

    3) 一致性哈希(Consistent Hashing): 

  * 范围分片(Range Partition)

* 复制(Replication): 按照复制粒度, 可分为节点复制和分片复制

* 复制策略: 常见的是同步和异步, 异步可以有多种权衡策略

* 失效转移(Failover)

* 读扩展

* 向量时钟, WNR

* 事务

* 乐观锁

* 悲观锁

* 持久化(Durability)

<img style="max-width:120%" src="/assets/images/distributed/big_pic.png" />

---

## 概念

#### 一致性哈希

要解决的问题:

1. 散列的不变性
2. 异常以后的分散性

<img src="/assets/images/distributed/consistent_hash_1.png" />

问题:

1. 新加入一个节点后, 新节点和分离节点的负载是其他节点的一半, 负载不均衡
2. 性能高和性能低的机器, 负载需要不同

改进: 引入虚拟层, 虚拟节点

<img src="/assets/images/distributed/consistent_hash_2.png" />

稳定性分析: N 个node的集群, 新增一个node, 数据继续命中原node的概率

* 哈希取模: 1/(N+1)
* 一致性哈希: N/(N+1)

#### 向量时钟

使用向量时钟的Nosql: Dynamo, Riak

<img src="/assets/images/distributed/vector_clock.png" />

* R 一次成功读操作中最少参与的节点数目
* W 一次成功写操作中最少参与的节点数目
* N 节点数目

当W+R > N，写成功需要的副本数 + 读成功需要的副本数 > 副本总数，则能保证最终一致性.

#### 并发情况下的写丢失

<img width="50%" src="/assets/images/distributed/write_lost.png" />

解决方案:

1. 悲观锁
2. 乐观锁

---

## CouchBase

#### 集群结构

<img src="/assets/images/distributed/node_cb.png" />

#### 分片

<img src="/assets/images/distributed/partition_cb.png" />

#### 复制

<img src="/assets/images/distributed/replication_cb.png" />

#### Rebalance

<img src="/assets/images/distributed/rebalance_cb.png" />

---

## Elasticsearch

#### Node, 分片, 复制

<img src="/assets/images/distributed/node_partition_replication_es.png" />

#### Auto Sharding

<img src="/assets/images/distributed/auto_sharding_es.png" />

<img src="/assets/images/distributed/auto_sharding_es_2.png" />

#### 失效转移

<img src="/assets/images/distributed/failover_es.png" />

#### Client-Node 交互

协调node转发

<img src="/assets/images/distributed/client-node-es.png" />

mget:

<img src="/assets/images/distributed/client-node-mget-es.png" />

#### 副本同步/写安全/复制策略(sync)

协调node转发+同步复制(同步的目标是修改后的完整文档, 而不是修改操作)

<img src="/assets/images/distributed/sync-es.png" />

批量写:

<img src="/assets/images/distributed/sync-es-bulk-write-es.png" />


---
