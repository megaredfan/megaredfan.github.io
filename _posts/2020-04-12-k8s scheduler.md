---
layout: post
title: 'k8s的调度'
---

## 资源模型与资源管理
### 资源限额配置
k8s中资源调度的单位是pod，对资源的需求都设置在pod的属性里，其中主要有cpu和memory等。在k8s中，cpu属于“可压缩”的资源，意思是当cpu资源不足时，pod只会产生“饥饿”，而不会退出；memory属于“不可压缩”的资源，当memory资源不足时，pod就会因为OOM而被kill掉。同时，pod内可以包含多个container，所以在包含多个container的pod中，对资源的需求实际上是配置在每个container里的，这样一来pod的资源配置就是其包含的所有container的资源配置的总和。

k8s中pod对资源的配置，包含requests和limits两种类型：在调度的时候，调度器只会按照requests的值进行计算。而在真正设置cgroups限制的时候，kubelet则会按照limits的值来进行设置。这个设计的思路在于：用户在提交pod时，可以声明一个相对较小的requests值供调度器使用，而k8s真正设置给容器cgroups的，则是相对较大的limits值。

这样的设计实际上源于Borg论文中的“动态资源边界”的定义，即：容器化作业在提交时所设置的资源边界，并不一定是调度系统所必须严格遵守的，这是因为在实际场景中，大多数作业使用到的资源其实远小于它所请求的资源限额。基于这种假设，Borg在作业被提交后，会主动减小它的资源限额配置，以便容纳更多的作业、提升资源利用率。而当作业资源使用量增加到一定阈值时，Borg会通过“快速恢复”过程，还原作业原始的资源限额，防止出现异常情况。

###pod的Qos级别
根据pod对requests和limits的不同的设置方式，k8s会将pod划分为三个不同的QoS级别：
+ guaranteed：当pod中每个container都同时设置了requests和limits，并且requests和limits值相等的时候，这个pod就属于guaranteed类别。当pod仅设置了limits没有设置requests的时候，k8s会自动为它设置与limits相同的requests值，这也属于guaranteed
+ burstable：当pod不满足guaranteed的条件，但至少有一个container设置了requests。那么这个pod就会被划分到burstable类别
+ besteffort：如果一个pod既没有设置requests，也没有设置limits，那么它的QoS类别就是besteffort

###pod的eviction
QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet对pod进行eviction（即资源回收）时需要用到的。具体地说，当k8s所管理的宿主机上不可压缩资源短缺时，就有可能触发eviction。比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等等。

pod的eviction有两种类型：soft和hard；soft类型表示在资源不足时，先等待一段时间（可配置）再进行eviction，hard类型表示当资源不足时就立刻进行eviction。kubelet在对pod进行eviction时，会根据pod的QoS级别来选择要被驱逐的pod：besteffort -> burstable -> guaranteed。同时k8s会保证只有当guaranteed类别的pod的资源使用量超过了其limits的限制，或者宿主机本身正处于Memory Pressure状态时，guaranteed的pod才可能被选中进行eviction操作。

## 默认调度器
k8s中的默认调度器的职责，就是为新创建的pod分配一个合适的node。而在筛选合适的node的过程中，主要包含两个步骤：
1. 从集群的所有node中，挑选出可以运行这个pod的node；这个步骤会运行一个叫做predicate的算法来检查每个node
2. 从第1步筛选的结果中，挑选出一个最符合条件的node作为最终结果；这个步骤会运行一个叫做priority的算法来为每个node打分，然后选出得分最高的node
而对一个pod调度成功，实际上就是将pod的spec.nodeName字段填上合适的node的名字；随后对应node上的kubelet会监听到这个更新，然后创建出实际的node。

### 默认调度器的工作原理
k8s的调度器的核心，实际上就是两个相互独立的控制循环，称为Informer Path和Scheduling Path。
![](/img/scheduler.png)

Informer Path的主要内容就是监听etcd中各种资源类型的变化（Pod、Node、Service等等）以及与调度相关的API对象的变化。当一个待调度的pod被创建之后，Informer Path就会感知到并将其加入调度队列中，默认情况下，调度队列是一个优先级队列。

此外，默认调度器还要负责对调度器缓存（scheduler cache）进行更新。事实上，k8s调度部分进行性能优化的一个最根本原则，就是尽最大可能将集群信息Cache化，以便从根本上提高Predicate和Priority调度算法的执行效率。

Scheduling Path的主要逻辑，就是不断地从调度队列里出队一个pod。然后，调用predicates算法进行“过滤”。这一步“过滤”得到的一组node，就是所有可以运行这个pod的宿主机列表。当然，predicates算法需要的node信息，都是从scheduler cache里直接拿到的，这是调度器保证算法执行效率的主要手段之一。接下来，调度器就会再调用priorities算法为上述列表里的node打分，得分最高的node，就会作为这次调度的结果。调度算法执行完成后，调度器就需要将pod对象的nodeName字段的值，修改为上述node的名字。这个步骤称作 Bind。但是，为了不在关键调度路径里远程访问 APIServer，默认调度器在Bind阶段，只会更新scheduler cache里的pod和node的信息。这种基于“乐观”假设的API对象更新方式被称作Assume。

Assume之后，调度器才会创建一个goroutine来异步地向APIServer发起更新pod的请求，来真正完成bind操作。如果这次异步的bind过程失败了，scheduler cache 同步之后一切就会恢复正常。由于调度器的“乐观”绑定的设计，当一个新的pod完成调度需要在某个节点上运行起来之前，该节点上的kubelet还会通过一个叫作Admit的操作来再次验证该pod是否确实能够运行在该节点上。这一步Admit操作，实际上就是把一组叫作GeneralPredicates的最基本的调度算法，比如：“资源是否可用”“端口是否冲突”等再执行一遍，作为kubelet端的二次确认。

### 默认调度策略
#### predicate
predicates在调度过程中的作用，可以理解为Filter，即：它按照调度策略，从当前集群的所有节点中，“过滤”出一系列符合条件的节点。这些节点，都是可以运行待调度pod的宿主机。在k8s中，默认的调度策略有如下三种：
+ GeneralPredicates：负责的是最基础的调度策略，比如资源是否足够，node的名字是否满足pod指定的名字，端口是否有冲突等等。
+ Volume相关的过滤规则
+ 宿主机相关的过滤规则
+pod相关的过滤规则

在具体执行的时候，当开始调度一个pod时，k8s调度器会同时启动16个goroutine，来并发地为集群里的所有node计算predicates，最后返回可以运行这个pod的宿主机列表。需要注意的是，在为每个node执行predicates时，调度器会按照固定的顺序来进行检查。这个顺序，是按照predicates本身的含义来确定的。比如，宿主机相关的predicates会被放在相对靠前的位置进行检查。要不然的话，在一台资源已经严重不足的宿主机上，上来就开始计算PodAffinityPredicate是没有实际意义的。
#### priority
完成了节点的“过滤”之后，priority阶段的工作就是为这些节点打分。这里打分的范围是0-10分，得分最高的节点就是最后被pod绑定的最佳节点。k8s中有很多打分规则，比如：
+ LeastRequestedPriority：打分公式为`score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2`，即选择空闲资源最多的宿主机
+ BalancedResourceAllocation：打分公式为`score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10`，其中每种资源的Fraction的定义是 ：请求的资源/节点上的可用资源。而variance算法的作用，则是计算每两种资源fraction之间的“距离”。而最后选择的，则是资源Fraction差距最小的节点。BalancedResourceAllocation选择的其实是调度完成后，所有节点里各种资源分配最均衡的那个节点，从而避免一个节点上 CPU 被大量分配、而 Memory 大量剩余的情况。
+nodeAffinityPriority：根据node亲和性打分
+ TaintTolerationPriority：根据污点容忍打分
+ InterPodAffinityPriority：根据pod间亲和性打分

### 优先级和抢占机制
优先级和抢占机制，解决的是pod调度失败时该怎么办的问题。当一个pod调度失败后，它就会被暂时“搁置”起来，直到pod被更新，或者集群状态发生变化，调度器才会对这个pod进行重新调度。但在有时候，我们希望的是这样一个场景。当一个高优先级的pod调度失败后，该pod并不会被“搁置”，而是会“挤走”某个node上的一些低优先级的pod。这样就可以保证这个高优先级pod的调度成功。
#### 优先级
在k8s中，我们可以先定义一个PriorityClass对象，其中指定了优先级的值（没有指定则默认为0），然后在pod中通过priorityClassName指定其要设置的优先级。而拥有较高优先级的pod，就会在调度过程中的优先级队列中优先出队，就体现了“优先级”的概念。
#### 抢占
而当一个高优先级的pod调度失败的时候，调度器的抢占能力就会被触发。这时，调度器就会试图从当前集群里寻找一个节点，使得当这个节点上的一个或者多个低优先级pod被删除后，待调度的高优先级pod就可以被调度到这个节点上。这个过程，就是“抢占”这个概念的主要体现。

当上述抢占过程发生时，抢占者并不会立刻被调度到被抢占的node上。事实上，调度器只会将抢占者的spec.nominatedNodeName字段，设置为被抢占的node的名字。然后，抢占者会重新进入下一个调度周期，然后在新的调度周期里来决定是不是要运行在被抢占的节点上。这当然也就意味着，即使在下一个调度周期，调度器也不会保证抢占者一定会运行在被抢占的节点上。

这样设计的一个重要原因是，调度器只会通过标准的DELETE API来删除被抢占的pod，所以，这些pod必然是有一定的“优雅退出”时间（默认是30s）的。而在这段时间里，其他的节点也是有可能变成可调度的，或者直接有新的节点被添加到这个集群中来。所以，鉴于优雅退出期间，集群的可调度性可能会发生的变化，把抢占者交给下一个调度周期再处理，是一个非常合理的选择。而在抢占者等待被调度的过程中，如果有其他更高优先级的pod也要抢占同一个节点，那么调度器就会清空原抢占者的 spec.nominatedNodeName 字段，从而允许更高优先级的抢占者执行抢占，并且，这也就使得原抢占者本身，也有机会去重新抢占其他节点。这些都是设置 nominatedNodeName 字段的主要目的。

而k8s调度器实现抢占算法的一个最重要的设计，就是在调度队列的实现里，使用了两个不同的队列。第一个队列，叫作activeQ。凡是在activeQ里的pod，都是下一个调度周期需要调度的对象。所以，当集群里新创建一个pod的时候，调度器会将这个pod入队到activeQ里面。而调度器不断从队列里出队（Pop）一个pod进行调度，实际上都是从activeQ里出队的。第二个队列，叫作unschedulableQ，专门用来存放调度失败的pod。而当一个unschedulableQ里的pod被更新之后，调度器会自动把这个pod移动到activeQ里。

当调度失败之后，抢占者就会被放进unschedulableQ里面。然后，这次失败事件就会触发调度器为抢占者寻找牺牲者的流程。第一步，调度器会检查这次失败事件的原因，来确认抢占是不是可以帮助抢占者找到一个新节点。这是因为有很多predicate的失败是不能通过抢占来解决的。比如PodFitsHost算法（负责的是检查pod的nodeSelector与node的名字是否匹配）。第二步，如果确定抢占可以发生，那么调度器就会把自己缓存的所有节点信息复制一份，然后使用这个副本来模拟抢占过程。这里的抢占过程很容易理解。调度器会检查缓存副本里的每一个节点，然后从该节点上最低优先级的pod开始，逐一“删除”这些pod。而每删除一个低优先级pod，调度器都会检查一下抢占者是否能够运行在该node上。一旦可以运行，调度器就记录下这个node的名字和被删除pod的列表，这就是一次抢占过程的结果了。当遍历完所有的节点之后，调度器会在上述模拟产生的所有抢占结果里做一个选择，找出最佳结果。而这一步的判断原则，就是尽量减少抢占对整个系统的影响。比如，需要抢占的pod越少越好，需要抢占的pod的优先级越低越好等等。在得到了最佳的抢占结果之后，这个结果里的node，就是即将被抢占的 Node；被删除的pod列表，就是牺牲者。所以接下来，调度器就可以真正开始抢占的操作了，这个过程可以分为三步。

1. 第一步，调度器会检查牺牲者列表，清理这些pod所携带的 nominatedNodeName 字段。
2. 第二步，调度器会把抢占者的nominatedNodeName，设置为被抢占的node的名字。
3. 第三步，调度器会开启一个goroutine，同步地删除牺牲者。

对于任意一个待调度pod来说，因为有上述抢占者的存在，它的调度过程其实是有一些特殊情况需要特殊处理的。具体来说，在为某一对pod和node执行predicate算法的时候，如果待检查的node是一个即将被抢占的节点，即调度队列里有nominatedNodeName字段值是该node名字的pod存在（可以称之为：“潜在的抢占者”）。那么调度器就会对这个node将同样的 predicate算法运行两遍。

第一遍，调度器会假设上述“潜在的抢占者”已经运行在这个节点上，然后执行predicate算法；第二遍，调度器会正常执行predicates算法，即不考虑任何“潜在的抢占者”。而只有这两遍predicate算法都能通过时，这个pod和node才会被认为是可以绑定（bind）的。