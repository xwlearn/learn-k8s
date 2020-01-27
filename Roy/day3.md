目标：12|牛刀小试：我的第一个容器化应用

总结：
1. 最好不要直接使用命令行的方式来运行容器，最佳方式是使用yaml文件的方式。利用`kubectl apply -f xxx.yaml`来运行容器。这样的好处是有一个文件记录了k8s到底run了什么
2. 一个API对象的yaml文件主要由metadata与spec两部分组成，前者存放的是这个对象的元数据，其字段Labels是提供给k8s过滤对象的，后者存放的是该对象的独有定义，用来描述其功能
3. 一个API对象的创建与更新最好统一使用`kubectl apply -f xxx.yaml`，对于用户而已，不需要关系该操作是创建还是更新
4. 一种API对象管理另一种API对象，这种方式叫“控制器”模式，也正是k8s进行容器编排的重要模式
5. 查看pod的创建细节，可以使用命令：`kubectl describe pod xxx`查看

## 操作笔记

一个配置文件示例
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

使用k8s来运行这个容器

```
[root@JD1 k8s]# kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment created
[root@JD1 k8s]# kubectl get pods -l app=nginx
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-54f57cf6bf-5q5w7   0/1     ContainerCreating   0          21s
nginx-deployment-54f57cf6bf-xw4d8   0/1     ContainerCreating   0          21s
```

再次查看pod状态

```
[root@JD1 ~]# kubectl get pods -l app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-5q5w7   1/1     Running   0          8m43s
nginx-deployment-54f57cf6bf-xw4d8   1/1     Running   0          8m43s
```

利用`kubectl describe`命令查看创建细节

```
[root@JD1 ~]# kubectl describe pod nginx-deployment-54f57cf6bf-5q5w7
Name:         nginx-deployment-54f57cf6bf-5q5w7
Namespace:    default
Priority:     0
Node:         jd3/10.0.0.5
Start Time:   Mon, 27 Jan 2020 21:50:55 +0800
Labels:       app=nginx
              pod-template-hash=54f57cf6bf
Annotations:  <none>
Status:       Running
IP:           10.244.2.3
IPs:
  IP:           10.244.2.3
Controlled By:  ReplicaSet/nginx-deployment-54f57cf6bf
Containers:
  nginx:
    Container ID:   docker://909ae27d1c2ee60b618b929b861afd1b8ad53139577c3e0d6da0c20783c88108
    Image:          nginx:1.7.9
    Image ID:       docker-pullable://docker.io/nginx@sha256:e3456c851a152494c3e4ff5fcc26f240206abac0c9d794affb40e0714846c451
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 27 Jan 2020 21:51:15 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8zw92 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-8zw92:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-8zw92
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10m   default-scheduler  Successfully assigned default/nginx-deployment-54f57cf6bf-5q5w7 to jd3
  Normal  Pulling    10m   kubelet, jd3       Pulling image "nginx:1.7.9"
  Normal  Pulled     10m   kubelet, jd3       Successfully pulled image "nginx:1.7.9"
  Normal  Created    10m   kubelet, jd3       Created container nginx
  Normal  Started    10m   kubelet, jd3       Started container nginx
```

当前nginx版本为1.7.9，假如我想升级为1.8，可以直接修改yaml文件，然后再次执行`kubectl apply -f xxx.yaml`即可

```
[root@JD1 k8s]# kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-deployment configured
```

查看pod状态，会逐步更新替换

```
[root@JD1 k8s]# kubectl get pods -l app=nginx
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-54f57cf6bf-5q5w7   1/1     Running             0          16m
nginx-deployment-54f57cf6bf-xw4d8   1/1     Running             0          16m
nginx-deployment-9f46bb5-q26nk      0/1     ContainerCreating   0          2m27s
```

会发现之前的pod还是Running状态，新版的在创建中，过一会儿再看pod状态

```
[root@JD1 ~]# kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
nginx-deployment-9f46bb5-q26nk   1/1     Running   0          33m
nginx-deployment-9f46bb5-qcwcq   1/1     Running   0          27m
```

已经替换完毕