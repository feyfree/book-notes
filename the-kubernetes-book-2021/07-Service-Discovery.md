# Service Discovery

There are two major components to service discovery:

* Registration

* Discovery

## 1. Service registration

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220424095925-service-discovery.png)

A few important things to note about service discovery in Kubernetes:

1. Kubernetes uses its internal DNS as a *service registry*

2. All Kubernetes Services automatically register their details with DNS

```bash
➜  ~ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-78fcd69978-pg8vq   1/1     Running   0          3d14h
coredns-78fcd69978-sn5v6   1/1     Running   0          3d14h

➜  ~ kubectl get deploy -n kube-system -l k8s-app=kube-dns -o wide
NAME      READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                              SELECTOR
coredns   2/2     2            2           3d14h   coredns      k8s.gcr.io/coredns/coredns:v1.8.4   k8s-app=kube-dns

➜  ~ kubectl get svc -n kube-system -l k8s-app=kube-dns -o wide
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE     SELECTOR
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3d14h   k8s-app=kube-dns

➜  ~  kubectl get ep -n kube-system -l k8s-app=kube-dns
NAME       ENDPOINTS                                            AGE
kube-dns   10.1.0.55:53,10.1.0.56:53,10.1.0.55:53 + 3 more...   3d20h
```

### 1.1 服务注册的流程

1. 写一个Service 的声明（Yaml文件）给到API Server
2. 基于一定的准入原则，校验这个服务声明的有效性（鉴权，授权等）
3. 这个服务被分配了一个固定的虚拟IP （ClusterIP）
4. 创建了一个Endpoints 对象， 用于存储基于满足这个服务标签下的健康的Pod
5. Pod 中的网络会处理 发送到ClusterIP 上的流量
6. 服务的名称和IP 会被注册到集群的DNS 上

### 1.2 The Service back-end

Kubernetes 会为每个服务，创建一个 Endpoints / EndpointSlice 对象

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220424103406-service-endpoint.png)

The kubelet agent on every node is watching the API server for new Endpoints/EndpointSlice objects. When it sees one, it creates local networking rules to redirect ClusterIP traffic to Pod IPs. In modern Linux-based Kubernetes clusters, the technology used to create these rules is the Linux IP Virtual Server (**IPVS**). Older versions used **iptables**.

At this point the Service is fully registered and ready to be used:

* Its front-end configuration is registered with DNS
* Its back-end label selector is created
* Its Endpoints object (or EndpointSlice) is created
* Nodes and kube-proxies have created the necessary local routing rules

### 1.3 Summarising service registration

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220424103613-service-registration.png)

You post a new Service resource manifest to the API server where it’s authenticated and authorized. The Service is allocated a ClusterIP and its configuration is persisted to the cluster store. An associated Endpoints or EndpointSlice object is created to hold the list of healthy Pod IPs matching the label selector. The cluster DNS is running as a Kubernetes-native application and watching the API server for new Service objects. It observes it and registers the appropriate DNS A and SRV records. Every node is running a kube-proxy that observes the new objects and creates local IPVS/iptables rules so traffic to the Service’s ClusterIP is routed to the Pods matching the Service’s label selector.

## 2. Service Discovery

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220424105431-service-discovery-demo.png)

服务发现要求

1. 需要了解其他应用的服务名
2. 如何将服务名转化为 IP 地址

### 2.1 如何通过cluster DNS 将应用名转化为IP地址

Kubernetes automatically configures every container so it can find and use the cluster DNS to convert Service names to IPs. It does this by populating every container’s /etc/resolv.conf file with the IP address of cluster DNS Service as well as any search domains that should be appended to unqualified names.

An “unqualified name” is a short name such as ent. Appending a search domain converts it to a fully qualified domain name (**FQDN**) such as ent.default.svc.cluster.local.

The following snippet shows a container that is configured to send DNS queries to the cluster DNS at 10.96.0.10. It also lists three search domains to append to unqualified names.

**进入容器内 查看 /etc/resolv.conf** 

```bash
root@jump:/# cat /etc/resolv.conf 
nameserver 10.96.0.10
search dev.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

$ kubectl get svc -n kube-system -l k8s-app=kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3d16h
```

**应用服务通信**

1. 知道对方的应用名
2. 应用名解析
3. 网络转发

### 2.2 Some network magic

ClusterIPs are on a *“special”* network called the *service network*, and there are no routes to it! This means containers send all ClusterIP traffic to their *default gateway*.

默认网关的作用: 将流量转发到对应的Pod

默认网关一般可以认为是路由器

1. 应用发送数据到默认网关 (Target: 某个ClusterIP)
2. 默认网关有路由表， 会选择将这个ClusterIP 继续转达到某个地方
3. 最后会将这个数据流量转发到对应的Node 上面去

The node doesn’t have a route to the *service network* either, so it sends it to its own default gateway. Doing thiscauses the traffic to be processed by the node’s kernel, which is where the magic happens…

Node 节点上面是没有完整的 路由路径的， 只能通过发送给网关， 由网关来完成数据的转发

**kube-proxy**

每个Kubernetes 节点上面都会运行一个 kube-proxy 的系统服务

At a high-level, kube-proxy is responsible for capturing traffic destined for ClusterIPs and redirecting it to the IP addresses of Pods matching the Service’s label selector.

1. kube-proxy 是一个 Pod-based 的应用，这个应用实现了可以监听 API server 上面的新的服务，新的Endpoints 对象。当发现有新的对象产生的时候， 它会创建本地的IPVS 规则， 告诉node 去拦截 指定了 Service ClusterIP 的流量， 然后转发给 独立的 Pod IPs
2. This means that every time a node’s kernel processes traffic headed for an address on the *service network*, a trap occurs, and the traffic is redirected to the IP of a healthy Pod matching the Service’s label selector

Kubernetes originally used iptables to do this trapping and load-balancing. However, it was replaced by IPVS in Kubernetes 1.11. The is because IPVS is a high-performance kernel-based L4 load-balancer that scales better than iptables and implements better load-balancing.

### 2.3 Summarising service discovery

 ![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220424133611-service-discovery-summary.png)

## 3. Service Discovery and Namespaces

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220424133931-service-discovery-and-namespaces.png)

### 3.1 Demo

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: enterprise
  namespace: dev
  labels:
    app: enterprise
spec:
  selector:
    matchLabels:
      app: enterprise
  replicas: 2
  template:
    metadata:
      labels:
        app: enterprise
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - image: nigelpoulton/k8sbook:text-dev
        name: enterprise-ctr
        ports:
        - containerPort: 8080
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: enterprise
  namespace: prod
  labels:
    app: enterprise
spec:
  selector:
    matchLabels:
      app: enterprise
  replicas: 2
  template:
    metadata:
      labels:
        app: enterprise
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - image: nigelpoulton/k8sbook:text-prod
        name: enterprise-ctr
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: ent
  namespace: dev
spec:
  ports:
  - port: 8080
  selector:
    app: enterprise
---
apiVersion: v1
kind: Service
metadata:
  name: ent
  namespace: prod
spec:
  ports:
  - port: 8080
  selector:
    app: enterprise
---
apiVersion: v1
kind: Pod
metadata:
  name: jump
  namespace: dev
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - image: ubuntu
    name: jump
    tty: true
    stdin: true
```

```bash
➜  ~ kubectl get all -n dev
NAME                              READY   STATUS    RESTARTS   AGE
pod/enterprise-584b544bb6-bv6zv   1/1     Running   0          168m
pod/enterprise-584b544bb6-t2w5d   1/1     Running   0          168m
pod/jump                          1/1     Running   0          168m

NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/ent   ClusterIP   10.98.204.110   <none>        8080/TCP   168m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/enterprise   2/2     2            2           168m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/enterprise-584b544bb6   2         2         2       168m


➜  ~ kubectl get all -n prod
NAME                             READY   STATUS    RESTARTS   AGE
pod/enterprise-8b6fdc8c4-59w8t   1/1     Running   0          168m
pod/enterprise-8b6fdc8c4-mnhgt   1/1     Running   0          168m

NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/ent   ClusterIP   10.97.77.237   <none>        8080/TCP   168m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/enterprise   2/2     2            2           168m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/enterprise-8b6fdc8c4   2         2         2       168m


➜  ~ kubectl exec -it jump --namespace dev -- bash
root@jump:/# apt-get install curl -y
root@jump:/# curl ent:8080
Hello form the DEV Namespace!
Hostname: enterprise-584b544bb6-t2w5d
root@jump:/# curl ent.dev.svc.cluster.local:8080
Hello form the DEV Namespace!
Hostname: enterprise-584b544bb6-t2w5d
root@jump:/# curl ent.prod.svc.cluster.local:8080
Hello form the PROD Namespace!
Hostname: enterprise-8b6fdc8c4-59w8t
```

### 3.2 Trouble Shooting

kube-dns 是 通过deploy 部署的， 所以删掉的话， 也会自动重启的

```bash
➜  ~ kubectl run -it dnsutils --image k8s.gcr.io/e2e-test-images/jessie-dnsutils:1.3
If you don't see a command prompt, try pressing enter.
root@dnsutils:/# nslookup kubernetes
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.96.0.1

root@dnsutils:/# 

```

