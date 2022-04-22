# Ingress 

## 1. NodePort 和 LoadBalancer 的问题

NodePorts only work on high port numbers (30000-32767) and require knowledge of node names or IPs. LoadBalancer Services fix this, but require a 1-to-1 mapping between an internal Service and a cloud load balancer. This means a cluster with 25 internet-facing apps will need 25 cloud load balancers, and cloud loadbalancers aren’t cheap. They may also be a finite resource – you may be limited to how many cloud load-balancers you can provision.

NodePort 的问题：

1. 外部访问的话， 需要知道node name (DNS 能识别出来这个)， 或者是IP

LoadBalancer 的问题

1. 解决了 需要node name/IP 的问题， 但是每个内部服务都要创建一个LoadBalancer ， 数量上来，开销比较大

## 2. Ingress 架构

**Architecture**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/202204221423050-ingress-loadbalancer.png)

**Host Based Routing**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422144036-ingress-host-based.png)

**总结**

Ingress 的协同机制：

1. 部署一个ClusterIP Service的LoadBalancer 
2. 用户部署一个Ingress 对象， 这个Ingress 用于引导流量去具体哪个后端的服务
3. Ingress一般需要你自己主动配置

## 3. 实战

**部署一个Nginx Ingress Controller**

[deploy-v1.1.0](https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.0/deploy/static/provider/cloud/deploy.yaml)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.0/deploy/static/provider/cloud/deploy.yaml
```

It installs abunch of Kubernetes constructs including a Namespace, ServiceAccounts, ConfigMap, Roles, RoleBindings, and more

查看pod

```bash
➜   kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
NAME                                      READY   STATUS              RESTARTS   AGE
ingress-nginx-admission-create--1-lmfrh   0/1     Completed           0          4s
ingress-nginx-admission-patch--1-qk4xz    0/1     Completed           0          4s
ingress-nginx-controller-54bfb9bb-ft8f8   0/1     ContainerCreating   0          4s

```

**Configure Ingress classes for clusters with multiple Ingress controllers**

```bash
➜   kubectl apply -f ig-class.yml 
ingressclass.networking.k8s.io/igc-nginx created
➜   kubectl get ingressclasses
NAME        CONTROLLER                     PARAMETERS   AGE
igc-nginx   nginx.org/ingress-controller   <none>       22s
nginx       k8s.io/ingress-nginx           <none>       3m53s
```

**Configuring host-based and path-based routing**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422163532-host-based-routing.png)

Traffic flow to the shield app using host-based routing will be as follows.

1. A client will send traffic to shield.mcu.com

2. Name resolution will send the traffic to the load-balancer’s public endpoint

3. Ingress will read the HTTP headers for the hostname (shield.mcu.com)

4. An Ingress rule will trigger and the traffic will be routed to the svc-shield ClusterIP backend

5. The ClusterIP Service will ensure the traffic reaches the shield Pod

**Deploy the apps**

```bash
➜  kubectl apply -f app.yml 
service/svc-shield created
service/svc-hydra created
pod/shield created
pod/hydra created
```

**Create the Ingress object**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcu-all
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: shield.mcu.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-shield
            port:
              number: 8080
  - host: hydra.mcu.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-hydra
            port:
              number: 8080
  - host: mcu.com
    http:
      paths:
      - path: /shield
        pathType: Prefix
        backend:
          service:
            name: svc-shield
            port:
              number: 8080
      - path: /hydra
        pathType: Prefix
        backend:
          service:
            name: svc-hydra
            port:
              number: 8080
```

`kubectl apply -f ig-all.yml`

**Inspecting Ingress objects**

```bash
➜  ~ kubectl get ing
NAME      CLASS   HOSTS                                  ADDRESS     PORTS   AGE
mcu-all   nginx   shield.mcu.com,hydra.mcu.com,mcu.com   localhost   80      50m

➜  ~ kubectl describe ing mcu-all
Name:             mcu-all
Namespace:        default
Address:          localhost
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host            Path  Backends
  ----            ----  --------
  shield.mcu.com  
                  /   svc-shield:8080 (10.1.0.125:8080)
  hydra.mcu.com   
                  /   svc-hydra:8080 (10.1.0.126:8080)
  mcu.com         
                  /shield   svc-shield:8080 (10.1.0.125:8080)
                  /hydra    svc-hydra:8080 (10.1.0.126:8080)
Annotations:      nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    50m (x2 over 51m)  nginx-ingress-controller  Scheduled for sync
```

**Configure DNS name resolution**

把域名配置到 /etc/hosts 中

**Test**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422164349-ingress.png)

```bash
➜  ~ kubectl get ns
NAME              STATUS   AGE
default           Active   45h
ingress-nginx     Active   85m
kube-node-lease   Active   45h
kube-public       Active   45h
kube-system       Active   45h

➜  ~ kubectl get pods -n ingress-nginx
NAME                                      READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create--1-lmfrh   0/1     Completed   0          86m
ingress-nginx-admission-patch--1-qk4xz    0/1     Completed   0          86m
ingress-nginx-controller-54bfb9bb-ft8f8   1/1     Running     0          86m
➜  ~ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.109.137.14   localhost     80:31463/TCP,443:31569/TCP   86m
ingress-nginx-controller-admission   ClusterIP      10.104.81.234   <none>        443/TCP                      86m

➜  ~ kubectl get pods
NAME     READY   STATUS    RESTARTS   AGE
hydra    1/1     Running   0          79m
shield   1/1     Running   0          79m
➜  ~ kubectl get svc 
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    45h
svc-hydra    ClusterIP   10.105.90.81    <none>        8080/TCP   79m
svc-shield   ClusterIP   10.101.237.41   <none>        8080/TCP   79m

➜  ~ kubectl get ingress                 
NAME      CLASS   HOSTS                                  ADDRESS     PORTS   AGE
mcu-all   nginx   shield.mcu.com,hydra.mcu.com,mcu.com   localhost   80      58m
```

