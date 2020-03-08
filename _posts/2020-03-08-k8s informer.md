---
layout: post
title: 'k8s client-go informer机制简介'
---

## 前言

`client-go` 是k8s提供的用于和k8s集群交互的sdk，根据k8s集群版本不同分为两个维护分支：`v0.x.y`版本（对应高于或等于0.17.0版本的k8s集群）和`kubernetes-1.x.y`版本（对应低于0.17.0版本的k8s集群）。其主要功能包括以下几个部分：

+ `kubernetes` 与k8s api通信
+ `discovery` 服务发现相关功能
+ `dynamic` 构造并向k8s api发送动态请求
+ `plugin/pkg/client/auth` 鉴权相关
+ `transport` 建立连接相关
+ `tools/cache` 客户端缓存的实现，主要在编写controller时用到，本文的介绍重点

由于client-go根据k8s服务端版本的不同区分了两个主要版本，具体安装时需要根据实际情况进行区分，具体安装方式可以参考这个[链接](https://github.com/kubernetes/client-go/blob/master/INSTALL.md)。

## k8s中的controller

controller在k8s中是一个重要的概念，一个controller主要负责追踪某个（或多个）资源的状态，并努力使其追踪的资源的状态达到预期的值。controller内部的逻辑主体是一个无限循环，称为“控制循环”。控制循环的大致流程如下：

1. 循环开始
2. 获取目标资源的当前状态和预期状态
3. 如果当前状态和预期状态有差异，尝试通过各种手段使当前状态向预期状态靠拢

当controller发现资源的当前状态和预期状态不同时，它可以直接操作资源使其状态发生变化，但是实际中更常见的做法是向k8s api server发送请求，触发其他事件来使资源发生变化。后者更值得考虑的原因在于，一个controller不应该完成许多工作，因为操作可能会失败，而复杂的控制循环则会使系统复杂度上升，难以维护。举个例子：k8s中的job controller，当它发现需要创建新的job实例时，并不会直接创建新的pod，而是向api server发送创建pod的请求，随后。。。

## informer

一个controller的逻辑主体就是在循环中进行"获取状态 -> 进行处理"的处理，所以client-go提供了从k8s api server获取资源状态的支持，也称为informer。

informer指的是client-go/tools/cache包下的逻辑的代称，它的主要功能就是是帮助controller与k8s api server交互，主要包括获取资源对象的信息以及根据资源对象的变更事件触发自定义的回调。它实际上包含几个组件：

+ reflector: reflector主要负责监听（watch）具体的资源类型，而watch具体是通过`ListAndWatch`实现。被监听的资源不限于内置的类型，也可以是自定义的资源类型。当reflector通过watch api接收到有新的资源实例被创建的通知时，它会主动通过list api获取到新的资源实例，然后将其放入到`Delta Fifo`队列中。

+ informer: informer主要的工作是循环地从`Delta Fifo`队列中取出对象，随后通过回调传递给controller，以及交给indexer缓存下来。

+ indexer: indexer主要提供索引的功能，而通常对象的索引是根据对象的label创建的。indexer通过线程安全的结构来存储对象，同时cache包下定义了默认的根据对象label生成索引的函数`MetaNamespaceKeyFunc`，具体格式为`<namespace>/<name>`。

## controller中使用informer的场景

下图是controller在实际中通过client-go与k8s api server交互的示意图：
![img](/img/client-go-controller-interaction.jpeg)

从示意图中我们可以看出client-go主要提供的两个机制：第6步的事件注册和触发机制以及第9步的根据key获取对象。同时我们可以看到，在controller中实际有两个reference：informer reference和index reference，它们分别对应前面提到的两个功能。

而informer内部的流程主要包括几个关键点：

1. reflector 通过list & watch 从api server获取资源的信息，其中初始化时通过list获取最新信息，随后依赖watch监听资源变化，同时伴随可配置的定时resync机制。
2. reflector 获取到的资源变更事件都通过`Delta Fifo`队列传递到informer
3. informer 从队列中获取到事件及对应的对象时，会触发对应的controller回调并将对象传递到indexer进行索引和缓存。
4. controller 获取对象时，会直接从indexer返回缓存，提高性能
5. 对象在controller内部传递时也通过key传递，而不是直接传递对象本身，在实际处理时才从indexer获取最新状态的对象

另外示意图中的几个组件也需要介绍一下：

+ Resource Event Handlers: controller注册的事件回调函数，informer会在触发这些回调时将对应的资源对象也传递给controller，推荐的做法是根据对象获取其对应的key，然后将key放入work queue中（而不是直接把对象放进queue），随后再由其他组件处理。

+ Work queue: 用于存放在事件回调中收到的对象的key，通过队列来解耦对象的接受和处理。

+ Process Item: 实际处理从队列中获取的对象的逻辑，在做实际操作前往往需要从队列中获得的key从indexer reference获取资源对象。
