# Kubernetes Preview

## 1. What's Kuberntes

Kubernetes was created by Google based on lessons learned running containers at scale for a lot of years. It was
donated to the community as an open-source project and is now the industry standard API for deploying and
managing cloud-native applications. It runs on any cloud or on-premises datacenter and abstracts the underlying
infrastructure. This allows you to build hybrid clouds, as well as migrate on and off the cloud and between
different clouds. It’s open-sourced under the Apache 2.0 license and lives within the Cloud Native Computing
Foundation (CNCF).



## 2. Concepts

kuberntes 相关概念， 从机器分工的角度可以分为 master node，worker node；node 上面运行着 pod， pod 内部运行着 container；

从 container 到 pod， 再到 node， kubernetes 实际上做的这么一件事情： 创建应用需要的资源， 并使之运行。



### 2.1 Control plane 

![Controller Plane](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220415140909.png  "")

control plane 通常我们称为 Master 节点， 上面是不运行应用的， control plane 实际上是负责整个集群的管理。

#### The API Server

The API server is the Grand Central of Kubernetes. All communication, between all components, must go through the API server. We’ll get into the detail later, but it’s important to understand that internal system components, as well as external user components, all communicate via the API server – all roads lead to the API Server.

It exposes a RESTful API that you POST YAML configuration files to over HTTPS. These YAML files, which we sometimes call manifests, describe the desired state of an application. This desired state includes things like which container image to use, which ports to expose, and how many Pod replicas to run. All requests to the API server are subject to authentication and authorization checks. Once these are done, the config in the YAML file is validated, persisted to the cluster store, and work is scheduled to the cluster.

### The cluster store

一般是etcd。 负责存储配置和集群的状态

#### The controller manager and controllers

Architecturally, it’s a controller of controllers, meaning it spawns all the independent controllers and monitors
them.

#### The scheduler

负责 Pod 的调度

The scheduler isn’t responsible for running tasks, just picking the nodes to run them. A task is normally a Pod/container.

### 2.2 Worker node

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220415141457.png)

#### kubelet

One of the main jobs of the kubelet is to watch the API server for new work tasks. Any time it sees one, it executes the task and maintains a reporting channel back to the control plane。

专注于自身node， 负责执行master 发送过来的任务

#### Container runtime

可以想象一下docker的一些概念， 拉镜像， 创建容器， 关闭容器

#### kube-proxy

This runs on every node and is responsible for local cluster networking. It ensures each node gets its own unique IP address, and it implements local iptables or IPVS rules to handle routing and load-balancing of traffic on the Pod network

### 2.3 Kubernetes DNS

The cluster’s DNS service has a static IP address that is hard-coded into every Pod on the cluster. This ensures every container and Pod can locate it and use it for discovery. Service registration is also automatic. This means apps don’t need to be coded with the intelligence to register with Kubernetes service discovery.

Cluster DNS is based on the open-source CoreDNS project (https://coredns.io/).

### 2.4 Pods

The point is, a Kubernetes Pod is a construct for running one or more containers. 

1. Pods themselves don’t actually run applications – applications always run in containers, the Pod is just a sandbox to run one or more containers. Keeping it high level, Pods ring-fence an area of the host OS, build a network stack, create a bunch of kernel namespaces, and run one or more containers.

   ![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220415142418.png)

2. Pod 是作为 Scaling 的单元

3. containers 全部启动， Pod 才算启动成功  

### 2.5 Deployments 

最常用的控制器， 类比其他的还有DaemonSets 还有 StatefulSets

1. self-healing
2. scaling
3. zero-downtime rollouts
4. versioned rollbacks

### 2.6 Services

![Services](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220418085250.png)

Services bring stable IP addresses and DNS names to the unstable world of Pods