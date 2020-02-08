# Day6 为什么我们需要Pod

Pod，是 Kubernetes 项目中最小的 API 对象。可以这样描述：Pod，是 Kubernetes 项目的原子调度单位。

> Docker容器本质:`Namespace 做隔离，Cgroups 做限制，rootfs 做文件系统`

按照容器为维度调度应用组时,会存在**成组调度**问题,导致存在"紧密协作"的容器组资源分配不足

> “超亲密关系”容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace（比如，一个容器要加入另一个容器的 Network Namespace）等等。

## Pod实现原理

Pod 在 Kubernetes 项目里还有更重要的意义，那就是：**容器设计模式**。为了理解这一层含义，必须了解一下Pod 的实现原理。  
**首先,关于 Pod 最重要的一个事实是：它只是一个逻辑概念**

Pod，其实是一组共享了某些资源的容器  
Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。

在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 `Infra` 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。

> Infra容器镜像为`k8s.gcr.io/pause`

**将来如果你要为 Kubernetes 开发一个网络插件时，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。**

在 Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

## Pod 容器设计模式

用组合的方式,解决容器间的耦合关系(sidecar)

## 总结

Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。

> pod是一个小家庭，它把密不可分的家庭成员(container)聚在一起，Infra container则是家长，掌管家中共通资源，家庭成员通过sidecar方式互帮互助，其乐融融～ -- Jeff.W评论
