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

We’ve already said that StatefulSets are for applications that need Pods to be predictable and long-lived. As a result, other parts of the application as well as other applications may need to connect directly to individual Pods. To make this possible,StatefulSets use a **headless Service** to create predictable DNS hostnames for every Pod replica. Other apps can then query DNS for the full list of Pod replicas and use these details to connect directly to Pods.

* headless services: without an IP address （**spec.clusterIP** set to None）[see headless-services](https://kubernetes.io/zh/docs/concepts/services-networking/service/#headless-services)

有时不需要或不想要负载均衡，以及单独的 Service IP。 遇到这种情况，可以通过指定 Cluster IP（`spec.clusterIP`）的值为 `"None"` 来创建 `Headless` Service。

你可以使用无头 Service 与其他服务发现机制进行接口，而不必与 Kubernetes 的实现捆绑在一起。

对这无头 Service 并不会分配 Cluster IP，kube-proxy 不会处理它们， 而且平台也不会为它们进行负载均衡和路由。 DNS 如何实现自动配置，依赖于 Service 是否定义了选择算符。

**带选择算符的服务**

对定义了选择算符的无头服务，Endpoint 控制器在 API 中创建了 Endpoints 记录， 并且修改 DNS 配置返回 A 记录（IP 地址），通过这个地址直接到达 `Service` 的后端 Pod 上。

**无选择算符的服务** 

对没有定义选择算符的无头服务，Endpoint 控制器不会创建 `Endpoints` 记录。 然而 DNS 系统会查找和配置，无论是：

- 对于 [`ExternalName`](https://kubernetes.io/zh/docs/concepts/services-networking/service/#external-name) 类型的服务，查找其 CNAME 记录
- 对所有其他类型的服务，查找与 Service 名称相同的任何 `Endpoints` 的记录

## 2. Hands-on with StatefulSets

* A Storage Class
* A headless Service
* A StatefulSet

### 2.1 Deploying the StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: flash
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-ssd
```

### 2.2 Creating a governing headless Service

```yaml
# Headless Service for StatefulSet Pod DNS names
apiVersion: v1
kind: Service
metadata:
  name: dullahan
  labels:
    app: web
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: web
```

### 2.3 Deploying the StatefulSets 

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: tkb-sts
spec:
  replicas: 3 
  selector:
    matchLabels:
      app: web
  serviceName: "dullahan"
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: ctr-web
        image: nginx:latest
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: webroot
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: webroot
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "flash"
      resources:
        requests:
          storage: 1Gi
```

When the StatefulSet object is deployed, it will create three Pod replicas and three PVCs.

 **spec.serviceName**

The spec.serviceName field designates the *governing Service*. This is the name of the headless Service created in the previous step and will create the DNS SRV records for each StatefulSet replica. It’s called the *governing* *Service* because it’s in charge of the DNS subdomain used by the StatefulSet.

 **spec.volumeClaimTemplates**

每个Pod 都是遵循同一个模板创建， 但是每个Pod 需要自己的存储， 所以每个Pod 都需要自己的PVC。

你需要对每个潜在的StatefulSet 的Pod预先创建PVC

### 2.4 Testing peer discovery

However, before testing this, it’s worth taking a moment to understand how DNS hostnames and DNS subdomains work with StatefulSets.

By default, Kubernetes places all objects within the cluster.local DNS subdomain. You can choose something different, but most lab environments use this domain, so we’ll assume it in this example. Within that domain, Kubernetes constructs DNS subdomains as follows:

<object-name>.<service-name>.<namespace>.svc.cluster.local

So far, you’ve got three Pods called tkb-sts-0, tkb-sts-1, and tkb-sts-2 governed by the dullahan headless Service.

This means the 3 Pods will have the following fully qualified DNS names:

* tkb-sts-0.dullahan.default.svc.cluster.local

* tkb-sts-1.dullahan.default.svc.cluster.local

* tkb-sts-2.dullahan.default.svc.cluster.local

### 2.5 Scaling 

1. scaling down and deleting Pod replicas does **not** delete associated PVCs
2. Each time a StatefulSet is scaled up, a Pod **and** a PVC is created

**spec.podManagementPolicy**

*  OrderedReady 严格的one by one
*  Parallel 类似Deployment   created and deleted in parallel

### 2.6 Rolling updates

StatefulSets support rolling updates. You update the image version in the YAML and re-post it to the API server. Once authenticated and authorized, the controller replaces old Pods with new ones. However, the process always starts with the highest numbered Pod and works down through the set, one-at-a-time, until all Pods are on the new version. The controller also waits for each new Pod to be running and ready before replacing the one with the next lowest index ordinal.

**spec.updateStrategy**

### 2.7 Pod Failure

You can see the new Pod has the same name as the failed one, but does it have the same PVC

新的Pod 会继承之前失败的Pod 的 PVC

一个被认为是失败了的Pod， 可能会重新恢复， 那这样它就会和它的继承者产生冲突， 可能会造成数据污染， 需要人工的介入

### 2.8 Delete 

At this point, the StatefulSet object is deleted, but the headless Service, StorageClass, and jump-pod still exist. You may want to delete them as well.

仅仅删除StatefulSets 还不够， 可能你还需要删除 headless Service, StorageClass 等等