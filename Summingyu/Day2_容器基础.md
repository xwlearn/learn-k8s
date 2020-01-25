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
