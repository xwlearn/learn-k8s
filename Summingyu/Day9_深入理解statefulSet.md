# 深入理解statefulSet

## 拓扑状态

Deployment只能针对简单的应用场景,认为一个应用的所有Pod，是完全一样的。所以，它们互相之间没有顺序，也无所谓运行在哪台宿主机上。  **“无状态应用”**（`Stateless Application`）
但是在实际应用场景中,是存在不对等关系的,例如"主从关系","主备关系"等。  
这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为 **“有状态应用”**（`Stateful Application`）。

基于**控制器设计模式**,k8s在Deployment基础上扩展出了对有状态应用支持的`statefulSet`编排模式

1. **拓扑状态**: 应用的多个实例之间不是完全对等的关系。
2. **存储状态**: 应用的多个实例分别绑定了不同的存储数据。

**StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。**

### Headless Service

Service 是 Kubernetes 项目中用来将一组 Pod 暴露给外界访问的一种机制。

1. **Service的VIP方式:** 请求访问Service的VIP,然后代理到Service后端对应的pod中
2. **Service的DNS方式:** 当访问`my-svc.my-namespace.svc.cluster.local`

`kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh`
