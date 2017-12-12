---
layout: post
tags : [nginx]
title: nginx 负载均衡

---

nginx的负载均衡策略可以划分为两大类：内置策略和扩展策略

内置策略:
* 加权轮询
* ip hash

扩展策略:
* fair
* url_hash

---

## 负载均衡配置指令

nginx负载均衡模块: `ngx_http_upstream_module`

### upstream name {...}

所属指令：http

定义一组用于实现nginx负载均衡的服务器，它们可以侦听在不同的端口。另外，可以混合使用侦听TCP与UNIX-domain套接字文件


```
upstream backend {
    server backend1.example.com       weight=5;
    server backend2.example.com:8080;
    server unix:/tmp/backend3;

    server backup1.example.com:8080   backup;
    server backup2.example.com:8080   backup;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

默认情况下，请求被分散在使用加权轮询的nginx负载均衡服务器上.
如果在于服务器通信时发生了一个错误，这个请求会被传递给下一个服务器.
以此类推至道所有的功能服务器都尝试过。如果不能从所有的这些nginx负载均衡服务器上获得回应，客户端将会获得最后一个链接的服务器的处理结果.

### server 地址 [参数]；

可以定义下面的参数:

* weight=number
* max_fails=number
* fail_time=time
* backup: 标记该服务器为备用服务器。当主服务器停止时，请求会被发送到它这里
* down: 标记服务器永久停机了；与指令ip_hash一起使用


---

## 负载均衡策略

### 轮询

Round Robin

```
upstream backserver {
  server 192.168.0.14;
  server 192.168.0.15;
}
```

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除

### 加权轮询

Weighted Round Robin

```
upstream backserver {
  server 192.168.0.14 weight=10;
  server 192.168.0.15 weight=10;
}
```

机器按照权重权限, 首先将请求都分配给权重最高的机器, 类似一种「深度优先」, 如果该机器成功建立连接 将会把权重减一重排.

如果后端某台服务器宕机，故障系统被自动剔除

当所有后端机器都down掉时，nginx会立即将所有机器的标志位清成初始状态，以避免造成所有的机器都处在timeout的状态，从而导致整个前端被夯住

### ip hash

基于客户端ip的负载均衡算法, IPV4的前3个八进制位和所有的IPV6地址被用作一个hash key.

```
upstream backserver {
  ip_hash;
  server 192.168.0.14:88;
  server 192.168.0.15:80;
}
```

hash值既与ip有关又与后端机器的数量有关, hash 算法可以产生1045个互异的value.

来自同一个IP的访客固定访问一个后端服务器，有效解决了动态网页存在的session共享问题。当然如果这个节点不可用了，会发到下个节点，而此时没有session同步的话就注销掉了

在实际的网络环境中，有大量的高校出口路由器ip、企业出口路由器ip等网络节点，这些节点带来的流量往往是普通用户的成百上千倍

如果nginx负载均衡器组里面的一个服务器要临时移除，它应该用参数down标记，来防止之前的客户端IP还往这个服务器上发请求

### fair

第三方负载均衡策略

```
upstream backserver {
  server server1;
  server server2;
  fair;
}
```

扩展策略, 默认不被编译进nginx内核, 其原理是根据后端服务器的响应时间判断负载情况，从中选出负载最轻的机器进行分流.

### url_hash

第三方负载均衡策略

按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效.

```
upstream backserver {
  server squid1:3128;
  server squid2:3128;
  hash $request_uri;
  hash_method crc32;
}
```


### session_sticky

依赖第三方模块 `nginx-sticky-module`

`sticky [name=route] [domain=.foo.bar] [path=/] [expires=1h] [hash=index|md5|sha1] [no_fallback];`

通过cookie黏贴的方式将来自同一个客户端（浏览器）的请求发送到同一个后端服务器上处理

这个模块并不合适不支持 Cookie 或手动禁用了cookie的浏览器，此时默认sticky就会切换成RR。它不能与ip_hash同时使用

`ngx_http_upstream_session_sticky_module` 也是类似的功能

### least_conn

Least Connections

跟踪和backend当前的活跃连接数目，最少的连接数目说明这个backend负载最轻，将请求分配给他

### 通用hash、一致性hash


---

## 后端服务器的健康检查

nginx自带是没有针对负载均衡后端节点的健康检查的，但是可以通过默认自带的 ngx_http_proxy_module 模块和 ngx_http_upstream_module 模块中的相关指令来完成当后端节点出现故障时，自动切换到下一个节点来提供访问

指令:

* weight
* max_fails
* fail_timeout
* backup
* max_conns
* proxy_next_upstream

`nginx_upstream_check_module` 是专门提供负载均衡器内节点的健康检查的外部模块

通过它可以用来检测后端 realserver 的健康状态。如果后端 realserver 不可用，则后面的请求就不会转发到该节点上


---

## 参考资料

* [加权轮询策略剖析](http://blog.csdn.net/xiajun07061225/article/details/9318871)
* [解析 Nginx 负载均衡策略](http://www.cnblogs.com/wpjamer/articles/6443332.html)
* [Nginx做负载均衡器以及Proxy缓存配置](https://mp.weixin.qq.com/s?__biz=MzA3MzYwNjQ3NA==&mid=401736802&idx=1&sn=9508d9a05b725c66a05e05d8dad6dec1#rd)
