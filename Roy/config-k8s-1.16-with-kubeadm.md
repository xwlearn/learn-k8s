## 准备工作

在京东云买了三台云主机，配置如下
| 主机名 | 角色     | 内网ip        | CPU核数 | 内存  | 磁盘   | 操作系统        | 内核       |
|-----|--------|-------------|-------|-----|------|-------------|----------|
| JD1 | master | 10\.0\.0\.3 | 2     | 4GB | 40GB | CentOS 7\.3 | 3\.10\.0 |
| JD2 | node   | 10\.0\.0\.4 | 2     | 4GB | 40GB | CentOS 7\.3 | 3\.10\.0 |
| JD3 | node   | 10\.0\.0\.5 | 2     | 4GB | 40GB | CentOS 7\.3 | 3\.10\.0 |

## 配置kubernetes yum源

京东云自带的yum源无kubernetes，需要添加阿里云的源

    vim /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    gpgcheck=0
    enable=1
    
    yum clean all
    yum makecache

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

配置kubectl开机启动

    systemctl enable kubelet

## 部署 Master节点
