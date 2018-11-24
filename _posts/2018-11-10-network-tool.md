---
layout: post
title: 网络工具
tags: [tcp, udp, network, tool]

---

## 1. ip

安装: `apt-get install iproute2`

`ip  [OPTIONS]  OBJECT  [COMMAND [ARGUMENTS]]`

使用: `COMMAND[ARGUMENTS]`

设置针对指定对象执行的操作, 一般情况下，ip支持对象的

* 增加(add)
* 删除(delete)
* 展示(show或者list)
* 设置(set)

常用参数:

*  -s -stats -statistics 　　　　 输出更为详尽的信息(如果这个选项出现两次或者多次，输出的信息将更为详尽)

实例解读:

```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
  link/ether 52:54:00:cc:a5:be brd ff:ff:ff:ff:ff:ff
  inet 10.0.0.12/24 brd 10.0.0.255 scope global eth0
     valid_lft forever preferred_lft forever
```

* 首行标志位
  * UP: 表示网卡处于启动的状态
  * BROADCAST: 表示这个网卡有广播地址，可以发送广播包
  * MULTICAST: 表示网卡可以发送多播包
  * LOWER_UP: 表示 L1 是启动的，也即网线插着
* mtu: 以太网最大传输单元
* qdisc: queueing discipline，中文叫排队规则
  * pfifo: 它不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列
  * pfifo_fast
* link/ether 网卡地址 brd 广播地址
* inet 网卡ip地址 brd 广播地址
* inet6 网卡ip V6地址


关于排队规则`pfifo_fast`:

* 它的队列包括三个波段（band）。在每个波段里面，使用先进先出规则,band 0 的优先级最高，band 2 的最低。如果 band 0 里面有数据包，系统就不会处理 band 1 里面的数据包，band 1 和 band 2 之间也是一样
* 数据包是按照服务类型（Type of Service，TOS）被分配到三个波段（band）里面的。TOS 是 IP 头里面的一个字段，代表了当前的包是高优先级的，还是低优先级的

OBJECT: 表示要管理或者要获取信息的对象, 所有的对象名都可以简写，例如：address可以简写为addr，甚至是a

### 1.1 link

网络设备, 可以启用/禁用网络设备, 改变mtu/mac地址等

查看网络设备状态:

* ip link list 显示网络设备的运行状态

* ip link show eth0 显示具体网络设备信息

  当不指定网络接口时，ip addr其实是ip addr show的简略写法

* ip link show up 只看激活的网络接口

启动指定网卡:

* ip link set down eth1

* ip link set up eth1

* ip link set dev lo up

添加网卡:

* ip link add veth-a type veth peer name veth-b 创建2张配对虚拟网卡

配置网卡到指定网络namespace:

* ip link set {设备名} netns {ns}

### 1.2 address

一个设备的协议（IP或者IPV6）地址, 管理设备与协议(ip/ipv6), 网关管理等

* ip addr show [dev] eth1

为网卡添加ip地址:

* ip addr add 10.0.0.1/24 dev eth1

* ip addr add 10.0.0.1/24 broadcast 10.0.0.255 dev eth1

* ip addr del 10.0.0.1/24 dev eth1

* ip -6 addr add 2003:0db5:0:f102::1/64 dev eth1 增加ipv6地址

*  ip -6 addr del 2002:0db5:0:f102::1/64 dev eth1 移除ipv6地址

可以使用iproute2给同一个接口分配多个IP地址，ifconfig则无法这么做

[1个网卡设置多个IP作用](https://blog.csdn.net/junqiand/article/details/80567078)

### 1.3 netns 网络namespace

* ip netns add {ns名称} 创建网络namespace, 默认会有lo回环接口

  namespace文件会出现在`/var/run/netns/`目录下

* ip netns list

* ip netns delete {ns名称}

* ip netns exec {ns名称} {cmd}  在该网络namespace下执行命令

### 1.4 neighbour

ARP或者NDISC缓冲区条目

* ip neigh list 显示邻居表

### 1.5 route

路由表条目

* ip route (show)

* ip route add default gw 20.0.0.1 增加默认网关

* ip route add default gw 20.0.0.1 dev {网卡名}  增加默认网关时指定从哪个网卡出去

* ip route del default 删除默认路由

* ip route list 显示核心路由表

路由表和设备无关?

### 1.6 rule

路由策略数据库中的规则

### 1.7 maddress

多播地址

### 1.8 mroute

多播路由缓冲区条目

### 1.9 tunnel

IP上的通道

---

## 2. ifconfig

安装: `apt-get install net-tools`

net-tools是Linux平台NET-3网络分发包，包括arp、hostname、ifconfig、netstat、rarp、route、plipconfig、slattach、mii-tool、iptunnel和ipmaddr工具

### ping

安装: `apt-get install iputils-ping`

---

## 3. route 命令

```
$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.135.128.1    0.0.0.0         UG    0      0        0 eth0
10.135.128.0    *               255.255.192.0   U     0      0        0 eth0
```

* Destination: 目标网络或目标主机
* Gateway: 网关地址，如果没有就显示星号(有的也显示0.0.0.0) 没有表示是**直连规则**
* Genmask: 网络掩码
* Iface: 接口，即eth0,eth0等网络接口名
* Flags:
  * G: 默认路由
  * H: 主机路由, 路由选择表中指向单个IP地址或主机名的路由记录, 子网掩码是255.255.255.255
  * N: 网络路由：网络路由是代表主机可以到达的网络, Destination 形如 `192.19.12`
  * U: 貌似是启用的意思

结果有点类似 `ip route show`

删除路由:

`route del -net 169.254.0.0 netmask 255.255.0.0 dev eth0`

增加路由:

`route add -net 192.168.100.0 netmask 255.255.255.0 dev eth0`

---

## 4. dig

```
% dig baidu.com

第一段是查询参数和统计
; <<>> DiG 9.10.6 <<>> baidu.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54724
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

第二段是查询内容
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;baidu.com.			IN	A  #查询该域名的A记录, A是address的缩写

第三段是DNS服务器的答复
;; ANSWER SECTION:
baidu.com.		197	IN	A	220.181.57.216
baidu.com.		197	IN	A	123.125.115.110
答复表示有2个A记录, 是TTL值（Time to live 的缩写），197表示缓存时间，此事件之内不用重新查询

第四段显示被查询域名的NS记录（Name Server的缩写），即哪些服务器负责管理该域名的DNS记录
AUTHORITY SECTION
?

第五段是上面几个NS域名服务器的IP地址，这是随着前一段一起返回的
?

第六段是DNS服务器的一些传输信息
;; Query time: 20 msec
;; SERVER: 192.168.101.2#53(192.168.101.2) 这里是只本机DNS server, 端口53
;; WHEN: Sat Nov 24 23:47:02 CST 2018
;; MSG SIZE  rcvd: 70
```


简化返回:

```
% dig +short baidu.com
220.181.57.216
123.125.115.110
```

指定查询某一DNS server:

```
dig @8.8.8.8 baidu.com
```

显示DNS的整个分级查询过程:

```
dig +trace baidu.com
```

单独查看每一级域名的NS记录 , 也可以加`+short`简化返回

```
dig ns com
dig +short ns com
```

逆向查询, ip -> 域名:
```
dig -x <ip>
```

---

## nslookup

TODO

---

## iptables

[iptable 详解](http://www.zsythink.net/archives/tag/iptables)

