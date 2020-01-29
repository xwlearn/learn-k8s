目标：13|为什么我们需要Pod？


Pod，是Kubernetes项目中最小的API对象，即原子调度单位。那么为什么我们需要Pod？

Docker容器本质：Namespace做隔离，Cgroups做限制，Rootfs做文件系统

容器的本质：进程。Kubernetes对应未来云计算的操作系统。

1. 应用均是以进程组的形式存在的，而容器是单进程模型，并不是说容器中只有一个进程，而是说容器中PID=1的那个进程，即应用进程没有多进程管理能力。通常Linux操作系统中PID=1的哪个进程是init或systemd，该进程能管理多个进程。
2. 因此进程组中每个进程需要分别用一个容器来运行，这样就涉及到资源调度的问题。Pod就是用来解决这个问题的。
3. Pod由多个容器组成，是k8s的最小调度单位，适合把有亲密性关系的容器放在一个Pod里，比如war包与tomcat，这种形式叫sidecar。
4. Pod底层有一个Infra容器，该容器有一个占用极小的镜像，叫Pause，会占用Network Namespace，随后Pod中其他容器会join这个Newwork Namespace。为什么要这样实现呢？这是因为如果想实现多容器共享Network Namespace及Volume,用Docker相关命令也能实现，只不过这样的话就会有一个问题，必然涉及到容器启动顺序问题。那么容器间就不是对等关系，而成了拓扑关系了。Pod就解决了这个问题。
