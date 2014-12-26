---
layout: post
title: "Redis集群的数据划分与扩容探讨"
date: 2014-12-26 10:23
comments: true
categories: Redis
author: 高磊
author_url: http://xueqiu.com/lostleon
---

首先界定一下，本文探讨的标的不是官方即将发布的Redis 3.0 Cluster，而是自行实现的一个生产环境中的Redis集群。这是一个典型的分布式系统，以下讨论将围绕CAP原理中，与P相关的数据划分和扩容两个问题展开。

### 目标

对这样一个系统，在数据划分和扩容方面最理想的目标是：

1. 数据划分针对客户端透明
2. 支持在线热扩容，扩容针对客户端透明
3. 支持纵向扩容、横向扩容

### 数据Sharding分片

Redis在部署时的拓扑结构完全取决于其适用的业务场景。基本上可以划分为两类：

##### a) Cache场景

对Cache场景，服务器端最常见的部署拓扑是**"一致性哈希"**。一致性哈希可以极大的优化机器增删时带来的哈希目标漂移问题。同时对于Hash目标漂移时产生的严重的数据倾斜，可以利用虚拟节点来优化。

基本上，物理节点有了一定规模后，只要不是同时挂多个节点，或者同时扩容多个节点，数据分片不会有太大的扰动。穿透过Cache的请求后端存储可以抗住即可。

基本可以认为，在Cache的场景下，Redis是可以比较完美的实现前述横向扩容目标的。

##### b) Storage场景

Storage场景下的数据分片则会复杂一些。原因是既然作为存储就要保证数据的高可用，要实现高可用，Redis自身提供了主从Replication机制，通过多副本来保证。Redis在主从复制高可用方面也经历了较长的迭代，从最初2.4版本备受诟病的全量同步，到2.6版本终于实现了增量同步，到2.8版本的Sentinel代理。

高可用有了保证后，我们回过头来看数据分片，最简单的方式是对Key哈希后按照分片数**"取模"**。

略微复杂一点的话可以使用**"预分片（Pre-Sharding)"**的方案，有人也称呼为按“桶”进行数据划分。阿里的Tair（<http://tair.taobao.org/>）框架，以及豌豆荚最近放出的Codis（<https://github.com/wandoulabs/codis>）对数据的划分都使用了Pre-Sharding方案。

Pre-Sharding方案实际上可以理解为预先分配一个相当大的集合，对Key哈希的结果落在这个集合中，集合的每个元素又与具体的物理节点存在多对一的路由映射关系，这张路由表由一个配置中心进行维护。阿里的Tair把它叫做ConfigServer中，豌豆荚的Codis把它叫做ConfigManager，一个意思。

回过头来再细想下，一致性哈希中的虚拟节点，实际上也可以归类到Pre-Sharding方案中。换句话说，只要是key经过两次哈希，第一次Hash到虚拟节点，第二次Hash到物理节点，都可以算作Pre-Sharding。只不过区别在于，一致性哈希的第二次Hash其路由表是按照算法固定的，Tair/Codis的第二次Hash其路由表是第三方可配的。

### 纵向扩容 Scale Up

Redis的优雅之处其中就包括对在线纵向扩容的支持，直接一系列config set maxmemory完事（当然别忘了同步改从库，以及rewrite conf文件）。不过config set maxmemory在2.4版本有个小bug就是不支持以K/M/G为单位，不知道后续版本修复了没有。

### 横向扩容 Scale Out

前已述，Cache场景下横向扩容是没有问题的。

Storage场景下，我们分别根据两种Sharding方式探讨两种方案

##### a) 取模Sharding

假如扩容前是模N，我们提出的方案：扩容后，对N的倍数进行模运算。

以扩容后模2N为例，具体操作为：

1. 首先对N个Master节点（如A、B、C），以1:1建立N个Slave同步（如D、E、F）
2. 然后关闭Slave库的ReadOnly，但主从关系和顺序保持不变，客户端改为模2N（A、B、C、D、E、F）
3. 然后断开主从
4. 最后异步清理掉2N个库中多余的50%的Key

![取模Sharding扩容](https://www.lucidchart.com/publicSegments/view/549c593d-b9ac-4e0d-8334-2ced0a00c2b2/image.png)

这个方案的可行之处在于

1. 扩容前后，虽然取模的结果变了，但是目标节点的数据仍在
2. 在第2步中，允许多个客户端加载模2N有先后，因为不存在Race关系。（反证法：如果存在Race关系，那么在没扩容没重启时也存在Race关系）

最后一步清理Key的方式有两种

1. 如果有定期做RDB备份的话，可以异步解析RDB，挑出其中冗余的50% Key，在低峰期删除。这种方式比较彻底。
2. 如果没有定期RDB备份的话，可以在低峰其起异步的工具不断randomkey()，并检查其是否冗余，若冗余则删除。一段时间后冗余Key的数量一定会大大下降，但是不彻底。

##### b) Pre-Sharding

Pre-Sharding中，横向扩容只需要去修改Hash路由表即可，增加物理节点仍然需要保证数据可访问，类似模倍数方案。

或者如果有Proxy中间件的话让中间件进行路由。在这里值得一提的是Codis通过patch Redis实现了以key为粒度的原子migrate操作，使得通过中间件进行路由极为便捷。

另外还有个情况要考虑，假如万一路由表用满了，Pre-Sharding也就退化为取模Sharding的模式了，还可以再采用模倍数方案。

### 横向扩容客户端透明

客户端与Redis物理节点间有两种方式的连接

##### a) 直连

直连的话，客户端必须从配置中心订阅到所需的节点变动通知。这个配置中心可以是Tair的配置中心，也可以是ZooKeeper/etcd等等，也可以是Sentinel。

其中Sentinel有个天生的问题，就是它的监控粒度是一套主从节点。如果在Storage场景中，按照取模的方式进行Shard，并使用Sentinel做配置管理。这时横向扩容的话Sentinel并不能有效的通知客户端节点Sharding数发生了变化，解决这个问题需要一部分hack工作。而ZooKeeper/etcd等通用配置中心则可定制化程度较高。

##### b) 通过Proxy中间件透传

类似Twitter的twemproxy（<https://github.com/twitter/twemproxy>），或者百度的的bdrp（<https://github.com/ops-baidu/bdrp>）或者前述的豌豆荚codis，以及京东的Redis集群都是使用了一层Proxy进行透传请求。这样只需要Proxy能够订阅到物理节点的变更，并自动加载即可。订阅方式同样可以走各种配置中心。

### 总结

针对Cache和Storage两种场景，对数据划分和扩容有不同的方案和取舍点，可以在便捷性和成本等各个方面进行权衡。

另外也可以留心下即将发布的Redis 3.0，在线下环境中折腾玩下。

```
我们在招聘 “高级架构师 / DBA / Java工程师” 等职位
详细JD可以参考 http://xueqiu.com/about/jobs
感兴趣的同学欢迎投简历到 gaolei@xueqiu.com
```
