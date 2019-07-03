# 扩展伸缩

Deployment 看似简单，但实际上，它实现了 Kubernetes 项目中一个非常重要的功能：Pod 的“水平扩展 / 收缩”（horizontal scaling out/in）。这个功能，是从 PaaS 时代开始，一个平台级项目就必须具备的编排能力。

## 滚动更新

Deployment 就需要遵循一种叫作“滚动更新”（rolling update）的方式，来升级现有的容器。

更重要的是，Deployment 控制器实际操纵的，正是这样的 ReplicaSet 对象，而不是 Pod 对象。

对于一个 Deployment 所管理的 Pod，它的 **ownerReferenc**e 是谁？

所以，这个问题的答案就是：ReplicaSet。

> 其中，“水平扩展 / 收缩”非常容易实现，Deployment Controller 只需要修改它所控制的 ReplicaSet 的 Pod 副本个数就可以了。

## “滚动更新”又是什么意思，是如何实现的呢？

然后，我们来检查一下 nginx-deployment 创建后的状态信息：

```
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```
在返回结果中，我们可以看到四个状态字段，它们的含义如下所示。

DESIRED：用户期望的 Pod 副本个数（spec.replicas 的值）；

CURRENT：当前处于 Running 状态的 Pod 的个数；

UP-TO-DATE：当前处于最新版本的 Pod 的个数，所谓最新版本指的是 Pod 的 Spec 部分与 Deployment 里 Pod 模板里定义的完全一致；

AVAILABLE：当前已经可用的 Pod 的个数，即：既是 Running 状态，又是最新版本，并且已经处于 Ready（健康检查正确）状态的 Pod 的个数。

可以看到，只有这个 AVAILABLE 字段，描述的才是用户所期望的最终状态。


而 ReplicaSet 的 DESIRED、CURRENT 和 READY 字段的含义，和 Deployment 中是一致的。所以，相比之下，Deployment 只是在 ReplicaSet 的基础上，添加了 UP-TO-DATE 这个跟版本有关的状态字段。

这个时候，如果我们修改了 Deployment 的 Pod 模板，“滚动更新”就会被自动触发。

kubectl edit 指令编辑完成后，保存退出，Kubernetes 就会立刻触发“滚动更新”的过程。你还可以通过 kubectl rollout status 指令查看 nginx-deployment 的状态变化

将一个集群中正在运行的多个 Pod 版本，交替地逐一升级的过程，就是“滚动更新”。


更进一步地，如果我想回滚到更早之前的版本，要怎么办呢？

首先，我需要使用 kubectl rollout history 命令，查看每次 Deployment 变更对应的版本。

# 总结
在今天这篇文章中，我为你详细讲解了 Deployment 这个 Kubernetes 项目中最基本的编排控制器的实现原理和使用方法。

通过这些讲解，你应该了解到：Deployment 实际上是一个两层控制器。首先，它通过ReplicaSet 的个数来描述应用的版本；然后，它再通过ReplicaSet 的属性（比如 replicas 的值），来保证 Pod 的副本数量。

## 备注：Deployment 控制 ReplicaSet（版本），ReplicaSet 控制 Pod（副本数）。这个两层控制关系一定要牢记。


## 自定义控制器
不过，相信你也能够感受到，Kubernetes 项目对 Deployment 的设计，实际上是代替我们完成了对“应用”的抽象，使得我们可以使用这个 Deployment 对象来描述应用，使用 kubectl rollout 命令控制应用的版本。

可是，在实际使用场景中，应用发布的流程往往千差万别，也可能有很多的定制化需求。比如，我的应用可能有会话黏连（session sticky），这就意味着“滚动更新”的时候，哪个 Pod 能下线，是不能随便选择的。

这种场景，光靠 Deployment 自己就很难应对了。对于这种需求，我在专栏后续文章中重点介绍的“自定义控制器”，就可以帮我们实现一个功能更加强大的 Deployment Controller。

# 金丝雀发布（Canary Deployment）和蓝绿发布（Blue-Green Deployment）？？？？