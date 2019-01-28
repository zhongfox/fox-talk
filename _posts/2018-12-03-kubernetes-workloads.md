---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes Workloads

---

K8s 编排对象可以分为两大类:

1. 在线业务/Long Running Task: Deployment, StatefulSet, DaemonSet
2. 离线业务/ Batch Job: Job

---

## 1. Pod

### 1.1 Pod 设计理念

原生容器的“单进程模型”: 

> 并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。可是，用户编写的应用，并不能够像正常操作系统里的 init 进程或者 systemd 那样拥有进程管理的功能。比如，你的应用是一个 Java Web 程序（PID=1），然后你执行 docker exec 在后台启动了一个 Nginx 进程（PID=3）。可是，当这个 Nginx 进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？

模型类比:

* 容器: Linux 进程

* pod: 对应 Linux 进程组(超亲密) or 虚拟机(凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的)

  Pod 作为kubernetes中的一等公民, 用来组织单容器之间的超亲密关系 

* Kubernetes: Linux 操作系统

Pod 特点:

* Pod 只是一个逻辑概念, 用于组织超亲密关系的容器
* Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起, 这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器
* Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关
* Pod 是 Kubernetes 里的原子调度单位
* 共享: Network Namespace, 
* Volume 的定义都设计在 Pod 层级, Pod 里的容器只要声明挂载这个 Volume，就可以共享这个 Volume 对应的宿主机目录

Init Container:

在 Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动

容器设计模式之一: sidecar

sidecar 指的就是我们可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作

实例: 

```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001
  volumes:
  - name: app-volume
    emptyDir: {}
```

Tomcat 容器是我们要使用的主容器，而 WAR 包容器的存在，只是为了给它提供一个 WAR 包而已。所以，我们用 Init Container 的方式优先运行 WAR 包容器，扮演了一个 sidecar 的角色

### 1.2 Pod 使用

Spec重要属性:

* shareProcessNamespace:  Pod 里的容器要共享 PID Namespace, 默认并不是共享的
* NodeSelector
* NodeName: 一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字
* Affinity
  * NodeAffinity
  * PodAffinity
  * PodAntiAffinity
* Containers 容器数组, 各item的重要属性:
  * name: shell
  * image: busybox
  * ImagePullPolicy: Always/Never/IfNotPresent, 默认 IfNotPresent
  * stdin: true/false 等同于设置了 docker run 里的 -i
  * tty: true/false 等同于设置了 docker run 里的 -t
* HostAliases：追加 Pod 的 hosts 文件（比如 /etc/hosts）里的内容, 要设置 hosts 文件里的内容，一定要通过这种方法。否则，如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容
* hostNetwork: true/false
* hostIPC: true/false
* hostPID: true/false
* Lifecycle:
  * postStart
  * preStop

Lifecycle 执行顺序:

容器启动后 -> (开始执行ENTRYPOINT (同时) 执行postStart)

容器被杀死之前（比如，收到了 SIGKILL 信号） -> 执行preStop -> (串行) 容器被杀死

如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态

### 1.3 Projected Volume

投射数据卷: 它们存在的意义不是为了存放容器里的数据，也不是用来进行容器和宿主机之间的数据交换。这些特殊 Volume 的作用，是为容器提供预先定义好的数据。所以，从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”（Project）进入容器当中的

#### 1.3.1 Secret；

* 通过挂载方式进入到容器里的 Secret，一旦其对应的 Etcd 里的数据被更新，这些 Volume 里的文件内容，同样也会被更新。其实，这是 kubelet 组件在定时维护这些 Volume
* 更新可能会有一定的延时。所以在编写应用程序时，在发起数据库连接的代码处写好重试和超时的逻辑，绝对是个好习惯

```
containers:
- name: test-secret-volume
  image: busybox
  args:
  - sleep
  - "86400"
  volumeMounts:
  - name: mysql-cred
    mountPath: "/projected-volume" # 明文密码分别存在与 /projected-volume/user 和 /projected-volume/pass
    readOnly: true
volumes:
- name: mysql-cred # 一个卷, 多个secret 对象, 多个文件
  projected:
    sources:
    - secret:
        name: user # 这是secret的名字, 需要独立创建
    - secret:
        name: pass # 这是secret的名字, 需要独立创建
```

#### 1.3.2 ConfigMap

* 支持自动更新

#### 1.3.3 Downward API

* 它的作用是：让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息
* Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息
* 支持自动更新

```
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo # /etc/podinfo/labels 文件中会有键值对的labels
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

支持的pod元数据:

```
1. 使用 fieldRef 可以声明使用:
spec.nodeName - 宿主机名字
status.hostIP - 宿主机 IP
metadata.name - Pod 的名字
metadata.namespace - Pod 的 Namespace
status.podIP - Pod 的 IP
spec.serviceAccountName - Pod 的 Service Account 的名字
metadata.uid - Pod 的 UID
metadata.labels['<KEY>'] - 指定 <KEY> 的 Label 值
metadata.annotations['<KEY>'] - 指定 <KEY> 的 Annotation 值
metadata.labels - Pod 的所有 Label
metadata.annotations - Pod 的所有 Annotation

2. 使用 resourceFieldRef 可以声明使用:
容器的 CPU limit
容器的 CPU request
容器的 memory limit
容器的 memory request

```

#### 1.3.4 ServiceAccountToken

是一种特殊的 Secret

```
Containers:
...
  Mounts:
    /var/run/secrets/kubernetes.io/serviceaccount from default-token-s8rbq (ro)
Volumes:
  default-token-s8rbq:
  Type:       Secret (a volume populated by a Secret)
  SecretName:  default-token-s8rbq
  Optional:    false
```

### 1.4 容器健康检查和恢复机制

#### 1.4.1 恢复机制

pod.spec.restartPolicy: Pod 恢复机制, 默认Always

* Always：在任何情况下，只要容器不在运行状态，就自动重启容器
* OnFailure: 只在容器 异常时才自动重启容器
* Never: 从来不重启容器

Kubernetes 中并没有 Docker 的 Stop 语义。所以虽然是 Restart（重启），但实际却是重新创建了容器

设置恢复策略注意事项:

* 执行任务类的pod 成功会进入 Succeeded 状态。如果用 restartPolicy=Always 策略强制重启这个 Pod 的容器可能不是用户想要的
* 如果你要关心这个容器退出后的上下文环境，比如容器退出后的日志、文件和目录，就需要将 restartPolicy 设置为 Never。因为一旦容器被自动重新创建，这些内容就有可能丢失掉了（被垃圾回收了）


对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数

Pod 的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。事实上，一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去, 除非使用 Deployment 这样的“控制器”来管理 Pod


#### 1.4.2 livenessProbe:

周期性检查, 决定pod的生命周期

支持 HTTP/TCP/CMD

#### 1.4.3 readinessProbe

readinessProbe 检查结果的成功与否，决定的这个 Pod 是不是能被通过 Service 的方式访问到，而并不影响 Pod 的生命周期

### 1.5 PodPreset

Pod 预设, 为符合预设选择器的pod 设置默认值:

```
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:               # 只影响符合选择器的pod
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: nginx
      ports:
        - containerPort: 80
```

PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义

---

## 2. Secret and ConfigMap

### 2.1 Secret

类型:

* Opaque 默认值, 用户自定义secret数据
* kubernetes.io/service-account-token: Secret.Data  属性是一个键值对map, 定义多组数据, 映射为多个文件, 典型的三个文件: `ca.crt namespace  token`
* kubernetes.io/dockerconfigjson: Secret.Data[".dockerconfigjson"] - a serialized ~/.docker/config.json file
* kubernetes.io/dockercfg: Secret.Data[".dockercfg"] - a serialized ~/.dockercfg file
* kubernetes.io/basic-auth: Secret.Data["username"] 和Secret.Data["password"]
* kubernetes.io/ssh-auth:  Secret.Data["ssh-privatekey"]
* kubernetes.io/tls: Secret.Data["tls.key"] - TLS private key 和 Secret.Data["tls.crt"] - TLS certificate
* bootstrap.kubernetes.io/token

通过secret文件创建, 一个文件一个secret

```
$ cat ./username.txt
admin
$ cat ./password.txt
c1oudc0w!

$ kubectl create secret generic user --from-file=./username.txt
$ kubectl create secret generic pass --from-file=./password.txt
```

通过yaml创建:

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:                     # 可以一次创建多个secret
  user: YWRtaW4=          # 要求这些数据必须是经过 Base64 转码的，以免出现明文密码的安全隐患
  pass: MWYyZDFlMmU2N2Rm
```


Secret 状态表:

```
$ kubectl get secrets
NAME           TYPE                                DATA      AGE
user          Opaque                                1         51s
pass          Opaque                                1         51s
```

### 2.2 ConfigMap

不需要加密的、应用所需的配置信息, ConfigMap 的用法几乎与 Secret 完全相同

从文件创建:

`kubectl create configmap ui-config --from-file=example/ui.properties`


---

## 3. ReplicaSet

ReplicaSet 的 DESIRED、CURRENT 和 READY 字段的含义，和 Deployment 中是一致的。所以，相比之下，Deployment 只是在 ReplicaSet 的基础上，添加了 UP-TO-DATE 这个跟版本有关的状态字段


TODO: 修改rs, 对pod, deployment的影响??

---


## 4. Deployment

Deployment 控制器实际操纵的，正是这样的 ReplicaSet 对象，而不是 Pod 对象

```
Pod.ownerReferences -> ReplicaSet
                       ReplicaSet.ownerReferences -> Deployment
```

状态表:

```
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```

* DESIRED: 用户期望的 Pod 副本个数（spec.replicas 的值）
* CURRENT: 当前处于 Running 状态的 Pod 的个数 (但版本不一定是最新的)
* UP-TO-DATE：处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致(但不一定是Running)
* AVAILABLE: 既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数 (CURRENT && UP-TO-DATE)

### 4.1 滚动更新:

将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程，就是“滚动更新”

查看滚动更新过程:

```
$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
```

Deployment 自动创建的RS:

```
$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-3167673210   3         3         3       20s
```

随机字符串叫作 pod-template-hash. ReplicaSet 会把这个随机字符串加在它所控制的所有 Pod 的标签里，从而保证这些 Pod 不会与集群里的其他 Pod 混淆


查看滚动更新历史:
```
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
```

滚回到上一版本:
```
$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment
```

查看指定版本对象细节:
```
$ kubectl rollout history deployment/nginx-deployment --revision=2
```

回滚到指定版本:
```
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.extensions/nginx-deployment
```

版本保留限制: `deployment.spec.revisionHistoryLimit`

### 4.2 滚动更新策略

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```

* maxSurge 指定的是除了 DESIRED 数量之外，在一次“滚动”中，Deployment 控制器还可以创建多少个新 Pod
* maxUnavailable 指的是，在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod
* 两个配置还可以用百分比形式来表示

### 4.3 暂停更新

对 Deployment 进行的每一次更新操作，都会生成一个新的 ReplicaSet 对象，对某些场景有些多余, 用户可能会对deployment多次修改, 但是只希望更新一次.

暂停更新:
```
$ kubectl rollout pause deployment/nginx-deployment
deployment.extensions/nginx-deployment paused
```
对 Deployment 的所有修改，都不会触发新的“滚动更新”，也不会创建新的 ReplicaSet

恢复更新:

```
$ kubectl rollout resume deploy/nginx-deployment
deployment.extensions/nginx-deployment resumed
```

---

## 5. Statefulset

重要属性:

* `spec.serviceName`: 用这个service的名字组装pod的网络标识(通常对应一个headless service, 但是service需要用户自己去创建)

* `spec.VolumeClaimTemplates` 是一个PVC的定义都设计在模板(数组), 这样就不需要单独提前去定义PVC. pvc name也是自动生成:

  StatefulSet.name + volumeClaimTemplate.name + 索引


StatefulSet 如何管理Pod:

* 拓扑状态维护:

  * Pod 命名: StatefulSet.name + 从0递增的整数
  * Pod 启动: 严格按照编号顺序进行, 上一个Ready之前, 下一个会一直处于Pending
  * 固定Pod的网络标识: 每个Pod都会设置DNS: `<StatefulSet 名字 >-< 编号 >`, 解析到该pod的ip, Pod 重建后, 网络标识不会变, 虽然pod ip 很可能变了

* 存储状态
  * 按照`spec.VolumeClaimTemplates`定义, 创建PVC, 会被分配一个与这个 Pod 完全一致的编号: `<PVC 名字 >-<StatefulSet 名字 >-< 编号 >`
  * Pod 重建后, 同样会绑定到之前的PVC和PV, Pod 删除不会删除PVC

---

## 6. DaemonSet

场景: node上的网络插件, 监控插件, 日志插件, 存储插件等

* 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上
* 每个节点上只有一个这样的 Pod 实例
* 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉
* 调谐过程通过DaemonSet Controller 来控制
* 通过pod的属性nodeAffinity来控制pod在node上的分布: DaemonSet 会自动给这个 Pod 加上一个 nodeAffinity，从而保证这个 Pod 只会在指定节点上启动

### 6.1 Toleration:

DaemonSet 会给这个 Pod 自动加上另外一个与调度相关的字段，叫作 tolerations。这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）

```
Pod ......
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```

在正常情况下，被标记了 unschedulable“污点”的 Node，是不会有任何 Pod 被调度上去的（effect: NoSchedule）。可是，DaemonSet 自动地给被管理的 Pod 加上了这个特殊的 Toleration，就使得这些 Pod 可以忽略这个限制，继而保证每个节点上都会被调度一个 Pod。当然，如果这个节点有故障的话，这个 Pod 可能会启动失败，而 DaemonSet 则会始终尝试下去，直到 Pod 启动成功

在 Kubernetes 项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为node.kubernetes.io/network-unavailable的“污点”。

而通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来


Master 污点:

```
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule

```

默认情况下，Kubernetes 集群不允许用户在 Master 节点部署 Pod。因为，Master 节点默认携带了一个叫作node-role.kubernetes.io/master的“污点”。所以，为了能在 Master 节点上部署 DaemonSet 的 Pod，我就必须让这个 Pod“容忍”这个“污点”

### 6.2 rollout

DaemonSet 同样支持rollout:

```
$ kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   2         2         2         2            2           <none>          1h
```

```
$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
```

升级(–record 会记录操作到历史中):

```
$ kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n=kube-system
```

查看滚动更新过程:

```
$ kubectl rollout status ds/fluentd-elasticsearch -n kube-system
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 1 of 2 updated pods are available...
daemon set "fluentd-elasticsearch" successfully rolled out
```

查看更新历史:

```
$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets "fluentd-elasticsearch"
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --namespace=kube-system --record=true
```

### 6.3 ControllerRevision

资源对象, 用于记录(DaemonSet等对象)历史版本, ControllerRevision 其实是一个通用的版本管理对象 

ControllerRevision 对象，实际上是在 Data 字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 Annotation 字段保存了创建这个对象所使用的 kubectl 命令

```
$ kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-64dc6799c9   daemonset.apps/fluentd-elasticsearch   2          1h
```

```
$ kubectl describe controllerrevision fluentd-elasticsearch-64dc6799c9 -n kube-system
Name:         fluentd-elasticsearch-64dc6799c9
Namespace:    kube-system
Labels:       controller-revision-hash=2087235575
              name=fluentd-elasticsearch
Annotations:  deprecated.daemonset.template.generation=2
              kubernetes.io/change-cause=kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
API Version:  apps/v1
Data:
  Spec:
    Template:
      $ Patch:  replace
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Name:  fluentd-elasticsearch
      Spec:
        Containers:
          Image:              k8s.gcr.io/fluentd-elasticsearch:v2.2.0
          Image Pull Policy:  IfNotPresent
          Name:               fluentd-elasticsearch
...
Revision:                  2
Events:                    <none>
```


回滚:

```
$ kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-system
daemonset.extensions/fluentd-elasticsearch rolled back
```

实际上相当于读取到了 Revision=1 的 ControllerRevision 对象保存的 Data 字段。而这个 Data 字段里保存的信息，就是 Revision=1 时这个 DaemonSet 的完整 API 对象

在执行完这次回滚完成后，你会发现，DaemonSet 的 Revision 并不会从 Revision=2 退回到 1，而是会增加成 Revision=3。这是因为，一个新的 ControllerRevision 被创建了出来

---

## 7. 离线业务

### 7.1 Job

Job Controller 控制的对象，直接就是 Pod, (spec.template)

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2
  completions: 4
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```


Job 对 Pod 的控制:

* Pod template, 被自动加上了一个 `controller-uid=< 一个随机字符串 >`这样的 Label
* Job.Selector, 同样设置了上述label用于定位pod

Job 对Pod的调谐过程:

* 需要创建的 Pod 数目 = 最终需要的 Pod 数目 - 实际在 Running 状态 Pod 数目 - 已经成功退出的 Pod 数目
* 单时创建的 Pod 数目受到并行度(spec.parallelism)的限制

#### 7.1.1 失败控制: restartPolicy

在 Job 对象里只允许被设置为 Never 或者 OnFailure

* `restartPolicy=Never`: job失败会导致不断地尝试创建一个新 Pod, 重试上限是 Job 对象的 spec.backoffLimit(默认值是 6)

  新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10 s、20 s、40 s …后

* restartPolicy=OnFailure: Job 失败不会重新创建pod, 但会不断地尝试重启 Pod 里的容器

#### 7.1.2 失败控制: spec.activeDeadlineSeconds

设置最长运行时间, 超时后对应 Pod 都会被终止。并且可以在 Pod 的状态里看到终止的原因是 reason: DeadlineExceeded

#### 7.1.3 并行控制

* spec.parallelism: 它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行

* spec.completions: 它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数

Job 状态表:

```
$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
pi        4         0            3s
```

* DESIRED: 对应 spec.completions
* SUCCESSFUL: 成功完成的pod数量

### 7.2 CronJob

CronJob 是一个 Job 对象的控制器（定义中的jobTemplate）

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

schedule: 标准的Unix Cron格式的表达式

CronJob 状态表:

```
$ kubectl get cronjob hello
NAME      SCHEDULE      SUSPEND   ACTIVE    LAST-SCHEDULE
hello     */1 * * * *   False     0         Thu, 6 Sep 2018 14:34:00 -070
```

#### 7.2.1 并行控制:

某个 Job 还没有执行完，按照调度新 Job 将被创建时, 是否允许并行由`concurrencyPolicy` 来控制:

* concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在
* concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过
* concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job

#### 7.2.2 失败控制

如果某一次 Job 创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss 的数目达到失败上限100时，那么 CronJob 会停止再创建这个 Job

时间窗口: spec.startingDeadlineSeconds 字段指定

---

## 8. 对象从属关系

子对象通过属性`metadata.ownerReferences` 指向父对象

### Garbage Collection

---
