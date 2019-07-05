---
layout: post
tags : [container, kubernetes, istio, 微服务, mesh]
title: 腾讯云TKE Mesh 实践

---

腾讯云 TKE Mesh 在 Kubecon-CloudNativecon China 2019 上对外发布, 以下是同场活动中「Application Level Networking」的现场demo部分.

demo 涉及代码: <https://github.com/TencentCloudContainerTeam/tcm-demo>

## 1. 背景介绍

现场演示:

1. 利用服务网格构建多分支环境
2. 通过全链路跟踪系统进行服务性能分析
3. 权重路由和灰度发布

------

#### 1.1 用户需求

<img src="https://i.loli.net/2019/06/26/5d12d0c7180bd67792.jpg" referrerpolicy="no-referrer"/>


第一个场景应用:多分支环境.

在具备一定规模的组织中, 这是一类常见的需求: 多人开发、并行测试、蓝绿部署等等场景, 不同角色的用户对环境都有逻辑上「独占」需求.

------

#### 1.2 全量环境复制

<img src="https://i.loli.net/2019/06/26/5d12d0c77330021873.jpg" referrerpolicy="no-referrer"/>

容器和kubernetes技术的流行, 极大的降低了测试、开发以及生产环境的一体化的门槛, 利用容器技术, 我们可以快速的复制并编排出一套新的运行环境,  这种全量复制方式对于小型的系统比较合适, 但是对于规模庞大, 结构复杂的大型系统, 这种方式会遇到很多挑战:

1. 首当其冲的是资源限制, 全量环境复制耗时和不确定性随着系统规模增长.
2. 环境依赖的数据存储, 外部接口, vm服务等模块复制的成本较高.
3. 全量分支环境管理成本极高, 特性和数据的冲突, 不一致难以避免.

------

#### 1.3 分支增量复制+流量控制


<img src="https://i.loli.net/2019/06/26/5d12d0c69a71f71249.jpg" referrerpolicy="no-referrer"/>

理想的情况是, 我们只维护一个主干环境, 通过增量且按需的方式复制, 获得分支环境. 新的分支环境既可以部署已有服务的新版本, 也可以对新服务进行部署, 通过流量控制策略灵活定义各环境中服务流量.

这种方式显而易见好处:

- 只维护一套稳定的基础环境
- 分支环境实现逻辑上「独享」
- 整体资源利用率高, 维护成本低

------

#### 1.4 在线商城系统

<img src="https://i.loli.net/2019/06/26/5d12d0c76072b95784.jpg" referrerpolicy="no-referrer"/>

所以让我们看看, 系统引入服务网格后, 可以如何优化这个场景.

首先我这里准备了一个demo系统, 这是一个在线电子商城, 属于多语言微服务系统, 包括nodejs, golang, ruby, python和java, 整个系统由大概10个微服务项目组成.

整个页面分三个区域: 用户信息, 推荐商品, 折扣商品:

用户访问商城首页时, mall服务会分别访问后端的服务users, recommend和discount, 分别获取用户信息, 推荐商品列表, 和折扣商品列表. 其中users服务又会从mongodb服务读写用户信息, recommend服务会从scores服务中查询商品的综合评分, 同时调用products服务获取商品详情, discount服务也会从products商品服务获取商品详情.

------

#### 1.5 多分支环境场景

<img src="https://i.loli.net/2019/06/26/5d12d0c77c13628439.jpg" referrerpolicy="no-referrer"/>

- jason希望验证推荐系统recommend 新版本v2, 这个版本在「推荐商品」区域上增加了一个banner.
- 与此同时, fox正打算验证另一个的feature: 包括对discount 和 products的修改, 同时引入了新的收藏服务favorites. 其中discount v2在「折扣商品」区域上新增一个banner, 同时products服务会通过调用新的服务favorites获取商品的收藏人数, 然后返回给前端页面.

------



## 2.  创建Demo 基准环境

#### 2.1 创建base namespace:

首先我们创建一个base namespace 用作基准测试环境:

```
kubectl create namespace base
```

#### 2.2 开启 base 命名空间中的sidecar自动注入功能

<img src="https://i.loli.net/2019/06/26/5d12d0c73f0e242509.jpg" referrerpolicy="no-referrer"/>

#### 2.3 安装base环境的服务

`install/base_apps.yaml` 包含了商城系统的9个微服务, 是k8s标准的无状态服务: deployment和service. 我这里直接用kubectl和yaml直接安装:

```
kubectl -nbase apply -f install/base_apps.yaml
```

稍等片刻查看各应用运行正常:

<img src="https://i.loli.net/2019/06/26/5d12d0c7724fb91360.jpg" referrerpolicy="no-referrer"/>

这时候网格里的服务还无法被外网访问到, 我们还需要对网格边缘流量进行配置.

#### 2.4 创建 ingress gateway

Istio 使用 Gateway 定义网格边缘的流量行为, 对比原生的k8s ingress, k8s ingress 依赖云平台解析和实现ingress rule, 同时k8s ingress rule只能定义简单的七层路由规则和virtual host. 而Istio Gateway资源本身可以配置L4-L6的功能，例如暴露的端口，TLS设置等；同时Gateway可以和绑定多个VirtualService，在VirtualService 中可以配置七层路由规则，这些七层路由规则包括根据按照服务版本对请求进行导流，故障注入，HTTP重定向，HTTP重写等所有Mesh内部支持的路由规则. Istio Gateway默认使用和数据面相同的代理envoy实现, 这使得2者的配置完全统一, 因此Gateway也具有其他高级的流控功能.

istio gateway 仅是一个描述边缘流控的CRD, 它要工作还需要配套的service, 包含proxy的pod, 以及绑定云平台LB, TKE Mesh Gateway对这些组件进行和整合, 用户只需要配置Gateway的基本流控规则, TKE Mesh会自动实现相关的service, proxy pod 以及腾讯云LB的配置.

<img src="https://i.loli.net/2019/06/26/5d12d0c743e7294107.jpg" referrerpolicy="no-referrer"/>



#### 2.5 创建mall v1版本

如果服务只有一个版本, 这里可以省略, 不过为了后面演示需要, 我们还是对mall应用创建一个版本v1:

<img src="https://i.loli.net/2019/06/26/5d12d0c7380da71705.jpg" referrerpolicy="no-referrer"/>



#### 2.6 创建mall 路由规则

gateway定义了入口流量的4层属性, 我们还需要一个virtualservice来定义七层路由, 将后端应用层服务挂到gateway上:

<img src="https://i.loli.net/2019/06/26/5d12d0c750e0f53146.jpg" referrerpolicy="no-referrer"/>

现在可以访问mall首页验证应用正常运行, 大家都是访问base 基准环境.

------



## 3. 构建jason分支环境

recommend 新版本v2 将在「推荐商品」区域增加一个的banner.

#### 3.1 jason namespace和应用部署

1. 创建名为`jason`的 namespace 用于放置隔离的分支测试环境, 并开启自动注入.

2. 创建新版的的recommend应用, 包括recommend v2版本的Deployment和Service, 这个版本在recommend区域新增了一个banner

   ```
   kubectl apply -f install/jason-apps.yaml
   ```

   <img src="https://i.loli.net/2019/06/26/5d12d11c49f9c94327.jpg" referrerpolicy="no-referrer"/>

#### 3.2 通过TKE Mesh控制台发布新的路由策略:

1. 登录用户`jason`访问recommend服务时路由到命名空间`jason`中的recommend v2版本;
2. 其他流量仍然路由到base环境中.

我们这里是要控制对recommend 的inbound流量

服务网格提供了大量的基于流量内容匹配的规则: 这里我们选择使用header, 因为我们的登录信息在cookie里

<img src="https://i.loli.net/2019/06/26/5d12d11ce761259487.jpg" referrerpolicy="no-referrer"/>

#### 3.3 进行jason登录验证

<img src="https://i.loli.net/2019/06/26/5d12d11d2322085818.jpg" referrerpolicy="no-referrer"/>

注意一点, recommend服务还依赖scores服务,  我们对jason分支环境, 只定义了recommend服务的inbound流量规则, 对于从recommend v2 服务流出的outbound流量, 仍然会访问base 环境中现存的scores服务, 这也符合我们的预期:

<img src="https://i.loli.net/2019/06/26/5d12d11cf218114995.jpg" referrerpolicy="no-referrer"/>

------



## 4. 构建fox分支环境

我们再来看看一个更复杂的情况: 分支环境包括多个服务, 而且有修改的服务, 也有新服务:

该场景中 discount 服务v2版本在discount区域新增一个banner, 同时 products 服务v2版本会新增一个数据接口调用: 从新服务favorites中查询商品的收藏人数.

对该分支环境的构建流程:

#### 4.1 fox namespace和应用部署

1. 创建名为`fox`的 namespace 用于放置隔离的分支测试环境
2. 创建discount v2版本和discount v2版本对应的Deployment和Service, 以及新服务rates v1的Deployment和Service.

```
kubectl apply -f install/fox-apps.yaml
```

<img src="https://i.loli.net/2019/06/26/5d12d11cc471439546.jpg" referrerpolicy="no-referrer"/>

#### 4.2 通过TKE Mesh控制台发布fox路由策略

1. 登录用户`fox`访问discount服务时路由到命名空间`fox`中的discount v2 版本, 其他流量仍然路由到base环境中.(图略)

2. 对于products服务的访问流量(inbound), 判断访问流量的来源, 如果属于`fox`环境, 则路由到fox下products, 其他流量仍路由到base环境

   <img src="https://i.loli.net/2019/06/26/5d12d11d09d1785730.jpg" referrerpolicy="no-referrer"/>

3. 新服务favorites只存在于namespace `fox`中,同样我们通过来源判断, 限制其访问流量(inbound), 只能来自`fox`环境. (图略)

(以上操作等价于`kubectl apply -f install/fox-routing.yaml`)

#### 4.3 应用成功后, 进行fox登录访问验证:

<img src="https://i.loli.net/2019/06/26/5d12d11d44e5a86113.jpg" referrerpolicy="no-referrer"/>

#### 4.4 查看服务拓扑变化

<img src="https://i.loli.net/2019/06/26/5d12d11d104b888896.jpg" referrerpolicy="no-referrer"/>



------

## 5. 通过全链路跟踪辅助系统分析

分布式系统和微服务架构的演进, 使得服务之间开始产生复杂的交互, 系统的能见度越来越弱. 用户看到的一次请求响应, 通常会触发多个系统间的RPC调用或存储操作. 而任何一个子系统的低效都会导致最终的响应缓慢.

服务网格的一个重要的特性就是通过遥测提升系统的可观测性, 其中就包括全链路跟踪系统,  通过遥测数据来描绘整个系统的流量特征.

常见的链路分析包括:

- 调用关系分析
- 异常分析
- 请求耗时分析
- RPC串行并行逻辑优化
- 请求合并优化

<img src="https://i.loli.net/2019/06/26/5d12d11d3ba7645992.jpg" referrerpolicy="no-referrer"/>



通过查分析以上全链路, 我们可以发现以下问题:

1. mall 服务调用users, recommend, discount服务时, 时间线是串行, 耗时是3个调用累加.(图中红色备注)
2. recommend服务调用scores服务是每次rpc查询单个商品, 6个商品总共发生6次请求往返, 耗时累加.(图中蓝色备注)
3. 调用users服务返回错误, 经查是因为业务查询不到输入用户, 返回404.



------

## 6. 权重路由和灰度发布

根据全链路跟踪我们分析出了系统问题所在, 接下来我们就要尝试改造:

当我们修改mall应用的代码, 将调用3个服务的逻辑从串行改为并行, 然后制作成v2版本镜像.

#### 6.1 创建mall v2 deployment

```
kubectl apply -f install/mall-v2-apps.yaml
```

<img src="https://i.loli.net/2019/06/26/5d12d11d1cd1781460.jpg" referrerpolicy="no-referrer"/>



#### 6.2 新建mall v2 版本:

<img src="https://i.loli.net/2019/06/26/5d12d1266aa3e62106.jpg" referrerpolicy="no-referrer"/>

创建以后仍然不会有访问流量, 因为之前的配置mall应用只会访问v1.

#### 6.3 灰度发布

修改mall 路由策略, 按照权重50-50 路由到v1和v2版本:

<img src="https://i.loli.net/2019/06/26/5d12d126b72d770874.jpg" referrerpolicy="no-referrer"/>

#### 6.4 验证

访问页面, 版本v1 和v2 随机切换, 验证新版本无异常, 可以查看基准环境拓扑变化:

<img src="https://i.loli.net/2019/06/26/5d12d126cf93777622.jpg" referrerpolicy="no-referrer"/>

查看v2版本的全链路跟踪, 确认请求性能得到优化:  mall 服务调用users, recommend, discount服务方式, 从串行改为了并行, 理论耗时从三次访问耗时之和降低为三次访问耗时的最大值.

<img src="https://i.loli.net/2019/06/26/5d12d126f070439180.jpg" referrerpolicy="no-referrer"/>

#### 6.4 完成发布

将流量全部切为 mall v2:

<img src="https://i.loli.net/2019/06/26/5d12d126e2c7549273.jpg" referrerpolicy="no-referrer"/>

验证系统运行符合预期.

对于问题「recommend服务调用scores服务是每次rpc查询单个商品, 6个商品总共发生6次请求往返, 耗时累加」, 现场不再演示修复过程, 感兴趣的读者可以在tke mesh上, 尝试解决该问题, 将6次请求往返优化为一次批量查询.
