---
title: "Docker 入门文档"
date: 2024-04-29T22:01:37+08:00
draft: false

tags:
- Docker
- Container

categories: 
- Cloud Native

authors:
- "linuzb"
---

## 前言

本文将简单介绍 Docker 的安装与使用过程

## docker 安装

docker 官方文档提供了各个系统的安装方式，参考 [Install Docker Engine](https://docs.docker.com/engine/install/)，但是由于国内网络环境，使用官方方式会比较慢，因此国内环境不推荐这种方式安装。下面我们介绍另一种国内安装方式。

自动化的安装方式，本节内容源自 [清华大学开源镜像站|Docker CE 软件仓库](https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/)，其他安装方式可以参考该文档。

Docker 提供了一个自动配置与安装的脚本，支持 Debian、RHEL、SUSE 系列及衍生系统的安装。

以下内容假定

- 您为 root 用户，或有 sudo 权限，或知道 root 密码；
- 您系统上有 curl 或 wget

```shell
export DOWNLOAD_URL="https://mirrors.tuna.tsinghua.edu.cn/docker-ce"
# 如您使用 curl
curl -fsSL https://get.docker.com/ | sh
# 如您使用 wget
wget -O- https://get.docker.com/ | sh
```

### 使用非 root 使用 docker

通过以上方式安装的docker只能通过 root 用户或者 sudo 权限使用。

接下来我们将配置非 root 用户使用 docker

1. 创建 docker 用户组

```shell
sudo groupadd docker
```

2. 将当前用户添加到 docker 用户组

```shell
sudo usermod -aG docker $USER
```

3. 激活用户组配置

```shell
newgrp docker
```

4. 测试 docker 命令

```shell
docker run hello-world
```

此命令将下载测镜像并在容器中运行。容器运行时，它会打印一条信息并退出。

### 镜像加速

国内从 Docker Hub 拉取镜像有时会遇到困难，此时可以配置镜像加速器。Docker 官方和国内很多云服务商都提供了国内加速器服务。

配置加速地址

创建或修改 `/etc/docker/daemon.json`：
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "https://dockerproxy.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://docker.nju.edu.cn"
    ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

[Docker Hub 镜像加速器列表](https://gist.github.com/y0ngb1n/7e8f16af3242c7815e7ca2f0833d3ea6)

## 获取镜像

[Docker Hub](https://hub.docker.com/search?q=&type=image) 上有大量的高质量的镜像可以用，我们可以提前在dockerhub 查找自己感兴趣的镜像 pull 到本地。

例如我们想下载一个 ubuntu镜像，版本为 18.04.
```shell
$ docker pull ubuntu:18.04
18.04: Pulling from library/ubuntu
92dc2a97ff99: Pull complete
be13a9d27eb8: Pull complete
c8299583700a: Pull complete
Digest: sha256:4bc3ae6596938cb0d9e5ac51a1152ec9dcac2a1c50829c74abd9c4361e321b26
Status: Downloaded newer image for ubuntu:18.04
docker.io/library/ubuntu:18.04
```

默认情况下，docker 会从 docker.io 这个地址获取镜像。如果我们想从其他镜像仓库获取镜像，我们可以使用命令
```shell
$ docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
```
例如，我想从阿里云镜像中心下载镜像，我可以这样
```shell
docker pull registry.cn-hangzhou.aliyuncs.com/linuzb/scipy-notebook:python-3.9.13-3893868
```

这里我们说一下镜像名称的格式。
- Docker 镜像仓库地址：地址的格式一般是 <域名/IP>[:端口号]。默认地址是 Docker Hub(docker.io)。上例是 `registry.cn-hangzhou.aliyuncs.com`
- 仓库名：如之前所说，这里的仓库名是两段式名称，即 <用户名>/<软件名>。对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。在上面例子中，用户名是 `linuzb`, 软件名是 `scipy-notebook`
- 标签：即不通版本。本例是 `python-3.9.13-3893868`。如果不提供标签，默认使用 `latest`。

## 操作容器

所需要的命令主要为 `docker run`。当利用 `docker run` 来创建容器时，会先检查镜像是否存在，如果不存在会自动下载。

这里举一个例子，详细教程可以参考 [Docker — 从入门到实践|操作容器](https://yeasy.gitbook.io/docker_practice/container/run)

### 启动一个 bash 终端

加入我想启动一个 ubuntu 18.04的环境，并通过 bash 进行一些命令操作。我可以使用以下docker命令

下面的命令则启动一个 bash 终端，允许用户进行交互。
```shell
$ docker run -t -i ubuntu:18.04 /bin/bash
root@af8bae53bdd3:/#
```
其中，`-t` 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， `-i` 则让容器的标准输入保持打开。

在交互模式下，用户可以通过所创建的终端来输入命令，例如
```shell
root@af8bae53bdd3:/# pwd
/
root@af8bae53bdd3:/# ls
bin boot dev etc home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var
```

## 存储

### 挂载一个主机目录作为数据卷

```shell
$ docker run -d -P \
    --name web \
    -v /src/webapp:/usr/share/nginx/html \
    nginx:alpine
```
上面的命令加载主机的 /src/webapp 目录到容器的 /usr/share/nginx/html目录。

## 使用 Dockerfile 定制镜像

Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

Dockerfile 指令的顺序很重要。Docker 构建由一系列有序的构建指令组成。Dockerfile 中的每条指令大致相当于一个镜像层。下图说明了 Dockerfile 如何转化为镜像中的层。

![image-layers](https://docs.docker.com/build/guide/images/layers.png)

镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。

深入理解可以参考[官方文档](https://docs.docker.com/build/guide/layers/)

### FROM 指定基础镜像

所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。

例如以下dockerfile 指定了 ubuntu 22.04 作为基础镜像，后续的安装命令都将基于 ubuntu 22.04
```dockerfile
FROM ubuntu:22.04
...
```

### RUN 执行命令
`RUN` 指令是用来执行命令行命令的。由于命令行的强大能力，`RUN` 指令在定制镜像时是最常用的指令之一。

以下 `RUN` 命令在基础镜像的基础上安装了 vim 软件
```shell
RUN apt update && apt install vim
```
前面我们提到，镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。 因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

因此我们的 `RUN` 命令可以优化成以下方式
```shell
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    vim \
    iputils-ping \
    telnet \
    netcat-traditional \
    dnsutils \
    traceroute \
    net-tools \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```
可以看到，命令最后我们使用 `apt-get clean` 和 `rm` 命令删除了一些无用的缓存。


### Dockerfile 指令详解

`RUN` 命令可以让我们定制化来自 `FROM` 的基础镜像。如果我们想向镜像中复制文件、配置自定义的启动命令、设置环境变量以及创建用户等更复杂的操作，我们该如何实现呢？

可以直接参考 [Docker — 从入门到实践|Dockerfile 指令详解](https://yeasy.gitbook.io/docker_practice/image/dockerfile)

### 构建镜像

我们知道，Dockerfile 定义了镜像最终的形态，那么如何构建镜像呢

我们在 Dockerfile 文件所在目录执行：
```
$ docker build -t ubuntu:22.04-vim .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM ubuntu:22.04
 ---> e43d811ce2f4
Step 2 : RUN apt-get update ...
 ---> Running in 9cdc27646c7b
 ---> 44aa4490ce2c
Removing intermediate container 9cdc27646c7b
Successfully built 44aa4490ce2c
```

这里我们使用了 docker build 命令进行镜像构建。其格式为
```shell
docker build [选项] <上下文路径/URL/->
```

在这里我们指定了最终镜像的名称 `-t ubuntu:22.04-vim`，构建成功后，我们可以像之前运行 `ubuntu:22.04` 那样来运行这个镜像，并且可以使用 vim 操作文件。

## 访问仓库

### dockerhub

参考 [Docker — 从入门到实践|Dockerhub](https://yeasy.gitbook.io/docker_practice/repository/dockerhub)

### 自动构建镜像

使用 github action 自动构建镜像

TODO： 待补充

参考
- git操作
- github action

## 参考

本文仅仅简单介绍了 docker 的使用方式，详细内容可参考以下文档。

- [docker 官方文档](https://docs.docker.com/get-docker/)
- [Docker — 从入门到实践](https://yeasy.gitbook.io/docker_practice/introduction)
- [dockerized](https://github.com/y0ngb1n/dockerized?tab=readme-ov-file)
- [Docker Hub 镜像加速器列表](https://gist.github.com/y0ngb1n/7e8f16af3242c7815e7ca2f0833d3ea6)