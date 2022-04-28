# API Security And RBAC

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428102750-rbac.png)

## 1. Authentication

Cluster details and credentials are stored in a kubeconfig file. Tools like kubectl read this file to know which cluster to send commands to, as well as which credentials to use. It’s usually stored in the following location.

* Windows: C:\Users<user>.kube\config

* Linux/Mac: /home/<user>/.kube/config

也可以使用命令行查看， 但是一些token，ca的信息被隐藏了

```bash
➜ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    namespace: default
    user: docker-desktop
  name: docker-desktop
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

1. clusters 标明K8S的集群信息， 包括CA， 包括API Server 的url
2. users 标明用户信息
3. context 封装用户 + 集群， The contexts section combines users and clusters, and the current-context is the cluster and user kubectl willuse for all commands.

## 2. Authorization (RBAC)

 [使用RBAC鉴权](https://kubernetes.io/zh/docs/reference/access-authn-authz/rbac/)

### 2.1 Role 和 ClusterRole 

RBAC 的 *Role* 或 *ClusterRole* 中包含一组代表相关权限的规则。 这些权限是纯粹累加的（不存在拒绝某操作的规则）。

Role 总是用来在某个[名字空间](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/namespaces/) 内设置访问权限；在你创建 Role 时，你必须指定该 Role 所属的名字空间。

与之相对，ClusterRole 则是一个集群作用域的资源。这两种资源的名字不同（Role 和 ClusterRole）是因为 Kubernetes 对象要么是名字空间作用域的，要么是集群作用域的， 不可两者兼具。

ClusterRole 有若干用法。你可以用它来：

1. 定义对某名字空间域对象的访问权限，并将在各个名字空间内完成授权；
2. 为名字空间作用域的对象设置访问权限，并跨所有名字空间执行授权；
3. 为集群作用域的资源定义访问权限。

如果你希望在名字空间内定义角色，应该使用 Role； 如果你希望定义集群范围的角色，应该使用 ClusterRole。



ClusterRole 可以和 Role 相同完成授权。 因为 ClusterRole 属于集群范围，所以它也可以为以下资源授予访问权限：

- 集群范围资源（比如 [节点（Node）](https://kubernetes.io/zh/docs/concepts/architecture/nodes/)）

- 非资源端点（比如 `/healthz`）

- 跨名字空间访问的名字空间作用域的资源（如 Pods）

  比如，你可以使用 ClusterRole 来允许某特定用户执行 `kubectl get pods --all-namespaces`

### 2.2 RoleBinding 和 ClusterRoleBinding 

角色绑定（Role Binding）是将角色中定义的权限赋予一个或者一组用户。 它包含若干 **主体**（用户、组或服务账户）的列表和对这些主体所获得的角色的引用。 RoleBinding 在指定的名字空间中执行授权，而 ClusterRoleBinding 在集群范围执行授权。

一个 RoleBinding 可以引用同一的名字空间中的任何 Role。 或者，一个 RoleBinding 可以引用某 ClusterRole 并将该 ClusterRole 绑定到 RoleBinding 所在的名字空间。 如果你希望将某 ClusterRole 绑定到集群中所有名字空间，你要使用 ClusterRoleBinding。

RoleBinding 或 ClusterRoleBinding 对象的名称必须是合法的 [路径区段名称](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/names#path-segment-names)。

### 2.3 Rules

```bash
➜  ~ kubectl api-resources -o wide
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND                             VERBS
bindings                                       v1                                     true         Binding                          [create]
componentstatuses                 cs           v1                                     false        ComponentStatus                  [get list]
configmaps                        cm           v1                                     true         ConfigMap                        [create delete deletecollection get list patch update watch]
endpoints                         ep           v1                                     true         Endpoints                        [create delete deletecollection get list patch update watch]
events                            ev           v1                                     true         Event                            [create delete deletecollection get list patch update watch]
limitranges                       limits       v1                                     true         LimitRange                       [create delete deletecollection get list patch update watch]
namespaces                        ns           v1                                     false        Namespace                        [create delete get list patch update watch]
nodes                             no           v1                                     false        Node                             [create delete deletecollection get list patch update watch]
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim            [create delete deletecollection get list patch update watch]
persistentvolumes                 pv           v1                                     false        PersistentVolume                 [create delete deletecollection get list patch update watch]
pods                              po           v1                                     true         Pod                              [create delete deletecollection get list patch update watch]
podtemplates                                   v1                                     true         PodTemplate                      [create delete deletecollection get list patch update watch]
replicationcontrollers            rc           v1                                     true         ReplicationController            [create delete deletecollection get list patch update watch]
resourcequotas                    quota        v1                                     true         ResourceQuota                    [create delete deletecollection get list patch update watch]
```

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428104207-rbac-roles.png)

### 2.4 Pre-created users and permissions

```bash
➜  ~ kubectl get clusterrolebindings
NAME                                                   ROLE                                                                               AGE
cluster-admin                                          ClusterRole/cluster-admin                                                          7d15h
ingress-nginx-admission                                ClusterRole/ingress-nginx-admission                                                5d19h
kubeadm:get-nodes                                      ClusterRole/kubeadm:get-nodes                                                      7d15h
kubeadm:kubelet-bootstrap                              ClusterRole/system:node-bootstrapper                                               7d15h
kubeadm:node-autoapprove-bootstrap                     ClusterRole/system:certificates.k8s.io:certificatesigningrequests:nodeclient       7d15h
kubeadm:node-autoapprove-certificate-rotation          ClusterRole/system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   7d15h
kubeadm:node-proxier                                   ClusterRole/system:node-proxier                                                    7d15h
storage-provisioner                                    ClusterRole/storage-provisioner                                                    7d15h
system:basic-user                                      ClusterRole/system:basic-user                                                      7d15h
system:controller:attachdetach-controller              ClusterRole/system:controller:attachdetach-controller                              7d15h
system:controller:certificate-controller               ClusterRole/system:controller:certificate-controller                               7d15h
system:controller:clusterrole-aggregation-controller   ClusterRole/system:controller:clusterrole-aggregation-controller                   7d15h
system:controller:cronjob-controller                   ClusterRole/system:controller:cronjob-controller                                   7d15h
system:controller:daemon-set-controller                ClusterRole/system:controller:daemon-set-controller                                7d15h
system:controller:deployment-controller                ClusterRole/system:controller:deployment-controller                                7d15h
system:controller:disruption-controller                ClusterRole/system:controller:disruption-controller                                7d15h
system:controller:endpoint-controller                  ClusterRole/system:controller:endpoint-controller                                  7d15h
system:controller:endpointslice-controller             ClusterRole/system:controller:endpointslice-controller                             7d15h
system:controller:endpointslicemirroring-controller    ClusterRole/system:controller:endpointslicemirroring-controller                    7d15h
system:controller:ephemeral-volume-controller          ClusterRole/system:controller:ephemeral-volume-controller                          7d15h
system:controller:expand-controller                    ClusterRole/system:controller:expand-controller                                    7d15h
system:controller:generic-garbage-collector            ClusterRole/system:controller:generic-garbage-collector                            7d15h
system:controller:horizontal-pod-autoscaler            ClusterRole/system:controller:horizontal-pod-autoscaler                            7d15h
system:controller:job-controller                       ClusterRole/system:controller:job-controller                                       7d15h
system:controller:namespace-controller                 ClusterRole/system:controller:namespace-controller                                 7d15h
system:controller:node-controller                      ClusterRole/system:controller:node-controller                                      7d15h
system:controller:persistent-volume-binder             ClusterRole/system:controller:persistent-volume-binder                             7d15h
system:controller:pod-garbage-collector                ClusterRole/system:controller:pod-garbage-collector                                7d15h
system:controller:pv-protection-controller             ClusterRole/system:controller:pv-protection-controller                             7d15h
system:controller:pvc-protection-controller            ClusterRole/system:controller:pvc-protection-controller                            7d15h
system:controller:replicaset-controller                ClusterRole/system:controller:replicaset-controller                                7d15h
system:controller:replication-controller               ClusterRole/system:controller:replication-controller                               7d15h
system:controller:resourcequota-controller             ClusterRole/system:controller:resourcequota-controller                             7d15h
system:controller:root-ca-cert-publisher               ClusterRole/system:controller:root-ca-cert-publisher                               7d15h
system:controller:route-controller                     ClusterRole/system:controller:route-controller                                     7d15h
system:controller:service-account-controller           ClusterRole/system:controller:service-account-controller                           7d15h
system:controller:service-controller                   ClusterRole/system:controller:service-controller                                   7d15h
system:controller:statefulset-controller               ClusterRole/system:controller:statefulset-controller                               7d15h
system:controller:ttl-after-finished-controller        ClusterRole/system:controller:ttl-after-finished-controller                        7d15h
system:controller:ttl-controller                       ClusterRole/system:controller:ttl-controller                                       7d15h
system:coredns                                         ClusterRole/system:coredns                                                         7d15h
system:discovery                                       ClusterRole/system:discovery                                                       7d15h
system:kube-controller-manager                         ClusterRole/system:kube-controller-manager                                         7d15h
system:kube-dns                                        ClusterRole/system:kube-dns                                                        7d15h
system:kube-scheduler                                  ClusterRole/system:kube-scheduler                                                  7d15h
system:monitoring                                      ClusterRole/system:monitoring                                                      7d15h
system:node                                            ClusterRole/system:node                                                            7d15h
system:node-proxier                                    ClusterRole/system:node-proxier                                                    7d15h
system:public-info-viewer                              ClusterRole/system:public-info-viewer                                              7d15h
system:service-account-issuer-discovery                ClusterRole/system:service-account-issuer-discovery                                7d15h
system:volume-scheduler                                ClusterRole/system:volume-scheduler                                                7d15h
vpnkit-controller                                      ClusterRole/vpnkit-controller                                                      7d15h
```

```bash
➜  ~ kubectl describe clusterrolebindings cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  cluster-admin
Subjects:
  Kind   Name            Namespace
  ----   ----            ---------
  Group  system:masters  
  
➜  ~ kubectl describe clusterrole cluster-admin
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources  Non-Resource URLs  Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
```

## 3. Admission Control

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428110150-api-handle-requests.png)

[使用准入控制器](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/)

[搞懂 Kubernetes 准入控制](https://segmentfault.com/a/1190000041037182)

```bash
➜  ~ kubectl describe pod/kube-apiserver-docker-desktop --namespace kube-system | grep admission
      --enable-admission-plugins=NodeRestriction
```

准入控制器是一段代码，它会在请求通过认证和授权之后、对象被持久化之前拦截到达 API 服务器的请求。控制器由下面的[列表](https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do)组成， 并编译进 `kube-apiserver` 二进制文件，并且只能由集群管理员配置。 在该列表中，有两个特殊的控制器：MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook。 它们根据 API 中的配置，分别执行变更和验证 [准入控制 webhook](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/#admission-webhooks)。

准入控制器可以执行 “验证（Validating）” 和/或 “变更（Mutating）” 操作。 变更（mutating）控制器可以根据被其接受的请求修改相关对象；验证（validating）控制器则不行。

准入控制器限制创建、删除、修改对象或连接到代理的请求，不限制读取对象的请求。

准入控制过程分为两个阶段。第一阶段，运行变更准入控制器。第二阶段，运行验证准入控制器。 再次提醒，某些控制器既是变更准入控制器又是验证准入控制器。

如果任何一个阶段的任何控制器拒绝了该请求，则整个请求将立即被拒绝，并向终端用户返回一个错误。

最后，除了对对象进行变更外，准入控制器还可以有其它作用：将相关资源作为请求处理的一部分进行变更。 增加使用配额就是一个典型的示例，说明了这样做的必要性。 此类用法都需要相应的回收或回调过程，因为任一准入控制器都无法确定某个请求能否通过所有其它准入控制器。

```bash
# 查看默认启用的插件
kube-apiserver -h | grep enable-admission-plugins
```

我们主要从两个角度来理解为什么我们需要准入控制器：

- 从安全的角度
  - 我们需要明确在 Kubernetes 集群中部署的镜像来源是否可信，以免遭受攻击；
  - 一般情况下，在 Pod 内尽量不使用 root 用户，或者尽量不开启特权容器等；

- 从治理的角度
  - 比如通过 label 对业务/服务进行区分，那么可以通过 admission controller 校验服务是否已经有对应的 label 存在之类的；
  - 比如添加资源配额限制 ，以免出现资源超卖之类的情况