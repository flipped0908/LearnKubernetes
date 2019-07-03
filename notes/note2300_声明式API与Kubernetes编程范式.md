# 声明式API与Kubernetes编程范式

这个 YAML 文件，正是 Kubernetes 声明式 API 所必须具备的一个要素。不过，是不是只要用 YAML 文件代替了命令行操作，就是声明式 API 了呢？

# 那么，到底什么才是“声明式 API”呢？

答案是，kubectl apply 命令。

实际上，你可以简单地理解为，kubectl replace 的执行过程，是使用新的 YAML 文件中的 API 对象，替换原有的 API 对象；而 kubectl apply，则是执行了一个对原有 API 对象的 PATCH 操作。

更进一步地，这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力。

# 接下来，我就以 Istio 项目为例，来为你讲解一下声明式 API 在实际使用时的重要意义。

 Istio 项目，实际上就是一个基于 Kubernetes 项目的微服务治理框架。它的架构非常清晰，如下所示：

 ![](img/2301.png)

 
 Istio 最根本的组件，是运行在每一个应用 Pod 里的 Envoy 容器。

 这个 Envoy 项目是 Lyft 公司推出的一个高性能 C++ 网络代理，也是 Lyft 公司对 Istio 项目的唯一贡献。

 而 Istio 项目，则把这个代理服务以 sidecar 容器的方式，运行在了每一个被治理的应用 Pod 中。我们知道，Pod 里的所有容器都共享同一个 Network Namespace。所以，Envoy 容器就能够通过配置 Pod 里的 iptables 规则，把整个 Pod 的进出流量接管下来。

 这时候，Istio 的控制层（Control Plane）里的 Pilot 组件，就能够通过调用每个 Envoy 容器的 API，对这个 Envoy 代理进行配置，从而实现微服务治理。

 更重要的是，在整个微服务治理的过程中，无论是对 Envoy 容器的部署，还是像上面这样对 Envoy 代理的配置，用户和应用都是完全“无感”的。

 实际上，Istio 项目使用的，是 Kubernetes 中的一个非常重要的功能，叫作 Dynamic Admission Control。


那么，Istio 又是如何在用户完全不知情的前提下完成这个操作的呢？

Istio 要做的，就是编写一个用来为 Pod“自动注入”Envoy 容器的 Initializer。

    首先，Istio 会将这个 Envoy 容器本身的定义，以 ConfigMap 的方式保存在 Kubernetes 当中。

    接下来，Istio 将一个编写好的 Initializer，作为一个 Pod 部署在 Kubernetes 中

    有了这个 TwoWayMergePatch 之后，Initializer 的代码就可以使用这个 patch 的数据，调用 Kubernetes 的 Client，发起一个 PATCH 请求。

    这也就意味着，当你在 Initializer 里完成了要做的操作后，一定要记得将这个 metadata.initializers.pending 标志清除掉。这一点，你在编写 Initializer 代码的时候一定要非常注意。



Istio 项目的核心，就是由无数个运行在应用 Pod 中的 Envoy 容器组成的服务代理网格。这也正是 Service Mesh 的含义。

        这个 Demo https://github.com/resouer/kubernetes-initializer-tutorial


Kubernetes“声明式 API”的独特之处：

    首先，所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”，我所期望的状态是什么样子。
    其次，“声明式 API”允许有多个 API 写端，以 PATCH 的方式对 API 对象进行修改，而无需关心本地原始 YAML 文件的内容。
    最后，也是最重要的，有了上述两个能力，Kubernetes 项目才可以基于对 API 对象的增、删、改、查，
    在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。


如何使用控制器模式，同 Kubernetes 里 API 对象的“增、删、改、查”进行协作，进而完成用户业务逻辑的编写过程。


# 总结
从“使用 Kubernetes 部署代码”，到“使用 Kubernetes 编写代码”的蜕变过程，正是你从一个 Kubernetes 用户，到 Kubernetes 玩家的晋级之路。

而，如何理解“Kubernetes 编程范式”，如何为 Kubernetes 添加自定义 API 对象，编写自定义控制器，正是这个晋级过程中的关键点，也是我要在后面几篇文章中分享的核心内容。

此外，基于今天这篇文章所讲述的 Istio 的工作原理，尽管 Istio 项目一直宣称它可以运行在非 Kubernetes 环境中，但我并不建议你花太多时间去做这个尝试。