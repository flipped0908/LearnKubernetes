# pod对象的概念
Pod，而不是容器，才是 Kubernetes 项目中的最小编排单位。将这个设计落实到 API 对象上，容器（Container）就成了 Pod 属性里的一个普通的字段。那么，一个很自然的问题就是：到底哪些属性属于 Pod 对象，而又有哪些属性属于 Container 呢？

比如，凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。

这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”。比如，配置这个“机器”的网卡（即：Pod 的网络定义），配置这个“机器”的磁盘（即：Pod 的存储定义），配置这个“机器”的防火墙（即：Pod 的安全定义）。更不用说，这台“机器”运行在哪个服务器之上（即：Pod 的调度）。

##  NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段

```
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
```

## NodeName
一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。

## HostAliases
HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容，用法如下：

```
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```
凡是跟容器的 Linux Namespace 相关的属性，也一定是 Pod 级别的。

类似地，凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义，比如：

#Container
Kubernetes 项目中对 Container 的定义，和 Docker 相比并没有什么太大区别。我在前面的容器技术概念入门系列文章中，和你分享的 Image（镜像）、Command（启动命令）、workingDir（容器的工作目录）、Ports（容器要开发的端口），以及 volumeMounts（容器要挂载的 Volume）都是构成 Kubernetes 项目中 Container 的主要字段。不过在这里，还有这么几个属性值得你额外关注。

## 首先，是 ImagePullPolicy 字段。
它定义了镜像拉取的策略。而

## 其次，是 Lifecycle 字段。
它定义的是 Container Lifecycle Hooks。顾名思义，Container Lifecycle Hooks 的作用，是在容器状态发生变化时触发一系列“钩子”。


# 一个的 Pod 对象在 Kubernetes 中的生命周期。

Pod 生命周期的变化，主要体现在 Pod API 对象的Status 部分，这是它除了 Metadata 和 Spec 之外的第三个重要字段。其中，pod.status.phase，就是 Pod 的当前状态，它有如下几种可能的情况：

Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。

Running。这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。

Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。

Failed。这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。

Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。


#### 源码
在学习完这篇文章后，我希望你能仔细阅读 $GOPATH/src/k8s.io/kubernetes/vendor/k8s.io/api/core/v1/types.go 里，type Pod struct ，尤其是 PodSpec 部分的内容。争取做到下次看到一个 Pod 的 YAML 文件时，不再需要查阅文档，就能做到把常用字段及其作用信手拈来。

