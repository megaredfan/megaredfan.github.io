---
layout: post
title: 'k8s networkpolicy、service与ingress'
---

## NetworkPolicy

k8s默认情况下，所有pod都是连通的。如果有隔离的需求，可以通过设置NetworkPolicy来实现。NetworkPolicy在namespace范围下工作，它可以根据podSelector选中某些pod。而被NetworkPolicy选中的pod就进入了“默认隔离”的状态，只有符合NetworkPolicy中规则的流量才会被放行。值得注意的是NetworkPolicy需要对应的CNI插件支持，也就是说NetworkPolicy只提供了隔离规则的声明，而实际根据声明进行隔离操作的则是具体的网络实现方案，比如Calico、Weave、kube-router等，但是Flannel并不支持。

### 配置的格式

一个完整的NetworkPolicy的定义如下所示：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: 
    name: test-network-policy
    namespace: default
spec:
    podSelector:
        matchLabels:
            role: db
    policyTypes:
    - Ingress
    - Egress
    ingress:
    - from:
        - ipBlock:
            cdir: 172.17.0.0/16
            expect:
                172.17.1.0/24
        - namespaceSelector:
            matchLabels:
                project: myproject
        - podSelector:
            matchLabels:
                role: frontend
        ports:
        - protocol: TCP
          port: 6379
    egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/24
        ports:
        - protocol: TCP
          port: 5978
```

首先spec中的podSelector字段表示这个配置只影响default namespace下的带role=db标签的pod。policyTypes字段中声明了这个配置既影响流入（Ingress）的流量也影响（Egress）流出的流量，意思是被选中的pod的流入流量和流出流量都变成了默认拒绝，只有符合下面规则的流量才会被放行。如果没有声明policyTypes，则默认只影响流入的流量；如果没有声明policyTypes但是规则中声明了egress规则，则也会影响流出的流量。

规则定义方面，示例配置中声明了目标端口是6379的TCP协议的流入流量，同时满足以下任意条件的流量才会被放行：

1. 源地址属于172.17.0.0/16网段但不属于172.17.1.0/24
2. 来源pod的namespace中有project=myproject标签
3. 来源pod有role=frontend标签

值得注意的是规则声明中from和to部分下的内容以`-`开头表示“或”的关系，没有`-`开头的规则则表示“与”的关系。

流出规则的作用类似，就不叙述了。

### 配置的实现

前面提到了，NetworkPolicy需要配套的网络方案实现，而在具体实现中，所有支持NetworkPolicy的插件都维护着一个NetworkPolicy Controller，通过控制循环的方式来NetworkPolicy对象的变化，然后在宿主机上进行iptables的配置。iptables的规则大概有几条：

1. 拦截同一台宿主机上通过CNI网桥进行通信的流量
2. 拦截跨宿主机通信的流量
3. 将拦截的流量转交给NetworkPolicy的规则进行匹配
4. 匹配失败的流量则直接拒绝

## Service

Service是k8s内置的资源类型，它的作用就是将一组pod通过一个单一的ip和dns记录暴露出去，同时提供负载均衡。

### 配置格式

一个简单的Service配置如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:
    name: hostnames
spec:
    selector:
        app: hostnames
    ports:
    - name: default
      protocol: TCP
      port: 80
      targetPort: 9376
```

示例中的Service表示它只代理携带了app=hostnames标签的pod，同时这个Service对外的端口是80，被代理的pod的端口则是9376。被代理的pod也被称为Endpoint，只有处于Running状态同时readinessProbe检查通过的Pod才会被Service纳入Endpoint列表中，而且当一个pod出现问题时，k8s会将其自动摘掉。当成功创建一个Service时，k8s会为其分配一个vip（也称为cluster ip），通过Service的vip就能顺利访问其代理的后端pod了。

此外，k8s还会为Service的vip创建一条DNS记录，格式为

除了通过selector指定pod以外，k8s还支持显示指定Service和Endpoint的绑定关系，这需要首先声明一个不带selector字段的Service，比如：

```ymal
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

然后再通过声明一个Endpoint类型对象，通过metadata中的name字段把它和Service关联起来：

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 192.0.2.42
    ports:
      - port: 9376
```
这样一来，当我们通过vip访问my-service的
Endpoint的声明中，ip不能为回环地址，也不能指定为其他Service的vip。


## Ingress