---
layout: post
tags : [container, kubernetes, istio, 微服务, mesh]
title: 服务监听pod id 在istio中路由异常分析
subtitle:   "istio 踩坑记一"

---

## 1. 问题背景

pilot会下发XDS给数据面的pod,当前pod 接收到的inbound cluster配置中, 路由终点`socket_address`将是`127.0.0.1`, 示图如下:

![](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-035449.jpg)



这个代表本机的常量是在istio的pilot源码中写死的. 

```
// LocalhostAddress for local binding
LocalhostAddress = "127.0.0.1"

// LocalhostIPv6Address for local binding
LocalhostIPv6Address = "::1"
```


先说结论, 对于绝大部分场景, 用户服务进程监听的ip是`0.0.0.0`. 这种服务可以透明加入istio 服务网格, 但是如果用户进程监听的本机具体ip(pod ip), 这种服务无法直接加入当前isito服务网格.

关于监听地址的区别:

- 0.0.0.0: 通常叫做通配地址`INADDR_ANY` , 表示任一网络地址, 如果程序省略监听ip, 这将作为默认值.

- localhost: 是一个域名, 通常在系统hosts文件中指向ip `127.0.0.1`(ipv4) 和 `::1`(ipv6), 理论上可以任意修改为其他ip

- 127.0.0.1: 这个地址通常分配给 loopback 接口。loopback 是一个特殊的网络接口

  但是使用127.0.0.1作为loopback接口的默认地址只是一个**惯例**, 理论上可以配置为其他ip.

我们可以做以下验证:

- 如果listen(0.0.0.0:XX), 则这个端口可以被外部网络访问, 也可以被本机访问, 包括使用本机具体ip访问、127.0.0.1 访问.
- 如果 listen(127.0.0.1:XX), 则这个端口只能被本机访问, 且访问发出的ip地址必须是127.0.0.1 , 使用本机具体ip无法访问.
- 如果 listen(本机具体ip:XX), 则这个端口只能使用本机具体ip访问, **即使client端在服务本机上, 也无法通过127.0.0.1访问**

在不考虑特殊防火墙规则的情况下, 我们可以推导这样的矩阵:

![image-20190711110015870](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-040032.png)

右下角的红框正是istio的问题: 出于安全考虑或者其他历史原因, 业务中往往存在 listen 本机具体ip这种方式, 因为istio中envoy把流量终点发往`127.0.0.1`, 我们会发现这种服务是无法加入istio 的服务网格.

------

## 2. 尝试修复: 转发终点改为pod ip

通过修改istio 下发xds的源码, 将下发的终点`socket_address`由`127.0.0.1`改为当前pod的ip地址, 核心部分代码见commit: [use pod ip as inbound end](https://github.com/zhongfox/istio/commit/dd0ef4e02f5661b4448d63c55ddcb9d86fe3ee34#diff-f507fb55b7edf4a03d3c7e9aebf64a1e), 构建并部署.

测试bookinfo, 发现访问不通, 查看生成的xds, 确认`socket_address`改动是生效的.

继续调试, 发现envoy 发出的pod ip 流量被iptables拦截回了envoy, 形成死循环:

![image-20190710211703635](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-040039.png)



------

## 3. Isito 数据面 iptables 分析

尝试解决以上问题, 我们需要先回顾下 Isito 用于流量透明拦截的机制:

数据面的每个Pod会被注入一个名为`istio-init` 的initContainer, initContrainer是K8S提供的机制，用于在Pod中执行一些初始化任务. Istio 默认注入的initContainer 会在容器网络空间中配置若干iptables, 这些iptables主要用于拦截用户容器的出入流量, 发到代理进程envoy, 因此envoy才有机会实现相关的流量管控.

在部署好istio和bookinfo的环境中, 我们尝试查看details服务对应pod的iptables规则:

```
# 先通过docker ps 找到details容器对应的id, 进入容器查看iptables
$ docker exec -it --privileged a81fa3ab95c2 sh
$ sudo iptables -t nat -L
```

![image-20190710194910115](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-040047.png)


重点关注链`ISTIO_OUTPUT`中的5条规则:

规则1表示: 如果destination不是127.0.0.1/32,  转给15001(envoy监听)

规则2和3表示: 如果是envoy本身(UID/GID 是istio-proxy)发出的流量, 不做处理.

规则4表示: pod内容器间如果显式使用127.0.0.1/32相互调用, 不做处理.

规则5表示: 剩下的流量全转给15001(envoy监听)

规则2, 3, 4, 5 都好理解, 但是为什么会有规则1? 上面调试不通的原因正式因为规则1拦截掉了envoy 发给 本机 pod ip的流量.

查看源码中的代码和注释:

```
if [ -z "${DISABLE_REDIRECTION_ON_LOCAL_LOOPBACK-}" ]; then
  # Redirect app calls back to itself via Envoy when using the service VIP or endpoint
  # address, e.g. appN => Envoy (client) => Envoy (server) => appN.
  iptables -t nat -A ISTIO_OUTPUT -o lo ! -d 127.0.0.1/32 -j ISTIO_REDIRECT
fi
```

原来, 规则1是希望在这里起作用: 假设当前Pod a属于service A, Pod 中用户容器通过服务名访问服务A, envoy中负载均衡逻辑将这次访问转发到了当前的pod id, istio 希望这种场景服务端仍然有流量管控能力. 如图示:

![image-20190711194210456](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-122132.png)

上图中output流量的第4步和第5步间的iptables规则, 正是规则1:

```
Chain ISTIO_OUTPUT (1 references)
target          prot opt source     destination
ISTIO_REDIRECT  all  --  anywhere   !localhost
......
```

我们尝试构造上面的场景: 

1. 先把pilot 还原为原来的版本(`socket_address`为`127.0.0.1`)
2. 在原生bookinfo项目中, details 服务是一个终点服务, 没有再调用其他服务, 现在修改details代码, 使得details 服务再次调用details服务本身的一个新的接口, 获取price接口, 在页面上展示一个价格. 相关代码: [add details price api](https://github.com/zhongfox/istio/commit/33c2f9002520f902282832a9f6400e564523c3fe#diff-1a10aee26c3ab4e908426aeb6cb63d92)

服务管控有很多能力, 我们这里只观察其中的遥测功能:

![image-20190710223507262](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-040056.png)

可以看到, 全链路监控中, 增加了2条details 调用details服务price接口的span, 分别是client side和server side, 可以看出原生pilot版本中, 调用自身的ip场景, 两端流量管控是生效的.

------

## 4. 再次改造: 转发终点+iptables调整

测试pilot下发终点`socket_address`为pod ip, 并且去掉iptables 规则1: 

![image-20190711195219221](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-122136.png)

1. 将pilot 修改为改造后的版本

2. 去掉iptables 规则1, 该规则可以通过编辑istio-init的env来取消:

   ```
   $ kubectl -n istio-system edit configmap istio-sidecar-injector
   # 对istio-init增加env `DISABLE_REDIRECTION_ON_LOCAL_LOOPBACK: true`
   ```

3. 重建bookinfo 项目, 使用改造过的「调用自身的ip场景」的details服务.


![image-20190710230302727](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-040120.png)

访问验证, 发现: details 调用details服务price接口的span, 只存在client端的一条, 而server 端的流量没有被拦截, 因此无法进行流量控制(这里是遥测).

------

## 5. 总结

这里是一个权衡取舍:

1. 如果保留iptables 规则 1, 可以获得「调用自身的ip场景」的两端流量管控能力. 但是无法满足「服务监听pod ip接入mesh」的需求
2. 如果改造pilot, 下发终点`socket_address`为pod ip, 同时去掉iptables 规则1, 可以实现「服务监听pod ip接入mesh」的需求, 但是对于「调用自身的ip场景」, 只会存在client端的流量管控能力

![image-20190711202120481](http://zhongfox-blogimage-1256048497.cos.ap-guangzhou.myqcloud.com/2019-07-11-122140.png)

如果业务强依赖「监听pod ip」, 而不愿意改造为「监听0.0.0.0」, 可以考虑实施方案2, 方案2中各种访问路由都是透明联通的, 损失的仅仅是在场景「pod调用自身service, 且service刚好负载均衡到当前pod」中的server side 流控, 在istio中, 大部分流控功能是在client side 实现的, 通常可以接受.

---

* github issue: <https://github.com/istio/istio/issues/14738>
* discuss.istio: <https://discuss.istio.io/t/user-process-listen-to-pod-ip-result-to-traffic-failure/2643>
