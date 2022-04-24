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