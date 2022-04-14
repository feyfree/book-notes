docker image 封装了应用运行时的底层依赖。image 是分层的， 而且像栈一样一层一层堆叠起来的。images 相当于为了运行应用， 封装了一个精简了的操作系统 + 应用所需文件。images 为了启动更快和更轻量， 一般来说都会设计很小。基于windows的镜像相对linux的相对大很多。

![](https://tcs.teambition.net/storage/312g79e7fd89a7e7e0d83cab211e0c1705e3?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzNDMzMiwiaWF0IjoxNjQ5OTI5NTMyLCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmc3OWU3ZmQ4OWE3ZTdlMGQ4M2NhYjIxMWUwYzE3MDVlMyJ9.5tyohGw43SW-NrwhTtiuiQjTJGaiTgF-hFXA1gRIyzA&download=image.png "")

# Images and Containers

镜像和容器是两个概念， 镜像相当于是一个文件，比如我们U盘里面的待安装的操作系统， 而容器更像是用来描述已经安好操作系统的一台电脑。同一份镜像， 可以用来启动多个容器。 Docker 里面， 如果容器没有被停掉和销毁的话， docker 是不允许用户将启动该容器的镜像删除的。

# Images are usually small

镜像通常很小，因为镜像在设计的时候尽量只封装应用需要的部分。打个比方， 镜像并不会像普通的Linux，Mac PC 一样， 内嵌好几个shell，供用户选择。

另外image 不包含 kernel， 所有的运行在docker主机上的容器，共享主机内核。

# Image Ops

## Commands

- **docker image pull** is the command to download images. We pull images from repositories inside of remote registries. By default, images will be pulled from repositories on Docker Hub. This command will pull the image tagged as latest from the alpine repository on Docker Hub: docker image pull alpine:latest. 

- **docker image ls** lists all of the images stored in your Docker host’s local image cache. To see the SHA256 digests of images add the --digests flag.

- ***docker image inspect*** is a thing of beauty! It gives you all of the glorious details of an image — layer data and metadata.

- ***docker manifest inspect*** allows you to inspect the manifest list of any image stored on Docker Hub. This will show the manifest list for the redis image: docker manifest inspect redis. 

- ***docker buildx*** is a Docker CLI plugin that extends the Docker CLI to support multi-arch builds.

- ***docker image rm*** is the command to delete images. This command shows how to delete the alpine:latest image — docker image rm alpine:latest. You cannot delete an image that is associated with a container in the running (Up) or stopped (Exited) states.

## Common

```shell
# 拉镜像
docker pull image nginx:apline

# 展示所有镜像
docker image ls

# A dangling image is an image that is no longer tagged, 
# and appears in listings as <none>:<none>.
docker image ls --filter dangling=true
```

## Advanced

```text
$ docker image ls --filter=reference="*:latest"
REPOSITORY TAG IMAGE ID CREATED SIZE
alpine latest f70734b6a266 3 days ago 5.61MB
redis latest a4d3716dbb72 3 days ago 98.3MB
busybox latest be5888e67be6 12 days ago 1.22MB

$ docker image ls --format "{{.Repository}}: {{.Tag}}: {{.Size}}"
alpine: latest: 5.61MB
redis: latest: 98.3MB
busybox: latest: 1.22MB

$ docker search alpine --filter "is-official=true"
NAME DESCRIPTION STARS OFFICIAL AUTOMATED
alpine A minimal Docker.. 6386 [OK]

$ docker search alpine --filter "is-automated=true"
NAME DESCRIPTION OFFICIAL AUTOMATED
anapsix/alpine-java Oracle Java 8 (and 7).. [OK]
frolvlad/alpine-glibc Alpine Docker image.. [OK]
alpine/git A simple git container.. [OK] \
<Snip>

$ docker image ls -q
bd3d4369aebc
4e38e38c8ce0

# 相当于删除了本机上面的所有镜像 （docker image prune）
$ docker image rm $(docker image ls -q) -f
Untagged: ubuntu:latest
Untagged: ubuntu@sha256:f4691c9...2128ae95a60369c506dd6e6f6ab
Deleted: sha256:bd3d4369aebc494...fa2645f5699037d7d8c6b415a10
Deleted: sha256:cd10a3b73e247dd...c3a71fcf5b6c2bb28d4f2e5360b
Deleted: sha256:4d4de39110cd250...28bfe816393d0f2e0dae82c363a
Deleted: sha256:6a89826eba8d895...cb0d7dba1ef62409f037c6e608b
Deleted: sha256:33efada9158c32d...195aa12859239d35e7fe9566056
Deleted: sha256:c8a75145fcc4e1a...4129005e461a43875a094b93412
Untagged: alpine:latest
Untagged: alpine@sha256:3dcdb92...313626d99b889d0626de158f73a
Deleted: sha256:4e38e38c8ce0b8d...6225e13b0bfe8cfa2321aec4bba
```

# Image Layers

![](https://tcs.teambition.net/storage/312g68269c4e71228a32f136bcece8e2d4af?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzNDMzMiwiaWF0IjoxNjQ5OTI5NTMyLCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmc2ODI2OWM0ZTcxMjI4YTMyZjEzNmJjZWNlOGUyZDRhZiJ9.6U3cr1p5h5jg1TrmQY0Mbwbj1SOwELgZuul_58cxR_k&download=image.png "")



```shell
$ docker image pull ubuntu:latest
latest: Pulling from library/ubuntu
952132ac251a: Pull complete
82659f8f1b76: Pull complete
c19118ca682d: Pull complete
8296858250fe: Pull complete
24e0251a0e2c: Pull complete
Digest: sha256:f4691c96e6bbaa99d...28ae95a60369c506dd6e6f6ab
Status: Downloaded newer image for ubuntu:latest
docker.io/ubuntu:latest
```

![](https://tcs.teambition.net/storage/312g5403aa63331b4e2adbc53d7a6c685064?Signature=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBcHBJRCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9hcHBJZCI6IjU5Mzc3MGZmODM5NjMyMDAyZTAzNThmMSIsIl9vcmdhbml6YXRpb25JZCI6IiIsImV4cCI6MTY1MDUzNDMzMiwiaWF0IjoxNjQ5OTI5NTMyLCJyZXNvdXJjZSI6Ii9zdG9yYWdlLzMxMmc1NDAzYWE2MzMzMWI0ZTJhZGJjNTNkN2E2YzY4NTA2NCJ9.D6cDaP4ue3Y1V7nXDtPmzW92rr3sJ8LX9kmfllUeTuI&download=image.png "")

```text
$ docker image inspect ubuntu:latest
[ {
"Id": "sha256:bd3d4369ae.......fa2645f5699037d7d8c6b415a10",
"RepoTags": [
"ubuntu:latest"
<Snip>
"RootFS": {
"Type": "layers",
"Layers": [
    "sha256:c8a75145fc...894129005e461a43875a094b93412",
    "sha256:c6f2b330b6...7214ed6aac305dd03f70b95cdc610",
    "sha256:055757a193...3a9565d78962c7f368d5ac5984998",
    "sha256:4837348061...12695f548406ea77feb5074e195e3",
    "sha256:0cad5e07ba...4bae4cfc66b376265e16c32a0aae9"
] } } ]
```

## Tips

1. For example, some Dockerfile instructions (“ENV”, “EXPOSE”, “CMD”, and “ENTRYPOINT”) add metadata to the image and do not result in permanent layers being created

1. Images layers are shared

1. You can pull image by digest (同一个tag可能存在历史版本， 可以通过digest 下载)

```text
$ docker image ls --digests alpine
REPOSITORY TAG DIGEST IMAGE ID CREATED SIZE
alpine latest sha256:9a839e63da...9ea4fb9a54 f70734b6a266 2 days ago 5.61MB


$ docker image rm alpine:latest
Untagged: alpine:latest
Untagged: alpine@sha256:c0537...7c0a7726c88e2bb7584dc96
Deleted: sha256:02674b9cb179d...abff0c2bf5ceca5bad72cd9
Deleted: sha256:e154057080f40...3823bab1be5b86926c6f860

$ docker image pull alpine@sha256:9a839e63da...9ea4fb9a54
sha256:9a839e63da...9ea4fb9a54: Pulling from library/alpine
cbdbe7a5bc2a: Pull complete
Digest: sha256:9a839e63da...9ea4fb9a54
Status: Downloaded newer image for alpine@sha256:9a839e63da...9ea4fb9a54
docker.io/library/alpine@sha256:9a839e63da...9ea4fb9a54
```

1. All official images have manifest lists

```shell
$ docker manifest inspect golang | grep 'architecture\|os'
        "architecture": "amd64",
        "os": "linux"
        "architecture": "arm",
        "os": "linux",
        "architecture": "arm64",
        "os": "linux",
        "architecture": "386",
        "os": "linux"
        "architecture": "ppc64le",
        "os": "linux"
        "architecture": "s390x",
        "os": "linux"
        "architecture": "amd64",
        "os": "windows",
        "os.version": "10.0.14393.3630"
        "architecture": "amd64",
        "os": "windows",
        "os.version": "10.0.17763.1158"
```



