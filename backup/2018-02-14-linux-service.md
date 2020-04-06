---
layout: post
tags : [linux]
title: Linux 服务管理

---

## Linux 启动流程

1. 加载BIOS: BIOS中包含了CPU的相关信息、设备启动顺序信息、硬盘信息、内存信息、时钟信息、PnP特性等等.

2. 读取MBR: 硬盘上第0磁道第一个扇区被称为MBR，也就是Master Boot Record，即主引导记录，它的大小是512字节，别看地方不大，可里面却存放了预启动信息、分区表信息. 系统找到BIOS所指定的硬盘的MBR后，就会将其复制到0×7c00地址所在的物理内存中。其实被复制到物理内存的内容就是Boot Loader.

3. Boot Loader: Boot Loader 就是在操作系统内核运行之前运行的一段小程序. 初始化硬件设备、建立内存空间的映射图，从而将系统的软硬件环境带到一个合适的状态，以便为最终调用操作系统内核做好一切准备。

4. 加载内核

5. 用户层init依据inittab文件来设定运行等级

   第一个运行的程序便是/sbin/init，该文件会读取/etc/inittab文件，并依据此文件来进行初始化工作

   /etc/inittab文件最主要的作用就是设定Linux的运行等级, 其设定形式是“：id:5:initdefault:”，这就表明Linux需要运行在等级5上

   同时, 该文件设定了不同runlevel要启动的服务脚本位置, 比如:
   
   `/etc/rc1.d/` ... `/etc/rc5.d/` `/etc/rc6.d/` (有的系统貌似是`/etc/rc.d/rc数字.d/`)这些目录下是链接文件, 链接到`/etc/init.d`下的具体服务脚本

   ```
    S01apport -> ../init.d/apport
    S01kexec -> ../init.d/kexec
    S01lvm2-lvmetad -> ../init.d/lvm2-lvmetad
    S01lvm2-lvmpolld -> ../init.d/lvm2-lvmpolld
    S01lxcfs -> ../init.d/lxcfs
    S01lxd -> ../init.d/lxd
    S01open-vm-tools -> ../init.d/open-vm-tools
    S01rsyslog -> ../init.d/rsyslog
    S01uuidd -> ../init.d/uuidd
    S01YDService -> ../init.d/YDService
    S02acpid -> ../init.d/acpid
    S02atd -> ../init.d/atd
    S02cron -> ../init.d/cron
    S02dbus -> ../init.d/dbus
    S02kdump-tools -> ../init.d/kdump-tools
    S02kexec-load -> ../init.d/kexec-load
    S02mdadm -> ../init.d/mdadm
    S02ntp -> ../init.d/ntp
    S02rsync -> ../init.d/rsync
    S02ssh -> ../init.d/ssh
    S02thermald -> ../init.d/thermald
    S03grub-common -> ../init.d/grub-common
    S03ondemand -> ../init.d/ondemand
    S03plymouth -> ../init.d/plymouth
    S03rc.local -> ../init.d/rc.local
    ```

   0~6 个 runlevel

   命令`runlevel`可以查看当前runlevel

6. init进程执行rc.sysinit: 它做的工作非常多，包括设定PATH、设定网络配置（/etc/sysconfig/network）、启动swap分区、设定/proc等

7. 启动内核模块: 依据/etc/modules.conf文件或/etc/modules.d目录下的文件来装载内核模块

8. 执行不同运行级别的脚本程序: 根据运行级别的不同，系统会运行rc0.d到rc6.d中的相应的脚本程序，来完成相应的初始化工作和启动相应的服务

9. 执行/etc/rc.d/rc.local

10. 执行/bin/login程序，进入登录状态

---

## service

service命令用于对系统服务进行管理，比如启动（start）、停止（stop）、重启（restart）、查看状态（status）

要把一个程序注册成系统服务，首先得给出一个供service命令调用的脚本文件放到目录/etc/init.d/(貌似有的系统是 "/etc/rc.d/init.d/")中去

实际上，脚本的内容是完全可以按照自己需要来编写

* `service httpd` 等价 `/etc/rc.d/init.d/httpd`
* `service httpd start` 等价 `/etc/rc.d/init.d/httpd  start`
* `service httpd stop` 等价 `/etc/rc.d/init.d/httpd  stop`

为什么要注册成为service服务: 

* 可以使用"service 服务名称"来进行管理，比如常常使用的命令”service httpd start”,就是httpd注册成为一个服务了，于是才不需要写一大串的原始服务路径
* 注册成系统服务，还有一个好处。可以使用chkconfig命令来控制运行级别。也就是控制什么级别下面是开启运行

常用命令:

* service --status-all
* service dockerd status: 貌似还能去读取systemd的服务
*

## chkconfig

功能貌似和 update-rc.d  差不多

chkconfig不是立即自动禁止或激活一个服务，它只是简单的改变了符号连接(应该是`/etc/rc.d/rc数字.d/*`链接到`/etc/init.d/*`)

* `chkconfig –level sphinx 3456`: 这个命令是设置在3、4、5、6运行级别下sphinx服务(也就是/etc/rc.d/init.d/sphinx这个脚本)是启动状态
* chkconfig {service} on: 设置指定服务service开机时自动启动
* chkconfig {service} off: 设置指定服务service开机时不自动启动



---


## systemd

Systemd 是 Linux 系统工具，用来启动守护进程，已成为大多数发行版的标准配置.

Systemd 默认从目录/etc/systemd/system/读取配置文件。但是，里面存放的大部分文件都是符号链接，指向目录/usr/lib/systemd/system/，真正的配置文件存放在那个目录

* systemctl status: unit 是否正在运行
* systemctl status  dockerd, 可以看到该服务的配置文件
* systemctl enable XXX.service: 建立上面两个目录的符号链接.
* systemctl is-active application.service 如`systemctl is-active dockerd.service`
* systemctl list-unit-files: 列出所有配置文件, 以及启动状态
  * enabled：已建立启动链接
  * disabled：没建立启动链接
  * static：该配置文件没有[Install]部分（无法执行），只能作为其他配置文件的依赖
  * masked：该配置文件被禁止建立启动链接

---

## 参考资料

* [linux中注册系统服务—service命令的原理通俗](https://www.cnblogs.com/wangtao_20/archive/2014/04/04/3645690.html)

* [Systemd 入门教程：命令篇](http://www.ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)

* [Ubuntu Service说明与使用方法](http://www.mikewootc.com/wiki/linux/usage/ubuntu_service_usage.html)
