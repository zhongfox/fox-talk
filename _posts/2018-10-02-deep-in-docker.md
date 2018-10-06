---
layout: post
tags : [docker, linux]
title: 深入理解Docker

---

## 1. Namespace 资源隔离

* PID: 进程编号

  PID namespace是一个树状结构, 顶层PID namespace叫做root namespace

  父namespace可以看到子namespace, 并可以通过信号等方式影响子namespace中进程, 反过来却不行

  父节点中的进程有权终止子节点的进程

* Network: 网络设备, 网络栈, 端口, 路由表, 防火墙, /proc/net, /sys/class/netsocket 等

  一个物理网络设备最多存在于一个network ns, 可以通过创建veth pair(虚拟网络设备对)实现在不同ns间通信

* Mount: 挂载点(文件系统)

  通过`/proc/{PID}/mounts` 查看所有挂载到当前namespace的文件系统

  进程在创建mount namespace时, 会把当前的文件结构复制给新的namespace, 默认情况下, 新的namespace中所有的mount操作只影响自己namespace的文件系统(私有挂载)

* User: 用户ID, 用户组ID, root目录, key以及特殊权限

  User namespace 是一个层层嵌套的树状结构, 子 user namespace 有一个父节点namespace

  子user namespace创建后, 第一个进程被赋予该namepace的全部权限, 但在创建它的父namespace中没有任何权限

  接下来还需要进行用户绑定操作: 通过在`/proc{pid}/uid_map` 和`/proc/{pid}/gid_map`中写入绑定信息: 这2个文件只允许拥有该namespac`CAP_SETUID`权限的进程写入一次, 不允许修改

* UTS: 主机名, 域名

* IPC: 信号量, 消息队列, 共享内存

  申请IPC资源就申请了一个全局唯一的32位ID, IPC namespace包括: 系统IPC标识符以及实现POSIX消息队列的文件系统

#### 在文件系统上查看进程所属namespace


```
root@VM-100-76-ubuntu:~# ls -al /proc/6328/ns
total 0
dr-x--x--x 2 root root 0 Oct  2 13:16 .
dr-xr-xr-x 9 root root 0 Oct  2 13:16 ..
lrwxrwxrwx 1 root root 0 Oct  2 13:17 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Oct  2 13:17 ipc -> ipc:[4026532472]
lrwxrwxrwx 1 root root 0 Oct  2 13:17 mnt -> mnt:[4026532470]
lrwxrwxrwx 1 root root 0 Oct  2 13:16 net -> net:[4026532475]
lrwxrwxrwx 1 root root 0 Oct  2 13:17 pid -> pid:[4026532473]
lrwxrwxrwx 1 root root 0 Oct  2 13:17 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Oct  2 13:17 uts -> uts:[4026532471]
```

只要打开的文件描述符(fd)一直存在, 那么就算该ns下的所有进程已经结束, 这个namespace也会一直存在, 后续进程也可以加入(TODO 何时删除的?)

在docker中, 通过文件描述符定位和加入一个存在的namespace 是最基本的方式


#### namespace 操作API

* clone 同时创建Namespace
* setns 加入一个已存在的namespace: 通常为了不影响进程调用者, 会在setns执行后使用clone创建子进程, 然后让原来进程结束, 比如`docker exec`
* unshare: 在原进程上进行namespace隔离, 不启动新进程, 加入新创建namepsace

---

## 2. Cgroups

作用:

* 对任务进行资源总额进行限制
* 优先级分配
* 资源统计
* 任务控制: 对任务挂起, 恢复等

特点:

* cgroups是内核附加在程序上的一系列hook, 通过程序运行时对资源的调度触发相应的hook实现资源的追踪和限制
* cgroups API 以一个伪文件系统的方式实现, 用户态可以通过文件操作实现cgroups的组织管理
* 操作单元可以细粒度到线程级别


Docker 相关Cgroup文件:

#### CPU

`/sys/fs/cgroup/cpu/docker/<容器的完整ID>/`

份额控制:

* `/cpu.shares`:  参数`--cpu-shares 100`

  指定容器所使用的CPU份额值, 是一个弹性的加权值

> 默认情况下，每个docker容器的cpu份额都是1024。单独一个容器的份额是没有意义的，只有在同时运行多个容器时，容器的cpu加权的效果才能体现出来。例如，两个容器A、B的cpu份额分别为1000和500，在cpu进行时间片分配的时候，容器A比容器B多一倍的机会获得CPU的时间片，但分配的结果取决于当时主机和其他容器的运行状态，实际上也无法保证容器A一定能获得CPU时间片。比如容器A的进程一直是空闲的，那么容器B是可以获取比容器A更多的CPU时间片的。极端情况下，比如说主机上只运行了一个容器，即使它的cpu份额只有50，它也可以独占整个主机的cpu资源

周期控制:

* `/cpu.cfs_period_us`:  参数`--cpu-period`

  指定容器对CPU的使用要在多长时间内做一次重新分配, 单位为微秒（μs）最小值为1000微秒，最大值为1秒（10^6 μs），默认值为0.1秒（100000 μs）

* `/cpu.cfs_quota_us`: 参数`--cpu-quota`, 单位为微秒（μs）, 值默认为-1，表示不做控制

  指定在这个(cpu-period)周期内，最多可以有多少时间用来跑这个容器


> 如果容器进程需要每1秒使用单个CPU的0.2秒时间，可以将cpu-period设置为1000000（即1秒），cpu-quota设置为200000（0.2秒）。当然，在多核情况下，如果允许容器进程需要完全占用两个CPU，则可以将cpu-period设置为100000（即0.1秒），cpu-quota设置为200000（0.2秒）

core控制:


* `/cpuset.cpus`: 参数 `--cpuset-cpus`

  `--cpuset-cpus 0-2 ubuntu，表示创建的容器只能用0、1、2这三个内核`

#### Memery

`/sys/fs/cgroup/memory/docker/<容器的完整ID>/`

* `/memory.limit_in_bytes` 内存配额, 参数`—memory 128m`

  除了–memory指定的内存大小以外，docker还为容器分配了同样大小的swap分区

* `/memory.memsw.limit_in_bytes`, swap分区大小, 参数`--memory-swap`

#### 磁盘IO配额

TODO

实例:

`docker run -d --cpu-period 100000 --cpu-quota 200000 --memory 256m --memory-swap 512m nginx`


```
# cat /sys/fs/cgroup/memory/docker/ba24b72e664d3197da87afb8db4cb6df5e30f5f615fcfe77306278e7ec82bc6f/memory.limit_in_bytes
268435456
# cat /sys/fs/cgroup/memory/docker/ba24b72e664d3197da87afb8db4cb6df5e30f5f615fcfe77306278e7ec82bc6f/tasks
17426
17486
# cat /sys/fs/cgroup/memory/docker/ba24b72e664d3197da87afb8db4cb6df5e30f5f615fcfe77306278e7ec82bc6f/memory.limit_in_bytes
268435456
# cat /sys/fs/cgroup/cpu/docker/ba24b72e664d3197da87afb8db4cb6df5e30f5f615fcfe77306278e7ec82bc6f/tasks
17426
17486
# cat /sys/fs/cgroup/cpu/docker/ba24b72e664d3197da87afb8db4cb6df5e30f5f615fcfe77306278e7ec82bc6f/cpu.cfs_period_us
100000
# cat /sys/fs/cgroup/cpu/docker/ba24b72e664d3197da87afb8db4cb6df5e30f5f615fcfe77306278e7ec82bc6f/cpu.cfs_quota_us
200000
```


参考: [docker容器资源配额控制](https://blog.csdn.net/horsefoot/article/details/51731543)

---

## 3. Docker 架构

### Docker daemon

后台启动一个API Server, 负责响应来自Docker Client的请求, 根据不同的请求分发到不同的模块, 翻译成系统调用完成容器管理操作

daemon 模块:

* 镜像管理模块:  下载镜像, 管理本地image, reference, layer等
  * distribution: 负责与Docker registry交互, 上传和下载镜像以及相关元数据
  * registry: 负责与Docker registry有关的身份验证, 镜像查找验证等
  * image:
  * reference:
  * layer:

* graphdriver: 镜像存储驱动, 将镜像文件存储到文件系统, 是所有与容器镜像相关操作的最终执行者. 目前docker 支持的graphdriver有: aufs, overlay, devicemapper, zfs, vfs, btrfs

* volumedriver: 管理数据卷, docker volumedriver默认实现是local, 默认将文件存储在docker根目录下volume目录下

* execdriver: 调用libcontainer 实现docker容器运行资源限制和用户指令, 是对linux操作系统的namespace cgroups等的二次封装

* network 模块: 调用libnetwork库, 创建并配置容器网络环境

独立模块:

* libcontainer
* libnetwork: 抽象出一个容器网络模型(CNM), 给调用者提供一个统一抽象接口

---

## 4. 镜像本地存储

* Docker镜像: 是一个只读的Docker容器模块(rootfs)
* Docker容器: 多个只读rootfs + init层 + 可读写的文件系统层,  进行联合挂载(Union Mount)

docker 镜像的特点:

* 分层: 实现共享
* 写时复制
* 内容寻址: 镜像层通过内容计算hash值
* 联合挂载: 实现分层合并


本地标识:

* layer.DiffID: DiffID = SHA256hex(uncompressed layer data)

* layer.ChainID:

  * For bottom layer: ChainID(layer0) = DiffID(layer0)
  * For other layers: ChainID(layerN) = SHA256hex(ChainID(layerN-1) + " " + DiffID(layerN))

* image.ID: SHA256hex(imageConfigJSON)

* layer.cacheID: 镜像层在存储驱动中的新标识ID, 随机生成

* mountID: 容器层在存储驱动总的新标识ID, 随机生成

### repository 元数据

`/var/lib/docker/image/{graphdriver}/repositories.json`

包括 tag名称和对应的IMAGE ID

```
{
	"Repositories": {
		"ccr.ccs.tencentyun.com/library/qcloud_ingress_controller": {
			"ccr.ccs.tencentyun.com/library/qcloud_ingress_controller:0.0.9": "sha256:ec63c169b87e4da22bfcb270bd71c1ee79beda4f81a58c8d30357d6149668722",
			"ccr.ccs.tencentyun.com/library/qcloud_ingress_controller@sha256:39585daede57590493f40dfe167fc7d9bb62c8087d20f9dc55fcb48816859bab": "sha256:ec63c169b87e4da22bfcb270bd71c1ee79beda4f81a58c8d30357d6149668722"
		},
		"nginx": {
			"nginx:latest": "sha256:bc26f1ed35cf942745343c09475050f0e4529f785efcfcba784d46296d5842c8", # IMAGE ID
			"nginx@sha256:e8ab8d42e0c34c104ac60b43ba60b19af08e19a0e6d50396bdfd4cef0347ba83": "sha256:bc26f1ed35cf942745343c09475050f0e4529f785efcfcba784d46296d5842c8"
		}
	}
}
```

### image 元数据

`/var/lib/docker# cat image/{graphdriver}/imagedb/content/sha256/{IMAGE ID}`

包括镜像架构, 操作系统, 镜像默认配置, 镜像历史, rootfs等信息

```
{
	"architecture": "amd64",
	"config": {
		"Hostname": "",
		"Domainname": "",
		"User": "",
		"AttachStdin": false,
		"AttachStdout": false,
		"AttachStderr": false,
		"ExposedPorts": {
			"80/tcp": {}
		},
		"Tty": false,
		"OpenStdin": false,
		"StdinOnce": false,
		"Env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "NGINX_VERSION=1.15.4-1~stretch", "NJS_VERSION=1.15.4.0.2.4-1~stretch"],
		"Cmd": ["nginx", "-g", "daemon off;"],
		"ArgsEscaped": true,
		"Image": "sha256:192fc9e2123e57b2e5dd41ff35f4c170231a14778ebbe4072f113c2c6f39dbf1",
		"Volumes": null,
		"WorkingDir": "",
		"Entrypoint": null,
		"OnBuild": [],
		"Labels": {
			"maintainer": "NGINX Docker Maintainers <docker-maint@nginx.com>"
		},
		"StopSignal": "SIGTERM"
	},
	"container": "41426166a96ac07c5f86a20c71c02427d884c2c3acc99a55d427ddf12f9db960",
	"container_config": {
		"Hostname": "41426166a96a",
		"Domainname": "",
		"User": "",
		"AttachStdin": false,
		"AttachStdout": false,
		"AttachStderr": false,
		"ExposedPorts": {
			"80/tcp": {}
		},
		"Tty": false,
		"OpenStdin": false,
		"StdinOnce": false,
		"Env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", "NGINX_VERSION=1.15.4-1~stretch", "NJS_VERSION=1.15.4.0.2.4-1~stretch"],
		"Cmd": ["/bin/sh", "-c", "#(nop) ", "CMD [\"nginx\" \"-g\" \"daemon off;\"]"],
		"ArgsEscaped": true,
		"Image": "sha256:192fc9e2123e57b2e5dd41ff35f4c170231a14778ebbe4072f113c2c6f39dbf1",
		"Volumes": null,
		"WorkingDir": "",
		"Entrypoint": null,
		"OnBuild": [],
		"Labels": {
			"maintainer": "NGINX Docker Maintainers <docker-maint@nginx.com>"
		},
		"StopSignal": "SIGTERM"
	},
	"created": "2018-09-25T17:21:29.327116583Z",
	"docker_version": "17.06.2-ce",
	"history": [{
		"created": "2018-09-04T21:21:34.335927291Z",
		"created_by": "/bin/sh -c #(nop) ADD file:e6ca98733431f75e97eb09758ba64065d213d51bd2070a95cf15f2ff5adccfc4 in / "
	}, {
   ......
	}],
	"os": "linux",
	"rootfs": {
		"type": "layers",
		"diff_ids": ["sha256:8b15606a9e3e430cb7ba739fde2fbb3734a19f8a59a825ffa877f9be49059817", "sha256:3bbff39fa30bc3eb281b0b07c6eff6c2f4c1c5489eb972871ab58411793a776b", "sha256:e8916cb59586ee8657a86aac49f93680fa1d796530111dfe92056770d5243429"]
	}
}
```

### layer 元数据

`/var/lib/docker# cat image/{graphdriver}/layerdb/`

TODO

### 容器读写层层创建过程

1. 随机生成容器层的mountID, 持久化到`image/aufs/layerdb/mounts/{container_id}/mount_id` 中
2. 分别在mnt和diff目录中创建mountID同名的子文件夹
3. 在layers目录中创建mountID同名的文件, 记录该层所依赖的其他层(mountID ? cacheID?)
4. 将diff中属于该容器镜像的所有层目录以只读的方式挂载到mnt下
5. 在diff目录中生成一个`<mountID>init`命名的目录, 作为最后一层只读层, 并进行挂载
6. 以mountID为名称的可读写层, 挂载到mnt下, 该层实际内容保存到diff下

init 层的目的: int层中文件与容器环境相关, 但不适合作为镜像文件, 因为这和具体容器特有


### 目录结构

```
$ sudo tree . -L 3
.
├── aufs 镜像层数据存储
│   ├── diff  所有层的存储目录(新下载的镜像层逐层保存在这里), 都是目录, 包括只读层和读写层, mountID 为目录名
│   │   ├── 024f2af668a25bd0efcb3c1ae53c95f780fe7f84a5f7652fc3603aa1daa284d2/
│   │   ├── 166d40dc45b1ff354282429d32117c98b025d8713de7be5dab7150eb98d4ce74/
│   │   ├── 67a2f06df297d09211a3349245d987bb3020f5c08697b1021e9a825ea9f1e304-init/
│   │   ├── 68f9cf205ca7aed2b2b82f50ae4c321aaa0ddb517d0d642d171a23d8b99a5c76/
│   │   ├── 73fdbb31d7fca4c78d41836ecb48fd07ba8737b83115c4c32f95d6090097456b-init/
│   │   ├── 80ba70a43d4b02b02456907c686ff0add12fcb567d6fa4c5ad673e492a5975f8/
│   ├── layers 上述层之间的关系, 文件内容为祖先层的mountID
│   │   ├── 67a2f06df297d09211a3349245d987bb3020f5c08697b1021e9a825ea9f1e304
│   │   ├── 67a2f06df297d09211a3349245d987bb3020f5c08697b1021e9a825ea9f1e304-init
│   │   ├── 73fdbb31d7fca4c78d41836ecb48fd07ba8737b83115c4c32f95d6090097456b
│   │   ├── 73fdbb31d7fca4c78d41836ecb48fd07ba8737b83115c4c32f95d6090097456b-init
│   │   ├── 81463e36170507752ec2225985604f9c4d506791b8783c85c24a9668b018c00d
│   │   ├── a5c23ffc07999599f2f94109c618ea797b9d4b8f2af6807ee5406df4ebf72993-init
│   │   └── fa04c9d948b44505cd0bdc0eb9087f0ca4e91e8856d49e90d56428c4848899b0
│   └── mnt 挂载点, , 都是目录, mountID 为目录名, 如果容器中写了一个新文件, 会出现在这里
│       ├── 80ba70a43d4b02b02456907c686ff0add12fcb567d6fa4c5ad673e492a5975f8-init/
│       ├── a5c23ffc07999599f2f94109c618ea797b9d4b8f2af6807ee5406df4ebf72993/
│       ├── ae13e9bbfcf13653047c7e61a14bf5563b7cbb07f68c4dc23bfa5f663993abe2-init/
│       └── fa04c9d948b44505cd0bdc0eb9087f0ca4e91e8856d49e90d56428c4848899b0/
├── builder
├── containerd
│   └── daemon
├── containers
│   ├── 3231802a57b6af43c3bb54eb4a67d473afcb62705beb4c19a5fcf98302aa6809
│   │   ├── 3231802a57b6af43c3bb54eb4a67d473afcb62705beb4c19a5fcf98302aa6809-json.log
│   │   ├── checkpoints
│   │   ├── config.v2.json
│   │   ├── hostconfig.json
│   │   ├── hostname
│   │   ├── hosts
│   │   ├── resolv.conf
│   │   ├── resolv.conf.hash
│   │   └── shm
│   └── f8a122aedda035c4f100d02e66c3be9264f0284a63cee8f829b2026514506b35
├── image 元数据存储
│   └── aufs
│       ├── distribution 
          ├── diffid-by-digest 保存了digest->diffid的映射关系，即distribution hashes和Content hashes的映射关系。这可以方便检查layer是否在本地已经存在
          │   └── sha256
          │       ├── e872a2642ce1d63f3283e81bb6503579808c01e2bf63412ef87135ecb0f04746 文件名是distribution hashes, 内容是Content hashes TODO 验证
          │       └── ef25ef6e9e7be10e07503c12c0309580d77eaffe56ab4376bda13eea5f1a5493
          └── v2metadata-by-diffid 保存了diffid -> (digest,repository)的映射关系，这可以方便查找layer的digest及其所属的repository
              └── sha256
                  ├── 833649a3e04c96faf218d8082b3533fa0674664f4b361c93cad91cf97222b733 内容 [{"Digest":"sha256:c0a04912aa5afc0...","SourceRepository":"10.x.x.x:5000/busybox"}]
                  └── 8600ee70176b569d0776833f106d239d56043cb854a5edbb74aff6c5e8d4782d
        ├── imagedb 保存image的信息
        │   ├── content
        │   │   └── sha256 (镜像ID -> 镜像配置json) 内容是镜像元数据配置(包括diff_ids), 用于算出imageID
        │   │       ├── 2a4cca5ac898476c2c47a8d6a17102e00241d6fa377fbe4d50787fe3d7a8d4d6
        │   │       └── f2a91732366c0332ccd7afd2a5c4ff2b9af81f549370f7a19acd460f87686bc7
        │   └── metadata
        │       └── sha256
│       ├── layerdb 保存layer的信息
          ├── mount
          │   ├── 3231802a57b6af43c3bb54eb4a67d473afcb62705beb4c19a5fcf98302aa6809 容器ID
          │   │   ├── init-id   init层在graphdriver中的mountID
          │   │   ├── mount-id  读写层在graphdriver中的mountID
          │   │   └── parent    容器层的父层镜像的chainID
          │   ├── 8b083d2fe47013176290c201bdce02135616ef5bcce52c4984132b94c3bea646
          │   │   ├── init-id
          │   │   ├── mount-id
          │   │   └── parent
          ├── sha256 目录名称为layer的chainID
          │   ├── 01c7acde7bc52fac47440a98e8a92994d337d7e55037dd903c9cf56ad22e2f81
          │   │   ├── cache-id
          │   │   ├── diff 保存当前layer的diff ID
          │   │   ├── parent 保存上一层layer的chainID
          │   │   ├── size
          │   │   └── tar-split.json.gz
          │   └── 1599365ecbaf5b106d874fcbce0a70283e3ecff771f0f826c7695fd85c2f7eaa
          │       ├── cache-id
          │       ├── diff
          │       ├── parent
          │       ├── size
          │       └── tar-split.json.gz
          └── tmp
│       └── repositories.json 镜像元数据, 包括镜像名, tag和 ImageID的对应关系
├── network
├── plugins
├── runtimes
├── swarm
├── tmp
├── trust
└── volumes
    └── metadata.db
```

---

## 4. Volume


默认目录: `/var/lib/docker/volumes`

如果容器中目标目录中存在数据, 将会被隐藏掉

```
docker volume create --name fox0

docker run -d \
  -v fox0:/data0 \                  # 挂载已存在的volume "Mounts.Type": "volume"
  -v fox1:/data1 \                  # 创建新volume并挂载 "Mounts.Type": "volume"
  -v /tmp/fox2:/data2 \             # 使用host path 挂载 (没有创建volume) "Mounts.Type": "bind"
  -v /data3 \                       # 创建随机命名volume并挂载 "Mounts.Type": "volume"
  -v /etc/hosts:/test_hosts \       # 使用host path file(必须存在) 挂载文件 (没有创建volume) "Mounts.Type": "bind"
  nginx

docker volume ls
DRIVER              VOLUME NAME
local               aa1087f9c83e16591914025b41e4488ad9b957736c99939c37338871b9c1c3a3
local               fox0
local               fox1
```

### 在dockerfile中添加volume

`VOLUME ["/data1", "/data2"]`

VOLUME 指令不能挂载主机中的目录和文件, 这是为了保证dockerfile可移植性

在dockerfile中, VOLUME 指令后对卷内容的修改无效, 可以将修改操作放到VOLUME指令之前, 卷的内容可以保留

### Mount

Docker 使用的volume挂载方式是`bind mount`

关于mount:

`--bind` mount --bind命令和硬连接很像，硬链接连接到同一个inode上面，只不过hard link无法连接目录，而mount --bind命令可以, 区别:

* 目标目录内容是被隐藏, 而不是删除
* mount --bind连接的两个目录的inode号码并不一样，只是被挂载目录的block被屏蔽掉，inode被重定向到挂载目录的inode（被挂载目录的inode和block依然没变）
* 两个目录的对应关系存在于内存里，一旦重启挂载关系就不存在了

`--rbind` 参数: TOOD

---

## 5. Docker Network

libnetwork 使用了 CNM(container network model), CNM 定义了容器虚拟化网络的模型:

* sandbox: 一个沙盒包括一个容器网络栈信息, 可以对容器的接口, 路由, DNS等进行管理, 沙盒实现: 如 linux namespace
* endpoint: 一个端点可以加入一个沙盒和一个网络, 端点实现: 如 veth pair, Open vSwitch内部端口等, 一个端点只能属于一个网络和沙盒
* network: 一组可以直联互通的端点. 实现如linux bridge, Vlan等

libnetwork 内置5中驱动:

* bridge
* host: 不会为容器创建网络协议栈, 即不创建新的network namespace, 容器进程共享宿主机网络环境
* overlay: 使用VXLAN 方式, 需要配置额外存储, 如Consul, etcd或者zk等
* remote
* null: 容器拥有自己的network namespace, 但并不为容器进行任何网络配置


### Bridge 驱动实现


启动2个docker后的宿主机器

`sudo docker run -it ubuntu /bin/bash`
`sudo docker run -p 4000:3000 ccr.ccs.tencentyun.com/fox-test/node-koa-demo:tag4`

```
$ ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:96:ae:f2:53
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2464 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5171 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:158607 (158.6 KB)  TX bytes:27187058 (27.1 MB)

eth0      Link encap:Ethernet  HWaddr 52:54:00:e4:66:50
          inet addr:10.135.151.206  Bcast:10.135.191.255  Mask:255.255.192.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2334152 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1977869 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:842043414 (842.0 MB)  TX bytes:234051187 (234.0 MB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

veth4593bc2 Link encap:Ethernet  HWaddr c6:88:13:25:52:ef
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:28 errors:0 dropped:0 overruns:0 frame:0
          TX packets:17 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2625 (2.6 KB)  TX bytes:32428 (32.4 KB)

vethe4094bd Link encap:Ethernet  HWaddr ba:94:26:07:c3:1a
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:2399 errors:0 dropped:0 overruns:0 frame:0
          TX packets:5155 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:187115 (187.1 KB)  TX bytes:27123582 (27.1 MB)
```

宿主机新增虚拟网卡docker0, 并配置了静态路由, 指定目标网络是docker网段的流量走docker0

```
$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
172.17.0.0      *               255.255.0.0     U     0      0        0 docker0 这条让宿主机器可以访问容器内网络, 流量由docker0网桥转发
```


宿主机创建docker0网桥, 通过veth pair 将容器和docker0连接:

```
查看网桥
ubuntu@VM-151-206-ubuntu:~$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024296aef253       no              veth4593bc2
                                                        vethe4094bd
```

在容器内, docker0 的ip将作为容器路由默认网关:

```
root@dabf60697f38:/# route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      *               255.255.0.0     U     0      0        0 eth0
```

查看宿主机的iptable:

```
~$ sudo iptables-save
# Generated by iptables-save v1.6.0 on Wed Jan 17 21:00:17 2018
*nat
:PREROUTING ACCEPT [491:38712]
:INPUT ACCEPT [477:37797]
:OUTPUT ACCEPT [1867:126055]
:POSTROUTING ACCEPT [1868:126115]
:DOCKER - [0:0]
-A PREROUTING -m addrtype --dst-type LOCAL -j DOCKER
-A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE 将宿主机网卡(即不是docke0网卡)出去的流量中源地址是172.17的做SNAT源地址转换
-A POSTROUTING -s 172.17.0.3/32 -d 172.17.0.3/32 -p tcp -m tcp --dport 3000 -j MASQUERADE 
-A DOCKER -i docker0 -j RETURN
-A DOCKER ! -i docker0 -p tcp -m tcp --dport 4000 -j DNAT --to-destination 172.17.0.3:3000 外部通过宿主机端口访问容器内部, 需要进行目的地址和端口映射

*filter
:INPUT ACCEPT [366284:30610359]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [368968:40992259]
:DOCKER - [0:0]
:DOCKER-ISOLATION - [0:0]
-A FORWARD -j DOCKER-ISOLATION
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT
-A DOCKER -d 172.17.0.3/32 ! -i docker0 -o docker0 -p tcp -m tcp --dport 3000 -j ACCEPT 允许宿主机网卡(非docker)发往容器内(docker0)
-A DOCKER-ISOLATION -j RETURN
```

docker 直连:

```
root@dabf60697f38:/# curl 172.17.0.3:3000
Hello World tag4root

root@dabf60697f38:/# arp
Address                  HWtype  HWaddress           Flags Mask            Iface
172.17.0.3               ether   02:42:ac:11:00:03   C                     eth0
172.17.0.1               ether   02:42:96:ae:f2:53   C                     eth0
```

---

`/var/lib/docker/containers/{container_id}/` todo
