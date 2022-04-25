# Kubernetes Storage

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220425133651-k8s-storage-plugion.png)

## 1. Concepts

### 1.1 CSI

Modern plugins are be based on the **Container Storage Interface (CSI)** which is an open standard aimed at providing a clean storage interface for container orchestrators such as Kubernetes. If you’re a developer writing storage plugins, the CSI abstracts the internal Kubernetes machinery and lets you develop *out-of-tree*

CSI 相当于定义了存储的接口规范， 然后各个存储厂家可以提供基于这个规范下面的插件， 好处是

1. 不需要和Kubernetes 绑定在一起开源
2. 厂商负责维护和更新自己的插件
3. 屏蔽底层细节

### 1.2 PV & PVC & SC

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220425134631-pv-demo.png)

A Kubernetes cluster is running on AWS and the AWS administrator has created a 25GB EBS volume called “ebs-vol”. The Kubernetes administrator creates a PV called “k8s-vol” that links back to the “ebs-vol” via the ebs.csi.aws.com CSI plugin. While that might sound complicated, it’s not. The PV is simply a way of representing the external storage asset on the Kubernetes cluster. Finally, the Pod uses a PVC to claim access to the PV and start using it.

The three core API resources in the persistent volume subsystem are:

*  Persistent Volumes (PV)

* Persistent Volume Claims (PVC)

*  Storage Classes (SC)

At a high level, **PVs** are how external storage assets are represented in Kubernetes. **PVCs** are like tickets that grant a Pod access to a PV. **SCs** make it all dynamic.

## 2. Details

StorageClasses (SC) let you dynamically create physical back-end storage resources that get automatically mapped to Persistent Volumes (PV) on Kubernetes. You define SCs in YAML files that reference a plugin and tie them to a particular tier of storage on a particular storage back-end. For example, *high-performance AWS SSD* *storage in the AWS Mumbai Region.* The SC needs a name, and you deploy it using kubectl apply. Once deployed, the SC watches the API server for new PVC objects referencing its name. When matching PVCs appear, the SC dynamically creates the required asset on the back-end storage system and maps it to a PV on Kubernetes. Apps can then claim it with a PVC

1. SC 负责动态创建物理存储和 K8S PV 的映射
2. SC 需要定义名称， 并且需要通过kubectl apply
3. 一旦SC 建立， SC 负责从API Server 监听 PVC 对象对 它名字的饮用
4. 一旦匹配上的PVC 创建了， SC 负责动态地在后台存储上面 分配必要的资源， 并且在Kubernetes 上面创建对应的PV与之映射
5. APP 可以用PVC 完成存储

### 2.1 Storage Classes (SC)

The basic workflow for deploying *and using* a StorageClass is as follows:

1. Have a storage back-end (can be cloud or on premises)

2. Create your Kubernetes cluster

3. Install and configure the CSI storage plugin

4. Create one or more StorageClasses on Kubernetes

5. Deploy Pods and PVCs that reference those StorageClasses

关于Storage Classes 需要注意的是

1. StorageClass objects are immutable – this means you can’t modify them after they’re deployed
2. *metadata.name* should be meaningful as it’s how **you** and other objects refer to the class
3. The parameters block is for plugin-specific values, and each plugin is free to support its own set of values. Configuring this section requires knowledge of the storage plugin and associated storage back-end. Each provisioner usually provides documentation.
4. You can configure as many StorageClasses as you need. However, each class can only relate to a single type of storage on a single back-end. For example, if you have a Kubernetes cluster with StorageOS and Portworx storage back-ends, you’ll need at least two StorageClasses

#### **举个例子**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220425140244-pv-pvc-sc-demo.png)

####  volume settings

There are a few other important settings you can configure in a StorageClass. We’ll cover:

*  Access mode

*  Reclaim policy

**Access mode**

Kubernetes supports three access modes for volumes.

* ReadWriteOnce (RWO)

* ReadWriteMany (RWM)

*  ReadOnlyMany (ROM)

ReadWriteOnce defines a PV that can only be bound as R/W by a single PVC. Attempts to bind it from multiple PVCs will fail.

只可以供一个PVC 读写

ReadWriteMany defines a PV that can be bound as R/W by multiple PVCs. This mode is usually only supported by file and object storage such as NFS. Block storage usually only supports RWO.

可以供给多个PVC 读写

ReadOnlyMany defines a PV that can be bound as R/O by multiple PVCs. It’s important to understand that a PV can only be opened in one mode – it’s not possible for a single PV to be bound to a PVC in ROM mode and another PVC in RWM mode.

可以供给多个PVC 读

**Reclaim policy**

A volume’s ReclaimPolicy tells Kubernetes how to deal with a PV when its PVC is released. Two policies currently exist:

* Delete

* Retain

Delete 是一个比较危险的定义， 通过SC创建的PVs 默认是delete， 如果PVC 被释放了， 默认PV 还有相关的数据存储都会被删掉， 需要注意

Retain 会保留这个PV 还有相关的数据， 但是PVC 是不会引用到这个PV了。 这种需要人工去释放掉原先的PV

### 2.2 Persistent Volumes (PV)

- PV is an abstraction for the physical storage device that attached to the cluster. PV is used to manage durable storage which needs actual physical storage.
- PV is independent of the lifecycle of the Pods. It means that data represented by a PV continue to exist as the cluster changes and as Pods are deleted and recreated.
- PV is not Namespaced, it is available to whole cluster. i.e. PV is accessible to all namespaces.
- PV is resource in the cluster that can be provisioned dynamically using Storage Classes, or they can be explicitly created by a cluster administrator.
- Unlike Volumes, the PVs lifecycle is managed by Kubernetes.

**Access Mode**

- ReadWriteOnce(RWO) — volume can be mounted as read-write by a single node.
- ReadOnlyMany(ROX) — volume can be mounted read-only by many nodes.
- ReadWriteMany(RWX) — volume can be mounted as read-write by many nodes.
- ReadWriteOncePod(RWOP) — volume can be mounted as read-write by a single Pod.

### 2.3 Persistent Volume Claims (PVC)

- PVC is binding between a Pod and PV. Pod request the Volume through the PVC.
- PVC is the request to provision persistent storage with a specific type and configuration.
- PVC is similar to a Pod. Pods consume node resources and PVC consume PV resources.
- Kubernetes looks for a PV that meets the criteria defined in the PVC, and if there is one, it matches claim to PV.
- PVCs describe the storage capacity and characteristics a pod requires, and the cluster attempts to match the request and provision the desired persistent volume.
- PVC must be in same namespace as the Pod. For each Pod, a PVC makes a storage consumption request within a namespace.
- Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany or ReadWriteMany).