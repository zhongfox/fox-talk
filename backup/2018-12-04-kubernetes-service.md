---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes Service, LoadBalancing, Networking

---

## 1. Service

```
apiVersion: v1
kind: Service
metadata:
  labels:
    name: app1
  name: app1
  namespace: default
spec:
  type: ClusterIP/NodePort/LoadBalancer
  clusterIP:            # 在type为ClusterIP时有效, 如果没有设置系统会自动提供一个
  ports:
  - name:               # 端口名称
    port: 8080          # service暴露在cluster ip上的端口, <cluster ip>:port 是提供给集群内部客户访问service的入口
    targetPort: 8080    # pod上的端口
    nodePort: 30062     # 节点port, 提供给集群外部客户访问service的入口
  selector:
    name: app1
```

### 1.1 ServiceType

* ClusterIP:

  仅使用一个集群内部的IP地址 - 这是默认值

  如果设置为`None`, 将创建一个Headless service

* NodePort: 在集群内部IP的基础上，在集群的每一个节点的端口上开放这个服务。你可以在任意<NodeIP>:NodePort地址上访问到这个服务 

  Kubernetes的master会从由启动参数配置的范围（默认是：30000-32767）中分配一个端口，然后每一个Node都会将这个端口（在每一个Node上相同的端口）代理到你的Service。这个端口会被写入你的Service的spec.ports[*].nodePort字段中

  也可以自行设定 nodePort

* LoadBalancer:

  在公有云提供的 Kubernetes 服务里，都使用了一个叫作 CloudProvider 的转接层，来跟公有云本身的 API 进行对接。所以，在 LoadBalancer 类型的 Service 被提交后，Kubernetes 就会调用 CloudProvider 在公有云上创建一个负载均衡服务，并且把被代理的 Pod 的 IP 地址配置给负载均衡服务做后端. (然后将LB ip 配置到EXTERNAL-IP ?) 

  EXTERNAL-IP 是一个LB, 端口是服务的port, 转发到node的nodeport

  ```
  NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE       SELECTOR
  node-koa-demo-service-2   LoadBalancer   172.16.255.211   119.28.109.191   4000:31750/TCP   87d       qcloud-app=node-koa-demo-service-2
  ```

  其中port 是 `服务port:主机nodeport`, 服务内部port同样用于EXTERNAL-IP 

  另外`Service.externalIPs` 可以直接设置, Kubernetes 要求 externalIPs 必须是至少能够路由到一个 Kubernetes 的节点.


* ExternalName:

  ```
  kind: Service
  apiVersion: v1
  metadata:
    name: my-service
  spec:
    type: ExternalName
    externalName: my.database.example.com
  ```

  在`kube-dns` 里添加了一条 CNAME 记录: `Service的DNS域名 -> my.database.example.com`

### 1.2 Endpoints

被 selector 选中的 Pod，就称为 Service 的 Endpoints，可以使用 `kubectl get ep` 查看

只有处于 Running 状态，且 readinessProbe 检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉

### 1.3 Headless Service

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

状态表:

```
kNAME               TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
nginx              ClusterIP      None            <none>          80/TCP           18s
```

Headless Service 被创建后并不会被分配一个 VIP，而是会以 DNS 记录的方式暴露出它所代理的 Pod, 它所代理的所有 Pod 的 IP 地址，都会被绑定一个这样格式的 DNS 记录:

```
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

### 1.5 Service 的 iptables 实现

Service 是由 kube-proxy 组件，加上 iptables 来共同实现的

1. `KUBE-SERVICES` Service 的入口链: 拦截service VIP 流量

   kube-proxy 通过 Service 的 Informer 感知 Service 对象的添加, 然后在本机创建iptables规则:

   在链`KUBE-SERVICES`新增接管该service VIP的流量

   `-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3`

   所有目的地址是 10.0.1.175:80 (VIP+service port)的 IP 包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3

2. `KUBE-SVC-(hash)` 负载均衡链: 均分流量到POD链

   新建的 该Service对应的iptables 链, 这些规则的数目应该与 Endpoints 数目一致:

   ```
   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
   ```

   通过随机流量均分到3条新链, 每个链对应一个pod的链

3. `KUBE-SEP-(hash)` Pod DNAT链: 这些规则应该与 Endpoints 一一对应

   ```
   -A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
   -A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

   -A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
   -A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

   -A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
   -A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
   ```

   做了2件事:

   * IP 包进行了DNAT, 目的地址改为该pod的的ip和端口
   * `--set-xmark` 打上标记`0x00004000`

NodePort 的 Service 实现:

NodePort 是由kube-proxy 进程监听的

1. NodePort `KUBE-NODEPORTS` Service 的入口链

   ```
   -A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
   ```

2. `KUBE-SVC-(hash)` 负载均衡链: 同上

3. `KUBE-SEP-(hash)` Pod DNAT链: 同上

4. `KUBE-POSTROUTING` 对离开本主机去其他node的, 由Service 转发出来的 IP 包进行SNAT

   ```
   -A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
   ```

   根据`mark==0x4000` 进行匹配需要处理的包, 该标记是第三步打上的

   SNAT的目的是让具体pod的返包原路返回, 而不是由pod直接返回给node, 不过这样pod 是不知道真实的clint ip.

   如果pod 需要知道client 真实的ip, 可以设置Service `externalTrafficPolicy: local`: 这时候，一台宿主机上的 iptables 规则，会设置为只将 IP 包转发给运行在这台宿主机上的 Pod, 这也就意味着如果在一台宿主机上，没有任何一个被代理的 Pod 存在，请求会直接被 DROP 掉


### 1.6 Service 的 IPVS 实现

TODO

---

## 2. Service 与 DNS

Service 有2中访问方式:

1. VIP
2. DNS: Service 对应的域名

### 2.1 DNS A 记录:

* Normal Service: 服务A 记录: `<service-name>.<namespace>.svc.cluster.local -> my-svc的VIP`

* Pod 之间的调用可以简写成 servicename.namespace，如果处于同一个命名空间下面，甚至可以只写成 servicename 即可访问

  原因在于pod的`/etc/resolv.conf` 有定义`search`

  ```
  cat /etc/resolv.conf
  nameserver 172.18.255.53
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5
  ```
* Headless Service: 服务A 记录: `<service-name>.<namespace>.svc.cluster.local -> my-svc所有pod ip集合`

* Headless Service pod 新增 A 记录: `<pod-hostname>.my-svc.my-namespace.svc.cluster.local -> 该pod ip`

  目的是通过pod name 和 svc name 确定pod 的ip

  网络标识产生的条件是service 需要存在. (先创建svc, 再创建pod, 或者先创建好pod, 再建svc 都行)

### 2.2 DNS SRV 记录:

> DNS SRV是DNS记录中一种，用来指定服务地址。与常见的A记录、cname不同的是，SRV中除了记录服务器的地址，还记录了服务的端口，并且可以设置每个服务地址的优先级和权重。访问服务的时候，本地的DNS resolver从DNS服务器查询到一个地址列表，根据优先级和权重，从中选取一个地址作为本次请求的目标地址

Service 命名端口: `<port-name>.<protocol>.<service-name>.<namespace>.svc.cluster.localhost`

* Normal Service: 解析出service 对应的A记录和该端口号
* Headless Service: 返回多个值(所有pod A记录)和该端口号

### 2.3 Pod 与DNS

TODO: (一定有吗) Pod: `<pod-ip-address>.<namespace>.pod.cluster.local`, 如: `1-2-3-4.default.pod.cluster.local`

对于headless service, Pod.subdomain 和 Pod.hostname 也会影响dns

Pod’s DNS Policy: pod 属性dnsPolicy

* Default: 集成该pod所在node的域名解析配置, 注意这不是dnsPolicy的默认配置
* ClusterFirst: 按照配置使用存根域和上游 DNS 服务器
* ClusterFirstWithHostNet
* None: allows a Pod to ignore DNS settings from the Kubernetes environment. All DNS settings are supposed to be provided using the dnsConfig field in the Pod Spec


Pod’s DNS Config:

```
service/networking/custom-dns.yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

### 2.4 DNS 配置

CoreDNS ConfigMap options

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           upstream
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

Stub-domain and upstream nameserver:

```
consul.local:53 {
        errors
        cache 30
        proxy . 10.150.0.1
    }
```


---

## 3. Ingress

Ingress 对象，其实就是 Kubernetes 项目对“反向代理”的一种抽象, 是全局的、为了代理不同后端 Service 而设置的负载均衡服务

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com # 必须是域名不能是IP
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80

```

Ingress Controller:  Ingress 只是一个统一的抽象, 具体实现需要从社区选用 Ingress Controller, 目前包括:  Nginx、HAProxy、Envoy、Traefik 等

Nginx Ingress Controller 示例:


* 名为nginx-configuration 的 ConfigMap:

  允许通过 Kubernetes 的 ConfigMap 对象来对上述 Nginx 配置文件进行定制

* 名为nginx-ingress-controller的 Deployment:

  启动POD: 是一个监听 Ingress 对象以及它所代理的后端 Service 变化的控制器

  1. 当新的 Ingress 对象由用户创建后
  2. nginx-ingress-controller 根据 Ingress 对象里定义的内容，生成一份对应的 Nginx 配置文件（/etc/nginx/nginx.conf），并使用这个配置文件启动一个 Nginx 服务
  3. 当Ingress 对象被更新，nginx-ingress-controller 就会更新这个配置文件

一个 Nginx Ingress Controller 为你提供的服务，其实是一个可以根据 Ingress 对象和被代理后端 Service 的变化，来自动进行更新的 Nginx 负载均衡器

---

## 4. Network Policies

* Kubernetes 里的 Pod 默认都是“允许所有”（Accept All）的，即：Pod 可以接收来自任何发送方的请求；或者，向任何接收方发送请求
* 一旦 Pod 被 NetworkPolicy 选中，那么这个 Pod 就会进入“拒绝所有”（Deny All）的状态，即：这个 Pod 既不允许被外界访问，也不允许对外界发起访问
* ingress: 定义若干准入策略
* egress: 定义若干准出策略, 规则定义的是目的ip/pod
* NetworkPolicy 需要CNI 网络插件支持才能起作用
* Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成 NetworkPolicy 对应的 iptable 规则来实现的

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:                  # 作用对象, 如果留空作用对象是该命名空间下所有pod
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:                       # 或关系
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:       # 选择对象始终是pod, 这个表示所有namespace中
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

Ingress, egress 也可以定义且关系:

```
...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
      podSelector:
        matchLabels:
          role: client
  ...
```


---

本地代理service: TODO

`kubectl proxy`

`http://localhost:8001/api/v1/namespaces/default/services/foxdemo-develop:80/proxy/`



---

TODO:

istio gateway 的 port和Service :

 istio-ingressgateway pod 没有init container, 不做iptables拦截, istio gateway 的 port和Service 会被配置为入口listener, 然后根据virtualservice, 把vs的配置作为virtualhost和router 放到这些listener中


pod ContainerPort 只影响其中的HostPort, 对service 的路由没啥影响

https://kubernetes.io/zh/docs/concepts/services-networking/service/#%E6%B2%A1%E6%9C%89-selector-%E7%9A%84-service

istio service entry:

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

只是增加了crd里的cluster:

```
{
version_info: "2019-05-16T13:11:25Z/16",
cluster: {
name: "outbound|80||httpbin.org",
type: "STRICT_DNS",
connect_timeout: "10s",
circuit_breakers: {
thresholds: [
{
max_retries: 1024
}
]
},
dns_refresh_rate: "5s",
dns_lookup_family: "V4_ONLY",
load_assignment: {
cluster_name: "outbound|80||httpbin.org",
endpoints: [
{
lb_endpoints: [
{
endpoint: {
address: {
socket_address: {
address: "httpbin.org",
port_value: 80
}
}
},
load_balancing_weight: 1
}
],
load_balancing_weight: 1
}
]
}
},
last_updated: "2019-05-16T13:11:25.534Z"
},
```

VirtualService 会增加内如到名为80的route里:

```
 apiVersion: networking.istio.io/v1alpha3
 kind: VirtualService
 metadata:
   name: httpbin-ext
 spec:
   hosts:
     - httpbin.org
   http:
   - timeout: 126s
     route:
       - destination:
           host: httpbin.org
         weight: 92
       - destination:
           host: httpbin.com
         weight: 8

```

```
route_config: {
name: "80",
virtual_hosts: [
{
name: "httpbin.org:80",
domains: [
"httpbin.org",
"httpbin.org:80"
],
routes: [
{
match: {
prefix: "/"
},
route: {
weighted_clusters: {
clusters: [
{
name: "outbound|80||httpbin.org",
weight: 92,
per_filter_config: {
mixer: {
forward_attributes: {
attributes: {
destination.service.host: {
string_value: "httpbin.org"
},
destination.service.name: {
string_value: "httpbin.org"
},
destination.service.namespace: {
string_value: "default"
}
}
},
mixer_attributes: {
attributes: {
destination.service.host: {
string_value: "httpbin.org"
},
destination.service.name: {
string_value: "httpbin.org"
},
destination.service.namespace: {
string_value: "default"
}
}
},
disable_check_calls: true
}
}
},
{
name: "outbound|80||httpbin.com",
weight: 8,
per_filter_config: {
mixer: {
forward_attributes: {
attributes: {
destination.service.host: {
string_value: "httpbin.com"
}
}
},
mixer_attributes: {
attributes: {
destination.service.host: {
string_value: "httpbin.com"
}
}
},
disable_check_calls: true
}
}
}
]
},
timeout: "126s",
retry_policy: {
retry_on: "connect-failure,refused-stream,unavailable,cancelled,resource-exhausted,retriable-status-codes",
num_retries: 2,
retry_host_predicate: [
{
name: "envoy.retry_host_predicates.previous_hosts"
}
],
host_selection_retry_max_attempts: "3",
retriable_status_codes: [
503
]
},
max_grpc_timeout: "126s"
},
metadata: {
filter_metadata: {
istio: {
config: "/apis/networking/v1alpha3/namespaces/default/virtual-service/httpbin-ext"
}
}
},
decorator: {
operation: "httpbin-ext:80/*"
},
per_filter_config: {
mixer: {
disable_check_calls: true
}
}
}
]
},
```

但并不会增加listener
