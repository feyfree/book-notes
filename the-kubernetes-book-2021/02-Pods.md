# Working with Pods

pod 是 K8S 的调度单元， 容器运行在Pod里面

## 1. Why Pods

我们开发应用并运行在 K8S 中， 一般会经过下面的步骤

1. 编码
2. 打包进镜像
3. 在Pod中运行镜像
4. 运行在K8S 中

### 1.1 Pod加强了容器

On the augmentation front, Pods augment containers in all of the following ways.

* Labels and annotations
* Restart policies
* Probes (startup probes, readiness probes, liveness probes, and potentially more)
* Affinity and anti-affinity rules
* Termination control
* Security policies
* Resource requests and limits

可以通过命令查看与Pod 相关的**配置方法**

```bash
kubectl explain pod --recursive
```

加上--recursive 显示很多

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220420170802-kubectl-explain.png)

查看特定的某个参数的含义

```bash
kubectl explain pod.spec.restartPolicy
```

```bash
➜  ~ kubectl explain pod.spec.restartPolicy
KIND:     Pod
VERSION:  v1

FIELD:    restartPolicy <string>

DESCRIPTION:
     Restart policy for all containers within the pod. One of Always, OnFailure,
     Never. Default to Always. More info:
     https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy
```

### 1.2 Pod 更方便调度

增加了一些针对亲和性的， 反亲和的规则； 增加了资源的限制（在Node上面运行）等等， 方便更好的控制

### 1.3 Pod 实现共享资源

On the sharing of resources front, Pods provide a *shared execution environment* for one or more containers. This shared execution environment includes things such as.

*  Shared filesystem

* Shared network stack (IP address and ports…)

* Shared memory

* Shared volumes

You’ll see it later, but every container in a Pod shares the Pod’s execution environment. So, if a Pod has two containers, both containers share the Pod’s IP address and can access any of the Pod’s volumes to share data



## 2. About Pod

1.  我们可以用Controller来定义Pod的，也可以单独写 Pod 类型的资源定义， 当然更建议通过Controller 来启动Pod。 单独建立的Pod 我们最好只在本地或者测试使用
2. 现实生产环境中的Pod内部一般是运行多个容器的，因为除了基本的应用镜像所属的容器， 还有一些Pod启动基础的容器
3. 部署容器的步骤一般是（如果是Controller启动的话， Pod上面还有一层Controller）
   1. Define it in a YAML *manifest file*
   2. Post the YAML to the API server
   3. The API server authenticates and authorizes the request
   4.  The configuration (YAML) is validated
   5. The scheduler deploys the Pod to a healthy node with enough available resources
   6. The local kubelet monitors it

## 3. Pod inside

Seriously though, a Pod is a collection of resources that containers running inside of it inherit and share. These

resources are actually Linux kernel namespaces, and include the following:

* **net namespace:** IP address, port range, routing table…
* **pid namespace:** isolated process tree
* **mnt namespace:** filesystems and volumes…
* **UTS namespace:** Hostname
* **IPC namespace:** Unix domain sockets and shared memory

### 3.1 Pod network

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220420173328-Inter-Pod-Communication.png)

Pod 与 Pod 之间通过 一个扁平化的网络 互相通信

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220420173540-containers.png)

Pod 里面多个containers 共享一个IP （Pause容器）对外部的 Pod container 进行通信， 通过端口区分

### 3.2 Pod 的部署

1. Pod 部署实际是一个原子操作， 意味着部署要么成功要么失败
2. Pod 状态变化 （Pending， Running 等等）
3. Pod 重启策略 （Always， Failure， Never）
4. Pod 是不可变的， 所以一般是一个新的Pod replace 老的Pod   （kubectl edit pod xxx 会发生什么）

## 4. Multi-container Pod

Kubernetes offers several well-defined multi-container Pod patterns.

* Sidecar pattern

* Adapter pattern

* Ambassador pattern

* Init pattern

### 4.1 Sidecar

Sidecar主张以额外的容器来扩展或增强主容器，而这个额外的容器被称为Sidecar容器 

[什么是Sidecar模式](https://blog.csdn.net/ZYQDuron/article/details/80757232)

Web-server容器可以与一个sidecar容易共同部署，该sidecar容器从文件系统中读取由Web-server容器生成的web-server日志，并将日志/stream发送到原称服务器（remote server）。Sidecar容器通过处理web-server日志来作为web-server容器的补充。当然，可能会有人说，为什么web-server不自己处理自己的日志？答案在于以下几点：

* 隔离（separation of concerns）：让每个容器都能够关注核心问题。比如web-server提供网页服务，而sidecar则处理web-server的日志，这有助于在解决问题时不与其他问题的处理发生冲突；

* 单一责任原则（single responsibility principle）：容器发生变化需要一个明确的理由。换句更容易理解的话来说，每个容器都应该是能够处理好“一件”事情的，而根据这一原则，我们应该让不同的容器分别展开工作，应该让它们处理的问题足够独立；

* 内聚性/可重用性（Cohesiveness/Reusability）：使用sidecar容器处理日志，这个容器按道理应该可以应用的其他地方重用

另外一个例子， 比如需要一个另外的Git 容器去定期去拉 仓库的数据， 实际上这个Git 容器可以是复用的

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220420183108.png)

### 4.2 Adapter

如其名， 实际上是一个便于主应用和外部服务交互的一个适配器， 本质上还是Sidecar 模式

### 4.3 Ambassador 

外交官模式， 实际上还是Sidecar 模式， 想象外交官（驻外使者）是干嘛的， 实际上相当于一个代理，用于与外界交互

### 4.4 Init

init 不是 sidecar， 在一个Pod的启动中， 如果定义了Init 容器的话， 会先启动Init 容器， 如果有多个， 会挨个建立， 相当于为主的应用服务器做一些铺垫的工作， 工作结束后，会自动退出

## 5. Pod Yaml & Commands

一个单个Pod Yaml 文件

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    zone: prod
    version: v1
spec:
  containers:
  - name: hello-ctr
    image: nigelpoulton/k8sbook:1.0
    ports:
    - containerPort: 8080
```

注意hello-pod 实际上是内部作为HostName， 相当于是DNS name

### 5.1 Commands

```bash
# 通过yaml 文件创建Pod
kubectl create -f pod.yml
kubectl apply -f pod.yml

# 查看Pod
kubectl get pod
kubectl get pod -o wide
kubectl get pod -o yaml

# 查看日志
kubectl logs hello-pod

# 如果有多个容器的话， 查看具体某个容器的日志
kubectl logs hello-pod -c {容器名称}

# 查看Pod的明细
kubectl describe pods hello-pod

# 进入pod执行命令
kubectl exec hello-pod -- ps aux

# 进入pod 并且以 交互模式shell, 多容器的话进行 -c 容器名 指定
kubectl exec -it hello-pod -- /bin/sh

# kubectl 修改pod   (kubernetes 会阻止你修改pod name， container port, container name 等)
kubectl edit pod hello-pod

# 删除pod
kubectl delete pod hello-pod

# 删除所有的pod
kubectl delete pod --all

# 删除svc
kubectl delete svc {svc名称}

```

### 5.2 Muti-Containers with init container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: initpod
  labels:
    app: initializer
spec:
  initContainers:
  - name: init-ctr
    image: busybox
    command: ['sh', '-c', 'until nslookup k8sbook; do echo waiting for k8sbook service; sleep 1; done; echo Service found!']
  containers:
    - name: web-ctr
      image: nigelpoulton/web-app:1.0
      ports:
        - containerPort: 8080
```

这个initContainer 里面会 nslookup k8sbook 说明需要依赖一个服务， 如果通过上面的yaml文件创建的话， Pod 是无法正常启动的。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: k8sbook
spec:
  selector:
    app: initializer
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

当这个Service 创建了，前面的initPod 才能正常启动

### 5.3 Muti-Containers with sidecar container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: git-sync
  labels:
    app: sidecar
spec:
  containers:
  - name: ctr-web
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/
  - name: ctr-sync
    image: k8s.gcr.io/git-sync:v3.1.6
    volumeMounts:
    - name: html
      mountPath: /tmp/git
    env:
    - name: GIT_SYNC_REPO
      value: https://github.com/feyfree/ps-sidecar.git
    - name: GIT_SYNC_BRANCH
      value: master
    - name: GIT_SYNC_DEPTH
      value: "1"
    - name: GIT_SYNC_DEST
      value: "html"
  volumes:
  - name: html
    emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: svc-sidecar
spec:
  selector:
    app: sidecar
  ports:
  - port: 80
  type: LoadBalancer
```

