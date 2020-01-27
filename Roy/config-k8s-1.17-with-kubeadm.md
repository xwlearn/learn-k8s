Table of Contents
=================

* [准备工作](#准备工作)
* [配置kubernetes yum源](#配置kubernetes-yum源)
* [安装 Docker](#安装-docker)
* [安装 kubeadm](#安装-kubeadm)
* [部署 Master节点](#部署-master节点)
     * [方法一：命令行](#方法一命令行)
     * [方法二：配置文件(推荐，本次也采用该方式)](#方法二配置文件推荐本次也采用该方式)
* [安装Pod Network](#安装pod-network)
* [添加worker节点至集群](#添加worker节点至集群)
* [拷贝admin.conf到worker节点](#拷贝adminconf到worker节点)

## 准备工作

在京东云买了三台云主机，配置如下


| 主机名 | 角色   | 内网ip      | CPU核数 | 内存 | 磁盘 | 操作系统    | 内核     |
| ------ | ------ | ----------- | ------- | ---- | ---- | ----------- | -------- |
| JD1    | master | 10\.0\.0\.3 | 2       | 4GB  | 40GB | CentOS 7\.3 | 3\.10\.0 |
| JD2    | worker   | 10\.0\.0\.4 | 2       | 4GB  | 40GB | CentOS 7\.3 | 3\.10\.0 |
| JD3    | worker   | 10\.0\.0\.5 | 2       | 4GB  | 40GB | CentOS 7\.3 | 3\.10\.0 |

## 配置kubernetes yum源

京东云自带的yum源无kubernetes，需要添加阿里云的源

    vim /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    gpgcheck=0
    enable=1
    
    yum clean all && yum makecache

测试

    [root@JD1 yum.repos.d]# yum list | grep kubeadm
    kubeadm.x86_64                            1.17.2-0                       kubernetes

## 安装 Docker

三台机器上都需要安装

    [root@JD1 ~]# yum install docker -y
    [root@JD1 ~]# systemctl start docker
    [root@JD1 ~]# systemctl enable docker
    Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.
    [root@JD1 yum.repos.d]# docker version
    Client:
     Version:         1.13.1
     API version:     1.26
     Package version: docker-1.13.1-103.git7f2769b.el7.centos.x86_64
     Go version:      go1.10.3
     Git commit:      7f2769b/1.13.1
     Built:           Sun Sep 15 14:06:47 2019
     OS/Arch:         linux/amd64
    
    Server:
     Version:         1.13.1
     API version:     1.26 (minimum version 1.12)
     Package version: docker-1.13.1-103.git7f2769b.el7.centos.x86_64
     Go version:      go1.10.3
     Git commit:      7f2769b/1.13.1
     Built:           Sun Sep 15 14:06:47 2019
     OS/Arch:         linux/amd64
     Experimental:    false

## 安装 kubeadm
三台机器上都需要安装

    yum install kubeadm -y
    ......
    Installed:
      kubeadm.x86_64 0:1.17.2-0
    
    Dependency Installed:
      conntrack-tools.x86_64 0:1.4.4-5.el7_7.2          cri-tools.x86_64 0:1.13.0-0
      kubectl.x86_64 0:1.17.2-0                         kubelet.x86_64 0:1.17.2-0
      kubernetes-cni.x86_64 0:0.7.5-0                   libnetfilter_cthelper.x86_64 0:1.0.0-10.el7_7.1
      libnetfilter_cttimeout.x86_64 0:1.0.0-6.el7_7.1   libnetfilter_queue.x86_64 0:1.0.2-2.el7_2
      socat.x86_64 0:1.7.3.2-2.el7

kubelet、kubectl、kubenetes-cni也跟着一起安装好了

配置kubelet开机启动

    systemctl enable kubelet

## 部署 Master节点
以jd1作为master节点，另外两台为worker节点
### 方法一：命令行

```
kubeadm init --kubernetes-version=v1.17.2  \
    --pod-network-cidr=10.244.0.0/16  \
    --service-cidr=10.96.0.0/12  \
    --apiserver-advertise-address=10.0.0.3
```
### 方法二：配置文件(推荐，本次也采用该方式)
使用kubeadm配置文件，由于本次下载的kubeadm版本过高，安装低版本k8s集群时报错，索性就安装最新版本的k8s了。

    # 生成配置文件
    kubeadm config print init-defaults ClusterConfiguration >kubeadm.yaml

修改默认镜像仓库，由于大家都懂得的原因，谷歌默认容器镜像地址`k8s.gcr.io`无法访问，修改为`registry.cn-hangzhou.aliyuncs.com/google_containers`

    vim kubeadm.yaml
    #修改 imageRepository: k8s.gcr.io
    #改为 imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
    #修改 advertiseAddress: 1.2.3.4
    #改为 advertiseAddress: 10.0.0.3

最终版本
```
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.0.3
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: jd1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.17.2
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

查看kubeadm config所需的镜像，更多[kubeadm config](https://v1-16.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-config/)命令

    [root@JD1 ~]# kubeadm config images list --config kubeadm.yaml
    W0126 20:54:01.849570    9619 validation.go:28] Cannot validate kube-proxy config - no validator is available
    W0126 20:54:01.849607    9619 validation.go:28] Cannot validate kubelet config - no validator is available
    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.2
    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.2
    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.2
    registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2
    registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
    registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
    registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5

提前下载好这些镜像

    [root@JD1 ~]# kubeadm config images pull --config kubeadm.yaml
    W0126 19:44:46.987395   27714 validation.go:28] Cannot validate kube-proxy config - no validator is available
    W0126 19:44:46.987429   27714 validation.go:28] Cannot validate kubelet config - no validator is available
    [config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.17.2
    [config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.17.2
    [config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.17.2
    [config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.17.2
    [config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1
    [config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.3-0
    [config/images] Pulled registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.6.5

初始化master节点

    [root@JD1 ~]# kubeadm init --config kubeadm.yaml
    W0126 20:57:30.795523   10460 validation.go:28] Cannot validate kube-proxy config - no validator is available
    W0126 20:57:30.795570   10460 validation.go:28] Cannot validate kubelet config - no validator is available
    [init] Using Kubernetes version: v1.17.2
    [preflight] Running pre-flight checks
    [preflight] Pulling images required for setting up a Kubernetes cluster
    [preflight] This might take a minute or two, depending on the speed of your internet connection
    [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Starting the kubelet
    [certs] Using certificateDir folder "/etc/kubernetes/pki"
    [certs] Generating "ca" certificate and key
    [certs] Generating "apiserver" certificate and key
    [certs] apiserver serving cert is signed for DNS names [jd1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.0.0.3]
    [certs] Generating "apiserver-kubelet-client" certificate and key
    [certs] Generating "front-proxy-ca" certificate and key
    [certs] Generating "front-proxy-client" certificate and key
    [certs] Generating "etcd/ca" certificate and key
    [certs] Generating "etcd/server" certificate and key
    [certs] etcd/server serving cert is signed for DNS names [jd1 localhost] and IPs [10.0.0.3 127.0.0.1 ::1]
    [certs] Generating "etcd/peer" certificate and key
    [certs] etcd/peer serving cert is signed for DNS names [jd1 localhost] and IPs [10.0.0.3 127.0.0.1 ::1]
    [certs] Generating "etcd/healthcheck-client" certificate and key
    [certs] Generating "apiserver-etcd-client" certificate and key
    [certs] Generating "sa" key and public key
    [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
    [kubeconfig] Writing "admin.conf" kubeconfig file
    [kubeconfig] Writing "kubelet.conf" kubeconfig file
    [kubeconfig] Writing "controller-manager.conf" kubeconfig file
    [kubeconfig] Writing "scheduler.conf" kubeconfig file
    [control-plane] Using manifest folder "/etc/kubernetes/manifests"
    [control-plane] Creating static Pod manifest for "kube-apiserver"
    [control-plane] Creating static Pod manifest for "kube-controller-manager"
    W0126 20:57:34.329698   10460 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
    [control-plane] Creating static Pod manifest for "kube-scheduler"
    W0126 20:57:34.330430   10460 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
    [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
    [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
    [apiclient] All control plane components are healthy after 15.001757 seconds
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config-1.17" in namespace kube-system with the configuration for the kubelets in the cluster
    [upload-certs] Skipping phase. Please see --upload-certs
    [mark-control-plane] Marking the node jd1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
    [mark-control-plane] Marking the node jd1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
    [bootstrap-token] Using token: abcdef.0123456789abcdef
    [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy
    
    Your Kubernetes control-plane has initialized successfully!
    
    To start using your cluster, you need to run the following as a regular user:
    
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/
    
    Then you can join any number of worker nodes by running the following on each as root:
    
    kubeadm join 10.0.0.3:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:2957de566c5bcf9bee9fb2211d1bf0d9cb85eeefb2c5eed35443390728e45957

配置常规用户

    [root@JD1 ~]# mkdir -p $HOME/.kube
    [root@JD1 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    [root@JD1 ~]# chown $(id -u):$(id -g) $HOME/.kube/config

查看集群状态

    [root@JD1 ~]# kubectl get cs
    NAME                 STATUS    MESSAGE             ERROR
    scheduler            Healthy   ok
    controller-manager   Healthy   ok
    etcd-0               Healthy   {"health":"true"}

## 安装Pod Network

接下来安装flannel network add-on

    [root@JD1 ~]# mkdir -p ~/k8s/
    [root@JD1 ~]# cd ~/k8s
    [root@JD1 k8s]# curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                     Dload  Upload   Total   Spent    Left  Speed
    100 14416  100 14416    0     0   6772      0  0:00:02  0:00:02 --:--:--  6774
    [root@JD1 k8s]# ls
    kube-flannel.yml
    [root@JD1 k8s]# kubectl apply -f  kube-flannel.yml
    podsecuritypolicy.policy/psp.flannel.unprivileged created
    clusterrole.rbac.authorization.k8s.io/flannel created
    clusterrolebinding.rbac.authorization.k8s.io/flannel created
    serviceaccount/flannel created
    configmap/kube-flannel-cfg created
    daemonset.apps/kube-flannel-ds-amd64 created
    daemonset.apps/kube-flannel-ds-arm64 created
    daemonset.apps/kube-flannel-ds-arm created
    daemonset.apps/kube-flannel-ds-ppc64le created
    daemonset.apps/kube-flannel-ds-s390x created

## 添加worker节点至集群

在另外两台机器上执行如下命令即可

    [root@JD3 ~]# kubeadm join 10.0.0.3:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:2957de566c5bcf9bee9fb2211d1bf0d9cb85eeefb2c5eed35443390728e45957
    
    W0126 21:15:58.144981    6559 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
    [preflight] Running pre-flight checks
    [preflight] Reading configuration from the cluster...
    [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
    [kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Starting the kubelet
    [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
    
    This node has joined the cluster:
    * Certificate signing request was sent to apiserver and a response was received.
    * The Kubelet was informed of the new secure connection details.
    
    Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

查看节点信息

在master节点查看

    [root@JD1 ~]# kubectl get nodes
    NAME   STATUS   ROLES    AGE     VERSION
    jd1    Ready    master   19m     v1.17.2
    jd2    Ready    <none>   9m55s   v1.17.2
    jd3    Ready    <none>   105s    v1.17.2

查看集群状态信息

    [root@JD1 ~]# kubectl cluster-info
    Kubernetes master is running at https://10.0.0.3:6443
    KubeDNS is running at https://10.0.0.3:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    
    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

查看k8s集群server端与client端的版本信息

    [root@JD1 ~]# kubectl version --short=true
    Client Version: v1.17.2
    Server Version: v1.17.2

## 拷贝admin.conf到worker节点

worker节点运行kubectl命令报错

    The connection to the server localhost:8080 was refused - did you specify the right host or port?

kubectl命令需要使用kubernetes-admin来运行，将主节点中的/etc/kubernetes/admin.conf文件拷贝到worker节点相同目录下

    [root@JD2 ~]# scp root@jd1:/etc/kubernetes/admin.conf /etc/kubernetes/
    

然后执行

    [root@JD2 ~]# mkdir -p $HOME/.kube
    [root@JD2 ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    [root@JD2 ~]# chown $(id -u):$(id -g) $HOME/.kube/config
jd3主机也作同样操作，再次执行kubectl命令查看
```
# 检查nodes
[root@JD2 ~]# kubectl get nodes
NAME   STATUS   ROLES    AGE   VERSION
jd1    Ready    master   63m   v1.17.2
jd2    Ready    <none>   52m   v1.17.2
jd3    Ready    <none>   52m   v1.17.2

# 检查pods
[root@JD2 ~]# kubectl get pods -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS   AGE
kube-system            coredns-7f9c544f75-hc954                     1/1     Running   0          64m
kube-system            coredns-7f9c544f75-nlrkx                     1/1     Running   0          64m
kube-system            etcd-jd1                                     1/1     Running   0          64m
kube-system            kube-apiserver-jd1                           1/1     Running   0          64m
kube-system            kube-controller-manager-jd1                  1/1     Running   0          64m
kube-system            kube-flannel-ds-amd64-27b72                  1/1     Running   0          57m
kube-system            kube-flannel-ds-amd64-27l7c                  1/1     Running   0          53m
kube-system            kube-flannel-ds-amd64-7bg5p                  1/1     Running   0          53m
kube-system            kube-proxy-44698                             1/1     Running   0          53m
kube-system            kube-proxy-flx2c                             1/1     Running   0          64m
kube-system            kube-proxy-kk2nd                             1/1     Running   0          53m
kube-system            kube-scheduler-jd1                           1/1     Running   0          64m
kubernetes-dashboard   dashboard-metrics-scraper-7b64584c5c-jwmhv   1/1     Running   0          47m
kubernetes-dashboard   kubernetes-dashboard-566f567dc7-ktdpf        1/1     Running   0          47m
```
