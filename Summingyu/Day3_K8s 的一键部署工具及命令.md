# Day3 K8s 的一键部署工具及命令

要真正发挥容器技术的实力，你就不能仅仅局限于对 Linux 容器本身的钻研和使用。

## kubeadm介绍

可以使用[kubeadm](https://github.com/kubernetes/kubeadm)工具

```bash
# 创建一个Master节点
kubeadm init

# 将一个Node节点加入到当前集群中
kubeadm join <Master节点的IP和端口>
```

### kubeadm工作原理

kubeadm 的方案是:把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。

安装方法:`apt-get install kubeadm`

### kubeadm init 的工作流程

**kubeadm 首先要做的，是一系列的检查工作，以确定这台机器可以用来部署 Kubernetes。**  
Preflight Checks 包括了很多方面，比如：

- Linux 内核的版本必须是否是 3.10 以上？
- Linux Cgroups 模块是否可用？
- 机器的 hostname 是否标准？
- 在 Kubernetes 项目里，机器的名字以及一切存储在 Etcd 中的 API 对象，都必须使用标准的 DNS 命名（RFC 1123）。
- 用户安装的 kubeadm 和 kubelet 的版本是否匹配？
- 机器上是不是已经安装了 Kubernetes 的二进制文件？
- Kubernetes 的工作端口 10250/10251/10252 端口是不是已经被占用？
- ip、mount 等 Linux 指令是否存在？
- Docker 是否已经安装？
- ……

**在通过了 Preflight Checks 之后，kubeadm 要为你做的，是生成 Kubernetes 对外提供服务所需的各种证书和对应的目录。**  
ubeadm 为 Kubernetes 项目生成的证书文件都放在 Master 节点的 `/etc/kubernetes/pki` 目录下。

**证书生成后，kubeadm 接下来会为其他组件生成访问 kube-apiserver 所需的配置文件。**  

```bash
ls /etc/kubernetes/
admin.conf  controller-manager.conf  kubelet.conf  scheduler.conf
```

**接下来，kubeadm 会为 Master 组件生成 Pod 配置文件。**  
Pod配置文件是yaml格式存放在`/etc/kubernetes/manifests/`目录下

```bash
$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

ubelet 就会自动创建这些 YAML 文件中定义的 Pod，即 Master 组件的容器。  
Master 容器启动后，kubeadm 会通过检查 `localhost:6443/healthz` 这个 Master 组件的健康检查 URL，等待 Master 组件完全运行起来。

**然后，kubeadm 就会为集群生成一个 bootstrap token。**  
只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 kubeadm join 加入到这个集群当中。  

**在 token 生成之后，kubeadm 会将 ca.crt 等 Master 节点的重要信息，通过 ConfigMap 的方式保存在 Etcd 当中，供后续部署 Node 节点使用。**  

**kubeadm init 的最后一步，就是安装默认插件。**  
Kubernetes 默认 kube-proxy 和 DNS 这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和 DNS 功能。

### kubeadm join 的工作流程

任何一台机器想要成为 Kubernetes 集群中的一个节点，就必须在集群的 kube-apiserver 上注册  
要想跟 apiserver 打交道，这台机器就必须要获取到相应的证书文件（CA 文件）。  
为了能够一键安装，我们就不能让用户去 Master 节点上手动拷贝这些文件。所以，kubeadm 至少需要发起一次“不安全模式”的访问到 kube-apiserver，从而拿到保存在 ConfigMap 中的 cluster-info（它保存了 APIServer 的授权信息）。  
**bootstrap token，扮演的就是这个过程中的安全验证的角色。**

### 配置 kubeadm 的部署参数

强烈推荐在使用 kubeadm init 部署 Master 节点时，使用下面这条指令：

```bash
kubeadm init --config kubeadm.yaml
```

通过制定这样一个部署参数配置文件，就可以很方便地在这个文件里填写各种自定义的部署参数了。

```yaml
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
api:
  advertiseAddress: 192.168.0.102
  bindPort: 6443
  ...
etcd:
  local:
    dataDir: /var/lib/etcd
    image: ""
imageRepository: k8s.gcr.io
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    ...
kubeletConfiguration:
  baseConfig:
    address: 0.0.0.0
    ...
networking:
  dnsDomain: cluster.local
  podSubnet: ""
  serviceSubnet: 10.96.0.0/12
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  ...
```
