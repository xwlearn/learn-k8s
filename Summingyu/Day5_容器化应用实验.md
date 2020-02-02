# 容器化应用实验

使用 Kubernetes 的必备技能：编写配置文件。

> 这些配置文件可以是 YAML 或者 JSON 格式的。  
> 这么做最直接的好处是，你会有一个文件能记录下 Kubernetes 到底“run”了什么。  
> **当应用本身发生变化时，开发人员和运维人员可以依靠容器镜像来进行同步；当应用部署参数发生变化时，这些 YAML 文件就是他们相互沟通和信任的媒介。**

```bash
kubectl create -f 我的配置文件
```

> Pod 就是 Kubernetes 世界里的“应用”；而一个应用，可以由多个容器组成。

```yaml
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

可以使用`kubectl get pods`查看pod

```bash
# kubectl get pods -l app=nginx
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-f6495cd87-rpw44   1/1     Running   0          4d4h
nginx-deployment-f6495cd87-wpx25   1/1     Running   0          4d4h
```

> 在命令行中，所有 key-value 格式的参数，都使用“=”而非“:”表示。

可以使用 `kubectl describe` 命令，查看一个 API 对象的细节

```bash
# kubectl describe pod nginx-deployment-f6495cd87-rpw44
Name:         nginx-deployment-f6495cd87-rpw44
Namespace:    default
Priority:     0
Node:         jd2/10.0.0.4
Start Time:   Wed, 29 Jan 2020 18:08:58 +0800
Labels:       app=nginx
              pod-template-hash=f6495cd87
Annotations:  <none>
Status:       Running
IP:           10.244.1.5
IPs:
  IP:           10.244.1.5
Controlled By:  ReplicaSet/nginx-deployment-f6495cd87
Containers:
  nginx:
    Container ID:   docker://e786970bf2e07936d4351f670d798118034a4f75ac53355a88112092b2c6f44c
    Image:          nginx:1.8
    Image ID:       docker-pullable://docker.io/nginx@sha256:c97ee70c4048fe79765f7c2ec0931957c2898f47400128f4f3640d0ae5d60d10
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 29 Jan 2020 18:08:59 +0800
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
Events:          <none>
```

> 如果有异常发生，你一定要第一时间查看这些 Events，往往可以看到非常详细的错误信息。

`kubectl replace`  指令可以进行更新容器

```bash
 kubectl replace -f nginx-deployment.yaml
```

> 推荐是用`kubectl apply`命令来实现容器的创建和更新。  
> 这样的操作方法，是 Kubernetes“声明式 API”所推荐的使用方法。  
> 也就是说，作为用户，你不必关心当前的操作是创建，还是更新，你执行的命令始终是 kubectl apply，而 Kubernetes 则会根据 YAML 文件的内容变化，自动进行具体的处理。

```bash
kubectl apply -f nginx-deployment.yaml

# 修改nginx-deployment.yaml的内容

kubectl apply -f nginx-deployment.yaml
```

想要从 Kubernetes 集群中删除这个 Nginx Deployment 的话，直接执行

```bash
kubectl delete -f nginx-deployment.yaml
```
