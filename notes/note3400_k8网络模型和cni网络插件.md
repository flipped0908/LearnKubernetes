
# CNI 网络插件

上面这个流程，也正是 Kubernetes 对容器网络的主要处理方法。只不过，Kubernetes 是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0。这个网桥的名字就叫作：CNI 网桥，它在宿主机上的设备名称默认是：cni0。

Kubernetes 之所以要设置这样一个与 docker0 网桥功能几乎一样的 CNI 网桥，主要原因包括两个方面：

    一方面，Kubernetes 项目并没有使用 Docker 的网络模型（CNM），所以它并不希望、也不具备配置 docker0 网桥的能力；  

    另一方面，这还与 Kubernetes 如何配置 Pod，也就是 Infra 容器的 Network Namespace 密切相关。


CNI 的设计思想，就是：Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈。

我们在部署 Kubernetes 的时候，有一个步骤是安装 kubernetes-cni 包，它的目的就是在宿主机上安装CNI 插件所需的基础可执行文件。

 
 在宿主机的 /opt/cni/bin 

 CNI 的基础可执行文件，按照功能可以分为三类：

    第一类，叫作 Main 插件，它是用来创建具体网络设备的二进制文件。

    第二类，叫作 IPAM（IP Address Management）插件，它是负责分配 IP 地址的二进制文件

    第三类，是由 CNI 社区维护的内置 CNI 插件。

 如果要实现一个给 Kubernetes 用的容器网络方案，其实需要做两部分工作

    首先，实现这个网络方案本身。这一部分需要编写的，其实就是 flanneld 进程里的主要逻辑。

    然后，实现该网络方案对应的 CNI 插件。


## 一个 CNI 插件的工作原理

当 kubelet 组件需要创建 Pod 的时候，它第一个创建的一定是 Infra 容器。所以在这一步，dockershim 就会先调用 Docker API 创建并启动 Infra 容器，紧接着执行一个叫作 SetUpPod 的方法。这个方法的作用就是：为 CNI 插件准备参数，然后调用 CNI 插件为 Infra 容器配置网络。

第一部分，是由 dockershim 设置的一组 CNI 环境变量。

这个 ADD 和 DEL 操作，就是 CNI 插件唯一需要实现的两个方法。

第二部分，则是 dockershim 从 CNI 配置文件里加载到的、默认插件的配置信息。


# 总结 

在本篇文章中，我为你详细讲解了 Kubernetes 中 CNI 网络的实现原理。根据这个原理，你其实就很容易理解所谓的“Kubernetes 网络模型”了：

所有容器都可以直接使用 IP 地址与其他容器通信，而无需使用 NAT。

所有宿主机都可以直接使用 IP 地址与所有容器通信，而无需使用 NAT。反之亦然。

容器自己“看到”的自己的 IP 地址，和别人（宿主机或者容器）看到的地址是完全一样的。

可以看到，这个网络模型，其实可以用一个字总结，那就是“通”。

容器与容器之间要“通”，容器与宿主机之间也要“通”。并且，Kubernetes 要求这个“通”，还必须是直接基于容器和宿主机的 IP 地址来进行的。

