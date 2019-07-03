## 搭建一个完整的Kubernetes集群

实践的目标：
```
在所有节点上安装 Docker 和 kubeadm；

部署 Kubernetes Master；

部署容器网络插件；

部署 Kubernetes Worker；

部署 Dashboard 可视化插件；

部署容器存储插件。
```


## 安装 kubeadm 和 Docker

apt-get install -y docker.io kubeadm

## 部署 Kubernetes 的 Master 节点

给 kubeadm 用的 YAML 文件（名叫：kubeadm.yaml）：
```
apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
controllerManagerExtraArgs:
  horizontal-pod-autoscaler-use-rest-clients: "true"
  horizontal-pod-autoscaler-sync-period: "10s"
  node-monitor-grace-period: "10s"
apiServerExtraArgs:
  runtime-config: "api/all=true"
kubernetesVersion: "stable-1.11"
```

$ kubeadm init --config kubeadm.yaml

生成
kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry --discovery-token-ca-cert-hash sha256:00eb62a2a6020f94132e3fe1ab721349bbcd3e9b94da9654cfe15f2985ebd711

kubeadm 还会提示我们第一次使用 Kubernetes 集群所需要的配置命令：
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

查看节点
kubectl get 命令来查看当前唯一一个节点的状态了：
```
$ kubectl get nodes
 
NAME      STATUS     ROLES     AGE       VERSION
master    NotReady   master    1d        v1.11.1
```


kubectl describe 来查看这个节点（Node）对象的详细信息、状态和事件（Event），我们来试一下
```
$ kubectl describe node master
...
Conditions:
...
 
Ready   False ... KubeletNotReady  runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

 kubectl 检查这个节点上各个系统 Pod 的状态
 ```
 $ kubectl get pods -n kube-system
 
NAME               READY   STATUS   RESTARTS  AGE
coredns-78fcdf6894-j9s52     0/1    Pending  0     1h
coredns-78fcdf6894-jm4wf     0/1    Pending  0     1h
etcd-master           1/1    Running  0     2s
kube-apiserver-master      1/1    Running  0     1s
kube-controller-manager-master  0/1    Pending  0     1s
kube-proxy-xbd47         1/1    NodeLost  0     1h
kube-scheduler-master      1/1    Running  0     1s
 ```

## 部署网络插件

 $ kubectl apply -f https://git.io/weave-kube-1.6


## 部署 Kubernetes 的 Worker 节点
第一步，在所有 Worker 节点上执行“安装 kubeadm 和 Docker”一节的所有步骤。

第二步，执行部署 Master 节点时生成的 kubeadm join 指令：
```
$ kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry --discovery-token-ca-cert-hash sha256:00eb62a2a6020f94132e3fe1ab721349bbcd3e9b94da9654cfe15f2985ebd711

```

## 通过 Taint/Toleration 调整 Master 执行 Pod 的策略

为节点打上“污点”（Taint）的命令是：
$ kubectl taint nodes node1 foo=bar:NoSchedule

## 部署 Dashboard 可视化插件

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

## 部署容器存储插件
容器持久化存储。

如果你在某一台机器上启动的一个容器，显然无法看到其他机器上的容器在它们的数据卷里写入的文件。这是容器最典型的特征之一：无状态。

而容器的持久化存储，就是用来保存容器存储状态的重要手段：存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。这样，无论你在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。这就是“持久化”的含义。

由于 Kubernetes 本身的松耦合设计，绝大多数存储项目，比如 Ceph、GlusterFS、NFS 等，都可以为 Kubernetes 提供持久化存储能力。在这次的部署实战中，我会选择部署一个很重要的 Kubernetes 存储插件项目：Rook。

Rook 项目是一个基于 Ceph 的 Kubernetes 存储插件（它后期也在加入对更多存储实现的支持）。不过，不同于对 Ceph 的简单封装，Rook 在自己的实现中加入了水平扩展、迁移、灾难备份、监控等大量的企业级功能，使得这个项目变成了一个完整的、生产级别可用的容器存储插件。

得益于容器化技术，用两条指令，Rook 就可以把复杂的 Ceph 存储后端部署起来：

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml

>备注：其实，在很多时候，大家说的所谓“云原生”，就是“Kubernetes 原生”的意思。而像 Rook、Istio 这样的项目，正是贯彻这个思路的典范。在我们后面讲解了声明式 API 之后，相信你对这些项目的设计思想会有更深刻的体会。

## thinking
基于 Kubernetes 开展工作时，你一定要优先考虑这两个问题：

我的工作是不是可以容器化？

我的工作是不是可以借助 Kubernetes API 和可扩展机制来完成？