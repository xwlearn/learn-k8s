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

### docker灵魂--一致性

由于 rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。  
对一个应用来说，操作系统本身才是它运行所需要的最完整的“依赖库”。  
这种深入到操作系统级别的运行环境一致性，打通了应用在本地开发和远端执行环境之间难以逾越的鸿沟.  

> Docker 在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。
> 这是使用了Union File System来实现增量rootfs

## 重新认识Docker容器

使用Dockerfile构建镜像
app.py内容如下:

```python
from flask import Flask
import socket
import os

app = Flask(__name__)

@app.route('/')
def hello():
    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>"           
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname())
    
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

requirements.txt 内容如下:

```bash
$ cat requirements.txt
Flask
```

Dockerfile内容:

```Dockerfile
# 使用官方提供的Python开发镜像作为基础镜像
FROM python:2.7-slim

# 将工作目录切换为/app
WORKDIR /app

# 将当前目录下的所有内容复制到/app下
ADD . /app

# 使用pip命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的80端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个Python应用的启动命令
CMD ["python", "app.py"]
```

执行命令`docker build -t helloworld`命令

```bash
[root@JD1 test_docker]# docker build -t helloworld .  
Sending build context to Docker daemon 4.096 kB
Step 1/7 : FROM python:2.7-slim
Trying to pull repository docker.io/library/python ...
2.7-slim: Pulling from docker.io/library/python
8ec398bc0356: Pull complete
1cfa5cc2566c: Pull complete
642c5fbdca36: Pull complete
df3df7d23149: Pull complete
Digest: sha256:8ba6b057f6f93f888cf74e4b7ca57a62cbda108b28ea6c91eab60e387b650027
Status: Downloaded newer image for docker.io/python:2.7-slim
 ---> e2df0a42e2de
Step 2/7 : WORKDIR /app
 ---> 33f29ee4f8ed
Removing intermediate container ab565fb401a8
Step 3/7 : ADD . /app
 ---> 7ec575bf3708
Removing intermediate container ffb8608c3779
Step 4/7 : RUN pip install --trusted-host pypi.python.org -r requirements.txt
 ---> Running in 61d1899776c8

DEPRECATION: Python 2.7 reached the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 is no longer maintained. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Collecting Flask
  Downloading Flask-1.1.1-py2.py3-none-any.whl (94 kB)
Collecting click>=5.1
  Downloading Click-7.0-py2.py3-none-any.whl (81 kB)
Collecting itsdangerous>=0.24
  Downloading itsdangerous-1.1.0-py2.py3-none-any.whl (16 kB)
Collecting Werkzeug>=0.15
  Downloading Werkzeug-0.16.1-py2.py3-none-any.whl (327 kB)
Collecting Jinja2>=2.10.1
  Downloading Jinja2-2.11.0-py2.py3-none-any.whl (126 kB)
Collecting MarkupSafe>=0.23
  Downloading MarkupSafe-1.1.1-cp27-cp27mu-manylinux1_x86_64.whl (24 kB)
Installing collected packages: click, itsdangerous, Werkzeug, MarkupSafe, Jinja2, Flask
Successfully installed Flask-1.1.1 Jinja2-2.11.0 MarkupSafe-1.1.1 Werkzeug-0.16.1 click-7.0 itsdangerous-1.1.0
 ---> 4947fa09fef7
Removing intermediate container 61d1899776c8
Step 5/7 : EXPOSE 80
 ---> Running in 45e972716102
 ---> 035553f4d3a1
Removing intermediate container 45e972716102
Step 6/7 : ENV NAME World
 ---> Running in 0db6453dd852
 ---> e548857ef3e4
Removing intermediate container 0db6453dd852
Step 7/7 : CMD python app.py
 ---> Running in 7c2f0861a5ce
 ---> e650b4a76f93
Removing intermediate container 7c2f0861a5ce
Successfully built e650b4a76f93

```

运行容器:

```bash
[root@JD1 ~]# docker run -p 4000:80 helloworld
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```

访问4000端口

```bash
[root@JD1 ~]# curl http://127.0.0.1:4000
<h3>Hello World!</h3><b>Hostname:</b> d17baeaf1ebf<br/>[root@JD1 ~]#
```
