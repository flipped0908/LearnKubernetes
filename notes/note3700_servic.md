## 容器的发现

Service 机制，以及 Kubernetes 里的 DNS 插件，都是在帮助你解决同样一个问题，即：如何找到我的某一个容器？

这个问题在平台级项目中，往往就被称作服务发现，即：当我的一个服务（Pod）的 IP 地址是不固定的且没办法提前获知时，我该如何通过一个固定的方式访问到这个 Pod 呢？

而我在这里讲解的、ClusterIP 模式的 Service 为你提供的，就是一个 Pod 的稳定的 IP 地址，即 VIP。并且，这里 Pod 和 Service 的关系是可以通过 Label 确定的。

而 Headless Service 为你提供的，则是一个 Pod 的稳定的 DNS 名字，并且，这个名字是可以通过 Pod 名字和 Service 名字拼接出来的。

## 从外界连通Service与Service调试“三板斧”
Service 的访问信息在 Kubernetes 集群之外，其实是无效的。

在使用 Kubernetes 的 Service 时，一个必须要面对和解决的问题就是：如何从外部（Kubernetes 集群之外），访问到 Kubernetes 里创建的 Service？

最常用的一种方式就是：NodePort

Kubernetes 在 1.7 之后支持的一个新特性，叫作 ExternalName

> 在理解了 Kubernetes Service 机制的工作原理之后，很多与 Service 相关的问题，其实都可以通过分析 Service 在宿主机上对应的 iptables 规则（或者 IPVS 配置）得到解决。

##  谈谈Service与Ingress

这种全局的、为了代理不同后端 Service 而设置的负载均衡服务，就是 Kubernetes 里的 Ingress 服务。

Ingress 的功能其实很容易理解：所谓 Ingress，就是 Service 的“Service”。