# The Kubernetes API

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428161139-api-request-flow.png)

At the time of writing, **Protobuf** is mainly used for internal cluster traffic, whereas JSON is used when communicating with external clients

Protobuf 相当于 JSON 而言 更高效， 但是可读性相对弱一点， 以及不易于排查 （相对JSON）

## 1. API Server

The API server exposes the API over a secure RESTful interface using HTTPS. It acts as the front-end to the API and is a bit like Grand Central for Kubernetes – everything talks to everything else via REST API calls to the API server. For example:

* All kubectl commands go to the API server (creating, retrieving, updating, deleting objects)

* All node Kubelets watch the API server for new tasks and report status to the API server

* All control plane services communicate via the API server (components don’t talk directly each other)

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428161705-rest-api.png)

```bash
➜  ~ kubectl cluster-info
Kubernetes control plane is running at https://kubernetes.docker.internal:6443
CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### 1.1 REST

REST requests comprise a *verb* and a *path* to a resource. Verbs relate to actions, and are the standard HTTP methods you saw in the previous table. Paths are a URI path to the resource in the API

```bash
# 会生成一个进程，然后可以通过 localhost:9000 + 路径去访问相关资源
# 关闭的话， 通过ps 命令查看 进程ID 然后手动 kill 掉
$ kubectl proxy --port 9000 &
[1] 14774
Starting to serve on 127.0.0.1:9000
```

```json
{
    "kind": "Namespace",
    "apiVersion": "v1",
    "metadata": {
      "name": "shield",
      "labels": {
        "chapter": "api"
      }
    }
}
```

可以通过这个JSON 发送到 9000 端口

```bash
$ curl -X POST -H "Content-Type: application/json" \
--data-binary @ns.json http://localhost:9000/api/v1/namespaces
```

相当于 是 kubectl apply -f 类似的yaml 的底层的操作

## 2. API

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428162619-k8s-api-category.png)

### 2.1 The Core API group

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428162715-core-api-group.png)

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428162746-api-url-path.png)

### 2.2 Named API groups

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220428162830-named-api-groups.png)

可以通过

`kubectl api-resources` 查看所有的资源， 里面有分组的信息

查看api 的版本

```bash
➜  ~ kubectl api-versions
admissionregistration.k8s.io/v1
apiextensions.k8s.io/v1
apiregistration.k8s.io/v1
apps/v1
authentication.k8s.io/v1
authorization.k8s.io/v1
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
batch/v1
batch/v1beta1
certificates.k8s.io/v1
coordination.k8s.io/v1
discovery.k8s.io/v1
discovery.k8s.io/v1beta1
events.k8s.io/v1
events.k8s.io/v1beta1
flowcontrol.apiserver.k8s.io/v1beta1
networking.k8s.io/v1
node.k8s.io/v1
node.k8s.io/v1beta1
policy/v1
policy/v1beta1
rbac.authorization.k8s.io/v1
scheduling.k8s.io/v1
storage.k8s.io/v1
storage.k8s.io/v1beta1
v1
```

组合脚本， 查看对应资源对应的 version

```bash
$ for kind in `kubectl api-resources | tail +2 | awk '{ print $1 }'`; \
do kubectl explain $kind; done | grep -e "KIND:" -e "VERSION:"
```

### 2.3 Extending the api

**crd.yml**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: books.nigelpoulton.com
spec:
  group: nigelpoulton.com      
  scope: Cluster            
  names:
    plural: books      
    singular: book     
    kind: Book         
    shortNames:
    - bk                       
  versions:                    
    - name: v1                  
      served: true             
      storage: true            
      schema:                  
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                bookTitle:
                  type: string
                topic:
                  type: string
                edition:
                  type: integer
      additionalPrinterColumns:
      - name: Title
        type: string
        description: Title of the book
        jsonPath: .spec.bookTitle
      - name: Edition
        type: integer
        description: Edition number
        jsonPath: .spec.edition
```

```bash
➜  kubectl apply -f crd.yml 
customresourcedefinition.apiextensions.k8s.io/books.nigelpoulton.com created
➜  kubectl api-resources | grep books
books                             bk           nigelpoulton.com/v1                    false        Book
➜  kubectl explain books
KIND:     Book
VERSION:  nigelpoulton.com/v1

DESCRIPTION:
     <empty>

FIELDS:
		...
```

**tkb.yml**

```
apiVersion: nigelpoulton.com/v1
kind: Book
metadata:
  name: tkb
spec:
  bookTitle: "The Kubernetes Book"
  topic: Kubernetes
  edition: 2
```

```bash
➜  kubectl apply -f tkb.yml 
book.nigelpoulton.com/tkb created
➜  kubectl get bk
NAME   TITLE                 EDITION
tkb    The Kubernetes Book   2


➜  ~ kubectl proxy --port 9000 &
[1] 4579
➜  ~ Starting to serve on 127.0.0.1:9000

➜  ~ 
➜  ~ 
➜  ~ 
➜  ~  curl http://localhost:9000/apis/nigelpoulton.com/v1/
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "nigelpoulton.com/v1",
  "resources": [
    {
      "name": "books",
      "singularName": "book",
      "namespaced": false,
      "kind": "Book",
      "verbs": [
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "create",
        "update",
        "watch"
      ],
      "shortNames": [
        "bk"
      ],
      "storageVersionHash": "F2QdXaP5vh4="
    }
  ]
}
```

