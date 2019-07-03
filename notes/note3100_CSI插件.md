
# CSI 插件

DigitalOcean 是业界知名的“最简”公有云服务，即：它只提供虚拟机、存储、网络等为数不多的几个基础功能，其他功能一概不管。而这，恰恰就使得 DigitalOcean 成了我们在公有云上实践 Kubernetes 的最佳选择。

# 让我们运行在 DigitalOcean 上的 Kubernetes 集群能够使用它的块存储服务，作为容器的持久化存储。

## Kubernetes 又是如何知道一个 CSI 插件的名字的呢？

这就需要从 CSI 插件的第一个服务 CSI Identity 说起了。

CSI Identity 服务中，最重要的接口是 GetPluginInfo，它返回的就是这个插件的名字和版本号，

另外一个GetPluginCapabilities 接口也很重要。这个接口返回的是这个 CSI 插件的“能力”。

CSI Identity 服务还提供了一个 Probe 接口。Kubernetes 会调用它来检查这个 CSI 插件是否正常工作。

## 编写 CSI 插件的第二个服务，即 CSI Controller 服务了

“Provision 阶段”对应的接口，是 CreateVolume 和 DeleteVolume，它们的调用者是 External Provisoner

Attach 阶段”对应的接口是 ControllerPublishVolume 和 ControllerUnpublishVolume

# 编写 CSI Node 服务

CSI Node 服务对应的，是 Volume 管理流程里的“Mount 阶段”。它的代码实现，在 node.go 文件里。

kubelet 的 VolumeManagerReconciler 控制循环会直接调用 CSI Node 服务来完成 Volume 的“Mount 阶段”。

这个“Mount 阶段”的处理其实被细分成了 NodeStageVolume 和 NodePublishVolume 这两个接口

 kubelet 的 VolumeManagerReconciler 控制循环中，这两步操作分别叫作MountDevice 和 SetUp。

 

# 部署 CSI 插件的常用原则是：

第一，通过 DaemonSet 在每个节点上都启动一个 CSI 插件，来为 kubelet 提供 CSI Node 服务。

第二，通过 StatefulSet 在任意一个节点上再启动一个 CSI 插件，为 External Components 提供 CSI Controller 服务。