---
layout: post
title: 'k8s NetworkPolicy和Service'
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

此外，k8s还会为Service的vip创建一条DNS记录，格式为..svc.cluster.local，当访问其对应的DNS记录时，解析到的实际是Service对应的vip地址。然而对于制定了clusterIP=None的Headless Service来说，其DNS对应的记录解析到的结果是其代理的pod的ip的集合。对于ClusterIP模式的Service来说，它代理的pod会自动分配一个DNS记录，格式是..pod.cluster.local，这条记录则指向pod的ip地址。

除了通过selector指定pod以外，k8s还支持显示指定Service和Endpoint的绑定关系，这需要首先声明一个不带selector字段的Service，比如：

```yaml
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

这样一来，当我们通过vip访问my-service时，请求就会直接转发到192.0.2.42:9376

### 具体实现

Service实际上是通过kube-proxy组件加上iptables来实现的。Service提供多种代理模式：用户空间代理、iptables代理以及IPVS代理。

#### 本地进程代理

这个模式下，kube-proxy会监听Service和Endpoint的变化，然后在宿主机上启动一个代理端口（随机选择），当pod中发起连接到代理端口的连接时，这个连接会被桥接到Service实际的被代理pod上，在进行负载均衡时，还会参考`SessionAffinity`相关的配置。

然后kube-proxy会设置一系列的iptables规则，将发往Service vip的流量转发到代理端口上。

默认情况下，kube-proxy使用round-robin的规则来进行负载均衡。

#### iptables代理

这个模式下，kube-proxy会监听Service和Endpoint的变化，然后在宿主机上设置iptables规则，将发往Service vip的流量转发到实际的被代理的pod上，这个过程中会随机选择一个pod。这个模式比起本地代理性能会更高，也更为可靠。

当被选中的pod无法响应时，本次请求就会直接失败；而本地代理模式下，代理进程会检测到失败并将连接切换到新的pod上。当然在这里可以通过pod的readinessProbe来进行pod的健康检查。

#### ipvs代理

这个模式下，kube-proxy会监听Service和Endpoint的变化，然后通过`netlink`创建和定期维护ipvs规则。当pod需要访问Service时，ipvs会直接将流量转发到对应的目标pod。ipvs模式比起前两个模式性能更好，同时对大流量的场景支持的也更好（iptables会占用很多宿主机的CPU资源）。除此之外，ipvs模式还支持多种负载均衡策略比如round-robin、least connection、destination hash、source hash等等。

值得注意的是要使用ipvs模式需要宿主机支持ipvs功能，如果设置了ipvs模式但是宿主机不支持ipvs功能，则会fallback到iptables代理模式。

### 集群外访问Service

很明显Service相关的路由信息等只有在k8s集群内的才能访问到，而在k8s外想要访问Service，则需要一些额外的配置。

#### NodePort

如果将Service的type字段设置为NodePort，k8s就会在每个node将那个端口代理到指定的Service中。可以在spec.ports.nodePort字段中要求分配的端口，如果不指定则会分配一个随机的端口（具体范围是--service-node-port-range标志指定的范围内，默认值：30000-32767）。比如：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - nodePort: 8080
    targetPort: 80
    protocol: TCP
    name: http
  - nodePort: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```

采用NodePort方式部署Service，我们可以在集群内任意node通过指定端口访问实际的Service了。

#### LoadBalancer

除了NodePort方式，我们还可以利用一些公有云提供的负载均衡能力来对外暴露Service。通过把Service的type设置为LoadBalancer，可以触发外部负载均衡的创建，来自外部的流量就能被转发到实际的pod了。

#### ExternalIP

k8s还支持为Service指定ExternalIP属性，可以将一个ip直接绑定到Service，这样访问这个ip时就可以直接访问到Service代理的pod中。但这个特性要求ExternalIP中设置的ip必须能够路由到k8s中的一个node上。

#### ExternalName

k8s 1.7版本之后，支持了一个叫做ExternalName的新特性，将Service的type设置为ExternalName并设置了externalName属性之后，就可以将Service和externalName直接关联起来。这个方式实际上是将Service的DNS解析结果设置为externalName的解析结果，所以k8s要求externalName必须是一个DNS域名格式。

ExternalName的作用更类似于将外部的服务包装成一个k8s的Service对象，比如我有一个数据库集群并不部署在k8s中，但是我可以将数据库集群的vip绑定到一个Service，这样k8s内的pod就可以通过这个Service访问这个数据库集群了。

### Ingress

除了上述的将Service对外暴露的方式之外，k8s还提供了一个统一的抽象--Ingress。简单来说，Ingress就是多个Service的上层路由，可以将不同的流量路由到不同的Service上，而这只需要配置一个Ingress对象即可。举个例子：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

上面的配置中，spec.hosts下配置了Ingress的入口，也就是cafe.example.com，也就是当访问cafe.example.com这个host时，实际上就会访问到这个Ingress对象。rules下配置了两个规则，表示根据访问的路径不同，Ingress需要把流量转发到不同的Service下。

和其他资源类型类似，Ingress类型也需要对应的controller配合使用，Ingress的定义在k8s中创建之后，需要有具体的controller来创建具体的代理。目前Ingress controller的实现有很多，包括 Google Cloud Load Balancer， Nginx，Contour，Istio等等。他们都会根据Ingress的配置创建出实际的可工作的服务代理。

通过配置Ingress对象配合实际的Ingress controller实现，我们就可以省去将每个Service对外暴露的步骤，直接将Service配置到Ingress的下游即可。