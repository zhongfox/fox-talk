---
layout: post
tags : [nosql, elasticsearch, 分布式]
title: Elasticsearch 实战
header-img: img/pic/2016/09/mohai.jpg

---

## 节点类型

* 主(master)节点: node.master: true
* 数据(data)节点: node.data: true
* 客户端节点: node.master和node.data都设置为false, 作为客户端节点，可以响应用户的情况，并把相关操作发送到其他节点
* 部落节点: TODO

* 协调节点: 并非通过配置控制的类型, 而是接受请求的节点, 任何节点都可以是协调节点, 协调节点将请求转发到数据节点，每个数据节点在本地执行请求并把结果传输给协调节点，然后协调节点收集各个数据节点的结果转换成一个单一的请求结果返回

虽然主节点也可以协调节点，路由搜索和从客户端新增数据到数据节点，但最好不要使用这些专用的主节点。一个重要的原则是，尽可能做尽量少的工作。创建一个独立的主节点的配置为：

```
node.master: true
node.data: false
```

脑裂是确保小集群无法工作, 避免数据一致性:

`discovery.zen.minimum_master_nodes: 通常设置原则是（master_eligible_nodes / 2）+ 1`

## Client 类型

Java 提供了2种客户端:

* Transport Client: 作为一个集群和应用程序之间的通信层。它知道 API 并能自动帮你在节点之间轮询，帮你嗅探集群等等, 属于集群外

* Node Client: 是一个集群中的节点（但不保存数据，不能成为主节点）。因为它是一个节点，它知道整个集群状态（所有节点驻留，分片分布在哪些节点，等等）。 这意味着它可以执行 APIs 但少了一个网络跃点

---

## 索引模板

创建索引模板

```
curl -XPUT 192.168.80.10:9200/_template/template_1 -d '
{
    "template" : "zhouls*",
    "order" : 0,
    "settings" : {
        "number_of_shards" : 1
    },
    "mappings" : {
        "type1" : {
            "_source" : { "enabled" : false }
        }
    }
}'
```

当存在多个索引模板时并且某个索引两者都匹配时，settings和mpapings将合成一个配置应用在这个索引上。合并的顺序可由索引模板的order属性来控制

* settings
* mpapings
* order: 多模板合并时控制优先级

删除索引模板

`curl -XDELETE 192.168.80.10:9200/_template/template_1`

模板配置文件:

位置: `config/templates/template_1.json`

---

## 索引别名

增加索引别名

```
curl -XPOST 'http://192.168.80.10:9200/_aliases' -d '
{
    "actions" : [
        { "add" : { "index" : "存在的索引名", "alias" : "新的索引别名" } }
    ]
}'
```

索引和别名是多对多

一个索引可以有多个别名, 一个别名对应多个索引:

```
curl -XPOST 'http://192.168.80.10:9200/_aliases' -d '
{
    "actions" : [
        { "add" : { "index" : "zhouls", "alias" : "zhouls_all" } },
        { "add" : { "index" : "zhouls10", "alias" : "zhouls_all" } }
    ]
}'
```

删除索引别名


```
curl -XPOST 'http://192.168.80.10:9200/_aliases' -d '
{
    "actions" : [
        { "remove" : { "index" : "zhouls", "alias" : "zhouls_all" } }
    ]
}'
```

---

## 管理和监控API

* `_cluster/health` 集群监控检查

* `_nodes/stats` 按照节点维度展示节点信息, 其中包括索引部分信息, jvm等等

* `_cluster/stats` 集群统计, 分为索引部分和node部分

* `具体索引名字/_stats` 指定索引统计, 索引名字可以是逗号分隔的多个.

* `_cluster/pending_tasks` 查看等待中的任务, 只出现在元数据变动的次数比主节点能处理的还快的情况下

* `/_cat`

  ```
  /_cat/allocation
  /_cat/shards
  /_cat/shards/{index}
  /_cat/master
  /_cat/nodes
  /_cat/indices
  /_cat/indices/{index}
  /_cat/segments
  /_cat/segments/{index}
  /_cat/count
  /_cat/count/{index}
  /_cat/recovery
  /_cat/recovery/{index}
  /_cat/health
  /_cat/pending_tasks
  /_cat/aliases
  /_cat/aliases/{alias}
  /_cat/thread_pool
  /_cat/plugins
  /_cat/fielddata
  /_cat/fielddata/{fields}
  ```

  加`?v`可以看到表头

  加`?help`查看帮助


---

## 插件

* http://127.0.0.1:9200/_plugin/bigdesk

* https://github.com/lmenezes/elasticsearch-kopf

















---

## 参考资料

* [节点类型](https://my.oschina.net/secisland/blog/618911)
* [索引模板和索引别名](http://www.cnblogs.com/zlslch/p/6478168.html)
