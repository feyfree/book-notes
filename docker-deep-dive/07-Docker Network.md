Docker 创建一个容器的时候，会执行如下操作：

- 创建一对虚拟接口，分别放到本地主机和新容器中；

- 本地主机一端桥接到默认的 docker0 或指定网桥上，并具有一个唯一的名字，如 veth65f9；

- 容器一端放到新容器中，并修改名字作为 eth0，这个接口只在容器的命名空间可见；

- 从网桥可用地址段中获取一个空闲地址分配给容器的 eth0，并配置默认路由到桥接网卡 veth65f9。

完成这些之后，容器就可以使用 eth0 虚拟网卡来连接其他容器和其他网络。

可以在 `docker run` 的时候通过 `--net` 参数来指定容器的网络配置，有4个可选值：

- `--net=bridge` 这个是默认值，连接到默认的网桥。

- `--net=host` 告诉 Docker 不要将容器网络放到隔离的命名空间中，即不要容器化容器内的网络。此时容器使用本地主机的网络，它拥有完全的本地主机接口访问权限。容器进程可以跟主机其它 root 进程一样可以打开低范围的端口，可以访问本地网络服务比如 D-bus，还可以让容器做一些影响整个主机系统的事情，比如重启主机。因此使用这个选项的时候要非常小心。如果进一步的使用 `--privileged=true`，容器会被允许直接配置主机的网络堆栈。

- `--net=container:NAME_or_ID` 让 Docker 将新建容器的进程放到一个已存在容器的网络栈中，新容器进程有自己的文件系统、进程列表和资源限制，但会和已存在的容器共享 IP 地址和端口等网络资源，两者进程可以直接通过 `lo` 环回接口通信。

- `--net=none` 让 Docker 将新容器放到隔离的网络栈中，但是不进行网络配置。之后，用户可以自己进行配置。

# **网络配置细节**

用户使用 `--net=none` 后，可以自行配置网络，让容器达到跟平常一样具有访问网络的权限。通过这个过程，可以了解 Docker 配置网络的细节。首先，启动一个 `/bin/bash` 容器，指定 `--net=none` 参数。

```shell
$ docker run -i -t --rm --net=none base /bin/bash
root@63f36fc01b5f:/#
```

在本地主机查找容器的进程 id，并为它创建网络命名空间。

```shell
$ docker inspect -f '{{.State.Pid}}' 63f36fc01b5f
2778
$ pid=2778
$ sudo mkdir -p /var/run/netns
$ sudo ln -s /proc/$pid/ns/net /var/run/netns/$pid
```

检查桥接网卡的 IP 和子网掩码信息。

```shell
$ ip addr show docker0
21: docker0: ...
inet 172.17.42.1/16 scope global docker0
...
```

创建一对 “veth pair” 接口 A 和 B，绑定 A 到网桥 `docker0`，并启用它

```shell
$ sudo ip link add A type veth peer name B
$ sudo brctl addif docker0 A
$ sudo ip link set A up
```

将B放到容器的网络命名空间，命名为 eth0，启动它并配置一个可用 IP（桥接网段）和默认网关。

```shell
$ sudo ip link set B netns $pid
$ sudo ip netns exec $pid ip link set dev B name eth0
$ sudo ip netns exec $pid ip link set eth0 up
$ sudo ip netns exec $pid ip addr add 172.17.42.99/16 dev eth0
$ sudo ip netns exec $pid ip route add default via 172.17.42.1
```

以上，就是 Docker 配置网络的具体过程。

当容器结束后，Docker 会清空容器，容器内的 eth0 会随网络命名空间一起被清除，A 接口也被自动从 `docker0` 卸载。此外，用户可以使用 `ip netns exec` 命令来在指定网络命名空间中进行配置，从而配置容器内的网络。

# single-host bridge networks

The simplest type of Docker network is the single-host bridge network.

The name tells us two things:

- Single-host tells us it only exists on a single Docker host and can only connect containers that are on the same host.

- Bridge tells us that it’s an implementation of an 802.1d bridge (layer 2 switch).

## create network

```shell
# create a bridge network
[root@VM_0_13_centos ~]# docker network create -d bridge localnet
856c9146edc788314fca02298fd35f344581e57d8aa906de7dc5a220e815a44b

# 查看网桥状态 ('yum install brctl' first)
[root@VM_0_13_centos ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
br-856c9146edc7         8000.0242526da736       no
br-ff87b9d22f34         8000.0242a5215bec       no              veth76c8025
                                                        veth87bbe54
docker0         8000.02423270305a       no


```



## run a container using network created

```shell
# 运行一个容器 以localnet bridge 形式 
[root@VM_0_13_centos ~]# docker container run -d --name c1 --network localnet alpine sleep 1d
Unable to find image 'alpine:latest' locally
Trying to pull repository docker.io/library/alpine ... 
latest: Pulling from docker.io/library/alpine
df9b9388f04a: Already exists 
Digest: sha256:4edbd2beb5f78b1014028f4fbb99f3237d9561100b6881aabbf5acce2c4f9454
Status: Downloaded newer image for docker.io/alpine:latest
b4168f8308fd344482f4bd3d36594402872c6ff0f0b8f20e167c70562c845239

[root@VM_0_13_centos ~]# docker network inspect localnet --format '{{json .Containers}}'
{
  "b4168f8308fd344482f4bd3d36594402872c6ff0f0b8f20e167c70562c845239": {
    "Name": "c1",
    "EndpointID": "400cff760ab285a1b6146fa4b006fe887264900b14a295391b5396111870582e",
    "MacAddress": "02:42:ac:14:00:02",
    "IPv4Address": "172.20.0.2/16",
    "IPv6Address": ""
  }
}

[root@VM_0_13_centos ~]# brctl show
bridge name     bridge id               STP enabled     interfaces
br-856c9146edc7         8000.0242526da736       no              vethc3adc99
br-ff87b9d22f34         8000.0242a5215bec       no              veth76c8025
                                                        veth87bbe54
docker0         8000.02423270305a       no
```

## test connection

```shell
# 运行一个容器 以localnet bridge 形式 
[root@VM_0_13_centos ~]# docker container run -it --name c2 --network localnet alpine sh
/ # ping c1
PING c1 (172.20.0.2): 56 data bytes
64 bytes from 172.20.0.2: seq=0 ttl=64 time=0.077 ms
64 bytes from 172.20.0.2: seq=1 ttl=64 time=0.100 ms
64 bytes from 172.20.0.2: seq=2 ttl=64 time=0.072 ms
64 bytes from 172.20.0.2: seq=3 ttl=64 time=0.098 ms
64 bytes from 172.20.0.2: seq=4 ttl=64 time=0.084 ms
64 bytes from 172.20.0.2: seq=5 ttl=64 time=0.112 ms
^C
--- c1 ping statistics ---
6 packets transmitted, 6 packets received, 0% packet loss
round-trip min/avg/max = 0.072/0.090/0.112 ms
/ # 
```

**Beware**: 

The default bridge network on Linux does not support name resolution via the Docker DNS service. All other user-defined bridge networks do. The following demo will work because the container is on the user-defined localnet network.

```shell
[root@VM_0_13_centos ~]# docker network inspect localnet
[
    {
        "Name": "localnet",
        "Id": "856c9146edc788314fca02298fd35f344581e57d8aa906de7dc5a220e815a44b",
        "Created": "2022-04-06T15:10:10.122153007+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.20.0.0/16",
                    "Gateway": "172.20.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "6ccb806a8ff36044e3325a6aa1707864282f81d50970b83aa3c4aa51275e9118": {
                "Name": "c2",
                "EndpointID": "d22d6b32298febd55e292fe5da08a5f20d86df5b17cdaa9c7f39bd6437d0ed4d",
                "MacAddress": "02:42:ac:14:00:03",
                "IPv4Address": "172.20.0.3/16",
                "IPv6Address": ""
            },
            "b4168f8308fd344482f4bd3d36594402872c6ff0f0b8f20e167c70562c845239": {
                "Name": "c1",
                "EndpointID": "400cff760ab285a1b6146fa4b006fe887264900b14a295391b5396111870582e",
                "MacAddress": "02:42:ac:14:00:02",
                "IPv4Address": "172.20.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

```

## run a web server

```shell
[root@VM_0_13_centos ~]# docker container run -d --name web --network localnet --publish 5050:80 nginx
77417238f1cb0b744b674b30a91afe546cd12555ce2866907bf6972dfc6d1532
[root@VM_0_13_centos ~]# docker port web
80/tcp -> 0.0.0.0:5050
[root@VM_0_13_centos ~]# curl localhost:5050
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
[root@VM_0_13_centos ~]# docker network inspect localnet
[
    {
        "Name": "localnet",
        "Id": "856c9146edc788314fca02298fd35f344581e57d8aa906de7dc5a220e815a44b",
        "Created": "2022-04-06T15:10:10.122153007+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.20.0.0/16",
                    "Gateway": "172.20.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "6ccb806a8ff36044e3325a6aa1707864282f81d50970b83aa3c4aa51275e9118": {
                "Name": "c2",
                "EndpointID": "d22d6b32298febd55e292fe5da08a5f20d86df5b17cdaa9c7f39bd6437d0ed4d",
                "MacAddress": "02:42:ac:14:00:03",
                "IPv4Address": "172.20.0.3/16",
                "IPv6Address": ""
            },
            "77417238f1cb0b744b674b30a91afe546cd12555ce2866907bf6972dfc6d1532": {
                "Name": "web",
                "EndpointID": "e41a1f424a5ebf5d3fbb5eb38f00c79fe800319828fdb7236a09ed2799e33e3f",
                "MacAddress": "02:42:ac:14:00:04",
                "IPv4Address": "172.20.0.4/16",
                "IPv6Address": ""
            },
            "b4168f8308fd344482f4bd3d36594402872c6ff0f0b8f20e167c70562c845239": {
                "Name": "c1",
                "EndpointID": "400cff760ab285a1b6146fa4b006fe887264900b14a295391b5396111870582e",
                "MacAddress": "02:42:ac:14:00:02",
                "IPv4Address": "172.20.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

```shell
# 在 容器c1 里面
/ # curl 172.20.0.4:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

# multi-host overlay networks

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426163525-multi-host-overlay-network.png)

## macvlan

On the positive side, MACVLAN performance is good as it doesn’t require port mappings or additional bridges — you connect the container interface through to the hosts interface (or a sub-interface). However, on the negative side, it requires the host NIC to be in **promiscuous mode**, which isn’t always allowed on corporate networks and public cloud platforms. So MACVLAN is great for your corporate data center networks (assuming your network team can accommodate promiscuous mode), but it might not work in the public cloud.

网卡需要运行在**混杂模式**

macvlan 的demo 这里就不展开了

# service discovery

![](https://raw.githubusercontent.com/feyfree/my-github-images/main/20220426163605-docker-service-discovery.png)



Let’s step through the process.

- Step 1: The ping c2 command invokes the local DNS resolver to resolve the name “c2” to an IP address. All Docker containers have a local DNS resolver.

-  Step 2: If the local resolver doesn’t have an IP address for “c2” in its local cache, it initiates a recursive query to the Docker DNS server. The local resolver is pre-configured to know how to reach the Docker DNS server.

- Step 3: The Docker DNS server holds name-to-IP mappings for all containers created with the --name or --net-alias flags. This means it knows the IP address of container “c2”.

-  Step 4: The DNS server returns the IP address of “c2” to the local resolver in “c1”. It does this because the two containers are on the same network — if they were on different networks this would not work.

- Step 5: The ping command issues the ICMP echo request packets to the IP address of “c2”.

```text
# --dns 指定 dns 服务器, --dns-search 作为补充
$ docker container run -it --name c1 --dns=8.8.8.8 --dns-search=nigelpoulton.com alpine sh
```

