---
title: docker的swarm模式使用
date: 2017-08-07 14:31:43
categories:
- 技术杂谈
tags:
- Docker
---

由于部门的java业务的增大，之前的jenkins的服务部署方式很难满足目前的业务需求，同时也很难进行管理，服务的扩容与缩容都需要登录服务器上手动操作，增加了对于整体的Java的运维难度，

![](http://wx4.sinaimg.cn/mw1024/78d85414ly1fpeppae8ubj21kw0jajtg.jpg)

<!-- more -->

## Docker安装
Docker的安装官网上提供了两种方法，本文主要是采用下载rpm包自己手动安装的方法。
1. 卸载旧的docker版本
```bash
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
```
2. 从docker的[安装包下载地址](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)下载需要的版本
```bash
wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-17.12.0.ce-1.el7.centos.x86_64.rpm
```
3. 安装Docker CE
```bash
$ sudo yum install /path/to/package.rpm
```
4. 启动Docker
```bash
$ sudo systemctl start docker
```

## Swarm Mode安装

### 创建Swarm
在已经安装好 Docker Engine 的服务器上创建新的swarm
```bash
docker swarm init --advertise-addr <MANAGER-IP>
```
在成功运行以上命令后，可以在终端见到如下输出
```bash
$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### 节点添加

```bash
docker swarm join --token  \
SWMTKN-1-159xuoyndybto5w9o9m5tku1w9cqsxkow76powlup7o7c3gel6-4j5ri0czte9v \
 i6x9usblmano5 10.77.96.202:2377 \
```

### 节点下线
有些时候需要维护一个节点，此时此节点可能会网络断开或者需要关机，造成节点上服务可用。使用docker node update --availability drain <NODE-ID>将节点下线，swarm会将当前节点上的容器关闭并在其他节点上启动。当维护完成，需要上线是，将节点状态修改为active状态即可,命令如下：docker node update --availability active <NODE-ID>，举例说明
```bash
docker node update --availability drain worker1
docker node update --availability active worker1
```

## 服务发布
### 服务创建
首先简单的创建一个服务，如下所示
```bash
$ docker service create --replicas 1 --name helloworld alpine ping docker.com
```
* docker service create命令创建一个 service.
* --name标签命名service为helloworld.
* --replicas标签来详细声明1个运行实体.
* 参数alpine ping docker.com定义执行pingg docker.com作为alpine容器的服务.

利用docker service ls 命令来看全部的运行的服务
```bash
$docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
6oiyost9upwn        helloworld          replicated          1/1                 alpine:latest
```

### 服务审查
```bash
[manager1]$ docker service inspect --pretty helloworld

ID:		9uk4639qpg7npwf3fn2aasksr
Name:		helloworld
Service Mode:	REPLICATED
 Replicas:		1
Placement:
UpdateConfig:
Parallelism:	1
ContainerSpec:
Image:		alpine
Args:	ping docker.com
Resources:
Endpoint Mode:  vip
```
运行 docker service ps SERVICE-ID 命令来查看服务运行在哪个节点
```bash
$ docker service ps helloworld
NAME                                    IMAGE   NODE     DESIRED STATE  LAST STATE
helloworld.1.8p1vev3fq5zm0mi8g0as41w35  alpine  worker2  Running        Running 3 minutes
```
> 可能存在于 manager 节点，也可能存在于 worker 节点

### 服务扩容
运行如下命令对服务进行扩容处理
```bash
$ docker service scale <SERVICE-ID>=<NUMBER-OF-TASKS>
```
举例子如下
```bash
$docker service scale helloworld=3
helloworld scaled to 3
overall progress: 3 out of 3 tasks
1/3: running   [==================================================>]
2/3: running   [==================================================>]
3/3: running   [==================================================>]
verify: Service converged
```
查看扩容后的服务节点如下
```bash
$docker service ps helloworld
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE               ERROR               PORTS
vkmxhd3kjeiv        helloworld.1        alpine:latest       bx-web010           Running             Running about an hour ago
01040syvpdkb        helloworld.2        alpine:latest       docker004           Running             Running 29 seconds ago
8fvigfzx2lv3        helloworld.3        alpine:latest       docker003           Running             Running 42 seconds ago
```

### 服务删除
```bash
$ docker service rm helloworld
helloworld
```

## 服务路由
Docker引擎Swarm集群模式使得可以轻松地发布服务端口，使其可用于集群外的资源。所有节点都参与进入路由网。路由网格使得Swam集群中的每个节点能够接受在Swarm集群中运行的任何服务中已发布端口上的连接，即使节点上没有任何任务正在运行。 路由网络将所有接入请求路由到可用节点上的已发布端口到活动容器中。

为了在Swarm集群中使用接入网络，在启用Swarm集群模式之前， 你需要在Swarm集群节点之间打开以下端口：

* 7946端口， 主要用于容器网络发现；
* 4789端口， 主要用于容器接入发布网络。

你还需要在Swarm集群节点和任意外部资源之间打开发布端口，例如外部负载均衡应用，以便于它们能够访问所需要的端口。





## 参考文献
1. [Docker官网](https://docs.docker.com/install)
