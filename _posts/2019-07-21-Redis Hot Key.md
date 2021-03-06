---
layout: post
title: "redis热点"
---

## redis简介

redis是一个用C语言编写的开源的、基于内存的键值对存储数据库。redis支持多种数据结构如string、hash、list、set等等，同时还支持原子操作。相比起其他kv类型的数据库产品，redis有两个突出的特点：

1. redis支持多种数据结构，并且可以在复杂的数据结构上进行原子操作
2. redis支持数据的持久化，包括RDB和AOF格式

除此之外，redis的另一个着重宣传的优势就是快，因为redis对数据的操作都是基于内存的，同时redis采用单线程模型，所有的命令都在同一个线程中执行，避免了线程切换的影响。

## redis使用的限制

由于redis独特的单线程模型，在使用redis时需要注意几个地方，避免出现性能下降。

### 避免耗时的操作

前面提到，redis只用一个线程处理所有的命令，所以应避免出现耗时的操作，否则会阻塞整个redis实例，影响性能。

### 避免大key

大key指的是一个key对应的value过大的情况，比如string的长度非常大，或者hash、list、set、zset类型中元素个数非常多等等。redis中的大key会导致查询、删除等操作变慢，网卡带宽占满等问题，影响其他查询。

### 避免KEYS遍历

redis中的KEYS命令可以根据给定的正则表达式匹配所有符合的key，时间复杂度为O(N)，在key的数量特别多时耗时会变长，直接导致其他命令阻塞甚至实例崩溃。

## redis热点产生的原因

在实际的生产中，我们通常会通过集群的模式部署redis来保证性能和可用性。redis集群的原理是通过分片来扩展服务能力，但不支持同时处理多个key的命令，来自客户端的请求会根据一定的规则路由到集群中的某个实例，而实例与实例之间不进行数据交换。

热点通常是由实际业务中的热门商品、热点新闻或者突发事件引起的，这是用户会大量而且集中的访问某个数据。而在服务端访问数据时，就会根据数据分片规则访问某个redis实例，这时由于请求过于集中，所有的请求都会落到同一个实例，就产生了redis热点。

## redis热点的危害

redis热点实际上就是某个redis实例的负载过高，进而导致以下几个问题：

1. 主机网卡被打满，影响主机上的其他服务
2. 请求量过大，导致redis实例崩溃
3. redis失效导致业务降级到读取DB，进而导致DB也崩溃，业务雪崩

## redis热点的解决方法

### 避免直接读DB

这是比较简单的方法，业务的服务端维护一个本地缓存，当redis负载过高或者崩溃时不降级到DB，而是返回本地缓存中的数据。但是这个方案有以下几个问题：

1. 需要一个判断是否返回本地缓存的机制
2. 需要维护本地缓存

### 读写分离

读写分离的方案需要引入一个代理层，在redis客户端与redis集群之间增加一个代理，来自客户端的请求会先经过代理层做负载均衡和路由。然后redis集群中的实例分为读和写两类，读节点只负责读取数据。当出现redis请求热点时，读节点负载就会上升，这是我们可以扩容出更多的读节点来分担负载，避免redis实例崩溃。但是这种方案由以下几个要求：

1. 需要维护代理层和负载均衡
2. 需要实时监控redis负载
3. 需要支持节点扩缩容

### 热点数据缓存

另一种可以选择的方案就是对热点数据进行缓存，这里可以选择在代理层进行缓存或者是在客户端缓存。当发现有redis热点时，可以及时从缓存读取，避免直接向redis实例发起请求。这个方案有以下几个要求：

1. 需要维护本地缓存
2. 需要redis热点发现机制

### 热点的发现

热点的发现可以通过统计的方式实现。我们可以在代理层周期统计每个key的访问情况，当超过指定阈值时key就成为了热点key。
