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