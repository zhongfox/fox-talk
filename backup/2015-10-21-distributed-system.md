---
layout: post
tags : [分布式, web架构]
title: 分布式系统笔记

---

## 概述

### 系统分类

* 集中式
* 分布式: 硬件或者软件分布在不同的网络计算机上, 彼此仅通过消息传递进行通信和协调的系统

### 特征

* 分布性

* 对等性: 没有控制系统的主机, 也没有被控制的从机

  副本概念: 数据或者服务的一种冗余方式

  * 数据副本
  * 服务副本

* 并发性

* 缺乏全局时钟: 缺乏全局时钟序列控制

* 故障总是会发生

### 分布式系统目的

* 增强系统**可用性**, 防止单点不可用
* 提高系统整体性能, **可扩展性**, 通过负载均衡提供分布式服务

### 一致性分类

* 强一致性: 体验好, 性能影响大
* 弱一致性: 一致性延迟一定时间达成
* 最终一致性: 是弱一致性的一个特例

### 分布式环境中的问题

* 通信异常:
  * 网络延迟: 单机内存访问延迟在10ns左右, 分布式在0.1~1ms, 100+倍的延迟
  * 消息丢失普遍

* 网络分区(脑裂): 只有部分节点可以正常通信, 小部分集群需要完成原本完整集群的功能

* 节点故障

### 消息三态

* 成功
* 失败
* 超时: 通信的发起方是无法确定请求是否被成功处理

---

### 事务ACID特征

单机数据库事务可以满足:

* 原子性: 只允许成功失败2中状态
* 一致性: 事务执行不能破坏数据库的完整性和一致性
* 隔离性: 并发事务相互隔离
  SQL的4个事务隔离级别: TODO
* 持久性: 事务提交后, 对数据库的变更是永久的

### 分布式事务理论

ACID模型可以保证本地事务或者集中式事务的数据严格一致性

但ACID对于分布式系统来说, 可能造成系统的可用性和严格一致性的冲突

#### CAP 理论

一个分布式系统最多同时满足一致性(C),可用性(A), 分区容错性(C)中的2个

* C 一致性: 指数据强一致性
  放弃C: 放弃强一致性, 保证最终一致性

* A 可用性: 在有限的时间内, 返回结果
  放弃A: 出现脑裂, 服务在一定时间不可用

* P 分区容错性: 出现部分网络分区(子网络或者异地机房)故障, 需要保证可用性和一致性
  放弃P: 部署在同一机房, 意味着放弃可扩展性

分区容错性是不可舍弃的, 这是分布式的特点, 因此分布式系统通常在一致性和可用性间寻求平衡

#### BaSE 理论

是对CAP重一致性和可用性权衡的一种结果

* Basically Available: 基本可用
  * 相应时间损失
  * 功能上损失(服务降级)

* Soft state: 软状态
  不影响系统可用性的, 存在于数据副本中的中间状态

* Eventually consistent: 最终一致性
  存在5类最终一致性 TODO

----

## 数据库 undo/redo log 原理

undo/redo 用于保证事务特性, 主要是原子性和持久性, 要么全做, 要么不做, 并且修改都能持久化

### undo log

在崩溃的系统恢复后, 没有commit的操作借助undo log进行回滚.

事务进行时使用undo log:

1. undo log记录 START T
2. undo log记录 旧值
3. 写新值到数据库, 持久化
4. undo log记录commit T

宕机恢复后使用undo log回滚:

1. 扫描undo log, 找到start但是没有commit的事务
2. 对没有commit的事务, 用undo log旧值回滚

缺点：每个事务提交前将数据和Undo Log写入磁盘，这样会导致大量的磁盘IO，因此性能很低

## redo log

事务进行时使用redo log:

1. redo log记录START T
2. redo log记录新值
3. redo log记录COMMIT T
4. 新值写入数据库 

使用redo log重做事务:

1. 扫描日志，找到所有已经COMMIT的事务；
2. 对于已经COMMIT的事务，根据redo log重做事务；

当系统崩溃时，虽然数据没有持久化，但是RedoLog已经持久化。系统可以根据RedoLog的内容，将所有数据恢复到最新的状态

---

## 一致性协议

### 2PC

1. 阶段一: 提交事务请求/投票

  * 1) 事务询问: 协调者向所以参与者发送事务内容, 询问是否可以执行事务, 并等待应答
  * 2) 执行事务: 各个参与者执行事务, 并写入undo/redo, 以备可能的回滚
  * 3) 事务反馈: 各参与者如果成功, 返回yes, 失败返回no

  问题一(阻塞): 此后所有参与者都处于同步阻塞, 无法进行其他任何操作, 等待协调者反馈来解锁

2. 阶段二: 执行阶段

  有2种可能:

  * 2.1 执行事务提交(果所有参与者反馈都是yes):

    * 1) 发送提交请求: 向所有参与者发送commit
    * 2) 事务提交: 参与者接到commit后, 执行提交, 然后释放事务资源
    * 3) 反馈事务提交: 参与者向协调者发送ACK
    * 4) 完成事务: 协调者接到所有的ACK, 完成事务

    问题二(数据不一致): 步骤3,4期间, 如果部分commit丢失, 或者部分执行者崩溃, 因为后续没有再回滚的机制, 会导致系统中部分执行者提交, 出现数据不一致


  * 2.2 中断事务(有任何no的响应, 或者协调者**等待超时**)

    * 1) 发送回滚: 协调者向所有参与者发送Rollback
    * 2) 事务回滚: 执行者接到Rollback后, 使用undo log回滚, 然后释放事务资源
    * 3) 返回回滚结果: 参与者回滚完毕, 发送ACK
    * 4) 中断事务: 协调者接到所有的ACK, 完成事务中断

问题三(单点问题): 协调者是单点, 一旦协调者崩溃, 整个2PC无法进行; 另外如果单点问题出现在阶段二, 所有执行者将阻塞, 资源无法释放.

问题四(太过保守): 在协调者等待参与者答复时, 需要依赖超时判断是否中断事务, 容错机制需要完善

问题五(脑裂导致的数据不一致): 协调者发送commit后宕机, 唯一接收到这条消息的参与者也宕机, 后续即使从新选出协调者, 这条事务是否执行, 也无法知道. 这是2PC无法解决的问题

### 3PC

缓解阻塞问题, 但是还是有数据不一致

改进:

* 在二阶段中间插入了准备阶段(PreCommit)
* 投票阶段执行者没有开始执行事务, 不阻塞, 通常来讲，投票通过以后，准备阶段失败的概率会比较低.
* 协调者和参与者中都增加等待超时机制, 降低阻塞影响

1. 阶段一: CanCommit

  * 1) 协调者向所有参与者发送CanCommit询问
  * 2) 参与者判断自身情况后应答yes或者no

  改进一: 降低了阻塞范围

2. 阶段二: PreCommit

  有2种可能:

  * 2.1 执行事务预提交(协调者收到所以yes)

    * 1) 协调者向所有参与者发送preCommit, 进入preCommit阶段
    * 2) 参与者接到preCommit后执行事务, 写undo/redo 日志
    * 3) 各参与者向协调者反馈执行结果ACK, 等等后续commit或者abort, **参与者增加等待超时机制**

  * 2.2 中断事务(协调者接收到no, 或者等待yes/no超时)

    * 1) 协调者发送中断请求abort
    * 2) 中断事务: 参与者收到abort, 或者**参与者等待超时**, 都中断事务

  改进2: 参与者增加超时机制, 在协调者单点故障时, 可以继续完成一致性(所有都abort?)

3. 阶段三: doCommit

  两种可能:

  * 3.1 执行提交(2.1后, 协调者接到所有ack)

    * 1) 向所有执行者发送doCommit
    * 2) 执行者接到doCommit后, 正式提交, 然后释放事务资源
    * 3) 执行者发送ack, 协调者接到所有ack后, 完成事务

    执行者的超时机制: 在本阶段开始时如果因为协调者或网络问题，导致参与者迟迟不能收到来自协调者的commit或rollback请求，那么参与者将不会如两阶段提交中那样陷入阻塞，而是等待超时后继续commit(理由: 成功提交的几率很大), 相对于两阶段提交虽然降低了同步阻塞，但仍然无法避免数据的不一致性

  问题一: 数据不一致仍然可能, 如果此阶段, 协调者发出的rollback消息因为网络异常部分丢失, 参与者在等待超时后会自动commit, 将造成数据不一致.

  * 3.2 中断事务(2.1后协调者等待超时, 执行回执有失败的情况)

    * 1) 协调者发送rollback
    * 2) 事务回滚: 参与者接到rollback后, 通过undo日志回滚, 释放资源
    * 3) 参与者反馈事务回滚结果, ack到协调者
    * 4) 协调者接到所有ack后, 中断事务

2017.3.3补充:

[关于分布式事务、两阶段提交协议、三阶提交协议](http://www.hollischuang.com/archives/681)
[深入理解分布式系统的2PC和3PC](http://www.hollischuang.com/archives/1580)
[分布式事务2PC && 3PC](http://int64.me/2016/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A12PC%20&&%203PC.html)

---

## Paxos

无论是二阶段提交还是三阶段提交都无法彻底解决分布式的一致性问题.

> there is only one consensus protocol, and that’s Paxos” – all other approaches are just broken versions of Paxos.

世上只有一种一致性算法，那就是Paxos

---

## Raft

<http://thesecretlivesofdata.com/raft/>


---

## 分布式系统杂记

[事务与分布式事务](http://www.jianshu.com/p/d322cba3add4)

在分段事务中, 如文中先记录交易(存Transaction), 在更新卖家和买家数据, 2pc基本可以保证数据一致性, 如果采用BaSE方式, 先更新Transaction, 然后用消息队列处理买家和卖家的更新, 需要注意: 需要在买家卖家更新成功后再删除消息. 防止处理消息失败造成的数据不一致.

[牺牲一致性来换取分布式架构的可伸缩性](http://www.infoq.com/cn/news/2008/03/ebaybase)

> 出价竞拍本身就是一个很有意思的问题，原子性并不是重点，更多的是关系到在拍卖关键的最后几秒钟里不要阻塞任何出价人。如果改成在显示时刻而不是在出价时刻计算最高出价人和最高出价，就会变得非常简单。所有出价都被插入到一个单独的子表，插入操作不太会引起资源争用的情况。每次显示产品的时候，再重新取回所有的出价，并且在这个时候应用业务逻辑来决定最高的出价人

> 做为一名架构师，你必须决定什么时候应当为了满足系统的一个制约因素的要求而放松对另一个制约因素的要求

> 你必须打破常规模式，比如ACID和分布式事务。乐于寻找机会放松一些约束，即使传统上认为是不能放松的。

> 还有两条简单的原则：把每样东西都设计成分离的；考虑BASE、而不是ACID


---

## 参考资料

* [分布式事务 - 两阶段提交与三阶段提交](https://my.oschina.net/wangzhenchao/blog/736909)
* [undo log与redo log原理分析](http://blog.chinaunix.net/uid-20196318-id-3812190.html)
* [分布式协议之两阶段提交协议（2PC）和改进三阶段提交协议（3PC）](http://www.mamicode.com/info-detail-890945.html)
* [分布式系统的事务处理](http://coolshell.cn/articles/10910.html)
