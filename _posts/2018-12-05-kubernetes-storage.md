---
layout: post
tags : [container, kubernetes, docker]
title: Kubernetes Storage

---

## 1. Kubernetes Volume

Volume 并不是K8s 中独立的资源对象, 是定义在Pod上的属性, 然后被Pod的中各容器独立使用. volume 的具体类型实现可能是对象, 如persistentVolumeClaim, Secret等

volume类型:

* emptydir
* hostpath
* projected
* Secret
* nfs
* local
* cephfs
* csi
* gitRepo
* persistentVolumeClaim
......


```
% kubectl describe pod XXXX

Containers:
  con1:
    Image:         ccr.ccs.tencentyun.com/ciserver/cibuilder:v1.0.5
    Mounts:
      /var/run/docker.sock from dockersock-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-qgsb7 (ro)
Volumes:                                            # volume 是定义到Pod上
  dockersock-volume:
    Type:  HostPath (bare host directory volume)
    Path:  /var/run/docker.sock
  default-token-qgsb7:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-qgsb7
    Optional:    false
```

* Pod 中的Volume和 Pod 中的 Container 关系是多对多

* Kubernetes Volume 属于Pod, COntainer 使用 所属Pod的Volume: Volume 与 Pod 的生命周期相同，但与容器的生命周期不相关

* 当 Pod 被删除时，Volume 才会被清理。并且数据是否丢失取决于 Volume 的具体类型:

  * 数据清除类型: emptyDir, secret,  gitRepo ......
  * 数据持久化类型: hostpath, PV ......

---

## 2. 非持久化存储方式

### 2.1 emptyDir

* 主要用于多个容器之间共享数据等
* Kubernetes 会在 Node 上自动分配一个目录，因此无需指定 Node 宿主机上对应的目录文件。这个目录的初始内容为空
* 当 Pod 从 Node 上移除（Pod 被删除或者 Pod 发生迁移）时，emptyDir 中的数据会被永久删除
* 缺省情况下，emptryDir 是使用主机磁盘进行存储的。你也可以使用其它介质作为存储，比如：网络存储、内存等。设置 emptyDir.medium 字段的值为 Memory 就可以使用内存进行存储

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### 2.2 hostPath

主要场景:

* 运行中的容器需要访问 Docker 内部的容器，使用 /var/lib/docker 来做为 hostPath 让容器内应用可以直接访问 Docker 的文件系统
* 和 DaemonSet 搭配使用，用来操作主机文件

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```


---

## 3. 持久化存储方式

> 而所谓的“持久化 Volume”，指的就是这个宿主机上的目录，具备“持久性”。即：这个目录里面的内容，既不会因为容器的删除而被清理掉，也不会跟当前的宿主机绑定。这样，当容器被重启或者在其他节点上重建出来之后，它仍然能够通过挂载这个 Volume，访问到这些内容

### 3.1 PV

PersistentVolume（持久化卷）, 是对底层的共享存储的一种抽象

* 通常由集群管理员进行创建和配置, PV 也是集群资源的一种

* PersistentVolume 通过插件机制实现与共享存储的对接, 支持多种插件如: NFS, GCEPersistentDisk, AWSElasticBlockStore, AzureFile, CephFS 等等

* PV 的生命周期独立于 Pod，当使用它的 Pod 销毁时对 PV 没有影响

* PV属性:

  * 存储类型
  * 存储大小 Capacity
  * 访问模式 AccessModes
  * 回收策略 PersistentVolumeReclaimPolicy

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name:  pv1-nfs
spec:
  capacity:                             # 通用属性, 存储大小
    storage: 1Gi
  accessModes:                          # 通过属性, 访问模式
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:                                   # 对应着不同的PV插件
    path: /data/kubernetes
    server: 192.168.100.213
```

访问模式:

* ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载。
* ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载。
* ReadWriteMany（RWX）：读写权限，可以被多个节点挂载

一些 PV 可能支持多种访问模式，但是在挂载的时候只能使用一种访问模式，多种访问模式是不会生效的

三种访问模式在各个PV插件中并非都支持

回收策略: 用于定义当pvc 释放pv后, 对应的pv和数据如何处理

* Retain（保留）- 保留数据，需要管理员手工清理数据。PV 对象仍然存在, 数据仍然保留, pv 是否可重新claim???
* Delete（删除）- 删除PV对象, 同时删除关联的外部存储资源
* Recycle（回收）deprecated - 清除 PV 中的数据，效果相当于执行 rm -rf /thevoluem/* 。PV 可以重新被Claim

PV 状态:

* Available（可用）：表示可用状态，还未被任何 PVC 绑定。
* Bound（已绑定）：表示 PV 已经被 PVC 绑定。
* Released（已释放）：PVC 被删除，但是资源还未被集群reclaimed
* Failed（失败）： 表示该 PV 的自动回收失败

```
$ kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv1-nfs   1Gi        RWO            Recycle          Available                                      46s
```

### 3.2 PVC

PersistentVolumeClaim（持久化卷声明）,  是用户对存储资源的一种请求

* PV 和 PVC 的绑定是系统自动完成的，不需要显示指定要绑定的 PV。系统会根据 PVC 中定义的要求去查找处于 Available 状态的 PV
* 如果 PVC 申请的容量大小小于 PV 提供的大小，PV 同样会分配该 PV 所有容量给 PVC，如果 PVC 申请的容量大小大于 PV 提供的大小，此次申请就会绑定失败(PVC pending)
* PV 和 PVC 一对一, PVC 和 Pod 多对对, 一个PVC 可以被多个pod使用, pvc 是独立声明的对象, pod定义只是声明要使用某些pvc
* `StatefulSet.Pod.spec.VolumeClaimTemplates` 是一个PVC的定义都设计在模板(数组), 这样就不需要单独提前去定义PVC.

PV 和PVC绑定的条件:

* PV 和 PVC 的 spec 字段, 满足资源要求
* PV 和 PVC 的 storageClassName 字段必须一样

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc1-nfs
spec:
  accessModes:      # 这些需求要能和找到的PV匹配上, 才会配对
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

pvc 状态:

```
$ kubectl get pvc
NAME       STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc1-nfs   Bound     pv1-nfs   1Gi        RWO                           34s
```

pod 使用pvc:

```
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
    imagePullPolicy: IfNotPresent
    ports:
    - containerPort: 80
      name: web
    volumeMounts:
    - name: www                         # 挂Volume方式不变
      mountPath: /usr/share/nginx/html
  volumes:
  - name: www
    persistentVolumeClaim:              # PVC 作为一种Volume类型
      claimName: pvc2-nfs
```

使用 subPath 对同一个 PV 进行隔离:

```
volumeMounts:
- name: www
  subPath: nginx-pvc-test # 这会在PV的目录下创建这个子目录, 挂载到容器的mountPath下
  mountPath: /usr/share/nginx/html
```

PVC 中的数据持久化:

* 先删除PVC的使用对象(pod), 对PVC 没有影响, 数据仍然保留
* 直接删除PVC对象, PVC会进入terminating状态, 如果仍有对象在使用pvc, 数据仍然可用, 直到使用对象结束, 此时pvc才会删除.

### 3.3 StorageClass

* 资源对象, StorageClass 对象的作用，其实就是创建 PV 的模板

* 当用户通过 PVC 对存储资源进行申请时，StorageClass 会使用 Provisioner（不同 Volume 对应不同的 Provisioner）来自动创建用户所需 PV。这样应用就可以随时申请到合适的存储资源，而不用担心集群管理员没有事先分配好需要的 PV。


流程:

1. 创建PV自动配置程序: Provisioner

   通常用一个pod实现

2. 管理员创建StorageClass

   ```
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
   name: course-nfs-storage
   provisioner: fuseim.pri/ifs # 用于找到Provisioner
   ```

   ```
   $ kubectl get storageclass
   NAME                 PROVISIONER      AGE
   course-nfs-storage   fuseim.pri/ifs   1m
   ```



3. 用户通过PVC 申请 PV

   PVC 如何能找到对应的 StorageClass

   方法一:

   ```
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
   name: test-pvc
   annotations:
   volume.beta.kubernetes.io/storage-class: "course-nfs-storage" # 和StorageClass 名字一致
   spec:
   accessModes:
   - ReadWriteMany
   resources:
   requests:
   storage: 100Mi
   ```

   方法二:

   把名为 course-nfs-storage 的 StorageClass 设置为 Kubernetes 的默认后端存储

4. StorageClass 自动创建PV

5. 创建的PV和PVC绑定


TODO

* 自动创建的 PV 以 ${namespace}-${pvcName}-${pvName} 这样的命名格式创建在后端存储服务器上的共享数据目录中
* 自动创建的 PV 被回收后会以 archieved-${namespace}-${pvcName}-${pvName} 这样的命名格式存在后端存储服务器上


### 3.4 Kubernetes 访问存储资源的方式

* 直接访问

  移植性比较差，可扩展能力差。把 Volume 的基本信息完全暴露给用户，有安全隐患

* 静态PV

  * 集群管理员提前手动创建一些 PV。它们带有可供集群用户使用的实际存储的细节，之后便可用于 PVC 消费
  * 请求的 PVC 必须要与管理员创建的 PV 保持一致，如：存储大小和访问模式，否则不能将 PVC 绑定到 PV 上

* 动态PV

  当集群管理员创建的静态 PV 都不匹配用户的 PVC 时，PVC 请求存储类 StorageClass，StorageClass 动态的为 PVC 创建所需的 PV

---

## 4. PV 持久化过程

1. 当一个 Pod 调度到一个节点上之后，kubelet 就要负责为这个 Pod 创建它的 Volume 目录。默认情况下，kubelet 为 Volume 创建的目录是如下所示的一个宿主机上的路径

   `/var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 > `

2. Attach: 将远程存储挂载到node上, kubelet 具体的操作取决于Volume类型

   如google 磁盘服务: `$ gcloud compute instances attach-disk < 虚拟机名字 > --disk < 远程磁盘名字 >`

   对于远程存储, 如nfs, kubelet 可以跳过“第一阶段”（Attach）的操作，这是因为一般来说，远程文件存储并没有一个“存储设备”需要挂载在宿主机上

3. Mount: 格式化上阶段挂载的存储, 并将它挂载到宿主机指定的挂载点上, 也就是第一步中pod 的volume所在的位置

   ```
   # 通过 lsblk 命令获取磁盘设备 ID
   $ sudo lsblk
   # 格式化成 ext4 格式
   $ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/< 磁盘设备 ID>
   ???? 感觉缺一步
   # 挂载到挂载点
   $ sudo mkdir -p /var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >
   ```

4. kubelet 把这个 Volume 目录通过 CRI 里的 Mounts 参数，传递给 Docker，然后就可以为 Pod 里的容器挂载这个“持久化”的 Volume 了

   `$ docker run -v /var/lib/kubelet/pods/<Pod 的 ID>/volumes/kubernetes.io~<Volume 类型 >/<Volume 名字 >:/< 容器内的目标目录 > 我的镜像 ...  `


Attach 和 Mount 控制器:

* PV 的“两阶段处理”流程，是靠独立于 kubelet 主控制循环（Kubelet Sync Loop）之外的两个控制循环来实现的
* 第一阶段控制器: AttachDetachController, 不断地检查每一个 Pod 对应的 PV，和这个 Pod 所在宿主机之间挂载情况。从而决定，是否需要对这个 PV 进行 Attach（或者 Dettach）操作, Volume Controller 是 kube-controller-manager 的一部分。所以，AttachDetachController 也一定是运行在 Master 节点上的。当然，Attach 操作只需要调用公有云或者具体存储项目的 API，并不需要在具体的宿主机上执行操作
* 第二阶段控制器: VolumeManagerReconciler, 发生在 Pod 对应的宿主机上，所以它必须是 kubelet 组件的一部分, 是一个独立于 kubelet 主循环的 Goroutine, 避免了这些耗时的远程挂载操作拖慢 kubelet 的主控制循环，进而导致 Pod 的创建效率大幅下降的问题。实际上，kubelet 的一个主要设计原则，就是它的主控制循环绝对不可以被 block

---

### 参考资料:

* [浅谈 Kubernetes 数据持久化方案](https://mp.weixin.qq.com/s?__biz=MzI3MTI2NzkxMA==&mid=2247486206&idx=1&sn=99948873eeb1908d0cfa8a8bda1fb019&chksm=eac52bd7ddb2a2c1d632f06810bb92ae9fe3e5bd295623aed5ab0250c892dc098b77db35b3fa&token=1349413990&lang=zh_CN#rd)
