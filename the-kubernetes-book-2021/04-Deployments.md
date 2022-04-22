# Kubernetes Deployments

## 1. Deployments Theory

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220421113302-deployments.png)

### 1.1 Deployments and Pod

You post the Deployment object to the API server where, Kubernetes implements it and the Deployment controller watches it

相对于static pod, Deployment 相当于在Pod 上面增加了一个管理层， 有了这个管理层， Pod 可以更方便扩展， 还有更新

### 1.2 Deployments and ReplicatSets

Behind-the-scenes, Deployments rely heavily on another object called a ReplicaSet. While it’s usually recommended not to manage ReplicaSets directly (let the Deployment controller manage them), it’s important to understand the role they play

At a high-level, containers are a great way to package applications and dependencies. Pods allow containers to run on Kubernetes and enable co-scheduling and a bunch of other good stuff. ReplicaSets manage Pods and bring self-healing and scaling. Deployments manage ReplicaSets and add rollouts and rollbacks.

实际上是ReplicaSets 来完成了Pod 的 自我恢复和扩展的这种能力

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220421114211-deployments-rs.png)

**Rolling Updates**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422092710-rolling-update.png)

**Rollbacks**

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220422092749-rollback.png)

## 2. Create A Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-pod
          image: nigelpoulton/k8sbook:1.0
          ports:
            - containerPort: 8080
```

### 2.1 一些解释

* kind ：是 Deployment 类型
* spec： 一些属性
  * replicas: Pod副本数量
  * selector : 选择的条件
  * strategy：如何update
  * template： Pod 模板



**revisionHistoryLimit** tells Kubernetes to keep the configs of the previous 5 releases. This means the previous 5 ReplicaSet objects will be kept and you can rollback to any of them. 

**progressDeadlineSeconds** governs how long Kubernetes waits for each new Pod replica to start, before considering the rollout to have stalled. This config gives each Pod replica its own 5 minute window to come up.

**.spec.minReadySeconds** throttles the rate at which replicas are replaced. The one in the example tells Kubernetes that any new replica must be up and running for 10 seconds, without any issues, before it’s allowed to update/replace the next one in sequence. Longer waits give you a chance to spot problems and avoid updating all replicas to a dodgy version. In the real world, you should make the value large enough to trap common failures.

There is also a nested **.spec.strategy** map telling Kubernetes you want this Deployment to:

*  Update using the RollingUpdate strategy
* Never have more than one Pod below desired state (**maxUnavailable**: 1) 
* Never have more than one Pod above desired state (**maxSurge**: 1)

As the desired state of the app requests 10 replicas, maxSurge: 1 means you’ll never have more than 11 replicas during the update process, and maxUnavailable: 1 means you’ll never have less than 9. The net result is a rollout that updates two Pods at a time (the delta between 9 and 11 is 2).

### 2.2 Commands

**Deployments**

```bash
# 创建 deployment
kubectl apply -f deloy.yaml

# 查看deployment
kubectl get deploy hello-deploy

# 查看 详细 deploy 和 deployments 都可以
kubectl decribe deploy hello-deploy
```

**RS**

```bash
➜  deployments git:(master) ✗ kubectl get rs -o wide 
NAME                      DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                     SELECTOR
hello-deploy-85fd664fff   10        10        10      13m   hello-pod    nigelpoulton/k8sbook:1.0   app=hello-world,pod-template-hash=85fd664fff



➜  deployments git:(master) ✗ kubectl describe rs hello-deploy-85fd664fff        
Name:           hello-deploy-85fd664fff
Namespace:      default
Selector:       app=hello-world,pod-template-hash=85fd664fff
Labels:         app=hello-world
                pod-template-hash=85fd664fff
Annotations:    deployment.kubernetes.io/desired-replicas: 10
                deployment.kubernetes.io/max-replicas: 11
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/hello-deploy
Replicas:       10 current / 10 desired
Pods Status:    10 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=hello-world
           pod-template-hash=85fd664fff
  Containers:
   hello-pod:
    Image:        nigelpoulton/k8sbook:1.0
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-zjmvd
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-9smhw
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-bsr6t
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-f9w4r
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-hqr7x
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-5p6pn
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-cbgq8
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-ztqsz
  Normal  SuccessfulCreate  17m   replicaset-controller  Created pod: hello-deploy-85fd664fff-qrj56
  Normal  SuccessfulCreate  17m   replicaset-controller  (combined from similar events): Created pod: hello-deploy-85fd664fff-pxmgs

```

## 3. Scaling

```bash
➜  deployments git:(master) ✗ kubectl scale deploy hello-deploy --replicas 5
deployment.apps/hello-deploy scaled
➜  deployments git:(master) ✗ kubectl get pods 
NAME                            READY   STATUS    RESTARTS   AGE
hello-deploy-85fd664fff-5p6pn   1/1     Running   0          29m
hello-deploy-85fd664fff-9smhw   1/1     Running   0          29m
hello-deploy-85fd664fff-pxmgs   1/1     Running   0          29m
hello-deploy-85fd664fff-qrj56   1/1     Running   0          29m
hello-deploy-85fd664fff-ztqsz   1/1     Running   0          29m
```

## 4. Rolling update

修改Pod template 中的镜像版本 -> 2.0

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-pod
          image: nigelpoulton/k8sbook:2.0
          ports:
            - containerPort: 8080
```

### 4.1 Commands

```bash
# 查看更新状态
kubectl rollout status deployment hello-deploy

# 暂停rollout
kubectl rollout pause deploy hello-deploy

# resume
kubectl rollout resume deploy hello-deploy
```

## 5. Rollback

```bash
➜  deployments git:(master) ✗ kubectl rollout history deploy hello-deploy
deployment.apps/hello-deploy 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

如果一开始使用 

`kubectl apply --filename-deploy.yml --record=true`

进行滚动更新的话， 那个REVISION 2 的 CHANGE-CAUSE 会有这条记录



**查看RS**

```bash
➜  deployments git:(master) ✗ kubectl get rs           
NAME                      DESIRED   CURRENT   READY   AGE
hello-deploy-5445f6dcbb   10        10        10      12m
hello-deploy-85fd664fff   0         0         0       53m
```

```bash
➜  deployments git:(master) ✗ kubectl describe rs hello-deploy-85fd664fff
Name:           hello-deploy-85fd664fff
Namespace:      default
Selector:       app=hello-world,pod-template-hash=85fd664fff
Labels:         app=hello-world
                pod-template-hash=85fd664fff
Annotations:    deployment.kubernetes.io/desired-replicas: 10
                deployment.kubernetes.io/max-replicas: 11
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/hello-deploy
Replicas:       0 current / 0 desired
Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=hello-world
           pod-template-hash=85fd664fff
  Containers:
   hello-pod:
    Image:        nigelpoulton/k8sbook:1.0
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  54m    replicaset-controller  (combined from similar events): Created pod: hello-deploy-85fd664fff-pxmgs
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-9smhw
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-bsr6t
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-f9w4r
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-hqr7x
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-5p6pn
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-cbgq8
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-ztqsz
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-qrj56
  Normal  SuccessfulCreate  54m    replicaset-controller  Created pod: hello-deploy-85fd664fff-zjmvd
  Normal  SuccessfulDelete  25m    replicaset-controller  Deleted pod: hello-deploy-85fd664fff-cbgq8
  Normal  SuccessfulDelete  25m    replicaset-controller  Deleted pod: hello-deploy-85fd664fff-bsr6t
  Normal  SuccessfulDelete  25m    replicaset-controller  Deleted pod: hello-deploy-85fd664fff-f9w4r
  Normal  SuccessfulDelete  25m    replicaset-controller  Deleted pod: hello-deploy-85fd664fff-hqr7x
  Normal  SuccessfulDelete  25m    replicaset-controller  Deleted pod: hello-deploy-85fd664fff-zjmvd
  Normal  SuccessfulCreate  24m    replicaset-controller  Created pod: hello-deploy-85fd664fff-cnvnw
  Normal  SuccessfulCreate  24m    replicaset-controller  Created pod: hello-deploy-85fd664fff-8grlv
  Normal  SuccessfulCreate  24m    replicaset-controller  Created pod: hello-deploy-85fd664fff-9czfq
  Normal  SuccessfulCreate  24m    replicaset-controller  Created pod: hello-deploy-85fd664fff-6hxnx
  Normal  SuccessfulCreate  24m    replicaset-controller  Created pod: hello-deploy-85fd664fff-97h8t
  Normal  SuccessfulDelete  13m    replicaset-controller  Deleted pod: hello-deploy-85fd664fff-cnvnw
  Normal  SuccessfulDelete  9m45s  replicaset-controller  Deleted pod: hello-deploy-85fd664fff-9czfq
  Normal  SuccessfulDelete  9m45s  replicaset-controller  Deleted pod: hello-deploy-85fd664fff-6hxnx
  Normal  SuccessfulDelete  9m33s  replicaset-controller  Deleted pod: hello-deploy-85fd664fff-8grlv
  Normal  SuccessfulDelete  9m33s  replicaset-controller  Deleted pod: hello-deploy-85fd664fff-97h8t
  Normal  SuccessfulDelete  9m21s  replicaset-controller  Deleted pod: hello-deploy-85fd664fff-ztqsz
  Normal  SuccessfulDelete  9m21s  replicaset-controller  Deleted pod: hello-deploy-85fd664fff-9smhw
  Normal  SuccessfulDelete  9m9s   replicaset-controller  Deleted pod: hello-deploy-85fd664fff-qrj56
  Normal  SuccessfulDelete  9m9s   replicaset-controller  Deleted pod: hello-deploy-85fd664fff-pxmgs
  Normal  SuccessfulDelete  8m58s  replicaset-controller  (combined from similar events): Deleted pod: hello-deploy-85fd664fff-5p6pn
```

**回到版本1**

` kubectl rollout undo deployment hello-deploy --to-revision=1`

```bash
➜  deployments git:(master) ✗  kubectl rollout undo deployment hello-deploy --to-revision=1
deployment.apps/hello-deploy rolled back
➜  deployments git:(master) ✗ kubectl rollout status deployment hello-deploy
Waiting for deployment "hello-deploy" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "hello-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hello-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hello-deploy" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hello-deploy" rollout to finish: 1 old replicas are pending termination...
deployment "hello-deploy" successfully rolled out
```



