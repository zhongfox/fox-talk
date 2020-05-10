---
layout: post
tags : [container, kubernetes, istio, 微服务, mesh]
title: 基于服务网格的测试环境路由治理方案
header-img: assets/images/istio/shanghai_2.jpg

---

### 1. 场景

![image-20190731103825382](//zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-31-023827.png)

场景示例:

- 只维护一个环境:主干环境，通过增量复制组件得到分⽀支环境，进行测试
- 同一项目、服务多分支并行测试, 不同测试环境隔离
- 基于用户特征做AB test, 如地域、浏览器、cookie、referrer
- 流量特征在服务扇出的调用链中传递, 同一调用链中支持一致的路由规则

------



### 2. 需求

以上场景示例要求测试环境有以下能力:

- 服务版本定义能力
- 路由规则定义能力:
  - Version Base Routing
  - Content Base Routing
- 协议支持
- 链路信息传播(染色)

------

### 3. 服务版本定义能力

需要对同一服务, 定义不同版本, 如基础版本, 不同分支测试版本等.

k8s 提供基于label能力, 可以用于标记服务的不同workload, 原生提供服务多版本区分能力. 后续「版本」都是指k8s label base 版本.

**但是**, k8s 没有流量识别和路由定义的功能, ServiceMesh/Istio 提供相应功能.

------

### 4. 路由规则定义能力

#### 4.1 Version Base Routing:

istio 提供流量识别和路由定义的功能 (使用CRD VirtualService 定义), 可以支持「不同source 版本」 到不同「 destination版本」的路由定义, 该功能和上层服务协议无关, TCP/http 服务都可支持.

#### 4.2 Content Base Routing

对用户/方法名/host等信息的路由需求, 需要实现Content Base Routing.

istio提供了多种流量识别的能力, 其中包括常见的基于流量内容的识别, header/path/host/port/sni/网段等等

**但是**, istio的流量识别和操纵主要是基于http/http2/grpc实现, 现实是有大量其他RPC协议治理需求, 因此需要扩展istio/envoy支持的协议.

另外, 以上路由规则定义是作用于上下游端到端的服务流量, 要实现微服务中调用链流量治理(主要是路由规则和全链路跟踪), 还需要实现链路信息传播(染色传播).

------

### 6. 协议支持

#### 6.1 协议透明扩展

目前对服务网格协议扩展有几种可选方案, 这里主要讨论基于istio/envoy**协议透明扩展**方案:



![image-20190731103736410](//zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-31-023739.png)

目前istio 使用envoy作为sidecar流量代理, envoy通过filter chain 来实现流量的链式操纵, 该方案需要针对不同的私有协议, 开发一套envoy filter, 用于将私有协议透明转换为istio目前支持良好的协议, 如grpc.

配置生效过程:

1. 容器化的service A 和service B 接入mesh: 注册为k8s service, 注入sidecar
2. polit 会根据service 的端口命名, 识别serviceA/B 的(私有)协议类型, 对此特殊的(私有)协议, 下发包含「协议透明转换的filter」的xDS给sidecar
3. sidecar 接收到xDS, 会更新envoy config (配置filter chain)

流量流转过程:

1. Service A 发送私有协议, 被其sidecar拦截, sidecar配置的一个filter, 将私有协议转换为gRPC, 具体的, 需要将私有协议的destination, streamID等信息提取, 匹配, 并构造gRPC destination等头部, 把私有协议payload 放入gRPC 数据桢.
2. 流量以gRPC方式从 serviceA  sidecar 到 serviceB sidecar, 用户可以使用使用gRPC在istio体系的中治理能力.
3. Service B 对应的sidecar, 其最后一个filter 是将gRPC协议转换为指定私有协议.

![image-20190426084904072](//zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-31-024046.png)

方案主要优劣:

优: istio 控制面改造较小, 私有协议可以使用istio 大部分服务治理能力, 服务之间不需要关系协议转换, 基本零改造.

劣: 协议转换, 性能开销大

如果只针对测试环境, 性能问题应该可以接受.

#### 6.2 destination 信息

流量特征中最重要的是destination 信息, 除了ip和port外, 在RPC 中,往往还有destination信息, 不同RPC实现方式不同, 但语义相似:

| 协议   | destination (目的地)                       |
| ------ | ------------------------------------------ |
| HTTP 1 | Host header，Path                          |
| HTTP 2 | Header帧中的伪header `:authority`，`:path` |
| Dubbo  | payload 中的path/method                    |
| thirft | method                                     |

大部分协议有destination 信息, 上述协议转换的filter, 需要提取私有协议中的destination, 按照语义和gRPC 中destination进行互转.

#### 6.3 私有协议的限制和要求:

##### 6.3.1 要能够正确的拆包

envoy 需要实现私有协议序列化和反序列化, 必须要能够正确的拆包，然后以请求为单位进行转发，这是负载均衡的基础, 涉及tpc粘包拆包.

##### 6.3.2 destination信息可解析

首先, 协议需要有destination信息, 可以对应到http的host/path/header等语义

不同协议的destination有不同的实现方式, 有的在header里, 有的在payload/body里. 协议的destination需要能透明解析, 不依赖特定的IDL, 这涉及RPC序列化分类:

RPC序列化分类:

1. 序列化无类型: 如json
2. 类型信息在序列化结果中: 如java自带的序列化/hessian
3. client/server 持有序列化类型的IDL, 如thrift/grpc

其中1和2是可以支持解析destination(如果有的话). 对于3, 又可以分2种:

destination 解析不依赖IDL, 只有payload解析依赖IDL, 如grpc/thrift

destination 解析依赖client端和server端协商的 IDL, filter 无法透明获取 destination信息.

##### 6.3.3 需要支持自定义header, 实现染色信息操纵

RPC 自定义header 可以用来承载链路跟踪信息, 典型的是全链路跟踪系统中的trace信息传递, 染色信息可以认为是trace数据的一部分.

![](https://zhonghua.io/assets/images/hunter/baggage.png)

链路跟踪系统基本的要求是trace信息能在完整的调用链中传递, 具体实现方式在SDK类型的框架和sidecar类型框架中不太一致:

- SDK类型的框架:  可以做到透明的trace信息传递, 用户业务代码不需要修改, 但是要求引入指定的SDK, SDK实现了RPC调用的自动trace提取和注入. 但提取和注入和编程语言、RPC类型强相关.

- Sidecar 类型的框架: 无法做到透明的trace信息传递, 需要用户业务代码主动获取上个链路数据, 并主动注入到下个RPC header中. 虽然sidecar可以解析出和操纵 trace/染色信息, 但是无法透明实现, 原因主要是:

  1) Envoy是独立的Listener来处理Inbound和Outbound的请求。Inbound只会处理入的流量并将流量转发到本地的服务实例上。而Outbound就是根据服务发现找到对应的目标服务后端上。除了实在一个进程里外两个之间可以说没有任何关系。 对于sidecar 而言, inbound 流量和 outbound 流量是完全独立无关联的.

  2) App业务代码是处理请求不是转发请求, 前后2次RPC是否是父子关系, 只有App可以决定.

相关资料:

istio 全链路跟踪需要app 显式传递x-b3 headers: [istio DISTRIBUTED TRACINGOVERVIEW](https://istio.io/docs/tasks/telemetry/distributed-tracing/overview/)

envoy 跟踪传播:  [envoy Trace context propagation](<https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/tracing#trace-context-propagation>)

##### 6.3.4 需要有RequestId/Streamid, 以支持多路复

RequestId用来关联request和对应的response，请求报文中携带一个唯一的id值，应答报文中原值返回，以便在处理response时可以找到对应的request, 此信息不同RPC中叫法可能不同.

并非所有协议都有RequestId/Streamid, 如Dubbo/thrift/http2 带有此信息, 但是HTTP/1.1不支持多路复用并没有类似数据结构.

------

### 7. 社区类似场景和方案

#### 7.1 envoy 官方提供了http1和grpc 协议的转换:

1. client 端 http request 转grpc:

   场景: client端只支持http1, 但是要和grpc server 端通信:

   <https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/grpc_bridge>

2. server 端grpc 转http:

   场景: server 端只支持http1, 但是要和grpc client 端通信:

   <https://www.envoyproxy.io/docs/envoy/latest/configuration/http_filters/grpc_http1_bridge_filter#config-http-filters-grpc-bridge>

   假设http1 是一种私有协议的话, 这2个filter 功能非常类似我们要做的「协议透明转换」filter.

#### 7.2 蚂蚁金服service mesh产品 SOFAMesh 的通用协议扩展:

- 自研golang sidecar sofa-mosn 替代envoy作为数据面代理
- 自研通用协议 `X-PROTOCOL`, 以支持新的RPC协议低成本接入

出于技术栈和性能考虑, 蚂蚁用golang 重新实现了envoy的功能, 并实现了一套可扩展的私有协议接入方案.

SOFAMesh 相关资料:

- [SOFAMesh 的通用协议扩展](https://mp.weixin.qq.com/s?__biz=MzUzMzU5Mjc1Nw==&mid=2247484175&idx=1&sn=5cb26b1afe615ac7e06b2ccbee6235b3&chksm=faa0ecd5cdd765c3f285bcb3b23f4f1f3e27f6e99021ad4659480ccc47f9bf25a05107f4fee2&mpshare=1&scene=1&srcid=0828t5isWXmyeWhTeoAoeogw&pass_ticket=DqnjSkiuBZW9Oe68Fjiq%2Bqa6fFCyysQTR7Qgd8%2BX9FfooybAg7NXVAQdLmfG6gRX#rd)
- [SOFA github](http://github.com/alipay/sofa-mesh)

#### 7.3 有赞:

场景: 公司多语言技术栈: nodejs 使用http做RPC, java 使用dubbo 做RPC. nodejs项目需要调用java服务, 并做统一服务治理.

实现: 自研golang sidecar, 将nodejs 项目发出的http 转换为java dubbo 协议.

------

### 8. 方案预研大致工作

1. 预研私有协议选型: 序列化和反序列化、header、payload 格式等.

2. 开发envoy 协议转换filter, filter 实现destination信息, 染色信息提取、操纵;

   目前istio 使用的envoy 是改造过的, 代码: <https://github.com/istio/proxy>. 其中包括扩展的mixer client filter. 此方案filter 也应基于此repo扩展.

3. istio pilot 组件改造: 扩展service上新的私有协议的注册和识别, 在pilot和数据面envoy交互时, 针对私有协议的服务,pilot需要重组下发的xDS:

   在outbound listener 的filter chain 首, 加入「私有协议转gRPC」的filter

   在 inbound listener 的filter chain 尾, 加入「gRPC转私有协议」的filter

4. 环境搭建, 预研与验证.



