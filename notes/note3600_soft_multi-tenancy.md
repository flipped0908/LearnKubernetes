# 隔离

在 Kubernetes 里，网络隔离能力的定义，是依靠一种专门的 API 对象来描述的，即：NetworkPolicy。

而 NetworkPolicy 定义的规则，其实就是“白名单”。

在 ingress 字段里，我定义了 from 和 ports，即：允许流入的“白名单”和端口。其中，这个允许流入的“白名单”里，我指定了三种并列的情况，分别是：ipBlock、namespaceSelector 和 podSelector。


这个 NetworkPolicy 对象，指定的隔离规则如下所示：

    该隔离规则只对 default Namespace 下的，携带了 role=db 标签的 Pod 有效。限制的请求类型包括 ingress（流入）和 egress（流出）。

    Kubernetes 会拒绝任何访问被隔离 Pod 的请求，除非这个请求来自于以下“白名单”里的对象，并且访问的是被隔离 Pod 的 6379 端口。这些“白名单”对象包括：

    default Namespace 里的，携带了 role=fronted 标签的 Pod；

    任何 Namespace 里的、携带了 project=myproject 标签的 Pod；

    任何源地址属于 172.17.0.0/16 网段，且不属于 172.17.1.0/24 网段的请求。

    Kubernetes 会拒绝被隔离 Pod 对外发起任何请求，除非请求的目的地址属于 10.0.0.0/24 网段，并且访问的是该网段地址的 5978 端口。



 namespaceSelector 和 podSelector，其实是“与”（AND）的关系。所以说，这个 from 字段只定义了一种情况，只有 Namespace 和 Pod 同时满足条件，这个 NetworkPolicy 才会生效。

 果想要在使用 Flannel 的同时还使用 NetworkPolicy 的话，你就需要再额外安装一个网络插件，比如 Calico 项目，来负责执行 NetworkPolicy。

 那么，这些网络插件，又是如何根据 NetworkPolicy 对 Pod 进行隔离的呢？


可以看到，Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成 NetworkPolicy 对应的 iptable 规则来实现的。


在 CNI 网络插件中，上述需求可以通过设置两组 iptables 规则来实现。

第一组规则，负责“拦截”对被隔离 Pod 的访问请求。


而在经过路由之后，IP 包的去向就分为了两种：

第一种，继续在本机处理；  
第二种，被转发到其他目的地。

 IP 包的第一种去向。这时候，IP 包将继续向上层协议栈流动。在它进入传输层之前，Netfilter 会设置一个名叫 INPUT 的“检查点”。

 数据包在 Linux Netfilter 子系统里完整的流动过程


 也就是网络层的 iptables 链的工作流程。


 第一条 FORWARD 链“拦截”的是一种特殊情况

 而第二条 FORWARD 链“拦截”的则是最普遍的情况，即：容器跨主通信。这



# 总结
在本篇文章中，我主要和你分享了 Kubernetes 对 Pod 进行“隔离”的手段，即：NetworkPolicy。

可以看到，NetworkPolicy 实际上只是宿主机上的一系列 iptables 规则。这跟传统 IaaS 里面的安全组（Security Group）其实是非常类似的。

而基于上述讲述，你就会发现这样一个事实：

Kubernetes 的网络模型以及大多数容器网络实现，其实既不会保证容器之间二层网络的互通，也不会实现容器之间的二层网络隔离。这跟 IaaS 项目管理虚拟机的方式，是完全不同的。

所以说，Kubernetes 从底层的设计和实现上，更倾向于假设你已经有了一套完整的物理基础设施。然后，Kubernetes 负责在此基础上提供一种“弱多租户”（soft multi-tenancy）的能力。

并且，基于上述思路，Kubernetes 将来也不大可能把 Namespace 变成一个具有实质意义的隔离机制，或者把它映射成为“子网”或者“租户”。毕竟你可以看到，NetworkPolicy 对象的描述能力，要比基于 Namespace 的划分丰富得多。

这也是为什么，到目前为止，Kubernetes 项目在云计算生态里的定位，其实是基础设施与 PaaS 之间的中间层。这是非常符合“容器”这个本质上就是进程的抽象粒度的。

当然，随着 Kubernetes 社区以及 CNCF 生态的不断发展，Kubernetes 项目也已经开始逐步下探，“吃”掉了基础设施领域的很多“蛋糕”。这也正是容器生态继续发展的一个必然方向。