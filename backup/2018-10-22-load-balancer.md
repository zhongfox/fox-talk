---
layout: post
title: 关于负载均衡
tags: [loadbalancer, network]
header-img: img/pic/2015/10/kanasiquanjing.jpg

---

负载均衡器位于客户端和后端之间, 是一种网络技术，它在多个备选资源中做资源分配，以达到选择最优

* 网络技术: LB要解决的问题本质上是网络的问题，所以它实际上就是通过修改数据包中MAC地址、IP地址字段来实现数据包的“中转”
* 资源: 计算机, 交换机、存储设备等
* 最优: 依赖负载算法, 轮询、加权轮询、最小负载等

负载均衡器功能:

* 服务发现
* 健康检查
* 负载均衡

分布式系统中正确使用负载均衡会带来如下好处:

* 命名抽象
* 容错
* 低成本和高性能

负载均衡器VS代理:

不是所有的代理都是负载均衡器，但绝大多数代理都将负载均衡作为主要功能

---

## 负载均衡器拓扑的类型

* 中间代理
* 边缘代理
* 嵌入式客户端库: 负载均衡器直接嵌入到服务库中
* Sidecar代理

比较:

* 中间代理拓扑是最简单的负载平衡拓扑。缺点是：单点故障，伸缩瓶颈和黑箱操作
* 嵌入式客户端库拓扑提供了最好的性能和可扩展性，但是需要在每种语言中实现该库，并跨所有服务升级库
* Sidecar代理拓扑性能比嵌入式客户端库拓扑弱，但不受任何限制

---

## 四层负载均衡

主要是TCP、UDP、SCTP协议, 这种类型的负载均衡器不管数据包是什么，只是通过修改IP头部或者以太网头部的地址实现负载均衡

TCP的连接建立，即三次握手是客户端和服务器直接建立的，负载均衡设备只是起到一个类似路由器的转发动作. (只有一个tcp连接, 像一个路由)

1. 负载均衡设备在接收到第一个来自客户端的SYN 请求时
2. 选择一个最佳的服务器，并对报文中目标IP地址进行修改(改为后端服务器IP），直接转发给该服务器

在某些部署情况下，为保证服务器回包可以正确返回给负载均衡设备，在转发报文的同时可能还会对报文原来的源地址进行修改

### L4负载均衡类型:

* TCP / UDP终端负载均衡器

  使用两个离散的TCP连接：一个在客户端和负载均衡器之间，另一个在负载均衡器和后端之间

* TCP / UDP直通负载均衡器

  TCP连接不会被负载均衡器终止。而是在连接跟踪和网络地址转换（NAT）发生后，将每个连接的数据包转发到选定的后端

* 直接服务器返回（Direct Server Return，DSR）

  DSR构建在直通负载均衡器上。DSR是一种优化，只有入口/请求数据包才能通过负载均衡器。出口/响应数据包在负载平衡器周围直接返回到客户端

  负载均衡器通常使用通用路由封装（GRE）来封装从负载均衡器发送到后端的IP数据包，而不使用NAT 。因此，当后端收到封装的数据包时，可以对其进行解封装，并知道客户端的原始IP地址和TCP端口。这允许后端直接响应客户端，而无需响应数据包流经负载平衡器

### LVS 三种模式
* NAT模式中，后端real server返回数据包是返回给负载均衡器，负载均衡器再返回给客户端

  ```
  Clint(client_ip, client_mac)
  LB(VIP_0,VIP_mac_0) (inner_ip_0, inner_mac_0)
  RS1(inner_ip_1, inner_mac_1) 网关 inner_ip_0
  RS2(inner_ip_2, inner_mac_2) 网关 inner_ip_0
  ```

  * Client -> LB: 
  * LB 从inner_mac_0发出: (源: [client_ip, inner_mac_0], 目的: [inner_ip_1, inner_mac_1])
    Linux不会“校验”源IP地址是否是本机IP地址所以即便client_ip不在LB上数据包也是可以被发送的，此时的行为相当于“路由”
  * RS1 回包给网关LB: (源:[inner_ip_1, inner_mac_1] 目的:[client_ip, inner_mac_0])
  * LB 回包给Client

  和nat关系不大

* DR（Direct Routing）, real server返回数据是直接返回给客户端（通过额外的路由）；LB通过修改请求中目标地址MAC为选定的real server实现数据转发，这就要求LB和Real Server必须在同一个广播域内

  ```
  Clint(client_ip, client_mac)
  LB(VIP_0,VIP_mac_0) (inner_ip_0, inner_mac_0)
  RS1(inner_ip_1, inner_mac_1) (VIP_1)
  RS2(inner_ip_2, inner_mac_2) (VIP_2)
  ```

  * Client -> LB: 
  * LB 从inner_mac_0发出: (源: [client_ip, inner_mac_0], 目的: [vip_1, inner_mac_1])
  * RS1 回包给Client: (源:[vip_1?, inner_mac_1?] 目的:[client_ip, inner_mac_0])
   在操作系统中返回数据的时候是根据IP地址返回的而不是MAC地址

* TUN（IP Tunneling）模式中，RS返回的数据也是直接返回给客户端，这种模式通过Overlay协议（把一个IP数据包封装到另一个数据包内部叫Overlay）避免了DR的限制

---

## 七层负载均衡

主要是HTTP、GRPC, Mysql等协议, 这种负载均衡一般会把数据包内容解析出来后通过一定算法找到合适的服务器转发请求。它是针对某个特定协议所以不通用。比如Nginx只能用于HTTP而不适用于Mysql

也称为“内容交换”，也就是主要通过报文中的真正有意义的应用层内容，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器

1. 先代理最终的服务器和客户端建立连接(三次握手)
2. 接受到客户端发送的真正应用层内容的报文，然后再根据该报文中的特定字段，再加上负载均衡设备设置的服务器选择方式，决定最终选择的内部服务器

负载均衡和前端的客户端以及后端的服务器会分别建立TCP连接(2个tcp连接, 像一个代理)


### L7负载平衡类型

L7负载均衡器现在通常内置对高级负载平衡功能的支持，如超时，重试，速率限制，断路，屏蔽，缓冲，基于内容的路由等

---

## 参考

* [Introduction to modern network load balancing and proxying](https://mp.weixin.qq.com/s/LwmYMenIG77m8F_YaXqIAg?)
* [浅析负载均衡及LVS实现](https://mp.weixin.qq.com/s?__biz=MzIxMjAzMDA1MQ==&amp;mid=2648945888&amp;idx=1&amp;sn=b2a0f62db03d5cb7c56804bfa3ef68e5&amp;chksm=8f5b53ecb82cdafaa5624ecf7abf9f06f913b31bbface1021d5bbb05c55a58b57869227c5c4b#rd)
