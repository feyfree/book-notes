[被判“死刑”不到5年，Docker 翻身成为硅谷独角兽，新战略成功了？](https://mp.weixin.qq.com/s/i7YVcIGpKv9NhPLLMhDg3Q)

# 1. Docker Architecture

![](https://tcs.teambition.net/storage/312g109212a53d77c77db5dc81c1bcf74c2b?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzMTQ3OCwiaWF0IjoxNjQ5OTI2Njc4LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmcxMDkyMTJhNTNkNzdjNzdkYjVkYzgxYzFiY2Y3NGMyYiJ9.KjxCHyuf7XQUspPqcMfbIPP7DRg0OBCnfdi45vhc4y4&download=image.png "")

# 2. 一些名词

## OCI

The Open Container Initiative (OCI) was instrumental in standardizing the container runtime format and container image format. 

可以把它理解就是定义了一个文档，主要规定了容器镜像的结构、以及容器需要接收哪些操作指令，比如 create、start、stop、delete 等这些命令

## containerd

An industry-standard container runtime with an emphasis on simplicity, robustness and portability

## runc

runc is a CLI tool for spawning and running containers on Linux according to the OCI specification。

runc 就可以按照这个 OCI 文档来创建一个符合规范的容器，既然是标准肯定就有其他 OCI 实现，比如 Kata、gVisor 这些容器运行时都是符合 OCI 标准的

## libcontainer

Libcontainer 是 Docker 中用于容器管理的包，它基于 Go 语言实现，通过管理namespaces、cgroups、capabilities以及文件系统来进行容器控制。你可以使用 Libcontainer 创建容器，并对容器进行生命周期管理

[Docker背后的容器管理——Libcontainer深度解析_语言 & 开发_孙健波_InfoQ精选文章](https://www.infoq.cn/article/docker-container-management-libcontainer-depth-analysis)

Docker 被逼无耐将 libcontainer 捐献出来改名为 runc 的

## LXC

LXC，其名称来自Linux软件容器（Linux Containers）的缩写，一种操作系统层虚拟化（Operating system–level virtualization）技术，为Linux内核容器功能的一个用户空间接口。它将应用软件系统打包成一个软件容器（Container），内含应用软件本身的代码，以及所需要的操作系统核心和库。透过统一的名字空间和共享API来分配不同软件容器的可用硬件资源，创造出应用程序的独立沙箱执行环境，使得Linux用户可以容易的创建和管理系统或应用容器

# 3. Docker Engine

## High Level Overview

![](https://tcs.teambition.net/storage/312gc400a7421284e070621d59f7db95c0b5?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzMTQ3OCwiaWF0IjoxNjQ5OTI2Njc4LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmdjNDAwYTc0MjEyODRlMDcwNjIxZDU5ZjdkYjk1YzBiNSJ9.9gPT_XD8E8e-Hn4llSlCHUhRyDScTNMe-hQ5Sf3iXn0&download=image.png "")

## Old Version Architecture

![](https://tcs.teambition.net/storage/312gdeb3a9a7a250d3c8027e28e7bdcffb88?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzMTQ3OCwiaWF0IjoxNjQ5OTI2Njc4LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmdkZWIzYTlhN2EyNTBkM2M4MDI3ZTI4ZTdiZGNmZmI4OCJ9.IW6k3n8sdTdZYt1X-FpPljldfp4dOvCF18BBP4czFnY&download=image.png "")

When Docker was first released, the Docker engine had two major components:

- The Docker daemon (hereafter referred to as just “the daemon”)

- LXC

The Docker daemon was a monolithic binary. It contained all of the code for the Docker client, the Docker API, the container runtime, image builds, and much more.

一开始的Docker Daemon 很重。然后慢慢将containerd 和 runc 这部分模块化， 从原始的daemon 库中独立开来.

## Current Version Architecture

![](https://tcs.teambition.net/storage/312g55e6b14c12844c704b2affcfbc0dd2f9?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzMTQ3OCwiaWF0IjoxNjQ5OTI2Njc4LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmc1NWU2YjE0YzEyODQ0YzcwNGIyYWZmY2ZiYzBkZDJmOSJ9.pYGbtJEjhAhKfnM01BsuWiCGhE57cGes_CrFiKfKKao&download=image.png "")

## Starting a new container (example)

```shell
docker container run --name ctr1 -it alpine:latest sh
```

这条命令Docker 做了哪些工作

1. 解析这台命令， 并封装参数， 通过docker daemon 提供的API 发送给daemon

1. The API is implemented in the daemon and can be exposed over a local socket or the network. On Linux the
 socket is /var/run/docker.sock and on Windows it’s \pipe\docker_engine.

docker daemon 实际上是会监听这个socket上面的数据

1. docker daemon 收到指令后会让containerd 去创建容器 （此时daemon 不再负责下面的容器创建的事情）。daemon 和 containerd 通过GRPC 通信 （CRUD-Style）

1. Despite its name, containerd cannot actually create containers. It uses runc to do that. It converts the required. Docker image into an OCI bundle and tells runc to use this to create a new container.

实际上是runc 负责创建容器的

![](https://tcs.teambition.net/storage/312gd6f1f6dc97bd4f41ab38409881ef96a7?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzMTQ3OCwiaWF0IjoxNjQ5OTI2Njc4LCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmdkNmYxZjZkYzk3YmQ0ZjQxYWIzODQwOTg4MWVmOTZhNyJ9.UNyJ1LEWRjX9VV2jzFIpVFZpFr9OFaZ2Rf8kOoPE9zE&download=image.png "")

实际上当我们要创建一个容器的时候，现在 Docker Daemon 并不能直接帮我们创建了，而是请求 containerd 来创建一个容器，containerd 收到请求后，也并不会直接去操作容器，而是创建一个叫做 containerd-shim 的进程，让这个进程去操作容器，我们指定容器进程是需要一个父进程来做状态收集、维持 stdin 等 fd 打开等工作的，假如这个父进程就是 containerd，那如果 containerd 挂掉的话，整个宿主机上所有的容器都得退出了，而引入 containerd-shim 这个垫片就可以来规避这个问题了。

然后创建容器需要做一些 namespaces 和 cgroups 的配置，以及挂载 root 文件系统等操作，这些操作其实已经有了标准的规范，那就是 OCI（开放容器标准），runc 就是它的一个参考实现（Docker 被逼无耐将 libcontainer 捐献出来改名为 runc 的），这个标准其实就是一个文档，主要规定了容器镜像的结构、以及容器需要接收哪些操作指令，比如 create、start、stop、delete 等这些命令。runc 就可以按照这个 OCI 文档来创建一个符合规范的容器，既然是标准肯定就有其他 OCI 实现，比如 Kata、gVisor 这些容器运行时都是符合 OCI 标准的。

所以真正启动容器是通过 containerd-shim 去调用 runc 来启动容器的，runc 启动完容器后本身会直接退出，containerd-shim 则会成为容器进程的父进程, 负责收集容器进程的状态, 上报给 containerd, 并在容器中 pid 为 1 的进程退出后接管容器中的子进程进行清理, 确保不会出现僵尸进程。

而 Docker 将容器操作都迁移到 containerd 中去是因为当前做 Swarm，想要进军 PaaS 市场，做了这个架构切分，让 Docker Daemon 专门去负责上层的封装编排，当然后面的结果我们知道 Swarm 在 Kubernetes 面前是惨败，然后 Docker 公司就把 containerd 项目捐献给了 CNCF 基金会，这个也是现在的 Docker 架构

[一文搞懂容器运行时 Containerd](https://www.qikqiak.com/post/containerd-usage/)

## How it’s implemented on Linux

On a Linux system, the components we’ve discussed are implemented as separate binaries as follows:

-  dockerd (the Docker daemon)

-  docker-containerd (containerd)

- docker-containerd-shim (shim)

- docker-runc (runc)

You can see all of these on a Linux system by running a ps command on the Docker host. Obviously, some of them will only be present when the system has running containers.

## What's the point of daemon

Obviously, the answer to this question will change over time as more and more functionality is stripped out and modularized. However, at the time of writing, some of the major functionality that still exists in the daemon includes; image management, image builds, the REST API, authentication, security, core networking, and orchestration.



