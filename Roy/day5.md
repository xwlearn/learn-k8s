课程：14|深入解析Pod对象（一）：基本概念、15|深入解析Pod对象（二）：使用进阶

## 总结

why？
资源调度及容器设计模式
what？
一组容器组合。对应传统部署系统的“虚拟机”，所有与调度、网络、存储、安全等相关的属性都是Pod级别的。描述的是“机器”这个整体，而非里面运行的“程序”。
how?
查看pod：kubectl get pods
详细查看：kubectl describe pod xxx
日志查看：kubectl logs xxx
创建一个pod: kubectl create/apply -f xxx.yaml
更新一个pod：kubectl replace -f xxx.yaml
删除一个pod：kubectl delete xxx



### 几个重要字段

NodeSelector：用来筛选该Pod在哪些Pod上运行，筛选依据是labels
NodeName：有值的话，表示该Pod被调度过，但是可以修改，来欺骗调度器，这种“欺骗”主要是用来做调试。
HostAliases：定义Pod的hosts文件，容器中/etc/hosts一定要用这种方式来定义。
ImagePullPolicy：镜像拉取策略，默认值是Always
Lifecycle：定义了一系列钩子，不同的生命周期触发不同的动作。
Status：Pod的状态
- Pending
- Running
- Succeeded
- Failed
- Unknown

### Project volume
就是在容器启动之前就已经提供给容器的数据，好处是动态的，数据源变，容器中对应的数据也会跟着变

- Secret：加密数据，比如应用要读取的数据库的账号密码信息
- ConfigMap：除了不加密外，与Secret没有什么区别，相当于配置中心的作用
- ServiceAccountToken：一种特殊的Secret。Pod中假如想装一个客户端直接与Api server通信，那么就需要解决权限认证问题，这个ServiceAccount的作用就是如此，k8s会有一个默认的token。
- Downward API：可以获取Pod对象本身的一些信息

### 健康检查与故障恢复

健康检查探针
url、端口、文件等方式

Pod状态判断原则：
- 只要Pod的restartPolicy指定的策略是Always，那么Pod就会保持Running，并进行容器重启，否则，Pod就会进入Failed状态
- 对于多个容器的Pod，只有所有的容器都启动失败才会进入Failed，否则就会进入Running状态，并告知已成功启动多少个容器，还有多少个待启动

### PodPreset
pod的字段太多，没谁能都记住，有没有一种方式，可以自动填充Pod中字段呢，于是有了PodPreset。
设置失败，明天检查原因。貌似是初始化时未开启这个特性。