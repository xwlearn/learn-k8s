# 搭建一个完整的K8s集群

## 准备工作

1. 满足安装 Docker 项目所需的要求，比如 64 位的 Linux 操作系统、3.10 及以上的内核版本；
2. x86 或者 ARM 架构均可；
3. 机器之间网络互通，这是将来容器之间网络互通的前提；
4. 有外网访问权限，因为需要拉取镜像；
5. 能够访问到gcr.io、quay.io这两个 docker registry，因为有小部分镜像需要在这里拉取；
6. 单机可用资源建议 2 核 CPU、8 GB 内存或以上，再小的话问题也不大，但是能调度的 Pod 数量就比较有限了；
7. 30 GB 或以上的可用磁盘空间，这主要是留给 Docker 镜像和日志文件用的。

## 安装 kubeadm 和 Docker

添加 kubeadm 的源,进行安装

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

$ cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

apt-get update

apt-get install -y docker.io kubeadm
```

## 部署 Kubernetes 的 Master 节点

自定义配置文件kubeadm.yaml:

```yaml
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

执行安装命令:

```bash
kubeadm init --config kubeadm.yaml
```

部署完成后会生成一个命令

```bash
kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry --discovery-token-ca-cert-hash sha256:00eb62a2a6020f94132e3fe1ab721349bbcd3e9b94da9654cfe15f2985ebd711
```

kubeadm 还会提示我们第一次使用 Kubernetes 集群所需要的配置命令

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

使用`kubectl get nodes`可以查看节点状态

```bash
kubectl get nodes
NAME   STATUS   ROLES    AGE    VERSION
jd1    Ready    master   4d4h   v1.17.2
jd2    Ready    <none>   4d3h   v1.17.2
jd3    Ready    <none>   4d3h   v1.17.2
```

使用`kubectl describe node jd1`可以查看节点详情

使用`kubectl get pods -n kube-system`命令可以查看系统pod状态

```bash
kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
coredns-7f9c544f75-hc954      1/1     Running   0          4d4h
coredns-7f9c544f75-nlrkx      1/1     Running   0          4d4h
etcd-jd1                      1/1     Running   0          4d4h
kube-apiserver-jd1            1/1     Running   0          4d4h
kube-controller-manager-jd1   1/1     Running   1          4d4h
kube-flannel-ds-amd64-27b72   1/1     Running   0          4d4h
kube-flannel-ds-amd64-27l7c   1/1     Running   0          4d3h
kube-flannel-ds-amd64-7bg5p   1/1     Running   0          4d3h
kube-proxy-44698              1/1     Running   0          4d3h
kube-proxy-flx2c              1/1     Running   0          4d4h
kube-proxy-kk2nd              1/1     Running   0          4d3h
kube-scheduler-jd1            1/1     Running   1          4d4h
```

### 部署网络插件

在 Kubernetes 项目“一切皆容器”的设计理念指导下，部署网络插件非常简单，只需要执行一句 kubectl apply 指令，以 Weave 为例：

```bash
kubectl apply -f https://git.io/weave-kube-1.6
```