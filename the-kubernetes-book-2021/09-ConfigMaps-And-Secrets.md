# ConfigMaps And Secrets

应用与配置分离， 这样设计的好处

1. 应用镜像可以复用
2. 部署和测试相对简单
3. 修改的话，影响面清晰， 容易维护

## 1. ConfigMaps theory

ConfigMaps are first-class objects in the Kubernetes API under the *core* API group, and they’re v1. This tells us a lot of things:

1.  They’re stable (v1)

2.  They’ve been around for a while (the fact that they’re in the core API group)

3.  You can operate on them with the usual kubectl commands

4.  They can be defined and deployed via the usual YAML manifests

ConfigMaps 可以用来保存

1. 环境变量
2. 配置文件
3. 主机名 （HostNames）
4. 服务端口
5. 账号名称

ConfigMaps 不建议 存储敏感信息： 密码，密钥等等， 建议使用Secrets。 Secrets 在 K8S 中存储不是明文 （一般是base64）

### 1.1 How ConfigMaps work

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426110656-configmaps.png)

All of the methods work seamlessly with existing applications. In fact, all an application sees is its configuration data in either; an environment variable, an argument to a startup command, or a file in a filesystem. The application is unaware the data originally came from a ConfigMap.

最长用的是 volume， 因为 volume 的变化， 相当于是修改了能看见（It may take a minute for the changes to be updated in the running container.）

Behind the scenes, ConfigMaps are a map of key/value pairs and we call each key/value pair an *entry*. 

*  **Keys** are an arbitrary name that can be created from alphanumerics, dashes, dots, and underscores

*  **Values** can contain anything, including multiple lines with carriage returns

*  Keys and values are separated by a colon – key:value

## 2. ConfigMap Details

### 2.1 Creating configmaps imperatively

**from-literal**

```bash
➜  ~ kubectl create secret generic creds --from-literal user=nigelpoulton --from-literal pwd=Password123
➜  ~ kubectl get secrets creds -o yaml
apiVersion: v1
data:
  pwd: UGFzc3dvcmQxMjM=
  user: bmlnZWxwb3VsdG9u
kind: Secret
metadata:
  creationTimestamp: "2022-04-26T06:03:27Z"
  name: creds
  namespace: default
  resourceVersion: "219399"
  uid: 0127f149-c386-42b9-8988-8c942a5864a3
type: Opaque
```

**from-file**

```bash
➜  cat cmfile.txt
Kubernetes FTW

➜  kubectl create cm testmap2 --from-file cmfile.txt 

➜  kubectl get cm testmap2 -o yaml
apiVersion: v1
data:
  cmfile.txt: |
    Kubernetes FTW
kind: ConfigMap
metadata:
  creationTimestamp: "2022-04-26T01:46:58Z"
  name: testmap2
  namespace: default
  resourceVersion: "204001"
  uid: 31fd3651-9ccc-4d6c-bb74-62cfda441ba1

```

### 2.2 Creating ConfigMaps declaratively

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: test-config
data:
  test.conf: |
    env = plex-test
    endpoint = 0.0.0.0:31001
    char = utf8
    vault = PLEX/test
    log-size = 512M
```

相当于

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426141704-pipe-conf.png)

### 2.3 Injecting ConfigMap data into Pods and containers

**multimap.yaml**

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: multimap
data:
  given: Nigel
  family: Poulton
```

**By env**

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    chapter: configmaps
  name: envpod
spec:
  containers:
    - name: ctr1
      image: busybox
      command: ["sleep"]
      args: ["infinity"]
      env:
        - name: FIRSTNAME
          valueFrom:
            configMapKeyRef:
              name: multimap
              key: given
        - name: LASTNAME
          valueFrom:
            configMapKeyRef:
              name: multimap
              key: family
```

**By startup commands**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
  labels:
    chapter: configmaps
spec:
  restartPolicy: OnFailure
  containers:
    - name: args1
      image: busybox
      command: [ "/bin/sh", "-c", "echo First name $(FIRSTNAME) last name $(LASTNAME)", "wait" ]
      env:
        - name: FIRSTNAME
          valueFrom:
            configMapKeyRef:
              name: multimap
              key: given
        - name: LASTNAME
          valueFrom:
            configMapKeyRef:
              name: multimap
              key: family
```

**By volume**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cmvol
spec:
  volumes:
    - name: volmap
      configMap:
        name: multimap
  containers:
    - name: ctr
      image: nginx
      volumeMounts:
        - name: volmap
          mountPath: /etc/name
```

## 3. Secrets

Despite being designed for sensitive data, Kubernetes does not encrypt Secrets. It merely obscures them as base- 64 encoded values that can easily be decoded. Fortunately, it’s possible to configure encryption-at-rest with EncryptionConfiguration objects, and most service meshes encrypt network traffic.

A typical workflow for a Secret is as follows.

1. The Secret is created and persisted to the cluster store as an un-encrypted object

2. A Pod that uses it gets scheduled to a cluster node

3. The Secret is transferred over the network, un-encrypted, to the node

4. The kubelet on the node starts the Pod and its containers

5. The Secret is mounted into the container via an in-memory tmpfs filesystem and decoded from base64 to plain text

6. The application consumes it

7. If/when the Pod is deleted, the Secret is deleted form the node

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426142705-secrets.png)

### 3.1 Imperatively 

```bash
➜  kubectl create secret generic creds --from-literal user=nigelpoulton --from-literal pwd=Password123
➜  kubectl get secrets creds -o yaml
apiVersion: v1
data:
  pwd: UGFzc3dvcmQxMjM=
  user: bmlnZWxwb3VsdG9u
kind: Secret
metadata:
  creationTimestamp: "2022-04-26T06:03:27Z"
  name: creds
  namespace: default
  resourceVersion: "219399"
  uid: 0127f149-c386-42b9-8988-8c942a5864a3
type: Opaque
➜   echo UGFzc3dvcmQxMjM= | base64 -d
Password123
```

### 3.2 Using secrets in pods

**tkb-secret.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tkb-secret
  labels:
    chapter: configmaps
type: Opaque
data:
  username: bmlnZWxwb3VsdG9u
  password: UGFzc3dvcmQxMjM=
```

**secretpod.yaml**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
  labels:
    topic: secrets
spec:
  volumes:
  - name: secret-vol
    secret:
      secretName: tkb-secret
  containers:
  - name: secret-ctr
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: "/etc/tkb"
      readOnly: true
```

```bash
➜   kubectl create -f tkb-secret.yml 
secret/tkb-secret created
➜   kubectl apply -f secretpod.yml 
pod/secret-pod created
➜   kubectl exec secret-pod -- ls /etc/tkb
password
username
➜   kubectl exec secret-pod -- cat /etc/tkb/password
Password123
```

