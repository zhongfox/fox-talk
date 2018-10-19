---
layout: post
tags : [docker, kubernetes, linux, helm]
title: Kubernetes 笔记

---

## 资源

* [Kubernetes 调度器介绍](https://www.qikqiak.com/post/kube-scheduler-introduction/)

## Kubernetes 笔记

brew cask install minikube

minikube start

---

kubectl action resource 
kubectl action resource --help

资源如: node, container


kubectl cluster-info

kubectl get nodes

kubectl get deployments


---

## Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    ports:
    - name: 端口名称
      containerPort: 80
      hostPort:            # 当指定hostPort之后，同一台宿主机将无法启动该容器的第2份副本, 尽量不用
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

----

## Service

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

ServiceType:

* ClusterIP:

  仅使用一个集群内部的IP地址 - 这是默认值

* NodePort: 在集群内部IP的基础上，在集群的每一个节点的端口上开放这个服务。你可以在任意<NodeIP>:NodePort地址上访问到这个服务 

  Kubernetes的master会从由启动参数配置的范围（默认是：30000-32767）中分配一个端口，然后每一个Node都会将这个端口（在每一个Node上相同的端口）代理到你的Service。这个端口会被写入你的Service的spec.ports[*].nodePort字段中

  也可以自行设定 nodePort

* LoadBalancer:

  一种实现是: 增加EXTERNAL-IP, 是一个LB, 端口是服务的port, 转发到node的nodeport


```
NAME                      TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE       SELECTOR
node-koa-demo-service-2   LoadBalancer   172.16.255.211   119.28.109.191   4000:31750/TCP   87d       qcloud-app=node-koa-demo-service-2
```

其中port 是 `服务内部port:主机nodeport`, 服务内部port同样用于EXTERNAL-IP




本地代理service:

`kubectl proxy`

`http://localhost:8001/api/v1/namespaces/default/services/foxdemo-develop:80/proxy/`

---

## ReplicaSet


删除RC以及pod: `kubectl delete rs XXX`

仅删除RC: `kubectl delete rs XXX --cascade=false`


常用子命令:

kubectl sale rc XXX --replicas=3

### rolling-update

* `kubectl rolling-update new-rc-name -f rc.yaml`

  new-rc-name 需要和原来rc名字不同

  TODO: 怎么知道旧的rc是哪个?

* `kubectl rolling-update rc-name --image=xxxxxxxxxx`

   不需要改rc名字, k8s 会先创建一个临时新的rc, 滚动完再rename.


`kubectl rolling-update my-nginx --rollback`

更新历史和回滚?

---

## Deployment

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 25            # pod副本数量
  selector: .......
  minReadySeconds: 5      # 新创建的Pod状态为Ready持续的时间至少为.spec.minReadySeconds才认为Pod Available(Ready)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 3         # 可以为整数或者百分比，默认为desired Pods数的25%. Scale Up新的ReplicaSet时，按照比例计算出允许的MaxSurge，计算时向上取整(比如3.4，取4)
      maxUnavailable: 2  # 可以为整数或者百分比，默认为desired Pods数的25%. Scale Down旧的ReplicaSet时，按照比例计算出允许的maxUnavailable，计算时向下取整(比如3.6，取3)
  template:
```

在Deployment rollout时，需要保证Available(Ready) Pods数不低于 desired pods number - maxUnavailable; 保证所有的Pods数不多于 desired pods number + maxSurge

deployment 会自动创建RS, RS命名: `[DEPLOYMENT-NAME]-[POD-TEMPLATE-HASH-VALUE]`

### rollout

#### 策略:

* RollingUpdate: 默认策略,  会创建新的RS

  rollout history和ReplicaSet的对应关系，可以在kubectl describe rs $RSNAME返回的revision字段中得到，这里的revision就对应着rollout history返回的revison `deployment.kubernetes.io/revision=3`

  如果手动delete某个ReplicaSet，对应的rollout history就会被删除，也就是还说你无法回滚到这个revison了

* ReCreate: 先删除现存的pod, 然后在创建新的: todo 有没有新的rc和reversion???

#### 发起滚动升级:

修改模板定义: `kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1`

`--record` 参数(kubectl create 等命令也支持): 记录当前操作到annotations里, 和revision没关系

#### revision

revision 是在rollout发起时自动被创建的, reversion只在`Deployment’s pod template (.spec.template) ` 改变时被创建, `scaling the Deployment`并不会创建新的revision.

查看滚动更新历史:

`kubectl rollout history deployment/nginx-deployment`

`kubectl rollout history deployment/nginx-deployment --revision=2`

#### rollback

 all of the Deployment’s rollout history is kept in the system so that you can rollback anytime you want (you can change that by modifying revision history limit)

`kubectl rollout undo deployment/nginx-deployment`

`kubectl rollout undo deployment/nginx-deployment --to-revision=2`

回滚的时候也是按照滚动的机制进行的，同样要遵守maxSurge和maxUnavailable的约束。并不是一次性将所有的Pods删除，然后再一次性创建新的Pods

#### 常用命令:

查看 rollout 状态: `kubectl rollout status deployment/nginx-deployment`

scale: `kubectl scale deployment nginx-deployment --replicas=10`

---

内存和CPU限制:

* 指定 Container’s request, 没有指定Container’s limit: limit 使用 default memory limit for the namespace
* 指定 Container’s limit, 没指定Container’s request: request 和 limit 相同

---

crt to client-certificate-data: 进行base64编码

`cat flowtest.crt | base64`


查看证书: `openssl x509 -in ./flowtest.crt -text`


数字证书中主题(Subject)中字段的含义

公用名称 (Common Name) 简称：CN 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端证书则为证书申请者的姓名；
单位名称 (Organization Name) ：简称：O 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端单位证书则为证书申请者所在单位名称；


---

在Kubernetes中，授权有

* ABAC（基于属性的访问控制）
* RBAC（基于角色的访问控制）
* Webhook
* Node
* AlwaysDeny（一直拒绝）
* AlwaysAllow（一直允许）

## RBAC

### 角色和集群角色

在RBAC API中，角色包含代表权限集合的规则。在这里，权限只有被授予，而没有被拒绝的设置

* Role: 被用来授予访问单一命令空间中的资源
* ClusterRole: 定义集群范围的角色 

### 角色绑定和集群角色绑定

角色绑定用于将角色与一个或一组用户进行绑定，从而实现将对用户进行授权的目的。主体分为用户、组和服务帐户

角色绑定也分为角色普通角色绑定和集群角色绑定

* RoleBinding: 角色绑定只能引用同一个命名空间下的角色






---

## Helm


Helm 服务端正常安装完成后，Tiller默认被部署在kubernetes集群的kube-system命名空间下:

```
kubectl get pod -n kube-system -l app=helm
NAME                             READY     STATUS    RESTARTS   AGE
tiller-deploy-546cf9696c-wq5nl   1/1       Running   0          57m
```


主要概念:

* chart：包含了创建Kubernetes的一个应用实例的必要信息, tgz包
* config：包含了应用发布配置信息
* release：是一个 chart 及其配置的一个运行实例
* Repoistory: Helm 的软件仓库，Repository 本质上是一个 Web 服务器，该服务器保存了一系列的 Chart 软件包以供用户下载，并且提供了一个该 Repository 的 Chart 包的清单文件以供查询。Helm 可以同时管理多个不同的 Repository

Helm组件:

* Helm Client 是用户命令行工具
* Tiller Server是一个部署在Kubernetes集群内部的 server

client 管理 charts，而 server 管理发布 release

常用操作:


仓库交互:

* `helm repo list` 查看仓库列表
* `helm search mychart` 搜索指定包
* `helm serve &` 启动本地仓库, 指定绑定`helm serve --address 192.168.100.211:8879 &`


包管理:

* `helm lint mychart` 检查依赖和模板配置是否正确
* `helm package` 打包, 如 `helm package hello-helm`


release 管理:

* `helm install XXXX` 安装, 如`helm install ./hello-helm`
* `helm list` 查看release
* `helm ls --deleted` 使用 --deleted 参数来列出已经删除的 Release
* `helm delete releasename` 删除release

默认情况下已经删除的 Release 只是将状态标识为 DELETED 了 ，但该 Release 的历史信息还是继续被保存的, 如果要移除指定 Release 所有相关的 Kubernetes 资源和 Release 的历史记录，可以用如下命令:

* `helm delete --purge mike-test`

release版本管理:

* 查看release 历史: `helm history mike-test`
* 升级 `helm upgrade mike-test local/mychart` 你可以通过 --version 参数指定需要升级的版本号，如果没有指定版本号，则缺省使用最新版本
* 回滚: `helm rollback mike-test 1` 其中的参数 1 是 helm history 查看到 Release 的历史记录中 REVISION 对应的值


[Helm 入门指南](https://mp.weixin.qq.com/s/f-sHjfIGh2ESm0ovtY8DSw)









---------------

参考: <http://hustcat.github.io/docker-image-new-format/>

`/var/lib/docker`

---


----











---

## 认证

认证相关的配置: <https://github.com/docker/distribution/blob/master/docs/configuration.md>

其中token的配置:

* realm: 描述了认证的enpoint, 如`realm="https://registry.example.com:5001/auth"`
* service: `描述了持有资源服务器的名称` 应该表示registry自己
* issuer: token发布者, 发布者会把这个插入token, 所有registry需要设置一样的
* rootcertbundle: 证书绝对地址, 用于解开token

也可以使用环境变量如: `REGISTRY_AUTH_TOKEN_REALM` `REGISTRY_AUTH_TOKEN_SERVICE`


以docker-compose为例

```
registry:
  ports:
    - 5000:5000/tcp
  image: registry:2
  volumes:
    - "./certs:/certs:ro"
    - "./registry/storage:/var/lib/registry:rw"
  environment:
    - REGISTRY_AUTH=token
    - REGISTRY_AUTH_TOKEN_REALM=http://172.16.137.217:8080/auth
    - REGISTRY_AUTH_TOKEN_SERVICE="Docker registry"
    - REGISTRY_AUTH_TOKEN_ISSUER="Auth Service"
    - REGISTRY_HTTP_SECRET=secretkey
    - REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE=/certs/auth.crt
    - REGISTRY_LOG_LEVEL=debug
```

认证过程参考: <http://dockone.io/article/845> <http://yunlzheng.github.io/2016/11/29/docker-registry-details/>

1. docker client 尝试到registry中进行push/pull操作
2. registry会返回401未认证信息给client(未认证的前提下)，同时返回的信息中还包含了到哪里去认证的信息

   ```
   HTTP/1.1 401 Unauthorized
   Server: nginx/1.4.7
   Date: Sun, 22 Nov 2015 09:01:42 GMT
   Content-Type: application/json; charset=utf-8
   Content-Length: 87
   Connection: keep-alive
   Docker-Distribution-Api-Version: registry/2.0
   Www-Authenticate: Bearer realm="http://172.16.137.217:8080/auth",service="Docker registry",scope="repository:samalba/my-app:pull,push"
   X-Content-Type-Options: nosniff
   ```

   其中:realm, service 同registry的配置

   scope: 描述了client端操作资源的方法(push/pull) `scope="repository:shuyun/hello:push"`

   account: 描述了操作的账号 `account=admin`

3. client发送认证请求到认证服务器(authorization service)

   Docker Client提供用户输入用户名和密码后向auth server的Endpoint发送请求：`http://172.16.137.217:8080/auth?service=Docker registry&scope=repository:samalba/my-app:pull,push`

   同时在http head中包含用户相关的登录信息: `authorized: Basic YWtaW46cGzc3dvmcQ=`

4. 认证服务器(authorization service)返回token

   基于JWT协议规范使用私钥对返回内容签名生成相应的Token:

   ```
   HTTP/1.1 200 OK
   Content-Type: application/json

   {"token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImtpZCI6IlBZWU86VEVXVTpWN0pIOjI2SlY6QVFUWjpMSkMzOlNYVko6WEdIQTozNEYyOjJMQVE6WlJNSzpaN1E2In0.eyJpc3MiOiJhdXRoLmRvY2tlci5jb20iLCJzdWIiOiJqbGhhd24iLCJhdWQiOiJyZWdpc3RyeS5kb2NrZXIuY29tIiwiZXhwIjoxNDE1Mzg3MzE1LCJuYmYiOjE0MTUzODcwMTUsImlhdCI6MTQxNTM4NzAxNSwianRpIjoidFlKQ08xYzZjbnl5N2tBbjBjN3JLUGdiVjFIMWJGd3MiLCJhY2Nlc3MiOlt7InR5cGUiOiJyZXBvc2l0b3J5IiwibmFtZSI6InNhbWFsYmEvbXktYXBwIiwiYWN0aW9ucyI6WyJwdXNoIl19XX0.QhflHPfbd6eVF4lM9bwYpFZIV0PfikbyXuLx959ykRTBpe3CYnzs6YBK8FToVb5R47920PVLrh8zuLzdCr9t3w", "expires_in": 3600,"issued_at": "2009-11-10T23:00:00Z"}
   ```

5. client携带这附有token的请求，尝试请求registry

  `Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJFUzI1NiIsImtpZCI6IkJWM0Q6MkFWWjpVQjVaOktJQVA6SU5QTDo1RU42Ok40SjQ6Nk1XTzpEUktFOkJWUUs6M0ZKTDpQT1RMIn0.eyJpc3MiOiJhdXRoLmRvY2tlci5jb20iLCJzdWIiOiJCQ0NZOk9VNlo6UUVKNTpXTjJDOjJBVkM6WTdZRDpBM0xZOjQ1VVc6NE9HRDpLQUxMOkNOSjU6NUlVTCIsImF1ZCI6InJlZ2lzdHJ5LmRvY2tlci5jb20iLCJleHAiOjE0MTUzODczMTUsIm5iZiI6MTQxNTM4NzAxNSwiaWF0IjoxNDE1Mzg3MDE1LCJqdGkiOiJ0WUpDTzFjNmNueXk3a0FuMGM3cktQZ2JWMUgxYkZ3cyIsInNjb3BlIjoiamxoYXduOnJlcG9zaXRvcnk6c2FtYWxiYS9teS1hcHA6cHVzaCxwdWxsIGpsaGF3bjpuYW1lc3BhY2U6c2FtYWxiYTpwdWxsIn0.Y3zZSwaZPqy4y9oRBVRImZyv3m_S9XDHF1tWwN7mL52C_IiA73SJkWVNsvNqpJIn5h7A2F8biv_S2ppQ1lgkbw`

6. registry接受了认证的token并且使得client继续操作


### JWK

#### 头部（Header）

```
{
    "typ": "JWT", 当使用JWT时，typ固定为“JWT”
    "alg": "ES256", 签名算法
    "kid": "PYYO:TEWU:V7JH:26JV:AQTZ:LJC3:SXVJ:XGHA:34F2:2LAQ:ZRMK:Z7Q6" 和公钥相关, TODO
}
```

进行Base64编码，之后的字符串就成了JWT的Header（头部）

#### 载荷（Payload）

```
{
    "iss": "Auth Service", 该JWT的签发者 //需要注意必须与auth.token.issuer配置保持一致
    "sub": "some id", 该JWT所面向的用户 //根据业务系统的规则自定义生成即可
    "aud": "Docker registry", 接收该JWT的一方 // 从请求的service参数获取
    "exp": 1415387315, //过期时间
    "nbf": 1415387015, // not before 可选参数
    "iat": 1415387015, // 正式发行时间
    "jti": "tYJCO1c6cnyy7kAn0c7rKPgbV1H1bFws", //随机生成即可
    // access根据请求的scope获取，当然业务系统要判断用户的实际权限并在actions中放回
    "access": [
        {
            "type": "repository",
            "name": "samalba/my-app",
            "actions": [
                "pull",
                "push"
            ]
        }
    ]

}
```

将上面的JSON对象进行[base64编码]可以得到下面的字符串。这个字符串我们将它称作JWT的Payload

#### 签名（签名）

```
RSASHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  'auth.key file content  也就是秘钥'
)
```

最后把以上三个结果再次通过点号join在一起, 就是完整的JWT

----



---


## TODO 临时笔记 移走

####  MIME Type

资源的媒体类型, 由 Web 服务器告知浏览器的，更准确地说，是通过 Content-Type 来表示的, 浏览器区分它们，决定什么内容用什么形式来显示.

通常只有一些在互联网上获得广泛应用的格式才会获得一个 MIME Type，如果是某个客户端自己定义的格式，一般只能以 application/x- 开头

#### Http报头

* 通用报头
* 请求报头
* 响应报头
* 实体报头

请求方的http报头结构：通用报头|请求报头|实体报头
响应方的http报头结构：通用报头|响应报头|实体报头


*  Accept header: 属于请求头. 代表发送端（客户端）希望接受的数据类型.

*  Content-Type: 属于实体头, 客户端和服务器端都可以发送, 客户端用此表示上传的实体类型, 如POST/PUT, 服务器端用此表示返回的实体类型.

  对于HTML form submission, 通常的response Content-Type:

  * application/x-www-form-urlencoded (default, older, simpler, slightly less overhead for small amounts of simple ASCII text, no file upload support)
  * multipart/form-data (newer, adds support for file uploads, more efficient for large amounts of binary data or non-ASCII text)


参考:

<https://webmasters.stackexchange.com/questions/31212/difference-between-the-accept-and-content-type-http-headers>

<http://www.w3school.com.cn/media/media_mimeref.asp>




---

## HTTPS

https://www.cnblogs.com/xcloudbiz/articles/5526262.html

* Insecure Registry: 接收plain http访问的Registry.

  需要客户端的Docker daemon上配置: `DOCKER_OPTS="--insecure-registry 10.10.105.71:5000 ....`

* Secure Registry: transport采用tls, 需要为Registry配置tls所需的key和crt文件

  ```
  $ docker run -d -p 5000:5000 --restart=always --name registry \
  -v `pwd`/data:/var/lib/registry \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \ 证书
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \         私钥
  registry:2
  ```

---



