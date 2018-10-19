---
layout: post
tags : [spinnaker, ci, cd, devops]
title: 

---

## Immutable server pattern

CD 不仅仅是更好的编排

2015 年底开源

Netflix, Google, Microsoft

1. Immutable infrastructure

   It means that once you deploy, something, you don't change it

   When there's change, you rebuild it up from the host up, or container up, and then tear down the old one altogether

  * 解决环境间差异问题
  * 快速回滚到老版本
  * 更好的进行CI
  * 更好的自动化
  * 更好的伸缩性, 更容易进行大规模运维(更快的部署, 更好的支持部署策略)

2. Deployment strategy

   考虑潜在异常的可能, 提供更稳健的服务质量

   * 蓝绿
   * 蓝绿滚动
   * 灰度/金丝雀

3. Automation

   构建测试部署验证等过程是可确定的
   提升部署一致性(失败一致性, 成功一致性), 可重复性

   配置漂移

   实现: pipeline

4. Oprational Integration

---


注释：矿井中的金丝雀
17世纪，英国矿井工人发现，金丝雀对瓦斯这种气体十分敏感。空气中哪怕有极其微量的瓦斯，金丝雀也会停止歌唱；而当瓦斯含量超过一定限度时，虽然鲁钝的人类毫无察觉，金丝雀却早已毒发身亡。当时在采矿设备相对简陋的条件下，工人们每次下井都会带上一只金丝雀作为“瓦斯检测指标”，以便在危险状况下紧急撤离。


* 新版本部署目标资源:

  * 新实例(旧实例保留):

    需要额外资源(但易于回滚):

    [蓝绿部署] : 100%
    [蓝绿滚动部署] : n%
    [金丝雀] : 0% / 1%

  * 旧版本实例(先去除流量): [金丝雀]


begin loop

* 新版本部署实例step:

  * 1    [金丝雀]
  * n    [蓝绿滚动]
  * 50%
  * 100% [蓝绿部署]

* 新版本实例无流量验证:

  * 无
  * 健康检查/冒烟测试/功能测试 [蓝绿(滚动)部署]
  * 自动化测试 [金丝雀]

* 新版本流量增加(伴随老版本流量减少):

  * 1    [金丝雀]
  * n:   [蓝绿滚动]
  * 50%:
  * 100%: [蓝绿部署]

* 新版本实例流量验证:

  * 无
  * 健康检查/冒烟测试/功能测试    [蓝绿(滚动)部署]
  * 金丝雀分析: 系统基础数据(cpu,内存, 网络延迟, 状态码, 错误率, ), 计算得分, 计算可接受新版和旧版得分比率 [金丝雀]

  验证失败回滚(删除新实例, 流量100%切回旧实例)

end loop

(部署阶段性完成)

* 旧版本实例保留时间:

  * 无: [Highlander]
  * 保留N小时以便回滚之需 [蓝绿(滚动)部署] [金丝雀]


---

## Artifact

an artifact is an object that references an external resource:

a Docker image
a file stored in GitHub
an Amazon Machine Image (AMI)
a binary blob in S3, GCS, etc

Any of these can be fetched using a URI, and can be used within a Spinnaker pipeline.


artifact decoration:

* artifactAccount: Spinnaker account



## 工作流:


### Trigger

* Triggering on Pub/Sub messages
* Triggering on webhooks
* Triggering on receiving artifacts from GCS
* Triggering on receiving artifacts from GitHub


---

* Instance: Pod

  命名: appname-stack-servicegroupversion-random


  ```
  Labels:         app=foxv1app1
                  cluster=foxv1app1-c3
                  foxv1app1-c3-v001=true 所属service group ?????
                  load-balancer-foxv1app1-c1-svc1=false     当前生效sg
                  replication-controller=foxv1app1-c3-v001
                  stack=c3
                  version=1
  ```

* Server Group: Replica Set

  命名: appname-stack-servicegroupversion

  ```
  Selector:     app=foxv1app1,cluster=foxv1app1-c3,foxv1app1-c3-v000=true,replication-controller=foxv1app1-c3-v000,stack=c3,version=0
  Labels:       app=foxv1app1
                cluster=foxv1app1-c3
                foxv1app1-c3-v000=true    自己的版本
                load-balancer-foxv1app1-c1-svc1=true   所属LB????? 不能变动???
                replication-controller=foxv1app1-c3-v000 ?????????
                stack=c3
                version=0
  ```

   Docker Registry accounts associated with the Kubernetes Account

   `imagePullSecrets: name: ${DOCKER-REGISTRY}`

* Clusters

  Clusters are a logical grouping,

  one should not attempt to let Spinnaker’s orchestration (Red/Black, Highlander) manage Server Groups handled by Kubernetes’ orchestration (Rolling Update), since do not, and are not intended to work together

* Load Balancer: Service

  命名: appname-stack-servicename

  `selector: load-balancer-${LOAD-BALANCER}: true`

  service: `Selector: load-balancer-foxv1app1-c1-svc1=true`

  M:N relationship between pods and services

  LB 貌似还可以跨app

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


---

Highlander: 当新版本健康检查通过后, 删除就的实例
None: 新旧实例同时存在, 都有流量
r/B: 当新版本健康检查通过后, 禁用旧实例流量, 但是保留
