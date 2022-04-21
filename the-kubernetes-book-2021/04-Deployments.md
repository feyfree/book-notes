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