# 为什么需要pod

Pod，是 Kubernetes 项目中最小的 API 对象

Pod，是 Kubernetes 项目的原子调度单位

$ pstree -g

这条命令的作用，是展示当前系统中正在运行的进程的树状结构

```
gsh@ubuntu:~$ pstree -g
init(1)─┬─ModemManager(785)─┬─{ModemManager}(785)
        │                   └─{ModemManager}(785)
        ├─NetworkManager(806)─┬─dhclient(896)
        │                     ├─dnsmasq(1299)
        │                     ├─{NetworkManager}(806)
        │                     ├─{NetworkManager}(806)
        │                     └─{NetworkManager}(806)
        ├─accounts-daemon(469)─┬─{accounts-daemon}(469)
        │                      └─{accounts-daemon}(469)
        ├─acpid(1088)
        ├─avahi-daemon(558)───avahi-daemon(558)
        ├─bluetoothd(517)
        ├─colord(469)─┬─{colord}(469)
        │             └─{colord}(469)
        ├─cron(1078)
        ├─cups-browsed(616)
        ├─cupsd(2713)───dbus(2716)
        ├─dbus-daemon(469)
        ├─getty(1005)
        ├─getty(1011)
        ├─getty(1018)
        ├─getty(1019)
        ├─getty(1022)
        ├─getty(1228)
        ├─gnome-keyring-d(1782)─┬─{gnome-keyring-d}(1782)
        │                       ├─{gnome-keyring-d}(1782)
        │                       ├─{gnome-keyring-d}(1782)
        │                       ├─{gnome-keyring-d}(1782)
        │                       └─{gnome-keyring-d}(1782)
        ├─kerneloops(1142)
        ├─lightdm(1113)─┬─Xorg(1135)
```

Linux 操作系统只需要将信号，比如，SIGKILL 信号，发送给一个进程组，那么该进程组中的所有进程就都会收到这个信号而终止运行。

Kubernetes 项目所做的，其实就是将“进程组”的概念映射到了容器技术中，并使其成为了这个云计算“操作系统”里的“一等公民”。

>再次强调一下：容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。可是，用户编写的应用，并不能够像正常操作系统里的 init 进程或者 systemd 那样拥有进程管理的功能。比如，你的应用是一个 Java Web 程序（PID=1），然后你执行 docker exec 在后台启动了一个 Nginx 进程（PID=3）。可是，当这个 Nginx 进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？

## 一个成组调度的问题
假设我们的 Kubernetes 集群上有两个节点：node-1 上有 3 GB 可用内存，node-2 有 2.5 GB 可用内存。

这时，假设我要用 Docker Swarm 来运行这个 rsyslogd 程序。为了能够让这三个容器都运行在同一台机器上，我就必须在另外两个容器上设置一个 affinity=main（与 main 容器有亲密性）的约束，即：它们俩必须和 main 容器运行在同一台机器上。

然后，我顺序执行：“docker run main”“docker run imklog”和“docker run imuxsock”，创建这三个容器。

这样，这三个容器都会进入 Swarm 的待调度队列。然后，main 容器和 imklog 容器都先后出队并被调度到了 node-2 上（这个情况是完全有可能的）。

可是，当 imuxsock 容器出队开始被调度时，Swarm 就有点懵了：node-2 上的可用资源只有 0.5 GB 了，并不足以运行 imuxsock 容器；可是，根据 affinity=main 的约束，imuxsock 容器又只能运行在 node-2 上。

这就是一个典型的成组调度（gang scheduling）没有被妥善处理的例子。

## pod

到了 Kubernetes 项目里，这样的问题就迎刃而解了：Pod 是 Kubernetes 里的原子调度单位。这就意味着，Kubernetes 项目的调度器，是统一按照 Pod 而非容器的资源需求进行计算的。

像 imklog、imuxsock 和 main 函数主进程这样的三个容器，正是一个典型的由三个容器组成的 Pod。Kubernetes 项目在调度时，自然就会去选择可用内存等于 3 GB 的 node-1 节点进行绑定，而根本不会考虑 node-2。

像这样容器间的紧密协作，我们可以称为“超亲密关系”。

>而如果 Pod 的设计只是出于调度上的考虑，那么 Kubernetes 项目似乎完全没有必要非得把 Pod 作为“一等公民”吧？这不是故意增加用户的学习门槛吗？

## Pod 在 Kubernetes 项目里还有更重要的意义，那就是：容器设计模式。

Pod 的实现原理。

### 首先，关于 Pod 最重要的一个事实是：它只是一个逻辑概念。

Pod，其实是一组共享了某些资源的容器。

Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。


那这么来看的话，一个有 A、B 两个容器的 Pod，不就是等同于一个容器（容器 A）共享另外一个容器（容器 B）的网络和 Volume 的玩儿法么？

你有没有考虑过，如果真这样做的话，容器 B 就必须比容器 A 先启动，这样一个 Pod 里的多个容器就不是对等关系，而是拓扑关系了。
> 拓扑 是有依赖的意思么

#### 在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器

Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：k8s.gcr.io/pause。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有 100~200 KB 左右。

它们可以直接使用 localhost 进行通信；
它们看到的网络设备跟 Infra 容器看到的完全一样；
一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；
当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；
Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关。

> 因为将来如果你要为 Kubernetes 开发一个网络插件时，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。

有了这个设计之后，共享 Volume 就简单多了：Kubernetes 项目只要把所有 Volume 的定义都设计在 Pod 层级即可

# 明白了 Pod 的实现原理后，我们再来讨论“容器设计模式”

Pod 这种“超亲密关系”容器的设计思想，实际上就是希望，当用户想在一个容器里跑多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个 Pod 里的多个容器。

你就应该尽量尝试使用它来描述一些用单个容器难以解决的问题。

## 第一个最典型的例子是：WAR 包与 Web 服务器。

假如，你现在只能用 Docker 来做这件事情，那该如何处理这个组合关系呢？

一种方法是，把 WAR 包直接放在 Tomcat 镜像的 webapps 目录下，做成一个新的镜像运行起来。可是，这时候，如果你要更新 WAR 包的内容，或者要升级 Tomcat 镜像，就要重新制作一个新的发布镜像，非常麻烦。

另一种方法是，你压根儿不管 WAR 包，永远只发布一个 Tomcat 容器。不过，这个容器的 webapps 目录，就必须声明一个 hostPath 类型的 Volume，从而把宿主机上的 WAR 包挂载进 Tomcat 容器当中运行起来。不过，这样你就必须要解决一个问题，即：如何让每一台宿主机，都预先准备好这个存储有 WAR 包的目录呢？这样来看，你只能独立维护一套分布式存储系统了。


实际上，有了 Pod 之后，这样的问题就很容易解决了。我们可以把 WAR 包和 Tomcat 分别做成镜像，然后把它们作为一个 Pod 里的两个容器“组合”在一起。这个 Pod 的配置文件如下所示：
> war 包也可以做镜像？？

```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```
你可能已经注意到，WAR 包容器的类型不再是一个普通容器，而是一个 Init Container 类型的容器。

在 Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。



##### sidecar
像这样，我们就用一种“组合”方式，解决了 WAR 包与 Tomcat 容器之间耦合关系的问题。

实际上，这个所谓的“组合”操作，正是容器设计模式里最常用的一种模式，它的名字叫：sidecar。

顾名思义，sidecar 指的就是我们可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。

## 第二个例子，则是容器的日志收集。

但不要忘记，Pod 的另一个重要特性是，它的所有容器都共享同一个 Network Namespace。这就使得很多与 Pod 网络相关的配置和管理，也都可以交给 sidecar 完成，而完全无须干涉用户容器。这里最典型的例子莫过于 Istio 这个微服务治理项目了。



# 容器设计模

Kubernetes 社区曾经把“容器设计模式”这个理论，整理成了一篇小论文，
https://www.usenix.org/conference/hotcloud16/workshop-program/presentation/burns


## 总结

Pod 是 Kubernetes 项目与其他单容器项目相比最大的不同


事实上，直到现在，仍有很多人把容器跟虚拟机相提并论，他们把容器当做性能更好的虚拟机，喜欢讨论如何把应用从虚拟机无缝地迁移到容器中。

但实际上，无论是从具体的实现原理，还是从使用方法、特性、功能等方面，容器与虚拟机几乎没有任何相似的地方；也不存在一种普遍的方法，能够把虚拟机里的应用无缝迁移到容器中。因为，容器的性能优势，必然伴随着相应缺陷，即：它不能像虚拟机那样，完全模拟本地物理机环境中的部署方法。

最终还是要靠深入理解容器的本质，即：进程。


这也是当初 Swarm 项目无法成长起来的重要原因之一：一旦到了真正的生产环境上，Swarm 这种单容器的工作方式，就难以描述真实世界里复杂的应用架构了。


所以下一次，当你需要把一个运行在虚拟机里的应用迁移到 Docker 容器中时，一定要仔细分析到底有哪些进程（组件）运行在这个虚拟机里。

然后，你就可以把整个虚拟机想象成为一个 Pod，把这些进程分别做成容器镜像，把有顺序关系的容器，定义为 Init Container。这才是更加合理的、松耦合的容器编排诀窍，也是从传统应用架构，到“微服务架构”最自然的过渡方式。

注意：Pod 这个概念，提供的是一种编排思想，而不是具体的技术方案。所以，如果愿意的话，你完全可以使用虚拟机来作为 Pod 的实现，然后把用户容器都运行在这个虚拟机里。比如，Mirantis 公司的virtlet 项目就在干这个事情。甚至，你可以去实现一个带有 Init 进程的容器项目，来模拟传统应用的运行方式。这些工作，在 Kubernetes 中都是非常轻松的，也是我们后面讲解 CRI 时会提到的内容。


如果强行把整个应用塞到一个容器里，甚至不惜使用 Docker In Docker 这种在生产环境中后患无穷的解决方案，恐怕最后往往会得不偿失。

