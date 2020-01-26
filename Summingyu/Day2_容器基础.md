# 容器基础

## 从进程说起

进程的静态表现就是磁盘上的可执行程序,动态表现则是运行时计算机里的数据和状态的总和  
**容器**技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”。

### 启动一个docker容器

```bash
$ docker run -it busybox /bin/sh
/ #
# 命令翻译:请帮我启动一个容器，在容器里执行 /bin/sh，并且给我分配一个命令行终端跟这个容器交互
```

在docker启动的环境中中可以看到`/bin/sh`的PID是1,这是通过Linux的NameSpace机制实现的

```bash

/ # ps
PID  USER   TIME COMMAND
  1 root   0:00 /bin/sh
  10 root   0:00 ps
```

除了我们刚刚用到的 PID Namespace，Linux 操作系统还提供了 Mount、UTS、IPC、Network 和 User 这些 Namespace，用来对各种不同的进程上下文进行“障眼法”操作。

**所以说，容器，其实是一种特殊的进程而已。**

## 隔离与限制

在容器中运行的程序其实是运行在真实的OS上,只是被容器给欺骗了
> 想起了<楚门的世界>

“敏捷”和“高性能”是容器相较于虚拟机最大的优势，也是它能够在 PaaS 这种更细粒度的资源管理平台上大行其道的重要原因。

### Namespace 的弊端

Linux Namespace的弊端就是**隔离的不彻底**

首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。

> 如果你要在 Windows 宿主机上运行 Linux 容器，或者在低版本的 Linux 宿主机上运行高版本的 Linux 容器，都是行不通的。

其次，在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：时间。

### 限制(Cgroups)

Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。

```bash
# mount -t 命令:显示指定类型的文件系统
$ mount -t cgroup
cpuset on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cpu on /sys/fs/cgroup/cpu type cgroup (rw,nosuid,nodev,noexec,relatime,cpu)
cpuacct on /sys/fs/cgroup/cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct)
blkio on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
memory on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
...
```

```bash
$ echo 20000 > /sys/fs/cgroup/cpu/container/cpu.cfs_quota_us
# 限制container里面设定的pid只能占用cpu 20ms/100ms

$ echo 226 > /sys/fs/cgroup/cpu/container/tasks
# 写入需要限制的pid到tasks文件中
```

除 CPU 子系统外，Cgroups 的每一项子系统都有其独有的资源限制能力，比如：

- blkio，为​​​块​​​设​​​备​​​设​​​定​​​I/O 限​​​制，一般用于磁盘等设备；
- cpuset，为进程分配单独的 CPU 核和对应的内存节点；
- memory，为进程设定内存使用的限制。

```bash
$ docker run -it --cpu-period=100000 --cpu-quota=20000 ubuntu /bin/bash
# 加入对应参数限制启动docker容器命令
```

## 深入理解容器镜像

可以使用`chroot`命令改变根目录'/'的位置

> Mount Namespace 正是基于对 chroot 的不断改良才被发明出来的，它也是 Linux 操作系统里的第一个 Namespace。
> 为了能够让容器的这个根目录看起来更“真实”，我们一般会在这个容器的根目录下挂载一个完整操作系统的文件系统，比如 Ubuntu16.04 的 ISO。
> **而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。**

### docker核心原理

对 Docker 项目来说，它最核心的原理实际上就是为待创建的用户进程：

1. 启用 Linux Namespace 配置；
2. 设置指定的 Cgroups 参数；
3. 切换进程的根目录（Change Root）。

需要明确的是，rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。

### docker灵魂--系统内核
