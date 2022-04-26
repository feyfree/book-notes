# StatefulSets

## 1. The theory of StatefulSets

StatefulSets 和 Deployment 相似， 都是按照Controller 设计。 都支持自愈， 扩容， 更新等

和Deployment 不同的是

* Predictable and persistent Pod names

* Predictable and persistent DNS hostnames

* Predictable and persistent volume bindings

Pod Names + DNS HostNames + Volume Bindings 可以确定一个 Pod的状态， 组合起来可以看成是一个 “持久的ID”， 

在失败， 扩容， 或者其他调度的操作中， 这个ID是一只不变的， 这种设计比较符合那种要求Pod不能替换改变的那种应用。

As a quick example, failed Pods managed by a StatefulSet will be replaced by new Pods with the exact same Pod name, the exact same DNS hostname, and the exact same volumes. This is true even if the replacement Pod is started on a different cluster node. The same is not true of Pods managed by a Deployment.

### 1.1 StatefulSet Pod naming

Be aware that StatefulSets names need to be a valid DNS names, so no exotic characters

格式一般为 <StatefulSetName>-<Integer> 这种格式， integer 是从 0 开始

### 1.2 Ordered creation and deletion

创建的顺序是一个一个创建的

比如 tkb-sts-0 先创建

tkb-sts-1 在  tkb-sts-0  运行成功之后创建

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426221923-statefulsets-replicas.png)

**扩容**

比如扩容到5个， tkb-sts-4 在 3之后创建

**缩容**

从序号高到低依次关闭

### 1.3 Deleting StatefulSets

Firstly, deleting a StatefulSet does **not** terminate Pods in order. With this in mind, you may want to scale a StatefulSet to 0 replicas before deleting it.

不会按序删除，所以删除前最好将副本数量降低为 0

You can also use terminationGracePeriodSeconds to further control the way Pods are terminated. It’s common to set this to at least 10 seconds to give applications running in Pods a chance to flush local buffers and safely commit any writes that are still “in-flight”.

Pod 终止前最好留有 terminationGracePeriodSeconds 供优雅关闭， 这段时间可能会有 缓存的flush，或者说一些数据的写提交

### 1.4 StatefulSets and Volumes

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426224307-statefulsets-and-volumes.png)

Volumes are appropriately decoupled from Pods via the normal Persistent Volume Claim system. This means volumes have separate lifecycles to Pods, allowing them to survive Pod failures and termination operations. For example, any time a StatefulSet Pod fails or is terminated, associated volumes are unaffected. This allows replacement Pods to attach to the same storage as the Pods they’re replacing. This is true, even if replacement Pods are scheduled to different cluster nodes.

The same is true for scaling operations. If a StatefulSet Pod is deleted as part of a scale-down operation, subsequent scale-up operations will attach new Pods to the surviving volumes that match their names. This behavior can be a life-saver if you accidentally delete a StatefulSet Pod, especially if it’s the last replica!

### 1.5 Handling Failures

**Pod failure**

The same is true for scaling operations. If a StatefulSet Pod is deleted as part of a scale-down operation, subsequent scale-up operations will attach new Pods to the surviving volumes that match their names. This behavior can be a life-saver if you accidentally delete a StatefulSet Pod, especially if it’s the last replica!

**Node failure**

可能出现分区异常， 或者是kubelet 进程异常， 或者是node 重启了。 这样重启了的Node上面的Pod 和之前认为失败了， 然后重建的Pod 会产生冲突。 为了避免这种情况的话， 需要人工参与 （避免K8S替换掉失败的Node上面的Pod）

### 1.6 Network ID and headless Services

