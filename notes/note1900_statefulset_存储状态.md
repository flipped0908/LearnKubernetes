# 存储状态
## Persistent Volume Claim 的功能。
持久化存储项目（比如 Ceph、GlusterFS 等）

所谓“术业有专攻”，这些关于 Volume 的管理和远程持久化存储的知识，不仅超越了开发者的知识储备，还会有暴露公司基础设施秘密的风险。

一个声明了 Ceph RBD 类型 Volume 的 Pod：
```
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: kubernetes/pause
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '10.16.154.78:6789'
        - '10.16.154.82:6789'
        - '10.16.154.83:6789'
        pool: kube
        image: foo
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

，Kubernetes 项目引入了一组叫作 Persistent Volume Claim（PVC）和 Persistent Volume（PV）的 API 对象，大大降低了用户声明和使用持久化 Volume 的门槛。


举个例子，有了 PVC 之后，一个开发人员想要使用一个 Volume，只需要简单的两步即可。  
第一步：定义一个 PVC，声明想要的 Volume 的属性：  
第二步：在应用的 Pod 中，声明使用这个 PVC：

Kubernetes 中 PVC 和 PV 的设计，实际上类似于“接口”和“实现”的思想。

种解耦，就避免了因为向开发者暴露过多的存储系统细节而带来的隐患。

PVC、PV 的设计，也使得 StatefulSet 对存储状态的管理成为了可能

PVC 其实就是一种特殊的 Volume。



如果你使用 kubectl delete 命令删除这两个 Pod，这些 Volume 里的文件会不会丢失呢？

不会

这是怎么做到的呢？

分析一下 StatefulSet 控制器恢复这个 Pod 的过程 

    首先，当你把一个 Pod，比如 web-0，删除之后，这个 Pod 对应的 PVC 和 PV，并不会被删除，

    StatefulSet 控制器发现，一个名叫 web-0 的 Pod 消失了。所以，控制器就会重新创建一个新的、名字还是叫作 web-0 的 Pod 来，“纠正”这个不一致的情况

    在这个新的 web-0 Pod 被创建出来之后，Kubernetes 为它查找名叫 www-web-0 的 PVC 时，就会直接找到旧 Pod 遗留下来的同名的 PVC，进而找到跟这个 PVC 绑定在一起的 PV。

通过这种方式，Kubernetes 的 StatefulSet 就实现了对应用存储状态的管理。




## StatefulSet 的工作原理呢

首先，StatefulSet 的控制器直接管理的是 Pod

其次，Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录。

最后，StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 PVC。


# 总结

我们不难看出 StatefulSet 的设计思想：StatefulSet 其实就是一种特殊的 Deployment，而其独特之处在于，它的每个 Pod 都被编号了。而且，这个编号会体现在 Pod 的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识（即：在整个集群里唯一的、可被的访问身份）。

有了这个编号后，StatefulSet 就使用 Kubernetes 里的两个标准功能：Headless Service 和 PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护。
