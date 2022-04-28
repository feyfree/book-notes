# Kubernetes Services

[服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/)

## 1. Services Theory

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422105352-services.png)

With a Service in place, the Pods can scale up and down, they can fail, and they can be updated and rolled back. And clients will continue to access them without interruption. This is because the Service is observing the changes and updating its list of healthy Pods. **But it never changes its stable IP, DNS, and port**

### 1.1 Labels and loose coupling

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422105649-services-labels.png)

Selector 中 使用的Label 是 And 关系去匹配

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422105810-services-label-unmatch.png)

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422105855-services-labels-match2.png)

### 1.2 Services and Endpoint objects

Services 实际是通过Endpoint 对象去实现 存储 Services 对应Label 下面的健康的 Pod 的列表

每有一个健康的，属于这个Service 的标签下的Pod， K8S都会添加进入这个Service 对应的Endpoint 对象里面， 反之亦然

流量传递的逻辑 （Sending Traffic to Pods via a Service）

1. 集群的 DNS  通过Service Name 解析为IP地址
2. 发送流量到 这个IP 地址
3. 将流量分配给Endpoint Pod 列表下面的某个容器

**Note:** Recent versions of Kubernetes are replacing *Endpoints* objects with more efficient *Endpoint*

*slices*. The functionality is identical, but *Endpoint slices* are higher performance and more efficient.

```bash
➜  deployments git:(master) ✗ kubectl api-resources | grep End
endpoints                         ep           v1                                     true         Endpoints
endpointslices                                 discovery.k8s.io/v1                    true         EndpointSlice
```

### 1.3 Accessing Services from inside the cluster

A *ClusterIP* Service has a stable virtual IP address that is **only accessible from inside the cluster**. We call this a “ClusterIP”. It’s programmed into the network fabric and guaranteed to be stable for the life of the Service. *Programmed into the network fabric* is fancy way of saying the network *just knows about it* and you don’t need to bother with the details.

Anyway, the ClusterIP is registered against the name of the Service in the cluster’s internal DNS service. All Pods in the cluster are pre-programmed to use the cluster’s DNS service, meaning all Pods can convert Service names to ClusterIPs.

### 1.4 Accessing Services from outside the cluster

Kubernetes has two types of Service for requests originating from outside the cluster.

* NodePort

* LoadBalancer

**NodePort** 相当于在 Node 节点上开个端口， 然后从外部访问这个Node: 端口的话， 就会相当于访问到了内部Pod 对应的映射端口。 

**LoadBalancer Services** make external access even easier by integrating with an internet-facing load-balancer on your underlying cloud platform

### 1.5 Service Discovery

Kubernetes clusters run an internal DNS service that is the centre of service discovery. Service names are automatically registered with the cluster DNS, and every Pod and container is pre-configured to use the cluster DNS. This means every Pod/container can resolve every Service name to a ClusterIP and connect to the Pods behind it.

## 2. Service 实践

### 2.1 NodePort

**deploy.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: svc-test
spec:
  replicas: 10
  selector:
    matchLabels:
      chapter: services
  template:
    metadata:
      labels:
        chapter: services
    spec:
      containers:
      - name: hello-ctr
        image: nigelpoulton/k8sbook:1.0
        ports:
        - containerPort: 8080
```

**svc.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-test
  labels:
    chapter: services
spec:
# ipFamilyPolicy: PreferDualStack
# ipFamilies:
# - IPv4
# - IPv6
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
    targetPort: 8080
    protocol: TCP
  selector:
    chapter: services

```

**一些操作**

```bash
➜  services git:(master) ✗ kubectl apply -f deploy.yml 
deployment.apps/svc-test created
➜  services git:(master) ✗ kubectl apply -f svc.yml 
service/svc-test created
➜  services git:(master) ✗ kubectl get endpoints
NAME         ENDPOINTS                                                     AGE
kubernetes   192.168.65.4:6443                                             40h
svc-test     10.1.0.106:8080,10.1.0.107:8080,10.1.0.108:8080 + 7 more...   31s
➜  services git:(master) ✗ kubectl get endpointslices
NAME             ADDRESSTYPE   PORTS   ENDPOINTS                                      AGE
kubernetes       IPv4          6443    192.168.65.4                                   40h
svc-test-z7shj   IPv4          8080    10.1.0.108,10.1.0.109,10.1.0.106 + 7 more...   61s

➜  services git:(master) ✗ kubectl describe endpointslice svc-test-z7shj
Name:         svc-test-z7shj
Namespace:    default
Labels:       chapter=services
              endpointslice.kubernetes.io/managed-by=endpointslice-controller.k8s.io
              kubernetes.io/service-name=svc-test
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2022-04-22T03:46:13Z
AddressType:  IPv4
Ports:
  Name     Port  Protocol
  ----     ----  --------
  <unset>  8080  TCP
Endpoints:
  - Addresses:  10.1.0.108
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-w52ph
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.109
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-wcp8p
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.106
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-w2hq9
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.110
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-kmgh7
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.113
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-fc9nv
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.115
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-895zn
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.114
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-lpwn4
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.111
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-5vws8
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.107
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-j7x8x
    NodeName:   docker-desktop
    Zone:       <unset>
  - Addresses:  10.1.0.112
    Conditions:
      Ready:    true
    Hostname:   <unset>
    TargetRef:  Pod/svc-test-84db6ff656-s42pj
    NodeName:   docker-desktop
    Zone:       <unset>
Events:         <none>
```

### 2.2 LoadBalancer

**lb.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: cloud-lb
spec:
  type: LoadBalancer
  ports:
  - port: 9000
    targetPort: 8080
  selector:
    chapter: services
```

**一些操作**

```bash
➜  services git:(master) ✗ kubectl apply -f lb.yml 
service/cloud-lb created
➜  services git:(master) ✗ kubectl get svc --watch
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
cloud-lb     LoadBalancer   10.107.133.43    localhost     9000:31667/TCP   11s
kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP          40h
svc-test     NodePort       10.110.144.154   <none>        8080:30001/TCP   5m40s
```

**注意**

1. 因为我使用的是docker desktop 挂载的k8s,  因为External-IP 显示是localhost， 相当于是外网IP， 所以可以通过localhost 访问到这个SVC，所以从外部的Browser 中访问的话 使用 localhost:9000 就可以访问到
2. 为啥 localhost: 31667 无法访问， 因为这个相当于是NodePort 所以需要 是 NodeIP:NodePort 才能访问到。



### 2.3 清理操作

`kubectl delete -f deploy.yml -f svc.yml -f lb.yml`